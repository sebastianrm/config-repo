# config-repo — Repositorio de Configuración Centralizada

> **Repositorio Git de configuración centralizada** para la plataforma bancaria Yanki.
>
> Este repositorio contiene los archivos YAML que `config-server` (Spring Cloud Config Server) distribuye a todos los microservicios. Al desplegar el contenedor de `config-server`, se realiza un **clone automático** de este repositorio Git dentro del contenedor.

---

## 1. Propósito

| Responsabilidad | Detalle |
|---|---|
| **Fuente de verdad única** | Un solo repositorio Git contiene toda la configuración de la plataforma |
| **Distribución automática** | `config-server` clona el repositorio al arrancar y sirve la configuración a los microservicios |
| **Versionado** | Cada cambio de configuración queda registrado en el historial Git |
| **Secretos seguros** | Los secretos sensibles se inyectan por variables de entorno, nunca se hardcodean |
| **Merge jerárquico** | `application.yml` (común) se mergea con `<servicio>.yml` (específico) |

---

## 2. Cómo funciona

```
┌─────────────────────────────────────────────────────────┐
│  config-repo (repositorio Git remoto en GitHub)         │
│  → https://github.com/sebastianrm/config-repo.git       │
│  → Rama: main                                           │
│  → application.yml: configuración común compartida      │
│  → <servicio>.yml: configuración específica por servicio │
└────────────────────────┬────────────────────────────────┘
                         │ git clone (automático)
                         ▼
┌─────────────────────────────────────────────────────────┐
│  config-server (contenedor, puerto 8888)                │
│  → Clona automáticamente el repositorio Git al arrancar │
│  → SPRING_PROFILES_ACTIVE: git                          │
│  → CONFIG_GIT_URI: https://github.com/sebastianrm/config-repo.git │
│  → CONFIG_GIT_DEFAULT_LABEL: main                       │
│  → Sirve configuración a los microservicios             │
└────────────────────────┬────────────────────────────────┘
                         │ HTTP GET
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Cada microservicio                                     │
│  → Importa: optional:configserver:http://config-server:8888 │
│  → Recibe: application.yml + <servicio>.yml mergeados   │
│  → Sobrescribe secretos por variables de entorno        │
└─────────────────────────────────────────────────────────┘
```

### Flujo de despliegue

1. **`docker-compose.full.yml`** arranca `config-server` con `SPRING_PROFILES_ACTIVE: git`.
2. **`config-server`** clona automáticamente `https://github.com/sebastianrm/config-repo.git` (rama `main`) dentro del contenedor.
3. **Cada microservicio** importa configuración vía `optional:configserver:http://config-server:8888`.
4. **Spring Cloud Config** mergea `application.yml` (común) con `<servicio>.yml` (específico).
5. **Variables de entorno** sobrescriben los secretos sensibles (`JWT_SECRET`, `PASSWORD_SECRET`, etc.).

### Recarga de configuración

```bash
# Forzar recarga en config-server (después de un push al repositorio Git)
curl -X POST http://localhost:8888/actuator/refresh

# Forzar recarga en un microservicio específico
curl -X POST http://localhost:8082/actuator/refresh
```

---

## 3. Estructura de Archivos

```
config-repo/
├── README.md                    # ← ESTE ARCHIVO
├── application.yml              # Configuración común (Redis, Kafka, Eureka, JWT)
├── auth-service.yml             # Auth: MongoDB URI, puertos, JWT específico
├── api-gateway.yml              # Gateway: rutas, JWT, CORS, Resilience4j
├── accounts-service.yml         # Accounts: MongoDB URI, puertos
├── customers-service.yml        # Customers: MongoDB URI, puertos
├── credits-service.yml          # Credits: MongoDB URI, puertos
├── cards-service.yml            # Cards: MongoDB URI, puertos
├── transfers-service.yml        # Transfers: MongoDB URI, puertos
├── wallet-service.yml           # Wallet: MongoDB URI, puertos
└── reports-service.yml          # Reports: MongoDB URI, puertos
```

---

## 4. Descripción de Archivos

### 4.1. `application.yml` — Configuración Común

**Descripción:** Archivo compartido por **todos** los microservicios. Contiene la configuración de infraestructura común: Redis, Kafka, Eureka, Actuator, logging y JWT unificado.

| Sección | Propósito |
|---|---|
| `spring.data.redis` | Conexión a Redis (host, puerto, timeout) |
| `spring.kafka` | Productor y consumidor Kafka (serializadores, deserializadores, trusted packages) |
| `eureka.client` | Registro y descubrimiento de servicios en Eureka |
| `management.endpoints` | Exposición de health, info, metrics |
| `logging` | Nivel y patrón de logs común |
| `yanki.security.jwt` | JWT unificado (secret, issuer, expiration) |
| `yanki.auth.password` | Clave de encriptación de contraseñas |

**Variables de entorno sobrescritas:**
- `REDIS_HOST`, `REDIS_PORT`
- `KAFKA_BOOTSTRAP_SERVERS`
- `EUREKA_URL`
- `JWT_SECRET`, `JWT_ISSUER`, `JWT_EXPIRATION`
- `PASSWORD_SECRET`

### 4.2. `<servicio>.yml` — Configuración Específica por Servicio

**Descripción:** Cada archivo contiene la configuración específica de un microservicio: puerto, nombre de aplicación, URI de MongoDB, grupo de consumo Kafka y configuración de Springdoc/OpenAPI.

