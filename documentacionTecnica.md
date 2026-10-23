<USER_REQUEST>

# Documento Técnico Parcial (v1.0) - SplitVision

**Curso:** Construcción de Software II (SW707)

**Equipo:**

- Gonzalo Albornoz — 20232551K
- Luis Angel Vargas Ponce — 20231314E
- Farid Zamudio — 20231215G

**Fecha:** Octubre de 2026

---

## 1. Descripción del problema y usuarios

### 1.1. Contexto y problema del usuario

En grupos colaborativos, como compañeros de universidad, _roommates_ o grupos de viaje, la división de gastos compartidos suele hacerse a mano y es muy propensa a errores. Cuando varios integrantes asumen diferentes costos por ejemplo, uno paga las entradas, otro el almuerzo y otro el taxi, calcular cuánto debe pagar cada persona y quién le debe a quién se vuelve confuso y genera fricción en el grupo.

Aunque las boletas físicas o digitales son la fuente de verdad de los gastos, tener que pasar a mano cada plato y precio a una hoja de cálculo o chat resulta tedioso y aumenta la probabilidad de equivocarse en las cuentas.

### 1.2. Problema técnico de Construcción de Software

Para automatizar este proceso, el sistema enfrenta tres retos de construcción que van más allá de un CRUD tradicional: el control de concurrencia cuando varios usuarios registran o confirman pagos al mismo tiempo, el procesamiento asíncrono de lectura OCR tratando sus resultados como datos no confiables, y la precisión numérica para que no se pierda ni un solo céntimo al dividir las cuentas.

#### A. Condiciones de carrera y concurrencia en el ciclo de vida de pagos

El sistema modela las deudas manteniendo tres montos en centavos por cada obligación (`amount_cents`, `confirmed_cents` y `reserved_cents`), de los cuales se calcula el saldo disponible para pagar:

$$\mathrm{Saldo\_Disponible} = \mathrm{amount\_cents} - (\mathrm{confirmed\_cents} + \mathrm{reserved\_cents})$$

Este saldo compartido está expuesto a condiciones de carrera en dos situaciones muy comunes:

1. **Doble envío por lentitud de red móvil (_Double Submit_):** Cuando un deudor registra un pago desde su celular con señal inestable, la demora en la respuesta hace que suela presionar dos veces el botón de enviar o que el navegador reintente la petición. Dos solicitudes iguales llegan al servidor con pocos milisegundos de diferencia.
2. **Operaciones simultáneas entre Deudor y Acreedor:** Cuando un deudor registra un pago, este entra en estado `PENDIENTE` y reserva parte del saldo. Casi al mismo tiempo, el deudor podría intentar cancelar su pago (`POST /api/payments/:id/cancel`) justo cuando el acreedor está en su Bandeja de Pagos confirmándolo (`POST /api/payments/:id/confirm`) o rechazándolo (`POST /api/payments/:id/reject`), o bien cuando se ejecuta la compensación automática de deudas cruzadas del evento (`POST /api/events/:id/compensate`).

Si estas operaciones no se ejecutan de forma aislada y en orden, ocurre el problema de **Actualización Perdida (_Lost Update_)**, causado por verificar una condición y actuar después (_check-then-act_) sin bloquear el registro:

1. Dos peticiones leen al mismo tiempo el mismo saldo disponible de la deuda.
2. Ambas validan que el monto alcanza usando un dato que queda desactualizado al instante.
3. Ambas actualizan la base de datos pisándose entre sí.

Si esto pasa, el saldo disponible puede volverse negativo, los montos reservados no cuadran con los pagos reales y el balance del evento se rompe. Para evitarlo, el sistema aplica **bloqueos pesimistas a nivel de fila (`SELECT ... FOR UPDATE`)** dentro de transacciones SQL, obligando a que las peticiones sobre una misma deuda se procesen en fila india, y lo complementa con **llaves de idempotencia (`Idempotency-Key`)** para ignorar clics repetidos.

#### B. Cómputo intensivo asíncrono y datos no confiables de extracción OCR

Para ahorrarle al usuario el trabajo de digitar la boleta, el sistema incluye un subsistema de **Reconocimiento Óptico de Caracteres (OCR)** que lee los ítems y el total desde la foto del comprobante.

El diseño de este subsistema combina la posibilidad de consultar un servicio OCR externo por HTTP con un motor local de respaldo basado en **Tesseract.js (WebAssembly)**. Esto plantea varios retos de construcción:

- **Bloqueo del Event Loop por uso intensivo de CPU (_CPU-Bound_):** Analizar una imagen y ejecutar el reconocimiento óptico en Tesseract consume bastante procesador. Si lo hiciéramos dentro del mismo proceso de la API en Node.js, el _Event Loop_ se congelaría varios segundos por cada foto, dejando colgados a los demás usuarios que estén navegando en la app. Por eso, separamos la lectura en un **proceso Worker independiente** que se comunica con la API mediante una cola de trabajos guardada en PostgreSQL (`receipt_jobs`).
- **Gestión de memoria en tiempo de ejecución (_Memory Leaks_):** Los workers de Tesseract en WebAssembly reservan bloques grandes de memoria fuera del recolector de basura normal de JavaScript. Si no se destruyen explícitamente al terminar de leer cada boleta (`worker.terminate()`), la memoria RAM se llena rápido hasta tumbar el servidor.
- **Resultados del OCR como datos no confiables:** Las boletas reales suelen estar arrugadas, mal impresas o fotografiadas con sombras. Por ello, el texto devuelto por el OCR puede traer errores (como confundir letras con números o leer mal un precio). El sistema aplica **programación defensiva** al parsear el texto y obliga a que el usuario revise y confirme los datos en pantalla (estado `EXTRAÍDO`) antes de crear cualquier deuda.
- **Desacoplamiento mediante la interfaz `OcrClient`:** Para no atar el negocio a una sola librería, definimos el contrato `OcrClient` (`ocr.client.ts`), el cual tiene tres implementaciones intercambiables: `HttpOcrClient` (para conectarse a un servicio OCR externo), el cliente local con Tesseract.js (usado también como respaldo automático si cae el externo) y `FakeOcrClient` (que permite simular fallas, demoras y datos corruptos en las pruebas sin depender de internet).

#### C. Precisión aritmética en el cálculo de montos y conservación de masa

Dividir gastos entre varias personas casi nunca da una división exacta. Por ejemplo, al dividir S/ 100.00 en partes iguales entre 3 amigos, a cada uno le tocaría S/ 33.3333... Si simplemente redondeamos a S/ 33.33, la suma da S/ 99.99 y desaparece un céntimo (S/ 0.01).

Además, JavaScript maneja los números con decimales usando punto flotante binario (**IEEE 754**), donde operaciones simples como `0.1 + 0.2` dan `0.30000000000000004`, provocando descuadres en los saldos.

Para asegurar que las cuentas cuadren siempre al centavo, SplitVision aplica:

1. **Aritmética entera en centavos:** Nunca usamos decimales ni punto flotante para operar dinero. Todo monto se convierte desde texto a números enteros en centavos (`total_cents`, `amount_cents`).
2. **Invariante de conservación de masa (_Design by Contract_):** El motor de reparto verifica por contrato que la suma de las partes calculadas sea exactamente igual al total de la boleta:
   $$\sum_{i=1}^{n} \mathrm{parte}_i \equiv \mathrm{total\_comprobante}$$
3. **Reparto determinista de los céntimos sobrantes:** Los centavos que sobran en una división inexacta se reparten uno por uno siguiendo un orden fijo y predecible entre los participantes. Esta regla está probada con _Property-Based Testing_ (`fast-check`) corriendo 1,000 casos aleatorios en las pruebas automatizadas.

#### D. Relación entre los retos y las técnicas del curso

| Reto de construcción                         | Riesgo principal                                                                          | Técnica aplicada                                                                                                        | Sección en doc |
| :------------------------------------------- | :---------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------- | :------------: |
| **Concurrencia en la liquidación de deudas** | _Lost Update_, saldo negativo y pagos duplicados por doble clic o confirmación simultánea | Bloqueo pesimista a nivel de fila (`SELECT ... FOR UPDATE`), transacciones y llaves de idempotencia (`Idempotency-Key`) |       8        |
| **Cómputo intensivo de extracción OCR**      | Bloqueo del _Event Loop_ de la API y lentitud general del sistema                         | Arquitectura en dos procesos (API y Worker) y cola de tareas en PostgreSQL (`receipt_jobs`)                             |     3 y 5      |
| **Datos no confiables del OCR**              | Creación de deudas con montos falsos o caídas por lecturas corruptas                      | Programación defensiva, validación en el parser y revisión humana obligatoria en estado `EXTRAÍDO`                      |      6.2       |
| **Precisión numérica y reparto de residuos** | Pérdida de céntimos y saldos descuadrados por punto flotante IEEE 754                     | Operaciones enteras en centavos (`money.ts`), _Design by Contract_ y _Property-Based Testing_                           |    5 y 6.1     |
| **Fugas de memoria en el runtime**           | Caída del proceso por memoria acumulada en workers de WebAssembly                         | Control estricto del ciclo de vida del worker de OCR y liberación de recursos                                           |       5        |

### 1.3. Usuarios / Actores del sistema

#### 1.3.1. Usuarios finales (grupos colaborativos)

Son las personas registradas con una cuenta propia en la aplicación, que entran a sus grupos para subir boletas, revisar los montos leídos por el OCR y pagar o cobrar sus deudas. Dentro de un evento, un usuario puede actuar como:

- **Pagador (acreedor):** Es quien pagó la cuenta en el local y sube la foto del comprobante al evento. Revisa los ítems extraídos, elige cómo se va a dividir el gasto y luego confirma o rechaza los pagos que le envían sus amigos.
- **Participante (deudor):** Es el integrante del grupo a quien se le asigna una parte del gasto; puede ver cuánto debe, revisar los datos de cobro del acreedor (Yape, Plin, cuenta bancaria o QR) y registrar su pago subiendo la captura de su transferencia.

Estos roles cambian según la boleta: en una misma salida, un usuario puede ser pagador del almuerzo y deudor del taxi, por lo que el sistema también permite cruzar y compensar automáticamente las deudas mutuas.

#### 1.3.2. Subsistema de Extracción (Motor OCR en Worker)

Es el proceso en segundo plano encargado de tomar los trabajos pendientes de la tabla `receipt_jobs`, procesar las imágenes y extraer los platos y el total. Usa la interfaz `OcrClient` junto con un Circuit Breaker (`opossum`) para aislar fallos del motor de lectura sin afectar la estabilidad de la API principal.

