# Pedidos360 - Microservicio de Pagos (`cloud-backend-pago`)

Microservicio de **pagos** de la plataforma **Pedidos360**. Inicia y confirma transacciones con **Webpay Plus (Transbank, ambiente de integración)** y aplica una **regla antifraude síncrona** antes de contactar a la pasarela. Los intentos de pago y las alertas de fraude se publican en **Apache Kafka** para alimentar la analítica.

## 🛠️ Tecnologías Utilizadas

* **Lenguaje / Framework:** Java 17, Spring Boot 4.1.1 (`spring-boot-starter-webmvc`)
* **Seguridad:** Spring Security OAuth2 Resource Server (JWT de Azure AD)
* **Pasarela de pago:** Transbank SDK Java (Webpay Plus, integración)
* **Persistencia:** PostgreSQL (Spring Data JPA, base `db_pagos`)
* **Mensajería:** RabbitMQ y Apache Kafka (`spring-kafka`)
* **Pruebas:** JUnit 5 (`AntifraudeServiceTest`)
* **Contenedorización:** Docker

## 🔄 Flujo de Pago

```
Frontend ──► BFF (/api/v1/bff/pagos/*) ──► ms-pagos ──► Webpay Plus
                                              │
                                              ├─► Kafka: payment-attempts, fraud-alerts
                                              └─► PostgreSQL: alertas de fraude
```

1. El frontend llama al **BFF** (nunca directamente a este microservicio).
2. `POST /api/pagos/iniciar` evalúa el antifraude y, si el intento es válido, crea la transacción en Webpay y devuelve `url` + `token`.
3. El usuario paga en Webpay, que lo devuelve a la página `/callback` del frontend con `token_ws`.
4. `POST /api/pagos/commit` confirma el pago con Transbank y registra el resultado.

## 🛡️ Regla Antifraude

| Parámetro | Valor |
| :--- | :--- |
| Ventana deslizante | 10 segundos |
| Usos permitidos de la misma tarjeta | 3 (el 4.º intento se bloquea) |
| Duración del bloqueo | 30 minutos |
| Respuesta al bloquear | `429 Too Many Requests` |
| Clave de evaluación | `tarjeta:<hash>` o, si no viene tarjeta, `cliente:<clientId>` |

Cada bloqueo se guarda como **alerta** en la base de datos (`AlertaFraude`) y se publica en el tópico Kafka `fraud-alerts`. La publicación en Kafka es *best effort*: si Kafka no está disponible, el pago no se ve afectado.

## 📡 Endpoints

Todos requieren `Authorization: Bearer <token>` (Azure AD).

| Método | Ruta | Descripción |
| :--- | :--- | :--- |
| `POST` | `/api/pagos/iniciar` | Evalúa antifraude e inicia la transacción en Webpay. Devuelve `{ url, token }` o `429` si se bloquea. |
| `POST` | `/api/pagos/commit?token_ws=...` | Confirma el pago con Transbank. |
| `GET` | `/api/pagos/alertas` | Últimas 50 alertas antifraude. |

**Vía BFF** (lo que usa el frontend): `/api/v1/bff/pagos/iniciar`, `/api/v1/bff/pagos/commit` y `/api/v1/bff/pagos/alertas`.

Cuerpo de `iniciar`:

```json
{
  "clientId": "usuario-123",
  "amount": 15990,
  "transactionId": "orden-456",
  "tarjeta": "opcional: identificador de tarjeta usado por el antifraude"
}
```

## ⚙️ Instalación y Ejecución

### Requisitos Previos

* Java 17 y Maven 3.9+ (o el wrapper `./mvnw`)
* PostgreSQL con la base `db_pagos`
* RabbitMQ y Kafka accesibles
* Docker (opcional)

### Variables de Entorno

| Variable | Descripción | Valor por defecto |
| :--- | :--- | :--- |
| `AZURE_TENANT_ID` | ID del inquilino de Azure AD para validar el JWT | `78b145ef-56b9-4397-b87c-27b242a9fce5` |
| `DB_URL_PAGOS` / `SPRING_DATASOURCE_URL` | URL JDBC de PostgreSQL | `jdbc:postgresql://localhost:5432/db_pagos` |
| `DB_USER` / `DB_PASS` | Credenciales de la base de datos | `admin` / `admin` |
| `RABBITMQ_HOST` | Host de RabbitMQ | `localhost` |
| `RABBITMQ_USER` / `RABBITMQ_PASSWORD` | Credenciales de RabbitMQ | `admin` / `admin` |
| `KAFKA_HOST` | Servidor Kafka (`host:puerto`) | `localhost:9092` |
| `PAGOS_WEBPAY_RETURN_URL` | URL a la que Webpay devuelve al usuario (página `/callback` del frontend) | `http://localhost/desarrollo/callback` |

> **En AWS:** `PAGOS_WEBPAY_RETURN_URL` debe ser la URL pública real del frontend y `KAFKA_HOST` la IP privada de la EC2 donde corre Kafka.

### Ejecución Local

```bash
./mvnw spring-boot:run
```

El servicio queda disponible en `http://localhost:8088` (health: `/actuator/health`).

### Pruebas Unitarias

```bash
./mvnw test
```

### Despliegue con Docker

```bash
docker build -t pedidos360/ms-pagos:v1 .

docker run -d \
  --name ms-pagos \
  -p 8088:8088 \
  -e DB_URL_PAGOS="jdbc:postgresql://<IP_BD>:5432/db_pagos" \
  -e RABBITMQ_HOST="<IP_RABBITMQ>" \
  -e KAFKA_HOST="<IP_PRIVADA_KAFKA>:9092" \
  -e PAGOS_WEBPAY_RETURN_URL="https://<URL_PUBLICA>/desarrollo/callback" \
  pedidos360/ms-pagos:v1
```

## 💳 Pruebas con Webpay (integración)

Datos públicos de prueba de Transbank:

* **Tarjeta de crédito:** `4051 8856 0044 6623`, CVV `123`, cualquier fecha futura
* **Autenticación (RUT / clave):** `11.111.111-1` / `123`

Para probar el antifraude, repite el inicio de pago con la misma tarjeta más de 3 veces en menos de 10 segundos: el 4.º intento responde `429`.

---

## 🔗 Ecosistema de Repositorios

### Frontend

* [Frontend React](https://github.com/Raynagah/cloud-frontend.git)

### Backend

* [BFF Orchestrator](https://github.com/Raynagah/cloud-backend-bff)
* [Microservicio Producto](https://github.com/Raynagah/cloud-backend-producto)
* [Microservicio Carrito](https://github.com/Raynagah/cloud-backend-carrito)
* [Microservicio Usuarios](https://github.com/NBello26/ms-usuarios-cloud.git)
* [Microservicio Base de Datos](https://github.com/NBello26/ms-bd-cloud)
* [Microservicio Ordenes](https://github.com/Raynagah/cloud-backend-ordenes)
* [Microservicio Notificaciones](https://github.com/Raynagah/cloud-backend-notificaciones)
* [Microservicio Despacho](https://github.com/Raynagah/cloud-backend-despacho)
* [Microservicio Pago (Este repositorio)](https://github.com/Raynagah/cloud-backend-pago)
* [Microservicio Analítica](https://github.com/Raynagah/cloud-backend-analitica)
