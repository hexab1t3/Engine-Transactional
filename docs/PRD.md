# Documento de Requerimientos de Producto (PRD)

| Metadata | Detalle |
| :--- | :--- |
| **Proyecto:** | Core Transactional & Settlement Engine (Engine Transaccional de Alta Concurrencia)[cite: 1] |
| **Estado:** | `DRAFT` / En Revisión de Arquitectura |
| **Autor:** | Cloud-Native Product Engineer |
| **Versión:** | `1.0.0` |
| **Fecha:** | Septiembre 2026 |
| **Audiencia:** | Equipo de Ingeniería, Tech Leads, Arquitectos de Software |

---

## 1. Visión General y Problema de Negocio

### 1.1 Contexto
En sistemas financieros y e-commerce de alto tráfico, miles de peticiones HTTP llegan de forma simultánea solicitando debitar saldos, comprar boletos o reservar inventario limitado[cite: 1]. Sin los mecanismos de aislamiento y control de concurrencia adecuados, ocurren errores graves como:
* **Doble gasto (Double Spending):** Procesamiento duplicado de un cobro por retención/reintentos del cliente.
* **Condiciones de Carrera (Race Conditions):** Saldos negativos o sobreventas de inventario por lecturas/escrituras simultáneas en la base de datos[cite: 1].
* **Inconsistencia de Estado:** Fallos en la red que dejan transacciones a mitad de ejecución.

### 1.2 Objetivo del Producto
Diseñar y construir un **Engine Transaccional asíncrono, resiliente y de alta concurrencia** capaz de garantizar la consistencia estricta de saldos (cumplimiento de principios ACID)[cite: 1], prevenir peticiones duplicadas mediante llaves de idempotencia y desacoplar el procesamiento intensivo a través de colas de mensajes[cite: 1].

---

## 2. Objetivos de Negocio y Métricas (OKRs & SLAs)

* **Disponibilidad del Servicio (SLA):** 99.95% de uptime en la recepción de solicitudes.
* **Garantía de Idempotencia:** 0% de transacciones duplicadas registradas en la base de datos ante reintentos de peticiones.
* **Consistencia de Datos:** 0% de saldos negativos o inconsistentes bajo pruebas de estrés de alta concurrencia[cite: 1].
* **Latencia Ingestion (API):** $p99 < 50ms$ para la aceptación del evento y encolado en RabbitMQ[cite: 1].
* **Rendimiento (Throughput):** Soporte mínimo de **1,000 transacciones por segundo (TPS)** concurrentes por nodo sin degradación de base de datos.

---

## 3. Requerimientos Funcionales (RF)

| ID | Prioridad | Descripción del Requerimiento | Criterio de Aceptación |
| :--- | :--- | :--- | :--- |
| **RF-001** | **P0 (Crítico)** | **Ingesta de Transacciones:** La API debe exponer un endpoint `POST /v1/transactions` para recibir solicitudes de transferencias entre cuentas. | Recibe JSON con `source_account_id`, `destination_account_id`, `amount`, `currency` y devuelve `202 Accepted` con `transaction_id`. |
| **RF-002** | **P0 (Crítico)** | **Garantía de Idempotencia:** El sistema debe exigir el encabezado HTTP `X-Idempotency-Key` en cada petición transaccional. | Si llega una petición con una llave ya procesada en las últimas 24 hrs, debe retornar la respuesta previa almacenada en Redis sin volver a procesar la transacción. |
| **RF-003** | **P0 (Crítico)** | **Procesamiento Asíncrono:** La API debe validar el esquema básico, registrar la intención y publicar el evento en RabbitMQ para su procesamiento asíncrono en segundo plano[cite: 1]. | El cliente HTTP no espera la escritura en la base de datos PostgreSQL; la respuesta se entrega en $<50ms$. |
| **RF-004** | **P0 (Crítico)** | **Aislamiento Transaccional:** El servicio *Worker* debe consumir eventos de RabbitMQ y ejecutar la transferencia dentro de una transacción en PostgreSQL utilizando **Pessimistic Locking** (`SELECT ... FOR UPDATE`)[cite: 1]. | Ninguna otra transacción puede leer ni modificar los saldos de las cuentas involucradas hasta que finalice el `COMMIT` o `ROLLBACK`. |
| **RF-005** | **P0 (Crítico)** | **Validación de Reglas de Negocio:** Ninguna cuenta origen puede quedar con un saldo inferior a cero ($0.00$). | Si `balance < amount`, el worker aborta la transacción, registra el estado `FAILED` con motivo `INSUFFICIENT_FUNDS` y notifica evento de fallo. |
| **RF-006** | **P1 (Alto)** | **Consulta de Estado:** Exponer endpoint `GET /v1/transactions/{transaction_id}` para consultar el estado actual de la transacción (`PENDING`, `COMPLETED`, `FAILED`). | Retorna JSON con el desglose de la transacción y timestamp en ISO-8601. |
| **RF-007** | **P1 (Alto)** | **Manejo de Reintentos y DLQ:** Peticiones que fallen por errores transitorios de BD deben reintentarse hasta 3 veces con *Exponential Backoff*. | Si tras 3 reintentos persiste el fallo, el mensaje debe enviarse a una **Dead Letter Queue (DLQ)** de RabbitMQ para inspección manual sin bloquear la cola principal. |