| Archivo | Servicio | Puerto | BD MongoDB | Grupo Kafka |
|---|---|---|---|---|
| `auth-service.yml` | auth-service | 8081 | `yanki_auth` | `auth-service` |
| `api-gateway.yml` | api-gateway | 8080 | — | — |
| `accounts-service.yml` | accounts-service | 8083 | `yanki_accounts` | `accounts-service` |
| `customers-service.yml` | customers-service | 8082 | `yanki_customers` | `customers-service` |
| `credits-service.yml` | credits-service | 8084 | `yanki_credits` | `credits-service` |
| `cards-service.yml` | cards-service | 8085 | `yanki_cards` | `cards-service` |
| `transfers-service.yml` | transfers-service | 8086 | `yanki_transfers` | `transfers-service` |
| `wallet-service.yml` | wallet-service | 8087 | `yanki_wallet` | `wallet-service` |
| `reports-service.yml` | reports-service | 8088 | `yanki_reports` | `reports-service` |

**Estructura típica de un archivo por servicio:**
```yaml
server:
  port: ${SERVER_PORT:808X}
  shutdown: graceful

spring:
  application:
    name: <servicio>-service
  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://mongo:27017/yanki_<servicio>}
      database: yanki_<servicio>
      auto-index-creation: true
  kafka:
    consumer:
      group-id: ${KAFKA_GROUP_ID:<servicio>-service}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
    tags-sorter: alpha
```

### 4.3. `api-gateway.yml` — Configuración Especial del Gateway

**Descripción:** Archivo más extenso que contiene la configuración de enrutamiento, JWT, CORS y Resilience4j del API Gateway.

| Sección | Propósito |
|---|---|
| `spring.cloud.gateway.routes` | 9 rutas de enrutamiento a los microservicios backend |
| `yanki.security.jwt` | JWT para validación de tokens en el gateway |
| `yanki.security.public-paths` | Rutas públicas sin autenticación |
| `yanki.gateway.cors` | Configuración CORS (origins, methods, headers) |
| `resilience4j` | Circuit Breaker y TimeLimiter por defecto |
| `management.endpoints` | Exposición de health, info, gateway, metrics |

---

## 5. Reglas de Configuración

### 5.1. Reglas Inquebrantables

1. **NUNCA hardcodear secretos** en los archivos YAML.
   - ❌ `secret: ${JWT_SECRET:mi-clave-secreta}`
   - ✅ `secret: ${JWT_SECRET:}` (sin valor por defecto)
2. **Siempre usar placeholders** con `${VAR:}` para valores sensibles.
3. **Los valores por defecto** son solo para desarrollo local (`redis`, `kafka:29092`, etc.).
4. **`application.yml`** contiene solo configuración común; no mezclar configuración específica.
5. **Cada servicio** tiene su propio archivo `<servicio>.yml` con su configuración específica.
6. **El merge** lo realiza Spring Cloud Config automáticamente: `application.yml` + `<servicio>.yml`.

### 5.2. Convenciones de Nomenclatura

| Patrón | Ejemplo | Descripción |
|---|---|---|
| `<servicio>-service.yml` | `accounts-service.yml` | Archivo de configuración por servicio |
| `application.yml` | — | Archivo común compartido |
| `${VAR:default}` | `${SERVER_PORT:8083}` | Placeholder con valor por defecto |
| `${VAR:}` | `${JWT_SECRET:}` | Placeholder sin valor por defecto (secreto) |

---

## 6. Variables de Entorno

| Variable | Descripción | Archivos afectados |
|---|---|---|
| `JWT_SECRET` | Clave secreta para firmar/verificar tokens JWT | `application.yml`, `auth-service.yml`, `api-gateway.yml` |
| `PASSWORD_SECRET` | Clave para encriptación de contraseñas | `application.yml` |
| `REDIS_HOST` | Host del servidor Redis | `application.yml` |
| `REDIS_PORT` | Puerto del servidor Redis | `application.yml` |
| `KAFKA_BOOTSTRAP_SERVERS` | Servidores Kafka | `application.yml` |
| `EUREKA_URL` | URL del servicio Eureka | `application.yml` |
| `MONGODB_URI` | URI de conexión MongoDB | `<servicio>.yml` |
| `SERVER_PORT` | Puerto del servidor HTTP | `<servicio>.yml` |
| `KAFKA_GROUP_ID` | Grupo de consumo Kafka | `<servicio>.yml` |

---

## 7. Referencias

| Archivo | Sección | Descripción |
|---|---|---|
| [`../config-server/README.md`](../config-server/README.md) | Todo el documento | Config Server: propósito, endpoints, estructura, backend Git |
| [`../docker-compose.full.yml`](../docker-compose.full.yml) | `config-server` (línea 144) | Despliegue del contenedor config-server con Git remoto |
| [`../ARQUITECTURA-INFRAESTRUCTURA.md`](../ARQUITECTURA-INFRAESTRUCTURA.md) | Sección 3.2 | Config Server en la arquitectura de infraestructura |
| [`../AGENTS.md`](../AGENTS.md) | Regla de Oro de Configuración | Reglas de gestión de configuración y secretos |
| [`../README.md`](../README.md) | Sección 3.5 | Configuración Centralizada en el mapa de microservicios |

---

## 8. Historial

| Fecha | Versión | Autor | Cambio |
|---|---|---|---|
| 2026-11-22 | 1.0 | Sistema | Creación del README.md para config-repo |