### 1.4. Objetivo del prototipo (v1.0)

**Objetivo general:** Construir el prototipo funcional v1.0 de SplitVision que permita gestionar gastos grupales a partir de comprobantes y liquidar las deudas resultantes entre los participantes, asegurando que los saldos nunca se corrompan ante pagos simultáneos y que el sistema siga operando con normalidad si falla la lectura automática del OCR.

**Objetivos específicos:**

- Asegurar la consistencia de la tabla de deudas frente a pagos y confirmaciones simultáneas usando transacciones y bloqueos a nivel de fila.
- Mantener la fluidez y disponibilidad de la aplicación frente a demoras o fallos del OCR mediante el proceso Worker, un Circuit Breaker con respaldo local y una vía de ingreso manual.
- Garantizar mediante contratos (_Design by Contract_) que el reparto equitativo o por ítems conserve exactamente el total del comprobante sin perder centavos.
- Proteger el backend frente a archivos inválidos, parámetros manipulados o lecturas ruidosas del OCR aplicando programación defensiva.

---

## 2. Alcance y requisitos principales

### 2.1. Delimitación del Prototipo Funcional (Alcance v1.0)

El prototipo v1.0 cubre de punta a punta el flujo de negocio: desde el registro de usuarios y creación del grupo, pasando por el escaneo de boletas y división de ítems, hasta la compensación de saldos cruzados y confirmación de pagos con evidencia. Esta versión sirve como base sólida sobre la cual se trabajará la evolución hacia el Examen Final.

**Dentro del alcance (v1.0):**

- Registro e inicio de sesión con cuentas propias, sesiones en base de datos, protección contra fuerza bruta y validación CSRF (RF1).
- Creación de eventos, búsqueda de usuarios e invitaciones con aceptación o rechazo, además de configuración de métodos de cobro en el perfil (Yape, Plin, cuenta/CCI y código QR) (RF2).
- Subida de fotos de comprobantes validada por _magic bytes_, detección de imágenes duplicadas por hash SHA-256 y lectura asíncrona en el Worker con opción de reintento o ingreso manual si falla el OCR (RF3).
- Revisión y edición de los datos extraídos por el pagador, y cálculo del reparto en modo equitativo (`EQUITATIVE`) o por unidades de ítem (`BY_ITEM`) cumpliendo la conservación de centavos (RF4).
- Registro de pagos con evidencia fotográfica, estados de pago (`PENDIENTE`, `CONFIRMADO`, `RECHAZADO`, `CANCELADO`), bandeja de revisión para el acreedor y compensación automática de deudas mutuas, protegido con `SELECT ... FOR UPDATE` e `Idempotency-Key` (RF5).
- Consulta de deudas ("Debo" y "Me deben"), desglose de cruces mutuos, balance consolidado del evento y resumen listo para copiar a WhatsApp (RF6).
- Aplicación web SPA en React y TypeScript adaptable a celulares y escritorio, con soporte de tema claro/oscuro.

**Fuera del alcance (v1.0):**

- Cobros automáticos conectados directamente a bancos o tarjetas (las transferencias se hacen por Yape, Plin o banca móvil del usuario, y en SplitVision se sube la captura como evidencia).
- Custodia o retención de dinero real dentro de la plataforma.
- Inicio de sesión con redes sociales (Google/Facebook OAuth).
- Notificaciones push nativas en el celular.
- Publicación en tiendas móviles App Store o Google Play.

### 2.2. Modelo de Dominio y Operación (Sesiones)

#### A. Usuarios Persistentes y Sesiones Seguras:

Cada participante se registra con un `username` único (3 a 20 caracteres alfanuméricos o guion bajo), un correo electrónico único y una contraseña que se guarda hasheada con Argon2 (`@node-rs/argon2`). Además, cada usuario puede configurar en su perfil su celular, billetera preferida (`YAPE`, `PLIN`, `BIM`), banco, número de cuenta, CCI y subir la imagen de su código QR para facilitar que sus amigos le paguen.

Al iniciar sesión, el sistema registra una sesión en la tabla `sessions` asociada a una cookie `httpOnly` (`sid`) y un token `csrf` que el frontend envía en la cabecera `x-csrf-token` en cada operación que modifica datos. De esta forma sabemos con total exactitud qué usuario hizo cada acción.

#### B. Eventos e Invitaciones (Límites de Contexto):

Los gastos se agrupan dentro de "Eventos" (por ejemplo, "Salida a Barranco" o "Almuerzo FIIS"). Para sumar amigos al grupo, los participantes actuales les envían invitaciones (`event_invitations`) que los invitados aceptan o rechazan desde su Bandeja. El evento funciona como una barrera de seguridad: nadie que no sea participante activo (`event_participants`) puede ver ni tocar los comprobantes, saldos o deudas de ese grupo.

#### C. Comprobantes, Cola de Trabajos y Extracción:

Cuando se sube la imagen de una boleta, se crea el comprobante en estado `PROCESANDO` junto con una tarea en la tabla `receipt_jobs` dentro de la misma transacción. El Worker toma la tarea, ejecuta el OCR y pasa el comprobante a `EXTRAÍDO` (o a `FALLIDO` si la foto no se pudo leer tras los reintentos). Luego el pagador revisa los montos, elige el modo de reparto y confirma la boleta (`CONFIRMADO`).

#### D. Matriz de Deudas, Reserva de Saldo y Compensación Cruzada:

Al confirmar el reparto de una boleta, se crean registros en la tabla `debts` para cada participante (excepto para el mismo pagador). Cada deuda guarda en centavos el monto original (`amount_cents`), lo ya confirmado (`confirmed_cents`) y lo que está en revisión por pagos pendientes (`reserved_cents`).

Cuando el deudor envía un pago con su captura de pantalla, el pago queda `PENDIENTE` y ese monto se suma a `reserved_cents` para que no se pueda pagar dos veces. El acreedor lo revisa en su Bandeja de Pagos: si lo confirma, el monto pasa de `reserved_cents` a `confirmed_cents`; si lo rechaza (o el deudor lo cancela), se libera de `reserved_cents`. Además, si en un evento dos amigos se deben mutuamente por distintas boletas, el sistema cruza ambas deudas automáticamente registrando un abono interno con referencia `COMPENSACION`, para que solo tengan que transferirse la diferencia neta.

#### E. Entidades principales e invariantes

La base de datos relacional (`migrations/001_init.sql` a `006_billetera_y_cci.sql`) modela las siguientes tablas:

| Entidad (Tabla SQL)                                     | Descripción                                                | Atributos principales                                                                                                                                                                                      |
| :------------------------------------------------------ | :--------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Usuario (`users`)**                                   | Cuenta de usuario, perfil y datos de cobro                 | `id`, `username`, `email`, `password_hash`, `image_path`, `phone`, `preferred_wallet`, `bank_name`, `account_number`, `cci`, `qr_image_path`                                                               |
| **Sesión (`sessions`)**                                 | Sesiones activas y token anti-CSRF                         | `id`, `user_id`, `csrf`, `created_at`, `last_seen_at`, `expires_at`                                                                                                                                        |
| **Intentos de Login (`login_attempts`)**                | Registro de fallos para bloqueo por fuerza bruta           | `id`, `identifier`, `attempted_at`                                                                                                                                                                         |
| **Evento (`events`)**                                   | Grupo donde se comparten gastos                            | `id`, `name`, `created_by`, `image_path`, `created_at`                                                                                                                                                     |
| **Participante (`event_participants`)**                 | Miembros que pertenecen a un evento                        | `event_id`, `user_id`, `joined_at`                                                                                                                                                                         |
| **Invitación (`event_invitations`)**                    | Solicitudes para unirse a un evento                        | `id`, `event_id`, `invitee_id`, `inviter_id`, `status` (`PENDIENTE`, `ACEPTADA`, `RECHAZADA`)                                                                                                              |
| **Comprobante (`receipts`)**                            | Boleta o gasto subido a un evento                          | `id`, `event_id`, `payer_id`, `status` (`CARGADO`, `PROCESANDO`, `EXTRAÍDO`, `CONFIRMADO`, `FALLIDO`), `total_cents`, `tax_cents`, `tip_cents`, `discount_cents`, `image_path`, `image_hash`, `split_mode` |
| **Ítem (`receipt_items`)**                              | Platos o productos de un comprobante                       | `id`, `receipt_id`, `position`, `description`, `quantity`, `unit_price_cents`                                                                                                                              |
| **Trabajo OCR (`receipt_jobs`)**                        | Cola persistente de tareas para el Worker                  | `id`, `receipt_id`, `status` (`PENDING`, `RUNNING`, `DONE`, `FAILED`), `attempts`, `next_attempt_at`, `lease_owner`, `lease_expires_at`, `correlation_id`                                                  |
| **Reparto (`split_participants` / `item_assignments`)** | Asignación de quién participa o quién consumió cada unidad | `receipt_id`, `user_id`, `item_id`, `unit_index`                                                                                                                                                           |
| **Deuda (`debts`)**                                     | Saldo que un deudor le debe a un acreedor por una boleta   | `id`, `receipt_id`, `event_id`, `debtor_id`, `creditor_id`, `amount_cents`, `confirmed_cents`, `reserved_cents`                                                                                            |
| **Pago (`payments`)**                                   | Abono registrado con evidencia e idempotencia              | `id`, `debt_id`, `payer_id`, `amount_cents`, `method`, `reference`, `evidence_path`, `status`, `idempotency_key`                                                                                           |
| **Auditoría (`audit_logs`)**                            | Bitácora de operaciones importantes                        | `id`, `user_id`, `operation`, `resource`, `result`, `correlation_id`, `created_at`                                                                                                                         |

Las reglas matemáticas que el sistema mantiene siempre (**invariantes del dominio**, reforzadas incluso con `CHECK CONSTRAINTS` en PostgreSQL) son:

1. **Invariante de no sobregiro ni saldo negativo (`debts_balances_check`):** En todo momento, lo confirmado más lo reservado nunca supera el monto de la deuda:
   $$0 \le \mathrm{confirmed\_cents} + \mathrm{reserved\_cents} \le \mathrm{amount\_cents}$$
2. **Invariante de saldo disponible:** Lo que falta por abonar en una deuda se calcula siempre como:
   $$\mathrm{available\_cents} = \mathrm{amount\_cents} - (\mathrm{confirmed\_cents} + \mathrm{reserved\_cents})$$
3. **Invariante de conservación de masa monetaria:** Al confirmar un reparto, la suma de las deudas creadas más la parte del propio pagador es idéntica al total del comprobante:
   $$\sum_{i=1}^{k} \mathrm{deuda}_i + \mathrm{cuota\_pagador} \equiv \mathrm{total\_cents}$$

