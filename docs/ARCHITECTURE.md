# Especificación de Arquitectura de Sistemas

| Metadata | Detalle |
| :--- | :--- |
| **Proyecto:** | Core Transactional & Settlement Engine |
| **Estado:** | `APPROVED` / Diseño Técnico de Referencia |
| **Autor:** | Cloud-Native Product Engineer |
| **Versión:** | `1.0.0` |

---

## 1. Visión General de la Arquitectura

El sistema utiliza el patrón **Event-Driven Architecture (EDA)** desacoplado con **Clean Architecture** en el backend. Está diseñado para separar la recepción de peticiones de alta velocidad (I/O Bound) del procesamiento y liquidación atómica en la base de datos (CPU/Storage Bound).

```mermaid
graph TD
    Client[Cliente / App / Frontend] -->|1. POST /v1/transactions| API[API Ingestion Engine - Go/C#]
    API -->|2. Validar / Reservar Llave| Redis[(Redis Cache & Locks)]
    API -->|3. Publicar Evento Transaccional| RabbitMQ{RabbitMQ Exchange}
    RabbitMQ -->|4. Cola de Trabajo: tx.process| Worker[Worker Engine]
    RabbitMQ -->|5. Fallos Persistentes| DLQ[Dead Letter Queue: tx.dlq]
    Worker -->|6. Transacción ACID + Pessimistic Lock| Postgres[(PostgreSQL Core DB)]