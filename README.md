# SplitVision — Documentación

## Equipo

| Integrante | Código |
|------------|--------|
| Gonzalo Albornoz | 20232551K |
| Luis Angel Vargas Ponce | 20231314E |
| Farid Zamudio | 20231215G |

Repositorio de documentación del proyecto **SplitVision**, desarrollado para el curso **Construcción de Software II (SW707)**.

> Este repositorio contiene únicamente la documentación. El código fuente del prototipo vive en el repositorio **app**.

## ¿Qué es SplitVision?

SplitVision es un sistema para dividir gastos compartidos entre grupos (compañeros de universidad, *roommates*, grupos de viaje). El usuario sube la foto de un comprobante, un servicio externo de **OCR** extrae el monto, el usuario lo confirma y el sistema genera automáticamente las deudas entre los participantes, que luego se pueden saldar (total o parcialmente).

El proyecto no busca ser un CRUD convencional. Se enfoca en dos retos de construcción de software:

| Reto | Descripción | Mecanismos previstos |
|------|-------------|----------------------|
| **Concurrencia** | Varios usuarios pagando o actualizando la misma matriz de deudas al mismo tiempo (condiciones de carrera, *lost update*). | Transacciones, bloqueos (locks) a nivel de fila, aislamiento transaccional. |
| **Fallos de runtime / red** | El servicio OCR externo puede tener latencia, *timeouts* o caer. | Procesamiento en segundo plano (cola + worker), timeouts, manejo de excepciones, reintentos y recuperación. |

## Repositorios del proyecto

| Repositorio | Contenido |
|-------------|-----------|
| **app**  | Código fuente del prototipo funcional |
| **docs** | Documento técnico, diagramas y material de soporte. |

- Repositorio app: `https://github.com/SplitVisionFIIS/App`
- Repositorio docs: `https://github.com/SplitVisionFIIS/Docs`

---