### 2.3. Flujo de Negocio

El siguiente diagrama muestra la secuencia completa del flujo de negocio, incluyendo el trabajo asíncrono del Worker y el bloqueo en los pagos.

#### Diagrama de Flujo del negocio Simplificado

<div align="center">

<img src="assets/flujo_negocio_simple_v1.png" alt="Diagrama de Flujo del Negocio" width="400">

</div>

#### Diagrama de Flujo del negocio Detallado

![Diagrama de Flujo del Negocio](assets/flujo_negocio_detallado_v1.png)

### 2.4. Explicación del Flujo principal y comportamiento del sistema

Veamos el recorrido completo con un ejemplo práctico: Mateo, Lucía y Andrés salen a almorzar. Mateo paga la cuenta con su tarjeta y usa SplitVision para dividir el gasto y cobrarles a sus amigos.

#### Paso 1: Creación del Evento e Invitaciones

- **Acción del usuario:** Mateo crea el evento "Almuerzo Criollo", quedando registrado automáticamente como participante. Luego busca a Lucía y Andrés por su `username` o correo y les envía una invitación, la cual ellos aceptan desde su Bandeja (`/bandeja`).
- **Comportamiento del sistema:** Se inserta el evento en `events` y los vínculos en `event_participants` dentro de una transacción, dejando trazabilidad en `audit_logs`. A partir de ahí, solo ellos tres tienen permiso para ver o registrar gastos en ese grupo.

#### Paso 2: Subida del Comprobante y Encolado Asíncrono

- **Acción del usuario:** Mateo sube la foto de la boleta desde su celular.
- **Comportamiento del sistema:** El servicio `StorageService` revisa los primeros bytes del archivo (_magic bytes_) para confirmar que sea realmente una imagen JPG, PNG o WEBP válida de máximo 5 MB, calcula su hash SHA-256 para evitar que se suba dos veces la misma foto en el evento, y la guarda en disco con un identificador único (`comprobante-<uuid>.jpg`). Acto seguido, en una misma transacción SQL, crea el comprobante en estado `PROCESANDO` y encola un trabajo `PENDING` en `receipt_jobs`, respondiendo de inmediato al navegador sin bloquear la API.

#### Paso 3: Lectura OCR, Revisión Humana y Reparto con Contratos

- **Acción del usuario:** Cuando el Worker termina de leer la imagen, la pantalla de Mateo pasa automáticamente al formulario de revisión. Mateo verifica que los platos y el total coincidan con el papel, elige el modo de reparto (equitativo o asignando qué plato comió cada uno) y confirma el reparto.
- **Comportamiento del sistema:** El Worker reclama la tarea de `receipt_jobs` usando `FOR UPDATE SKIP LOCKED`, ejecuta el reconocimiento mediante el Circuit Breaker y pasa los datos por el parser defensivo (`recibo.parser.ts`). Al confirmar Mateo la división, `split-calculator.ts` reparte los montos en centavos aplicando el método del mayor residuo y comprueba por contrato que la suma de las cuotas sea exactamente igual a `total_cents` antes de guardar.

#### Paso 4: Generación de Deudas

- **Acción del usuario:** Lucía y Andrés entran a "Mis deudas" (`/deudas`) y ven cuánto le deben a Mateo (por ejemplo, S/ 100.00 cada uno si la boleta fue de S/ 300.00 en partes iguales).
- **Comportamiento del sistema:** Se insertan las filas en `debts` solo para Lucía y Andrés (la parte de Mateo no genera deuda porque él mismo pagó la boleta). Si existieran deudas en sentido contrario dentro del mismo evento, el sistema ejecuta `compensateCrossDebts` para descontarlas mutuamente.

#### Paso 5: Registro de Pagos y Control de Concurrencia

- **Acción del usuario:** Andrés le yapea S/ 100.00 a Mateo copiando su número desde la pantalla de pago (`/deudas/:id/pago`) y sube la captura del voucher. Por su lado, Lucía quiere hacer un abono parcial de S/ 60.00 sobre sus S/ 100.00, pero por mala señal móvil se disparan dos peticiones casi al mismo tiempo (o intenta mandar un pago de S/ 60.00 y otro de S/ 70.00 seguidos).
- **Comportamiento del sistema:** El backend protege el saldo con dos mecanismos:
  1. **Bloqueo pesimista (`SELECT ... FOR UPDATE`):** La base de datos pone en fila las peticiones que tocan la deuda de Lucía. La primera toma el bloqueo, verifica que S/ 60.00 $\le$ S/ 100.00 disponibles, crea el pago en estado `PENDIENTE`, sube `reserved_cents` a `6000` (dejando S/ 40.00 disponibles) y hace `COMMIT`. Cuando entra la segunda petición por S/ 70.00, lee el nuevo saldo disponible de S/ 40.00; como no alcanza, se rechaza con error **HTTP 409 Conflict** sin malograr los saldos.
  2. **Llave de idempotencia (`Idempotency-Key`):** Si las dos peticiones eran el mismo envío repetido con el mismo UUID en la cabecera `idempotency-key`, el índice único `payments_idempotent` detecta que ya se procesó y devuelve el pago existente (`existing: true`) sin volver a descontar saldo.

#### Paso 6: Confirmación en Bandeja de Pagos y Cierre

- **Acción del usuario:** Mateo abre su **Bandeja de Pagos** (`/bandeja/pagos`), revisa las capturas subidas por Andrés y Lucía, y presiona **"Confirmar"** en ambos pagos.
- **Comportamiento del sistema:** Dentro de una transacción con bloqueo sobre la deuda y el pago, el estado cambia a `CONFIRMADO`, el monto se resta de `reserved_cents` y se suma a `confirmed_cents`. El balance del evento (`/eventos/:id/balance`) se actualiza al instante mostrando las cuentas al día.

### 2.5. Requisitos Funcionales (v1.0)

#### RF1: Gestión de identidad, perfil de cobro y trazabilidad

- **Descripción:** El sistema permite registrar usuarios con `username` y correo únicos, iniciar sesión y configurar sus datos de cobro (billetera preferida, teléfono, cuenta bancaria/CCI y QR).
- **Criterio de aceptación técnico:** Las contraseñas se almacenan únicamente con hash Argon2. Toda ruta protegida exige una sesión activa validada por `SessionGuard` y token CSRF en operaciones de escritura (`POST`, `PUT`, `PATCH`, `DELETE`), registrando el `user_id` y `correlation_id` en `audit_logs`.

#### RF2: Agrupación y límites de contexto (eventos e invitaciones)

- **Descripción:** Un usuario puede crear un evento e invitar a otros usuarios registrados buscándolos por su nombre de usuario o correo.
- **Criterio de aceptación técnico:** Toda consulta o modificación dentro de un evento verifica primero con `assertParticipant(eventId, userId)` que el usuario pertenezca al grupo; de lo contrario, se rechaza la operación para impedir accesos no autorizados (IDOR).

#### RF3: Procesamiento asíncrono de extracción OCR

- **Descripción:** El sistema recibe fotos de comprobantes, las encola para que el Worker extraiga el total y los ítems en segundo plano, e informa el estado (`PROCESANDO`, `EXTRAÍDO`, `FALLIDO`, `CONFIRMADO`).
- **Criterio de aceptación técnico (SW707):** La lectura OCR corre fuera del proceso de la API usando la cola transaccional `receipt_jobs`. Si el OCR falla, el usuario cuenta con reintento manual y con un flujo de ingreso manual de respaldo.

#### RF4: Motor de fraccionamiento y generación de deuda (contratos)

- **Descripción:** Con los datos del comprobante revisados por el pagador, el sistema divide el gasto en partes iguales (`EQUITATIVE`) o por unidades consumidas (`BY_ITEM`) y crea las deudas hacia el pagador.
- **Criterio de aceptación técnico (SW707):** El calculador verifica por precondición que los ítems y cargos cuadren con el total, y por postcondición que la suma de las cuotas en centavos sea idéntica a `total_cents`.

#### RF5: Amortización, evidencia y control transaccional (concurrencia)

- **Descripción:** Los deudores registran pagos parciales o totales adjuntando evidencia gráfica (obligatoria para medios digitales), los cuales quedan pendientes hasta que el acreedor los confirme o rechace en su bandeja.
- **Criterio de aceptación técnico (SW707):**
  1. Toda operación sobre deudas y pagos bloquea las filas involucradas con `SELECT ... FOR UPDATE` para ejecutarse de manera secuencial y evitar _Lost Updates_.
  2. Si el monto supera el saldo disponible al momento de adquirir el bloqueo, se rechaza con HTTP 409 (`CONFLICT`).
  3. Se exige `Idempotency-Key` para evitar pagos duplicados ante reintentos de red.
  4. Las imágenes se validan por tamaño ($\le$ 5 MB) y _magic bytes_ reales antes de abrir la transacción en base de datos.

#### RF6: Consulta de deudas, compensación y balance del evento

- **Descripción:** Los participantes consultan lo que deben, lo que les deben, el detalle de compensaciones mutuas y la tabla de balance general del evento.
- **Criterio de aceptación técnico:** Los saldos mostrados respetan en todo momento la ecuación contable de la matriz de deudas y solo son accesibles para miembros del evento.

#### Trazabilidad entre requisitos y técnicas del curso

| Requisito | Técnica principal                                         | Sección |
| :-------: | :-------------------------------------------------------- | :-----: |
|    RF1    | Programación defensiva, manejo de errores y sesiones      | 6.2, 7  |
|    RF2    | Programación defensiva (aislamiento por evento)           |   6.2   |
|    RF3    | Tolerancia a fallas, análisis de problemas de runtime     |  5, 9   |
|    RF4    | _Design by Contract_ y conservación de centavos           | 5, 6.1  |
|    RF5    | Concurrencia, bloqueo pesimista e idempotencia            |    8    |
|    RF6    | _Design by Contract_ (invariantes), pruebas automatizadas | 6.1, 10 |

### 2.6. Requisitos no funcionales (v1.0)

- **RNF1. Consistencia e integridad:** Los saldos de todas las deudas cumplen siempre la invariante `confirmed_cents + reserved_cents <= amount_cents`, incluso bajo peticiones simultáneas.
- **RNF2. Disponibilidad ante fallas del OCR:** La API responde en milisegundos al subir un comprobante sin quedarse esperando al OCR, y si la lectura falla el usuario puede completarla manualmente sin bloquear el evento.
- **RNF3. Seguridad defensiva:** Contraseñas con Argon2, bloqueo temporal tras 5 intentos fallidos de login, cookies `httpOnly` + token CSRF, validación binaria de imágenes (_magic bytes_) y verificación de pertenencia al evento en cada endpoint.
- **RNF4. Trazabilidad:** Propagación de `correlationId` con `AsyncLocalStorage` en logs estructurados JSON y registro de acciones críticas en la tabla `audit_logs`.
- **RNF5. Verificabilidad:** Pruebas automatizadas unitarias, de propiedades con `fast-check`, de componentes en Vitest y de concurrencia/integración contra PostgreSQL real con Testcontainers.

