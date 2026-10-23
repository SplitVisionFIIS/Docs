# SplitVision — Documentación Técnica y Arquitectura

**Curso:** Construcción de Software II (SW707) — FIIS UNI  
**Fecha:** Octubre de 2026  
**Versión:** 1.0 (Entrega Parcial)

---

## Equipo

| Integrante                  | Código UNI |
| :-------------------------- | :--------- |
| **Gonzalo Albornoz**        | 20232551K  |
| **Luis Angel Vargas Ponce** | 20231314E  |
| **Farid Zamudio**           | 20231215G  |

---

## ¿Qué es SplitVision?

**SplitVision** es un sistema colaborativo para la división equitativa e individual de gastos compartidos entre grupos (compañeros de universidad, _roommates_ o viajes). Permite digitalizar boletas o comprobantes mediante **Reconocimiento Óptico de Caracteres (OCR)**, verificar las lecturas de forma interactiva y liquidar las deudas generadas mediante métodos de pago digitales (Yape, Plin, transferencias bancarias con CCI y efectivo).

El proyecto no es un CRUD convencional; aborda **tres retos centrales de construcción de software**:

1. **Concurrencia en la liquidación de deudas:** Prevención de condiciones de carrera y _Lost Updates_ ante pagos concurrentes, cancelaciones simultáneas o doble clic (_double-submit_) en redes móviles mediante bloqueos pesimistas (`SELECT ... FOR UPDATE`), transacciones ACID y llaves de idempotencia (`Idempotency-Key`).
2. **Cómputo intensivo y datos no confiables de OCR:** Desacoplamiento de la lectura de imágenes de la API principal mediante un **proceso Worker independiente** y cola persistente en PostgreSQL (`receipt_jobs` con `FOR UPDATE SKIP LOCKED`), integrando un _Circuit Breaker_ (`opossum`) con tolerancia a fallas y tratamiento defensivo de la lectura de la máquina (revisión humana obligatoria en estado `EXTRAÍDO`).
3. **Precisión aritmética y conservación de masa:** Eliminación absoluta de errores de redondeo por punto flotante binario (IEEE 754) utilizando **aritmética entera en centavos** (`amount_cents`), invariantes formales por contrato (_Design by Contract_) y distribución determinista de residuos monetarios verificada con _Property-Based Testing_ (`fast-check`, 1,000 corridas).

---

## Documento Principal

El documento técnico integral del proyecto se encuentra en:

📄 **[Documentación Técnica Parcial (v1.0) — documentacionTecnica.md](documentacionTecnica.md)**

### Índice de contenidos:

1. **Descripción del problema y usuarios:** Contexto, retos de ingeniería de software, actores y objetivos del prototipo.
2. **Alcance y requisitos principales:** Modelo de dominio, invariantes, estados de ciclo de vida (`CARGADO`, `PROCESANDO`, `EXTRAÍDO`, `CONFIRMADO`, `FALLIDO`), flujos de negocio y matriz de requisitos (RF/RNF).
3. **Arquitectura y decisiones técnicas:** Arquitectura de monolito modular, modelo C4 (Contexto, Contenedores y Componentes de API y Worker), stack tecnológico y justificación técnica (ADRs).
4. **Dependencias y reúso:** Auditoría de dependencias (`npm audit` con 0 vulnerabilidades en 571 paquetes backend y 695 frontend), control con `package-lock.json` y `overrides`.
5. **Problemas de runtime:** Fugas de memoria en WebAssembly/OCR, prevención de bloqueo del _Event Loop_ y mitigación de agotamiento del pool de conexiones.
6. **Contratos y programación defensiva:** Precondiciones, postcondiciones e invariantes (_Design by Contract_), inspección de _Magic Bytes_ (JPEG/PNG/WebP), prevención de _Path Traversal_ y control de acceso (anti-IDOR).
7. **Estrategia de errores y excepciones:** Taxonomía de errores tipados de dominio, filtro global NestJS, `Correlation ID` de extremo a extremo y traducción en cliente frontend.
8. **Concurrencia:** Bloqueo a nivel de fila en PostgreSQL, aislamiento transaccional y llaves de idempotencia contra reintentos de red.
9. **Tolerancia a fallas:** _Circuit Breaker_ sobre el motor OCR con fallback automático local, leases con recuperación de tareas caídas y degradación elegante a ingreso manual.
10. **Pruebas y resultados:** Suites E2E con PostgreSQL efímero en Docker (`Testcontainers`), pruebas de propiedades generativas (`fast-check`) y reproducción automatizada de carreras de concurrencia.
11. **Limitaciones y trabajo futuro:** Análisis de deuda técnica controlada en v1.0 y hoja de ruta hacia la entrega final (v2.0).

---

## Arquitectura del Sistema (Modelo C4)

Los diagramas visuales de la arquitectura están versionados en [`assets/c4/`](assets/c4/):

| Nivel C4    | Diagrama                            | Archivo                                                                |
| :---------- | :---------------------------------- | :--------------------------------------------------------------------- |
| **Nivel 1** | Contexto del Sistema                | [`assets/c4/Contexto.png`](assets/c4/Contexto.png)                     |
| **Nivel 2** | Contenedores (SPA, API, Worker, DB) | [`assets/c4/Contenedores.png`](assets/c4/Contenedores.png)             |
| **Nivel 3** | Componentes del Proceso API         | [`assets/c4/Componentes_API.png`](assets/c4/Componentes_API.png)       |
| **Nivel 3** | Componentes del Proceso Worker      | [`assets/c4/Componentes_Worker.png`](assets/c4/Componentes_Worker.png) |

---

## Repositorios del Proyecto

| Repositorio | Descripción                                                                         | Enlace                                                          |
| :---------- | :---------------------------------------------------------------------------------- | :-------------------------------------------------------------- |
| **App**     | Código fuente del prototipo funcional (Backend NestJS + Worker + Frontend React 19) | [SplitVisionFIIS/App](https://github.com/SplitVisionFIIS/App)   |
| **Docs**    | Documentación técnica, diagramas C4 y especificaciones del curso SW707              | [SplitVisionFIIS/Docs](https://github.com/SplitVisionFIIS/Docs) |