---

## 4. Requerimientos No Funcionales (RNF)

| ID | Categoría | Especificación Técnica |
| :--- | :--- | :--- |
| **RNF-001** | **Arquitectura** | **Clean Architecture / Hexagonal:** Separación estricta entre capa de Dominio, Casos de Uso, Controladores e Infraestructura (Postgres, Redis, RabbitMQ)[cite: 1]. |
| **RNF-002** | **Escalabilidad** | **Paridad Local/Nube (12-Factor App):** La aplicación debe ser 100% contenerizada con Docker y configurable mediante variables de entorno (`.env`)[cite: 1]. |
| **RNF-003** | **Seguridad** | **Sanitización y Principio de Menor Privilegio:** Ninguna credencial o secreto debe estar en duro en el código. Uso de TLS para comunicaciones inter-servicio en producción. |
| **RNF-004** | **Resiliencia** | **Circuit Breaker & Graceful Shutdown:** En caso de caída de la base de datos, los componentes deben apagar sus conexiones limpiamente sin congelar procesos colgados. |
| **RNF-005** | **Observabilidad** | **Structured Logging & Metrics:** Logs emitidos en formato JSON enriquecido (`trace_id`, `transaction_id`, `latency_ms`, `level`) compatibles con Datadog/CloudWatch. |
| **RNF-006** | **Pruebas y Calidad** | Cobertura de pruebas unitarias $>80\%$ e integración automatizada mediante GitHub Actions (CI/CD)[cite: 1]. |

---

## 5. Fuera del Alcance (Out of Scope)

Para mantener el enfoque en el motor de concurrencia y arquitectura de backend, quedan explícitamente excluidos del alcance inicial:
* Interfaz gráfica de usuario (Frontend web o app móvil).
* Integración directa con pasarelas de pago reales (Visa/Mastercard/SPEI).
* Módulo de facturación fiscal o conversión de divisas en tiempo real.

---

## 6. Riesgos del Proyecto y Estrategias de Mitigación

| Riesgo Técnico | Impacto | Mitigación Planteada |
| :--- | :--- | :--- |
| **Deadlocks en Base de Datos** por bloqueos cruzados en cuentas. | Alto | Ordenar siempre el bloqueo de cuentas por ID (ejemplo: siempre bloquear primero la cuenta con menor ID numérico). |
| **Saturación de Conexiones a PostgreSQL** bajo alta concurrencia. | Crítico | Configuración de Pool de Conexiones (*Connection Pooling*) estricto en la API/Worker y caché de lecturas en Redis[cite: 1]. |
| **Pérdida de Mensajes en RabbitMQ** ante reinicios del servidor. | Alto | Configurar colas y mensajes en modo **Durable / Persistent**. |