---

## 3. Arquitectura y decisiones técnicas

### 3.1. Vista general de la arquitectura

SplitVision está construido bajo una arquitectura de **monolito modular** compuesta por tres piezas principales que comparten el mismo repositorio: una aplicación web SPA en React (`/App/frontend`), un servidor API REST en NestJS (`src/main.ts`) y un proceso Worker en segundo plano (`src/worker.ts`), respaldados por una base de datos PostgreSQL y almacenamiento local de imágenes.

La API atiende todas las peticiones HTTP de los usuarios y ejecuta las transacciones financieras sobre la base de datos, mientras que el Worker se dedica exclusivamente a desencolar y procesar las imágenes de los comprobantes con el motor OCR.

### 3.2 C4 Nivel 1 – Diagrama de contexto

![Diagrama de contexto](assets/c4/Contexto.png)

### 3.3 C4 Nivel 2 – Diagrama de contenedores

![Diagrama de contenedores](assets/c4/Contenedores.png)

### 3.4 C4 Nivel 3 – Diagrama de componentes

![Diagrama de componentes1](assets/c4/Componentes_API.png)

![Diagrama de componentes2](assets/c4/Componentes_Worker.png)

### 3.5. Tecnologías utilizadas

#### 3.5.1. Frontend (`/App/frontend`)

| Tecnología            | Propósito                                                                           |
| :-------------------- | :---------------------------------------------------------------------------------- |
| React (v19)           | Construcción de las vistas y componentes interactivos de la interfaz                |
| TypeScript (v5.9)     | Tipado estático de respuestas de API, estados y propiedades                         |
| Vite (v8)             | Empaquetador y servidor de desarrollo rápido                                        |
| React Router DOM (v7) | Navegación entre páginas y protección de rutas (`RutaProtegida` / `RutaPublica`)    |
| TanStack Query (v5)   | Sincronización con el backend, caché y _polling_ automático de estados OCR y saldos |
| Tailwind CSS (v4)     | Sistema de diseño "Ledger" con variables CSS en OKLCH y soporte para modo oscuro    |

#### 3.5.2. Backend (`/App`)

| Tecnología        | Propósito                                                                                                                                 |
| :---------------- | :---------------------------------------------------------------------------------------------------------------------------------------- |
| Node.js (v22+)    | Entorno de ejecución para los procesos de la API y del Worker                                                                             |
| NestJS (v11)      | Estructura modular (`identidad`, `eventos`, `comprobantes`, `reparto`, `pagos`, `consultas`, `ocr`), guardias e inyección de dependencias |
| TypeORM & `pg`    | Conexión a PostgreSQL, manejo de transacciones explícitas (`withTx`) y consultas SQL parametrizadas con bloqueo pesimista                 |
| PostgreSQL 16/18  | Base de datos relacional transaccional y soporte de la cola de trabajos `receipt_jobs`                                                    |
| Tesseract.js (v7) | Motor OCR local en WebAssembly ejecutado dentro del Worker                                                                                |
| Opossum           | Circuit Breaker (`crearOcrBreaker`) para proteger la extracción OCR y activar el fallback local                                           |
| `@node-rs/argon2` | Hashing seguro de contraseñas                                                                                                             |
| Multer            | Recepción de archivos en memoria (`FileInterceptor`) previo a la validación de _magic bytes_ en `StorageService`                          |

#### 3.5.3. Pruebas y verificación

| Tecnología            | Propósito                                                                                       |
| :-------------------- | :---------------------------------------------------------------------------------------------- |
| Jest & `ts-jest`      | Pruebas unitarias, de integración y de concurrencia en el backend                               |
| Vitest (v5)           | Pruebas de componentes y flujos en el frontend                                                  |
| `fast-check`          | _Property-Based Testing_ para comprobar las invariantes del cálculo de reparto (1,000 corridas) |
| Testcontainers        | Contenedores PostgreSQL reales levantados automáticamente durante los tests                     |
| React Testing Library | Pruebas de interfaz simulando la interacción real del usuario                                   |

#### 3.5.4. Infraestructura

| Tecnología              | Propósito                                                       |
| :---------------------- | :-------------------------------------------------------------- |
| Docker & Docker Compose | Contenerización de PostgreSQL, API y Worker                     |
| Monorepo estructurado   | Código unificado con backend en `/App` y SPA en `/App/frontend` |

### 3.6 Decisiones técnicas

#### 3.6.1 Arquitectura monolítica modular

Elegimos un monolito modular en NestJS dividido por dominios de negocio (`identidad`, `eventos`, `comprobantes`, `reparto`, `pagos`, `consultas` y `ocr`). Esto nos permite tener transacciones SQL directas entre módulos sin la complejidad de red que implicaría separar microservicios para un equipo de 3 personas.

#### 3.6.2 Procesamiento OCR en un proceso Worker separado

La lectura de imágenes se separó en un ejecutable independiente (`worker.ts`) para que el consumo pesado de CPU y memoria de Tesseract.js nunca afecte el tiempo de respuesta de la API web.

#### 3.6.3 Cola transaccional sobre PostgreSQL (`receipt_jobs` con `FOR UPDATE SKIP LOCKED`)

En vez de instalar servicios adicionales como Redis o RabbitMQ, implementamos la cola de trabajos directamente en PostgreSQL con la tabla `receipt_jobs`. Para que un Worker tome una tarea disponible o recupere una tarea cuyo tiempo de arriendo (_lease_) expiró, ejecuta:

```sql
SELECT id FROM receipt_jobs
WHERE (status = 'PENDING' AND next_attempt_at <= now())
   OR (status = 'RUNNING' AND lease_expires_at < now())
ORDER BY next_attempt_at
FOR UPDATE SKIP LOCKED
LIMIT 1
```

Esto nos da tres ventajas claras:

1. **Sin choques entre workers:** `SKIP LOCKED` hace que si hay varios workers corriendo, cada uno tome una boleta distinta sin bloquearse.
2. **Atomicidad:** El comprobante y su trabajo OCR se insertan en la misma transacción (`withTx`), por lo que nunca queda una boleta subida sin su tarea en cola.
3. **Recuperación ante caídas:** Si el Worker se apaga a mitad de una lectura, cuando vence `lease_expires_at` la tarea vuelve a ser tomada automáticamente.

#### 3.6.4 Bloqueo pesimista y compensación de deudas cruzadas

Tanto el registro y resolución de pagos como la compensación automática de deudas mutuas (`compensateCrossDebts`) usan transacciones con `SELECT ... FOR UPDATE` sobre las filas de `debts`. Así aseguramos que dos operaciones concurrentes jamás modifiquen los saldos al mismo tiempo.

#### 3.6.5 Patrón Adapter y Circuit Breaker en el subsistema OCR

Mediante la interfaz `OcrClient` (`ocr.client.ts`) y el Circuit Breaker de `breaker.ts`, el Worker desacopla quién hace el reconocimiento óptico. Permite usar un proveedor externo (`HttpOcrClient`) protegido por `opossum` que salta automáticamente al motor local (`TesseractOcrClient`) si el servicio externo demora o cae, además de contar con `FakeOcrClient` para inyectar fallas controladas durante los tests.

#### 3.6.6 Verificación humana obligatoria del resultado OCR

Como ninguna lectura óptica de un ticket arrugado es 100% segura, el sistema nunca crea deudas directamente desde el OCR: siempre deja el comprobante en estado `EXTRAÍDO` para que el pagador revise los ítems y confirme los montos antes de pasar al reparto.

### 3.7 Modelo de datos

El diagrama Entidad-Relación refleja las tablas creadas por el motor de migraciones (`src/db/migrate.ts`), garantizando la integridad referencial desde que se crea el evento hasta que se audita cada pago.

![Entidad Relación](assets/modelo-datos/relacion.png)

---

## 4. Dependencias y reúso

Para gestionar las dependencias seguimos una regla clara: **usar librerías probadas para tareas de infraestructura o seguridad (como criptografía, conexión a base de datos o circuit breaker), y programar nosotros mismos toda la lógica del negocio (cálculos en centavos, reparto de residuos, cola SQL y control de concurrencia)**.

### 4.1 Gestión de dependencias directas

Tanto el backend (`/App`) como el frontend (`/App/frontend`) mantienen sus versiones fijadas con `package-lock.json` para asegurar que todo el equipo y el entorno Docker instalen exactamente los mismos paquetes.

#### 4.1.1 Dependencias directas del Backend (`/App`)

| Paquete                                                      | Versión               | Tipo | Justificación técnica                                                                                                                                       | Licencia   |
| :----------------------------------------------------------- | :-------------------- | :--- | :---------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------- |
| `@nestjs/core`, `@nestjs/common`, `@nestjs/platform-express` | `^11.0.0`             | Prod | Estructura modular del servidor HTTP, guardias de sesión (`SessionGuard`), interceptores de archivos y filtro global de excepciones (`AppExceptionFilter`). | MIT        |
| `@nestjs/typeorm`, `typeorm`                                 | `^11.0.0`, `^0.3.20`  | Prod | Mapeo de entidades y manejo del `QueryRunner` para transacciones explícitas (`withTx`).                                                                     | MIT        |
| `pg`                                                         | `^8.13.0`             | Prod | Driver oficial de PostgreSQL para el pool de conexiones y la ejecución de migraciones con bloqueo consultivo (`pg_advisory_lock`).                          | MIT        |
| `@node-rs/argon2`                                            | `^2.0.2`              | Prod | Hashing seguro de contraseñas en código nativo Rust, resistente a ataques por fuerza bruta con GPU.                                                         | MIT        |
| `tesseract.js`                                               | `^7.0.0`              | Prod | Motor OCR local en WebAssembly para leer boletas en español sin depender de servicios de pago en la nube.                                                   | Apache-2.0 |
| `opossum`                                                    | `^8.0.1`              | Prod | Implementación del patrón _Circuit Breaker_ para cortar llamadas colgadas al OCR y activar el fallback local.                                               | Apache-2.0 |
| `fast-check`                                                 | `^3.23.0`             | Dev  | Generador de casos aleatorios (_Property-Based Testing_) para poner a prueba las invariantes del motor de división de gastos.                               | MIT        |
| `testcontainers`                                             | `^12.2.1`             | Dev  | Levanta contenedores temporales de PostgreSQL en Docker para correr las pruebas de integración y concurrencia sobre una BD real.                            | MIT        |
| `jest`, `ts-jest`                                            | `^30.5.2`, `^29.4.14` | Dev  | Ejecución de pruebas automatizadas del backend.                                                                                                             | MIT        |

#### 4.1.2 Dependencias directas del Frontend (`/App/frontend`)

| Paquete                            | Versión             | Tipo     | Justificación técnica                                                                | Licencia |
| :--------------------------------- | :------------------ | :------- | :----------------------------------------------------------------------------------- | :------- |
| `react`, `react-dom`               | `^19.3.0`           | Prod     | Construcción de componentes y manejo de estado de la interfaz web.                   | MIT      |
| `react-router-dom`                 | `^7.18.4`           | Prod     | Enrutamiento declarativo entre las vistas de la SPA.                                 | MIT      |
| `@tanstack/react-query`            | `^5.104.1`          | Prod     | Manejo de peticiones al backend, caché e invalidación automática tras cada mutación. | MIT      |
| `tailwindcss`, `@tailwindcss/vite` | `^4.3.3`            | Dev/Prod | Estilos CSS basados en utilidades y variables de tema claro/oscuro.                  | MIT      |
| `vitest`, `@testing-library/react` | `^5.0.3`, `^16.3.3` | Dev      | Pruebas automatizadas de los componentes y páginas en React.                         | MIT      |

### 4.2 Análisis de dependencias transitivas y riesgos de seguridad

Revisamos el árbol completo de paquetes instalados ejecutando `npm audit` tanto en el backend como en el frontend.

#### 4.2.1 Estado de vulnerabilidades (Auditoría v1.0)

- **Backend (`/App`):** `found 0 vulnerabilities` en los 571 paquetes del árbol de dependencias.
- **Frontend (`/App/frontend`):** `found 0 vulnerabilities` en los 695 paquetes instalados.

#### 4.2.2 Medidas de control sobre dependencias transitivas

1. **Instalaciones reproducibles (`package-lock.json`):** Versionamos los archivos lock en Git para que `npm ci` instale exactamente las mismas sub-dependencias en cualquier máquina.
2. **Parcheo forzado con `overrides`:** Para no depender de que librerías de terceros actualicen sus propias dependencias internas, forzamos en el `package.json` versiones seguras de paquetes transitivos como `js-yaml`:
   ```json
   "overrides": {
     "js-yaml": "^4.1.0"
   }
   ```
3. **Aislamiento del binario WebAssembly:** Como `tesseract.js` ejecuta código complejo compilado en WebAssembly, lo mantuvimos aislado dentro del proceso Worker; así cualquier fallo interno al procesar una imagen pesada no afecta al servidor de la API.

### 4.3 Criterio de reúso frente a desarrollo propio

| Componente                                 | Decisión                                                           | Justificación técnica                                                                                                                                                                                                    |
| :----------------------------------------- | :----------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cifrado de contraseñas**                 | **Reúso:** `@node-rs/argon2`                                       | En seguridad nunca se debe inventar criptografía propia; Argon2 es el estándar actual recomendado para contraseñas.                                                                                                      |
| **Circuit Breaker**                        | **Reúso:** `opossum`                                               | Provee el control de estados (`CLOSED`, `OPEN`, `HALF-OPEN`), contadores de fallos y _timeouts_ ya probados en producción.                                                                                               |
| **Validación y normalización de entradas** | **Desarrollo propio:** validadores en servicios y `money.ts`       | En vez de depender de decoradores mágicos, programamos validaciones explícitas (`toCents`, `detectImageKind` por _magic bytes_ y expresiones regulares) que nos dan control total sobre los mensajes y códigos de error. |
| **Cola de trabajos asíncronos**            | **Desarrollo propio:** `receipt_jobs` con `FOR UPDATE SKIP LOCKED` | Evita levantar servicios extras como Redis y nos permite encolar el trabajo en la misma transacción SQL donde se guarda la boleta.                                                                                       |
| **Algoritmo de división de gastos**        | **Desarrollo propio:** `split-calculator.ts`                       | La regla para repartir centavos sobrantes sin perder dinero es parte central de nuestro dominio y debía estar bajo nuestro control y verificada por contratos.                                                           |
| **Control de concurrencia y compensación** | **Desarrollo propio:** `pagos.service.ts` y `eventos.service.ts`   | La lógica de saldos reservados (`reserved_cents`), confirmación en bandeja y cruce de deudas mutuas se programó a medida usando transacciones con `SELECT ... FOR UPDATE`.                                               |

---

## 5. Problemas de runtime

Cuando el software pasa de correr una prueba aislada a ejecutarse con usuarios reales subiendo fotos y registrando pagos al mismo tiempo, aparecen problemas de tiempo de ejecución (_runtime_) que no se ven al compilar. En SplitVision identificamos y solucionamos dos problemas importantes de runtime:

### 5.1 Problema 1: Fugas de memoria (_Memory Leaks_) y bloqueo del Event Loop por el motor OCR

#### A. Diagnóstico y causa técnica

Para leer las boletas localmente usamos `tesseract.js`, que carga en Node.js el motor Tesseract compilado en WebAssembly (`.wasm`) junto con el diccionario del idioma español (`spa.traineddata`, de más de 15 MB).

Durante las pruebas encontramos dos riesgos graves en tiempo de ejecución:

1. **Fuga de memoria nativa:** La memoria que reserva WebAssembly vive fuera del _Garbage Collector_ normal de JavaScript. Si creamos un worker de Tesseract por cada imagen y nos olvidamos de cerrarlo con `worker.terminate()` (por ejemplo, cuando ocurre una excepción a mitad de lectura), esos bloques de memoria nunca se liberan y tras unas cuantas boletas el proceso muere con el error:
   ```text
   FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory
   ```
2. **Congelamiento del Event Loop:** El reconocimiento óptico hace uso intensivo del procesador durante varios segundos. Si corriera dentro del proceso principal de la API, bloquearía el hilo de Node.js y ningún otro usuario podría ni siquiera iniciar sesión o ver sus eventos mientras se analiza una boleta.

#### B. Solución implementada

1. **Separación en proceso Worker (`src/worker.ts`):** Sacamos toda la ejecución del OCR a un proceso aparte que lee la tabla `receipt_jobs`. Así la API responde al instante y su _Event Loop_ queda libre para atender peticiones web.
2. **Liberación garantizada de recursos:** El cliente de Tesseract gestiona de forma controlada la vida del worker nativo asegurando su cierre incluso si la imagen lanza un error de lectura.
3. **Corte por tiempo límite (_Timeout_):** Envolvemos la llamada en el Circuit Breaker de `opossum` (`breaker.ts`) con un `timeoutMs` estricto para abortar cualquier lectura que se quede colgada por una imagen dañada.

---

### 5.2 Problema 2: Agotamiento del pool de conexiones y retención de bloqueos

#### A. Diagnóstico y causa técnica

El backend se conecta a PostgreSQL mediante un _Connection Pool_ con un número limitado de conexiones simultáneas. Cuando abrimos una transacción con `withTx` y ejecutamos un `SELECT ... FOR UPDATE`, esa conexión queda ocupada y la fila de la deuda queda bloqueada hasta que se haga `COMMIT` o `ROLLBACK`.

El problema de runtime ocurre si metemos operaciones lentas de disco o de cálculo pesado _adentro_ de la transacción abierta. Por ejemplo, al subir un comprobante o registrar un pago con foto de evidencia: si primero abrimos la transacción SQL, bloqueamos la deuda y recién ahí validamos los 5 MB de la imagen, calculamos su hash SHA-256 y escribimos el archivo en el disco duro, la transacción se queda abierta muchísimo más tiempo del necesario. Si varios usuarios suben pagos a la vez, se acaban las conexiones disponibles del pool y las demás peticiones fallan por _timeout_.

#### B. Solución implementada

Aplicamos la regla de **mantener las transacciones SQL lo más cortas posibles**, sacando todo el trabajo de archivos fuera de la transacción:

En `comprobantes.service.ts` (`upload`) y `pagos.service.ts`:

1. **Fase 1 — Fuera de la transacción (I/O de disco y hashing en memoria):**
   - Primero verificamos permisos rápidos y llamamos a `this.storage.saveImage(buffer, ...)`.
   - Ahí se validan los _magic bytes_, se calcula el hash SHA-256 y se escribe el archivo en `./data/images` sin tocar ninguna transacción ni bloquear filas en PostgreSQL.
2. **Fase 2 — Transacción ultrarrápida (`withTx`):**
   - Recién con el archivo ya guardado en disco, abrimos la transacción `READ COMMITTED`, tomamos el bloqueo `SELECT ... FOR UPDATE` (en el caso de pagos), ejecutamos únicamente los `INSERT`/`UPDATE` necesarios y hacemos `COMMIT` de inmediato.
3. **Limpieza compensatoria en caso de fallo:**
   - Si la transacción SQL falla (por ejemplo, porque el saldo disponible ya no alcanzaba o la imagen estaba duplicada), el bloque `catch` llama a `this.storage.delete(path)` para borrar la imagen recién creada y no dejar archivos basura huérfanos en el disco.

---

## 6. Contratos y programación defensiva

En el proyecto combinamos dos técnicas que cumplen roles distintos según en qué parte del código estemos:

- **Programación Defensiva (en la frontera externa):** Todo lo que viene de afuera (formularios del usuario, parámetros en la URL, imágenes subidas o el texto que escupe el OCR) se considera **no confiable**. Lo validamos y limpiamos apenas entra.
- **Design by Contract - DbC (en el núcleo interno):** Una vez que los datos ya están validados y dentro de nuestros servicios de dominio, nuestras funciones operan bajo contratos claros de _Precondiciones_, _Postcondiciones_ e _Invariantes_.

```text
[ Usuario / Navegador / Archivos / Motor OCR ]
               │
               ▼  (Datos externos no confiables)
   ┌───────────────────────┐
   │ PROGRAMACIÓN DEFENSIVA│ ──► Magic bytes (detectImageKind), regex, límites de 5 MB,
   └───────────────────────┘     aislamiento por evento (assertParticipant) y revisión humana de OCR.
               │
               ▼  (Datos ya sanitizados y en enteros)
   ┌───────────────────────┐
   │  DESIGN BY CONTRACT   │ ──► split-calculator.ts, pagos.service.ts, money.ts
   │         (DbC)         │     Precondiciones ──► Ejecución ──► Postcondiciones & Invariantes
   └───────────────────────┘
```

### 6.1. Contratos (Design by Contract)

Documentamos los **3 componentes principales** que trabajan bajo contratos en nuestro código:

#### Contrato 1: Motor de división de gastos y asignación de residuos (`src/modules/reparto/split-calculator.ts`)

Es el encargado de dividir el total de un comprobante entre los participantes, tanto en modo equitativo (`EQUITATIVE`) como por ítems (`BY_ITEM`).

- **Precondiciones:**
  1. `totalCents` debe ser un entero mayor a cero (`totalCents > 0`).
  2. En reparto equitativo, debe haber al menos un participante seleccionado (`participantIds.length > 0`).
  3. En reparto por ítems, todas las unidades (`1..quantity`) de cada ítem deben estar asignadas a un participante válido del evento, y el cuadre de ítems más impuestos/propina menos descuento debe coincidir con `totalCents`.
- **Postcondiciones:**
  1. Devuelve la lista de partes (`parts`) donde cada participante recibe un monto entero no negativo en centavos.
  2. El cálculo es 100% determinista: ante los mismos datos de entrada, los centavos sobrantes se asignan siempre a las mismas personas (desempatando por orden de `userId`).
- **Invariante de Dominio (conservación de masa monetaria):**
  La suma de los centavos asignados a todos los participantes es **exactamente igual** al total del comprobante, sin perder ni inventar ni un céntimo:
  $$\sum_{i=1}^{n} \text{parts}[i]\text{.cents} \equiv \text{totalCents}$$

Para que nunca falte un centavo (como en S/ 100.00 dividido entre 3), `distribute()` calcula primero la cuota entera base ($\lfloor 10000 / 3 \rfloor = 3333$ céntimos) y reparte el residuo ($10000 \bmod 3 = 1$ céntimo) sumando `1` céntimo a los primeros participantes según el orden determinista (quedando `3334`, `3333` y `3333`, cuya suma da `10000`).

---

#### Contrato 2: Registro y reserva de pagos (`src/modules/pagos/pagos.service.ts`)

Regula el momento en que un deudor registra un abono para pagar total o parcialmente su deuda.

- **Precondiciones:**
  1. `amountCents` debe ser un entero mayor a cero (`amountCents > 0`).
  2. El medio de pago debe ser uno de los valores válidos: `EFECTIVO`, `YAPE`, `PLIN`, `TRANSFERENCIA` u `OTRO`.
  3. Si el método es distinto de `EFECTIVO`, debe haberse adjuntado una imagen de evidencia válida.
  4. El usuario en sesión debe ser el deudor de esa obligación (`debt.debtor_id === userId`).
  5. Evaluado dentro del bloqueo `FOR UPDATE`, el monto no puede superar el saldo disponible actual:
     $$\mathrm{amountCents} \le \mathrm{amount\_cents} - (\mathrm{confirmed\_cents} + \mathrm{reserved\_cents})$$
- **Postcondiciones:**
  1. Se crea el pago en estado `PENDIENTE` asociado a su `idempotency_key`.
  2. El campo `reserved_cents` de la deuda aumenta exactamente en `amountCents`.
  3. Se inserta la traza de auditoría en `audit_logs` dentro de la misma transacción.
- **Invariante de Dominio:**
  Respaldada también por el constraint SQL `debts_balances_check`, en todo instante se cumple que:
  $$0 \le \mathrm{confirmed\_cents} + \mathrm{reserved\_cents} \le \mathrm{amount\_cents}$$

---

#### Contrato 3: Conversión y aritmética monetaria sin punto flotante (`src/common/money.ts`)

Para evitar errores clásicos de punto flotante en JavaScript (donde por ejemplo `19.99 * 100` en IEEE 754 da `1998.9999999999998`), el módulo `money.ts` convierte las entradas directamente mediante manipulación de cadenas de texto:

- **Precondiciones de `toCents(value, field)`:**
  1. `value` debe ser un `number` o `string` que, tras hacer `trim()`, cumpla estrictamente la expresión regular `/^\d{1,9}(\.\d{1,2})?$/` (número no negativo de hasta 9 dígitos enteros y máximo 2 decimales).
  2. Se rechaza cualquier valor negativo, texto con letras, `NaN`, `null` o con más de 2 decimales lanzando `validation()`.
- **Postcondiciones:**
  1. Separa la parte entera y decimal por el punto (`raw.split('.')`) y calcula el entero exacto en centavos mediante `Number(intPart) * 100 + Number((decPart + '00').slice(0, 2))`, verificando además `Number.isSafeInteger(cents)`.
  2. La función inversa `formatCents(cents)` devuelve siempre el texto formateado con 2 decimales exactos usando división entera y módulo (`Math.floor(abs / 100)` y `abs % 100`).
- **Invariante:**
  Toda columna monetaria en PostgreSQL (`total_cents`, `unit_price_cents`, `amount_cents`, `confirmed_cents`, `reserved_cents`) es de tipo `integer` y jamás almacena decimales.

---

### 6.2. Programación Defensiva

Implementamos cuatro barreras defensivas concretas en el código:

#### 1. Validación programática en la entrada de los servicios

Cada servicio valida y limpia sus parámetros antes de ejecutar consultas SQL:

- **Formatos estrictos con Regex:** Los nombres de usuario se validan con `/^[A-Za-z0-9_]{3,20}$/` (`auth.service.ts`), las referencias de pago con `/^[A-Za-z0-9]{1,50}$/` y los teléfonos/CCI en la configuración de perfil (`auth.service.ts`). Además, estas mismas reglas existen como `CHECK CONSTRAINTS` en PostgreSQL (`001_init.sql`) como segunda línea de defensa.
- **Rechazo inmediato con `AppException`:** Si un dato no cumple, se lanza la excepción tipada `validation('mensaje')`, que corta la ejecución y responde HTTP 400.

#### 2. Inspección binaria de archivos (_Magic Bytes_) y protección contra _Path Traversal_ (`src/modules/comprobantes/storage.service.ts`)

Cuando un usuario sube una imagen (boleta, voucher de pago, foto de evento, avatar o QR), no confiamos en la extensión del archivo ni en el header `Content-Type` del navegador, porque un atacante podría renombrar un script malicioso como `foto.jpg`:

- **Detección por _Magic Bytes_ (`detectImageKind`):** Leemos directamente los primeros bytes del `Buffer` en binario: `0xFF 0xD8 0xFF` para JPEG, `0x89 0x50 0x4E 0x47 0x0D 0x0A 0x1A 0x0A` para PNG, y los encabezados ASCII `RIFF` + `WEBP` para WebP. Si el contenido binario no coincide con ninguno de los tres, se rechaza el archivo.
- **Tope de 5 MB:** Se verifica que `buffer.length <= MAX_BYTES()` (5 MB por defecto).
- **Renombrado seguro y prevención de _Path Traversal_:** Ignoramos por completo el nombre original del archivo subido y generamos uno nuevo con `${prefix}-${randomUUID()}.${ext}`. Además, al leer o borrar un archivo en `read(path)` y `delete(path)`, comprobamos que la ruta absoluta resuelta empiece dentro de nuestro directorio autorizado (`!abs.startsWith(this.dir)`), bloqueando cualquier intento de leer archivos del sistema mediante `../../`.

#### 3. Parser defensivo y revisión humana para datos del OCR (`recibo.parser.ts` y `comprobantes.service.ts`)

- Todo payload que sale del OCR pasa por un parser defensivo que descarta ítems sin descripción, cantidades no enteras o precios negativos/cero, y si el resultado no tiene sentido lanza error para marcar la boleta como `FALLIDO` en lugar de meter basura en la base de datos.
- Aun cuando el OCR tiene éxito, la boleta queda en estado `EXTRAÍDO` y nunca genera deudas hasta que el pagador revise los campos en pantalla y confirme el reparto.

#### 4. Aislamiento por Evento y Sesiones con CSRF (`eventos.service.ts` y `session.guard.ts`)

- **Prevención de IDOR:** Antes de mostrar o editar cualquier evento, comprobante, deuda o balance, el backend ejecuta `assertParticipant(eventId, userId)` o `assertDebtParticipant(debtId, userId)`. Si el usuario autenticado no es miembro de ese grupo, el sistema responde `notFound('evento no encontrado')` (HTTP 404), ocultando incluso la existencia del recurso frente a terceros.
- **Protección CSRF y fuerza bruta:** `SessionGuard` exige en cada petición `POST`, `PUT`, `PATCH` o `DELETE` que la cabecera `x-csrf-token` coincida con el token guardado en la sesión de base de datos. Asimismo, `login` bloquea temporalmente la cuenta si detecta 5 intentos fallidos en los últimos 10 minutos en la tabla `login_attempts`.

---

## 7. Estrategia de errores y excepciones

Para evitar bloques `try/catch` desordenados en cada controlador o respuestas de error inconsistentes hacia el frontend, implementamos una política centralizada de manejo de errores, excepciones y trazabilidad en tres capas (`src/common/errors.ts`, `src/common/filters.ts` y `src/common/correlation.ts`).

### 7.1. Taxonomía de excepciones de dominio (`src/common/errors.ts`)

Definimos una clase única de excepción de negocio llamada `AppException`, que recibe un código semántico (`ErrorCode`), un mensaje claro para el usuario y detalles opcionales. Cada código tiene asignado un único estado HTTP:

| Código interno (`ErrorCode`) | Helper en código                       | HTTP Status | Cuándo se utiliza en SplitVision                                                                                                                                                                           |
| :--------------------------- | :------------------------------------- | :---------: | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `VALIDATION`                 | `validation(msg)`                      |   **400**   | Datos de entrada inválidos: montos negativos o con formato incorrecto en `toCents`, imágenes que superan 5 MB o no pasan _magic bytes_, contraseñas cortas.                                                |
| `UNAUTHORIZED`               | `unauthorized(msg)` / `forbidden(msg)` |   **401**   | Credenciales incorrectas en login, sesión expirada, falta de cookie `sid`, token CSRF inválido o intento de editar un comprobante por alguien que no es el pagador.                                        |
| `NOT_FOUND`                  | `notFound(msg)`                        |   **404**   | Recurso inexistente o usuario que intenta acceder a un evento, comprobante o deuda al que no pertenece (`assertParticipant`).                                                                              |
| `CONFLICT`                   | `conflict(msg)`                        |   **409**   | Violación de reglas de estado o concurrencia: intentar pagar más del saldo disponible, editar una boleta ya `CONFIRMADO`, registrar un username/correo duplicado o superar el límite de intentos de login. |
| `EXTERNAL`                   | `external(msg)` / `OcrExternalError`   |   **502**   | Fallo de comunicación o _timeout_ al invocar al servicio OCR externo.                                                                                                                                      |
| `INTERNAL`                   | Por defecto en fallos no esperados     |   **500**   | Errores inesperados de infraestructura o programación no previstos.                                                                                                                                        |

### 7.2. Filtro Global de Excepciones (`src/common/filters.ts`)

Registramos `AppExceptionFilter` como filtro global de NestJS (`@Catch()`). Su trabajo es interceptar cualquier excepción que ocurra durante una petición HTTP y transformarla en una respuesta JSON uniforme con la estructura:

```json
{
  "code": "CONFLICT",
  "message": "el monto supera el saldo disponible de la deuda",
  "correlationId": "b6d9f1a2-4c3e-4a8b-9d1e-2f5a7c8b9e01"
}
```

Además de capturar `AppException`, el filtro traduce automáticamente errores técnicos de librerías o de PostgreSQL para que nunca se filtren detalles internos ni _stack traces_ al navegador:

- **Errores de subida de archivos (`MulterError` / `BodyTooLargeError`):** Se convierten en `400 VALIDATION` indicando si el archivo superó los 5 MB.
- **UUIDs mal formados en URLs (código PostgreSQL `22P02`):** Si alguien envía `/api/events/abc-no-es-uuid`, PostgreSQL lanza el error `22P02`; el filtro lo atrapa y devuelve limpiamente `400 VALIDATION` (_"parámetro inválido: identificador con formato incorrecto"_) en vez de un error 500.
- **Choques de unicidad (código PostgreSQL `23505`):** Si dos peticiones simultáneas chocan contra un `UNIQUE INDEX`, se traduce automáticamente a `409 CONFLICT` (_"el recurso ya existe"_).

### 7.3. Trazabilidad con `Correlation ID` y Logging Estructurado (`src/common/correlation.ts`)

Cuando hay varios usuarios operando a la vez y un Worker corriendo en paralelo, un log suelto en consola no sirve si no sabemos a qué petición pertenece. Por eso implementamos:

1. **Propagación automática con `AsyncLocalStorage`:** Al entrar cada solicitud HTTP (o al encolarse un trabajo OCR en `receipt_jobs`), se genera un UUID de correlación (`correlationId`) y se ejecuta el flujo dentro de `runWithCorrelation(id, fn)`. Así, cualquier llamada a `logger.info`, `logger.warn`, `logger.error` o `audit()` en cualquier servicio obtiene automáticamente el mismo `correlationId` sin tener que pasarlo como argumento por todas las funciones.
2. **Niveles de log diferenciados en formato JSON:**
   - Los errores de cliente o de reglas de negocio (`status < 500`) se registran con `logger.warn('error controlado', { path, code, status, message, correlationId })`.
   - Los fallos graves (`status >= 500`) se registran con `logger.error('error no controlado', { path, exception, stack, correlationId })` incluyendo la traza completa para depuración interna, pero al usuario solo se le responde `"error interno"` junto con el `correlationId`.
3. **Bitácora de auditoría (`audit_logs`):** Todas las acciones sensibles (registro, login exitoso o fallido, creación de evento, subida/eliminación de boleta, confirmación de reparto, pagos y compensaciones) dejan un registro en la tabla `audit_logs` con el `user_id`, la operación y el `correlation_id`.

### 7.4. Manejo de errores en el Frontend (`client.ts` y `es-PE.ts`)

En el cliente React, el wrapper `apiFetch` (`src/api/client.ts`) captura toda respuesta no exitosa y lanza una instancia de `ApiError(status, code, message, correlationId)`:

- Si el servidor devuelve `401` en cualquier ruta distinta al login/registro, `apiFetch` dispara automáticamente el evento global `window.dispatchEvent(new CustomEvent('sesion:caducada'))`, el cual es escuchado por `SesionContext.tsx` para limpiar el estado del usuario y redirigirlo a `/login` sin dejar pantallas rotas.
- La función `mensajeDeError(err)` (`src/i18n/es-PE.ts`) traduce los códigos de error (`VALIDATION`, `CONFLICT`, `NETWORK`, etc.) a mensajes claros en español peruano que se muestran mediante los componentes `<Alerta>` y `<Toast>`.

---

## 8. Concurrencia

El manejo de concurrencia es el corazón transaccional de SplitVision. Como varios integrantes de un grupo pueden interactuar al mismo tiempo sobre las mismas deudas y comprobantes, implementamos cuatro mecanismos concretos en el código para evitar condiciones de carrera:

### 8.1. Bloqueo pesimista a nivel de fila (`SELECT ... FOR UPDATE`) en Deudas y Pagos

Todas las operaciones que modifican el estado de una deuda o de un pago se ejecutan dentro del helper transaccional `withTx(this.dataSource, async (em) => { ... })` (`src/common/db.ts`), que abre una transacción con nivel de aislamiento `READ COMMITTED`.

Dentro de la transacción, antes de verificar si el saldo alcanza o si un pago puede confirmarse/cancelarse, bloqueamos la fila de la deuda en PostgreSQL con `SELECT ... FOR UPDATE`:

1. **Al registrar un nuevo pago (`pagos.service.ts`):**
   Se bloquea la fila de `debts` correspondiente. Si dos peticiones concurrentes llegan al mismo tiempo para pagar la misma deuda, PostgreSQL le entrega el bloqueo a la primera y deja a la segunda esperando hasta que la primera haga `COMMIT`.
   - La primera petición lee `available = amount_cents - confirmed_cents - reserved_cents`, verifica que `amountCents <= available`, inserta el pago `PENDIENTE` y actualiza `reserved_cents = reserved_cents + amountCents`.
   - Recién cuando la primera hace `COMMIT`, la segunda petición despierta y lee los valores **ya actualizados** de `reserved_cents`. Si el nuevo saldo disponible ya no alcanza, lanza `conflict()` (HTTP 409) y hace `ROLLBACK`.
2. **Al confirmar, rechazar o cancelar un pago (`pagos.service.ts`):**
   Cuando el deudor intenta cancelar su pago pendiente al mismo instante en que el acreedor intenta confirmarlo desde su bandeja, el bloqueo `FOR UPDATE` serializa ambas acciones: la primera cambia el estado del pago de `PENDIENTE` a `CONFIRMADO` (o `CANCELADO`), y cuando entra la segunda ve que el pago ya no está `PENDIENTE`, rechazando la operación con HTTP 409 sin descontar ni liberar dos veces el saldo reservado.
3. **En la compensación automática de deudas cruzadas (`compensateCrossDebts` en `eventos.service.ts`):**
   Cuando se cruzan deudas mutuas entre participantes de un evento, el servicio ejecuta:
   ```sql
   SELECT id, debtor_id, creditor_id, amount_cents, confirmed_cents, reserved_cents
   FROM debts
   WHERE event_id = $1 AND (amount_cents - confirmed_cents - reserved_cents) > 0
   ORDER BY created_at ASC
   FOR UPDATE
   ```
   Al bloquear todas las deudas abiertas del evento ordenadas por `created_at ASC`, evitamos _deadlocks_ (interbloqueos) y garantizamos que ningún pago manual interfiera mientras se están calculando e insertando los abonos de `COMPENSACION`.

### 8.2. Llaves de Idempotencia (`Idempotency-Key`) frente a reintentos y _Double-Submit_

En redes móviles es muy común que el usuario presione "Enviar pago" y, por un microcorte de señal, la petición se envíe dos veces. Para resolverlo de extremo a extremo:

1. En el frontend (`Pago.tsx`), al enviar el formulario se genera un identificador único con `const idempotencyKey = crypto.randomUUID()` y se adjunta en la cabecera HTTP `idempotency-key`.
2. En la base de datos (`001_init.sql`), la tabla `payments` tiene el índice único compuesto:
   ```sql
   CREATE UNIQUE INDEX payments_idempotent ON payments (debt_id, idempotency_key);
   ```
3. Si llegan dos peticiones con la misma `(debt_id, idempotency_key)`, el backend detecta que ya existe ese registro y devuelve el pago previamente creado marcando `existing: true`, sin duplicar el cobro ni alterar `reserved_cents`.

El mismo principio de idempotencia se aplica al subir imágenes de comprobantes mediante el índice único `CREATE UNIQUE INDEX receipts_event_image_hash ON receipts (event_id, image_hash) WHERE image_hash IS NOT NULL` (`001_init.sql` y `comprobantes.service.ts`): si el usuario sube dos veces la misma foto en el mismo evento, el sistema detecta el mismo hash SHA-256 y devuelve el comprobante existente (`existing: true`) en lugar de duplicar la boleta.

### 8.3. Concurrencia sin contención en la cola de OCR (`FOR UPDATE SKIP LOCKED`) y Migraciones

- **Reclamo concurrente de tareas OCR:** Además del índice parcial `CREATE UNIQUE INDEX receipt_jobs_one_active ON receipt_jobs (receipt_id) WHERE status IN ('PENDING','RUNNING')` que impide encolar dos tareas activas para un mismo comprobante, los workers usan `FOR UPDATE SKIP LOCKED` para tomar trabajos en paralelo sin bloquearse entre sí.
- **Sincronización en el arranque (`pg_advisory_lock` en `src/db/migrate.ts`):** Cuando se levanta el entorno con Docker Compose, los contenedores de la `api` y del `worker` arrancan al mismo tiempo e intentan correr las migraciones SQL. Para evitar una condición de carrera creando tablas a la vez, `runMigrations()` ejecuta `await client.query('SELECT pg_advisory_lock(772001)')` antes de revisar `schema_migrations`, garantizando que un proceso espere al otro de forma ordenada.

---

## 9. Tolerancia a fallas

Dado que el reconocimiento óptico de imágenes puede fallar por fotos borrosas, demoras de red o saturación de recursos, diseñamos tres mecanismos de resiliencia que trabajan juntos para que el usuario nunca se quede bloqueado:

### 9.1. Mecanismo 1: Circuit Breaker con Fallback automático al OCR local (`src/modules/ocr/breaker.ts`)

Utilizamos la librería `opossum` en la función `crearOcrBreaker(externo, local, opts)` para envolver las llamadas al proveedor de OCR externo (`HttpOcrClient`):

- **Configuración del circuito:** Define un tiempo máximo de espera por lectura (`timeoutMs`), un porcentaje de fallos para abrir el circuito (`errorThresholdPercentage`), un mínimo de peticiones antes de evaluar (`volumeThreshold: 2`) y el tiempo de enfriamiento antes de volver a probar (`resetTimeout`).
- **Degradación automática (_Fallback_):** Si está activo `fallbackLocal`, cada vez que el servicio externo falla, tarda más del timeout o el circuito se encuentra abierto (`OPEN`), `opossum` ejecuta automáticamente `breaker.fallback((imagePath) => local.extract(imagePath))`, redirigiendo la lectura al motor local de Tesseract sin que la tarea falle.
- **Autorecuperación (`HALF-OPEN` $\rightarrow$ `CLOSED`):** Pasado el tiempo de `resetMs`, el circuito pasa a estado semiabierto (`halfOpen`), deja pasar una solicitud de prueba al servicio externo y, si responde bien, cierra el circuito (`close`) volviendo a la normalidad y registrando cada transición en los logs estructurados.

### 9.2. Mecanismo 2: Cola persistente con Arriendos (_Leases_) y Reintentos en `receipt_jobs`

Si tanto el OCR externo como el local fallan ante una imagen complicada, o si el proceso Worker se reinicia inesperadamente en medio de una lectura, la tabla `receipt_jobs` protege el trabajo mediante:

1. **Arriendo temporal (_Lease_ con expiración):** Cuando el Worker reclama una tarea, la pasa a estado `RUNNING`, anota su identificador en `lease_owner` y fija una fecha de vencimiento en `lease_expires_at`. Si el Worker sufre un _crash_ o se queda sin memoria antes de terminar, el arriendo vence (`lease_expires_at < now()`) y el trabajo vuelve a ser elegible automáticamente en el siguiente ciclo sin perderse.
2. **Conteo de intentos y retroceso (`attempts` y `next_attempt_at`):** Si ocurre un error durante la extracción, el Worker incrementa `attempts`, guarda la causa en `last_error` y programa el siguiente intento en `next_attempt_at`. Si se alcanza el número máximo de intentos sin éxito, la tarea se marca como `FAILED` y el comprobante pasa a estado `FALLIDO` informando el `fail_reason`.

### 9.3. Mecanismo 3: Recuperación asistida y degradación elegante a ingreso manual

En un caso real, si la boleta física tiene la tinta térmica borrada o está muy arrugada, ningún OCR del mundo podrá adivinar los números con certeza. Para que eso nunca detenga las cuentas del grupo, implementamos un flujo completo de recuperación en el frontend (`DiagnosticoLecturaFallida.tsx` y `SubirComprobante.tsx`) y en el backend (`comprobantes.service.ts`):

1. **Completar a mano reutilizando el comprobante fallido (`POST /api/events/:id/receipts/manual`):** El pagador puede pulsar _"Completar a mano"_, lo que lleva a `/eventos/:id/comprobantes/nuevo?fallido=:receiptId`. Al guardar los montos digitados, `createManual` actualiza ese mismo comprobante que estaba en estado `FALLIDO` pasándolo a `EXTRAÍDO` (`replaceItemsAndData(body.receiptId, data, 'FALLIDO')`), conservando la foto adjunta como respaldo visual para el grupo.
2. **Reintentar lectura OCR (`POST /api/receipts/:id/retry`):** Permite al pagador volver a encolar un nuevo trabajo `PENDING` en `receipt_jobs` si el fallo fue temporal.
3. **Subir otra toma o descartar (`DELETE /api/receipts/:id`):** Permite eliminar el comprobante fallido (limpiando también su archivo en disco) para tomar una foto con mejor luz.

---

## 10. Pruebas y resultados

Para demostrar que los contratos, el control de concurrencia y los mecanismos de tolerancia a fallas funcionan de verdad y no solo en teoría, construimos una suite de pruebas automatizadas en el backend (con **Jest**, **Testcontainers** y **fast-check**) y en el frontend (con **Vitest** y **React Testing Library**).

### 10.1. Estrategia de pruebas implementada

| Nivel de prueba                                                  | Herramientas                           | Qué verifica en SplitVision                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| :--------------------------------------------------------------- | :------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Pruebas basadas en propiedades (_Property-Based Testing_)** | `Jest` + `fast-check`                  | Ejecuta **1,000 combinaciones generadas al azar** de montos totales (hasta 100 millones de centavos), listas de participantes y cantidades de ítems sobre `split-calculator.ts`. Comprueba que **siempre** se cumpla la invariante $\sum \text{partes} \equiv \text{totalCents}$, que ninguna parte sea negativa y que el reparto sea determinista.                                                                                                                                                                        |
| **2. Pruebas de Concurrencia real en Base de Datos**             | `Jest` + `Testcontainers` (PostgreSQL) | Levanta un contenedor real de PostgreSQL, aplica las migraciones SQL (`001_init.sql` a `006_billetera_y_cci.sql`) y dispara múltiples peticiones simultáneas con `Promise.all` sobre una misma deuda. Demuestra que `SELECT ... FOR UPDATE` serializa los pagos, rechaza con `409 CONFLICT` los que exceden el saldo disponible, respeta la `Idempotency-Key` y mantiene intacta la invariante `confirmed_cents + reserved_cents <= amount_cents`.                                                                         |
| **3. Pruebas de Resiliencia e Inyección de Fallas OCR**          | `Jest` + `FakeOcrClient` + `opossum`   | Usando la variable `FAKE_OCR_MODE` en `FakeOcrClient` (`ocr.client.ts`), inyectamos sin necesidad de internet tres tipos de fallas: `fail` (error 503 del servicio externo), `timeout`/`slow` (demoras de 60s para disparar el timeout del Circuit Breaker) y `corrupt` (JSON malformado con totales negativos e ítems inválidos: `{ total: -5, items: 'no-es-una-lista' }`). Verifica que el Circuit Breaker active el fallback local o marque el comprobante como `FALLIDO` tras agotar reintentos sin tumbar el Worker. |
| **4. Pruebas de Seguridad y Programación Defensiva**             | `Jest` + Supertest                     | Verifica el rechazo de archivos que no tienen _magic bytes_ de imagen válida o superan 5 MB (`400 VALIDATION`), el bloqueo de acceso a eventos ajenos (`404 NOT FOUND`), el rechazo de mutaciones sin cabecera `x-csrf-token` (`401 UNAUTHORIZED`) y el bloqueo tras 5 intentos fallidos de login (`409 CONFLICT`).                                                                                                                                                                                                        |
| **5. Pruebas de Componentes y Flujos UI**                        | `Vitest` + `@testing-library/react`    | Verifica en el frontend la renderización de estados de comprobantes, el simulador interactivo de liquidación cruzada (`ReciboInteractivo.tsx`), el cálculo en vivo de cuotas en `Reparto.tsx` y el manejo de errores de sesión.                                                                                                                                                                                                                                                                                            |

### 10.2. Reproducción del escenario concurrente

El escenario crítico de concurrencia se reproduce de manera automatizada lanzando en paralelo varias promesas de pago contra una deuda de S/ 100.00 (`10000` centavos):

- **Caso A (Sobregiro concurrente):** Se lanzan al mismo tiempo dos pagos de S/ 60.00 (`6000` centavos) y S/ 70.00 (`7000` centavos) con distintas llaves de idempotencia. El test comprueba que exactamente uno responde éxito (`PENDIENTE`, dejando `reserved_cents = 6000`) y el segundo recibe `409 CONFLICT`, quedando la deuda sin sobregiro.
- **Caso B (Doble envío idempotente):** Se lanzan en simultáneo dos peticiones con la misma `Idempotency-Key` por S/ 60.00. El test verifica que solo se inserta una fila en `payments` y que `reserved_cents` sube una única vez a `6000`.
- **Caso C (Choque Deudor vs Acreedor):** Se lanza al mismo tiempo `cancel` (por el deudor) y `confirm` (por el acreedor) sobre un mismo pago `PENDIENTE`. El test verifica que solo una transición gana el bloqueo y la otra recibe `409 CONFLICT`, manteniendo `confirmed_cents + reserved_cents` consistente.

---

## 11. Limitaciones y trabajo futuro

### 11.1. Limitaciones actuales de la versión v1.0

1. **Actualización de estados mediante _Polling_ HTTP:** En esta versión v1.0, el frontend consulta periódicamente el estado del comprobante mientras está `PROCESANDO` (cada 1 segundo con TanStack Query) y refresca las bandejas/balances cada 5 segundos. Aunque es muy confiable y simple de mantener, genera peticiones repetidas cuando hay muchos clientes abiertos.
2. **Almacenamiento de imágenes en disco local (`./data/images`):** `StorageService` guarda las fotos en una carpeta local compartida por volumen entre la API y el Worker. Esto funciona muy bien en un despliegue de un solo nodo con Docker Compose, pero limita escalar el Worker en máquinas físicamente separadas sin un almacenamiento de objetos compartido.
3. **Observabilidad centrada en logs y auditoría en BD:** Contamos con logs JSON estructurados con `correlationId` y la tabla `audit_logs`, pero aún no se exponen métricas agregadas en tiempo real (como histogramas de latencia de OCR o tasa de apertura del Circuit Breaker) en un endpoint de monitoreo.
4. **Moneda única (PEN):** El formateador e ingreso de montos está enfocado exclusivamente en soles peruanos (`es-PE`, `S/`).

### 11.2. Evolución prevista hacia el Examen Final (v2.0)

Siguiendo la hoja de ruta del curso SW707 para las Unidades 3 y 4, el equipo evolucionará esta base v1.0 en los siguientes frentes:

- **Configuración externalizada y _Feature Flags_:** Incorporar un módulo de configuración dinámica y banderas de funcionalidades (_feature flags_) que permitan habilitar o deshabilitar en caliente el proveedor OCR externo, la auto-compensación de deudas o nuevos modos de reparto sin redesplegar el código.
- **Observabilidad completa (Métricas y Trazas):** Añadir recolección de métricas de rendimiento (tiempos de respuesta por endpoint, duración de lectura de Tesseract, uso de memoria del Worker y estado del Circuit Breaker) junto con trazas de extremo a extremo entre la API y el Worker.
- **Profiling y optimización de rendimiento:** Medir con herramientas de _profiling_ el consumo de CPU/RAM del Worker OCR y el tiempo de respuesta de las consultas de balance y compensación bajo volúmenes altos de comprobantes, aplicando optimizaciones medibles con un antes y después (_baseline vs optimizado_).
- **Mantenimiento, refactorización e ingeniería reversa:** Aplicar refactorizaciones respaldadas por nuestra suite de pruebas (por ejemplo, extraer máquinas de estados declarativas basadas en tablas para las transiciones de `ReceiptStatus` y `PaymentStatus`) y documentar los cambios según su categoría de mantenimiento (correctivo, adaptativo, perfectivo y preventivo). revisa
