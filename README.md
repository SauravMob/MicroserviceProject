# Microservice Project

A production-grade microservices architecture built with **Spring Boot 3.3.1** and **Spring Cloud 2023.0.2**, implementing service discovery, API gateway, distributed tracing, event-driven messaging, circuit breaking, and OAuth2 security.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [Microservices](#microservices)
- [APIs & Endpoints](#apis--endpoints)
- [Inter-Service Communication](#inter-service-communication)
- [Technologies & Tools](#technologies--tools)
- [Security](#security)
- [Databases](#databases)
- [Observability](#observability)
- [Resilience Patterns](#resilience-patterns)
- [Infrastructure (Docker)](#infrastructure-docker)
- [Configuration & Ports](#configuration--ports)
- [How to Run](#how-to-run)
- [Sample Data](#sample-data)
- [Order Placement Flow](#order-placement-flow)

---

## Architecture Overview

```
                        ┌─────────────────────────────┐
                        │         Keycloak             │
                        │   (OAuth2 / JWT Issuer)      │
                        │     localhost:8181           │
                        └────────────┬────────────────┘
                                     │ JWT Validation
                                     ▼
Client ──────────────▶  ┌─────────────────────────┐
                        │       API Gateway        │
                        │      localhost:9000      │
                        └──────┬──────┬────────────┘
                               │      │
               ┌───────────────┘      └──────────────────┐
               ▼                                         ▼
  ┌─────────────────────┐                  ┌──────────────────────┐
  │   Product Service   │                  │    Order Service     │
  │   localhost:8081    │                  │   localhost:8082     │
  │   (MongoDB)         │                  │   (MySQL)            │
  └─────────────────────┘                  └───────┬──────────────┘
                                                   │
                          ┌────────────────────────┼────────────────────┐
                          │                        │                    │
                          ▼ (WebClient + LB)       ▼ (Kafka)            │
              ┌────────────────────┐   ┌─────────────────────┐         │
              │ Inventory Service  │   │ Notification Service │         │
              │  localhost:8083    │   │  localhost:8085      │         │
              │  (MySQL)           │   │  (Kafka Consumer)    │         │
              └────────────────────┘   └─────────────────────┘         │
                                                                        │
  ┌─────────────────────────────────────────────────────────────────────┘
  │          All services register with:
  ▼
  ┌─────────────────────┐    ┌──────────────────┐    ┌───────────────────┐
  │  Discovery Server   │    │  Apache Kafka    │    │      Zipkin       │
  │  (Eureka)           │    │  localhost:9092  │    │  localhost:9411   │
  │  localhost:8761     │    └──────────────────┘    └───────────────────┘
  └─────────────────────┘
```

---

## Project Structure

```
MicroserviceProject/
├── pom.xml                          # Parent POM (Spring Boot 3.3.1, Java 17)
├── docker-compose.yml               # Infrastructure containers
│
├── api-gateway/                     # API Gateway (port 9000)
│   ├── pom.xml
│   └── src/main/
│       ├── java/apigateway/
│       │   ├── ApiGatewayApplication.java
│       │   └── config/SecurityConfig.java
│       └── resources/application.yml
│
├── discovery-server/                # Eureka Server (port 8761)
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/microservices/serviceDiscovery/
│       │   ├── DiscoveryServerApplication.java
│       │   └── config/SecurityConfig.java
│       └── resources/application.yml
│
├── productService/                  # Product Service (port 8081)
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/microservice/productService/
│       │   ├── ProductServiceApplication.java
│       │   ├── controller/ProductController.java
│       │   ├── service/ProductService.java
│       │   ├── model/Product.java
│       │   ├── dto/
│       │   │   ├── ProductRequest.java
│       │   │   └── ProductResponse.java
│       │   └── repo/ProductRepository.java
│       └── resources/application.yml
│
├── orderService/                    # Order Service (port 8082)
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/microservice/orderService/
│       │   ├── OrderServiceApplication.java
│       │   ├── controller/OrderController.java
│       │   ├── service/OrderService.java
│       │   ├── model/
│       │   │   ├── Order.java
│       │   │   └── OrderLineItems.java
│       │   ├── dto/
│       │   │   ├── OrderRequest.java
│       │   │   ├── OrderLineItemsDto.java
│       │   │   └── InventoryResponse.java
│       │   ├── repository/OrderRepository.java
│       │   ├── config/
│       │   │   ├── WebClientConfig.java
│       │   │   └── ManualConfig.java
│       │   ├── event/OrderPlacedEvent.java
│       │   └── listener/OrderPlacedEventListener.java
│       └── resources/application.yml
│
├── inventoryService/                # Inventory Service (port 8083)
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/microservices/inventoryService/
│       │   ├── InventoryServiceApplication.java
│       │   ├── controller/InventoryController.java
│       │   ├── service/InventoryService.java
│       │   ├── model/Inventory.java
│       │   ├── dto/InventoryResponse.java
│       │   └── repository/InventoryRepository.java
│       └── resources/application.yml
│
└── notificationService/             # Notification Service (port 8085)
    ├── pom.xml
    └── src/main/
        ├── java/com/microservices/notificationService/
        │   ├── NotificationServiceApplication.java
        │   └── OrderPlacedEvent.java
        └── resources/application.yml
```

---

## Microservices

### 1. API Gateway (port 9000)
The single entry point for all client requests. Handles routing, load balancing, and JWT-based authentication.

- Routes requests to downstream services using Spring Cloud Gateway
- Validates JWT tokens issued by Keycloak before forwarding
- Provides public access to the Eureka web dashboard at `/eureka/web`
- Integrates with Eureka for dynamic service discovery

**Routes configured:**

| Path | Target Service |
|------|----------------|
| `/api/product/**` | `lb://product-service` |
| `/api/order/**` | `lb://order-service` |
| `/eureka/web` | `http://localhost:8761` |
| `/eureka/**` | `http://localhost:8761` |

---

### 2. Discovery Server — Eureka (port 8761)
Service registry that all microservices register with, enabling dynamic service discovery.

- Standalone Eureka server (no cluster)
- Protected by HTTP Basic Auth (`eureka` / `password`)
- Web dashboard available at `http://localhost:8761`

---

### 3. Product Service (port 8081)
Manages the product catalog stored in MongoDB.

- Creates and retrieves product information
- Registered with Eureka as `product-service`
- Uses MongoDB for flexible document storage

---

### 4. Order Service (port 8082)
Handles customer orders. Coordinates with Inventory Service and triggers notifications via Kafka.

- Validates stock availability before placing an order
- Uses Resilience4j circuit breaker, retry, and time limiter
- Publishes `OrderPlacedEvent` to Kafka on successful order
- Stores orders in MySQL
- Registered with Eureka as `order-service`

---

### 5. Inventory Service (port 8083)
Tracks product stock levels and provides availability checks.

- Accepts multiple SKU codes in a single query
- Returns stock availability per SKU
- Stores inventory in MySQL
- Registered with Eureka as `inventory-service`

---

### 6. Notification Service (port 8085)
Event-driven service that consumes Kafka messages and sends notifications when orders are placed.

- Listens on Kafka topic `notificationTopic`
- Consumes `OrderPlacedEvent` messages (group-id: `notificationId`)
- Logs order placement notifications
- Registered with Eureka as `notification-service`

---

## APIs & Endpoints

All endpoints below are accessed through the **API Gateway** at `http://localhost:9000`.
Requests to protected routes require a valid **Bearer JWT token** in the `Authorization` header.

### Product Service

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| `POST` | `/api/product` | Create a new product | Yes |
| `GET` | `/api/product` | Get all products | Yes |

**POST `/api/product` — Request Body:**
```json
{
  "name": "iPhone 13",
  "description": "Apple iPhone 13 128GB",
  "price": 999.99
}
```

**GET `/api/product` — Response:**
```json
[
  {
    "id": "64abc123...",
    "name": "iPhone 13",
    "description": "Apple iPhone 13 128GB",
    "price": 999.99
  }
]
```

---

### Order Service

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| `POST` | `/api/order` | Place a new order | Yes |

**POST `/api/order` — Request Body:**
```json
{
  "orderLineItemsDtoList": [
    {
      "skuCode": "iphone_13",
      "price": 999.99,
      "quantity": 1
    }
  ]
}
```

**Response:**
- `"Order Placed"` — Order created successfully
- Fallback message — If inventory is unavailable or circuit breaker is open

---

### Inventory Service

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| `GET` | `/api/inventory` | Check stock for SKU codes | No (internal) |

**Query Parameters:** `skuCode` (multiple values supported)

**Example:** `GET /api/inventory?skuCode=iphone_13&skuCode=iphone_13_red`

**Response:**
```json
[
  { "skuCode": "iphone_13", "isInStock": true },
  { "skuCode": "iphone_13_red", "isInStock": false }
]
```

---

### Discovery Server Dashboard

| URL | Description | Auth Required |
|-----|-------------|---------------|
| `http://localhost:9000/eureka/web` | Eureka dashboard (via gateway) | No |
| `http://localhost:8761` | Eureka dashboard (direct) | Basic Auth |

---

## Inter-Service Communication

### Synchronous (HTTP)
**Order Service → Inventory Service**

Order Service uses a `LoadBalanced` `WebClient` to call the Inventory Service through Eureka service discovery. The `@CircuitBreaker`, `@TimeLimiter`, and `@Retry` annotations wrap this call for fault tolerance.

```
Order Service ──WebClient──▶ lb://inventory-service/api/inventory?skuCode=...
```

### Asynchronous (Kafka)
**Order Service → Notification Service**

When an order is placed, Order Service publishes an `OrderPlacedEvent` to the Kafka topic `notificationTopic`. Notification Service consumes this event independently.

```
Order Service ──Kafka──▶ [notificationTopic] ──▶ Notification Service
                         OrderPlacedEvent{orderNumber}
```

### Service Discovery (Eureka)
All services register with the Discovery Server on startup. The API Gateway and Order Service use `lb://` prefixed URIs to resolve services dynamically through Eureka.

---

## Technologies & Tools

### Core Framework
| Technology | Version | Usage |
|------------|---------|-------|
| Java | 17 | Language |
| Spring Boot | 3.3.1 | Application framework |
| Spring Cloud | 2023.0.2 | Microservices toolkit |
| Maven | 3.x | Build & dependency management |
| Lombok | Latest | Boilerplate code reduction |

### Spring Cloud Components
| Component | Usage |
|-----------|-------|
| Spring Cloud Gateway | API routing and load balancing |
| Spring Cloud Netflix Eureka Server | Service registry |
| Spring Cloud Netflix Eureka Client | Service registration & discovery |
| Spring Cloud Circuit Breaker (Resilience4j) | Fault tolerance |

### Data Layer
| Technology | Service | Usage |
|------------|---------|-------|
| Spring Data MongoDB | Product Service | Document storage |
| Spring Data JPA + Hibernate | Order, Inventory | Relational ORM |
| MySQL | Order, Inventory | Relational database |
| MongoDB | Product | NoSQL document database |

### Messaging
| Technology | Usage |
|------------|-------|
| Apache Kafka | Async event streaming |
| Apache Zookeeper | Kafka coordination |
| Spring Kafka | Kafka producer/consumer integration |

### Security
| Technology | Usage |
|------------|-------|
| Keycloak | OAuth2 authorization server / JWT issuer |
| Spring Security | Security framework |
| Spring OAuth2 Resource Server | JWT validation in API Gateway |
| HTTP Basic Auth | Discovery Server protection |

### Observability
| Technology | Usage |
|------------|-------|
| Spring Boot Actuator | Health checks, metrics endpoints |
| Micrometer Tracing (Brave) | Distributed tracing instrumentation |
| Zipkin | Distributed trace visualization |
| Prometheus | Metrics scraping (available via Docker) |
| Grafana | Metrics dashboards (available via Docker) |

### Resilience
| Technology | Usage |
|------------|-------|
| Resilience4j Circuit Breaker | Stops cascading failures |
| Resilience4j Retry | Automatic retry on failure |
| Resilience4j TimeLimiter | Prevents hanging calls |
| WebClient (Non-blocking) | Reactive HTTP client |
| CompletableFuture | Async order processing |

### Testing
| Technology | Usage |
|------------|-------|
| Spring Boot Test | Unit and integration testing |
| TestContainers | Containerized integration tests |

---

## Security

### API Gateway
- Acts as an **OAuth2 Resource Server**
- Validates JWTs issued by Keycloak at `http://localhost:8181/realms/spring-boot-microservices-realm`
- CSRF disabled (stateless JWT-based auth)
- **Public paths:** `/eureka/**`
- **Protected paths:** All other routes require a valid Bearer token

### Discovery Server
- Protected with **Spring Security HTTP Basic Auth**
- Credentials: `eureka` / `password`
- CSRF disabled for `/eureka/**` endpoint

### Getting a JWT Token (Keycloak)
```bash
curl -X POST http://localhost:8181/realms/spring-boot-microservices-realm/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=<client-id>" \
  -d "username=<username>" \
  -d "password=<password>" \
  -d "grant_type=password"
```

---

## Databases

### MongoDB — Product Service
| Config | Value |
|--------|-------|
| URI | `mongodb://localhost:27017/product-service` |
| Database | `product-service` |
| Collection | `product` |

### MySQL — Order Service
| Config | Value |
|--------|-------|
| URL | `jdbc:mysql://localhost:3306/orderService` |
| Username | `root` |
| Password | `saurav193` |
| DDL Auto | `update` |

### MySQL — Inventory Service
| Config | Value |
|--------|-------|
| URL | `jdbc:mysql://localhost:3306/inventoryService` |
| Username | `root` |
| Password | `saurav193` |
| DDL Auto | `create-drop` |

---

## Observability

All services are configured with distributed tracing:

- **Tracing endpoint:** `http://localhost:9411/api/v2/spans` (Zipkin)
- **Sampling probability:** `1.0` (100% of traces captured)
- Traces propagate across service boundaries using Brave/B3 propagation

**Actuator endpoints** are enabled on all services at `/actuator/*`, including:
- `/actuator/health` — Service health status
- `/actuator/metrics` — Application metrics
- `/actuator/circuitbreakers` — Circuit breaker state (Order Service)

---

## Resilience Patterns

### Circuit Breaker — Order Service → Inventory Service

| Property | Value |
|----------|-------|
| Name | `inventory` |
| Sliding Window Type | `COUNT_BASED` |
| Sliding Window Size | 5 calls |
| Failure Rate Threshold | 50% |
| Wait Duration (Open state) | 5 seconds |
| Calls in Half-Open State | 3 |
| Auto transition to Half-Open | Enabled |

### TimeLimiter
| Property | Value |
|----------|-------|
| Timeout Duration | 3 seconds |

### Retry
| Property | Value |
|----------|-------|
| Max Attempts | 3 |
| Wait Duration between retries | 5 seconds |

---

## Infrastructure (Docker)

The `docker-compose.yml` includes infrastructure services:

### Active Services
| Service | Image | Port |
|---------|-------|------|
| Zookeeper | `confluentinc/cp-zookeeper` | 2181 |
| Kafka Broker | `confluentinc/cp-kafka` | 9092 |

### Available (Commented Out)
| Service | Port | Purpose |
|---------|------|---------|
| MongoDB | 27017 | Product Service database |
| MySQL | 3306 | Order & Inventory databases |
| Keycloak | 8181 | OAuth2 / JWT authentication |
| Zipkin | 9411 | Distributed tracing UI |
| Prometheus | 9090 | Metrics collection |
| Grafana | 3000 | Metrics visualization |

To start infrastructure services:
```bash
docker-compose up -d zookeeper broker
```

---

## Configuration & Ports

| Service | Port | Eureka Name | Database |
|---------|------|-------------|----------|
| API Gateway | 9000 | `api-gateway` | — |
| Discovery Server | 8761 | — (server) | — |
| Product Service | 8081 | `product-service` | MongoDB |
| Order Service | 8082 | `order-service` | MySQL |
| Inventory Service | 8083 | `inventory-service` | MySQL |
| Notification Service | 8085 | `notification-service` | — |
| Kafka Broker | 9092 | — | — |
| Zookeeper | 2181 | — | — |
| Keycloak | 8181 | — | — |
| Zipkin | 9411 | — | — |

---

## How to Run

### Prerequisites
- Java 17+
- Maven 3.x
- Docker & Docker Compose
- MySQL running locally (port 3306)
- MongoDB running locally (port 27017)

### Step 1: Start Infrastructure
```bash
# From project root
docker-compose up -d
```
This starts Kafka and Zookeeper. For full observability, uncomment Zipkin/Prometheus/Grafana in `docker-compose.yml`.

### Step 2: Start MySQL & MongoDB
Ensure MySQL and MongoDB are running locally, then create the required databases:
```sql
CREATE DATABASE orderService;
CREATE DATABASE inventoryService;
```

### Step 3: Start Keycloak (Optional — required for auth)
```bash
# Uncomment Keycloak in docker-compose.yml and run:
docker-compose up -d keycloak
```
Configure a realm named `spring-boot-microservices-realm`.

### Step 4: Build the Project
```bash
mvn clean install -DskipTests
```

### Step 5: Start Services (in order)
```bash
# 1. Discovery Server (must start first)
cd discovery-server && mvn spring-boot:run

# 2. API Gateway
cd api-gateway && mvn spring-boot:run

# 3. Product Service
cd productService && mvn spring-boot:run

# 4. Inventory Service
cd inventoryService && mvn spring-boot:run

# 5. Order Service
cd orderService && mvn spring-boot:run

# 6. Notification Service
cd notificationService && mvn spring-boot:run
```

### Step 6: Verify
- Eureka Dashboard: `http://localhost:8761` (eureka/password)
- API Gateway: `http://localhost:9000`
- All 5 microservices should appear registered in the Eureka dashboard

---

## Sample Data

The Inventory Service pre-loads the following data on startup:

| SKU Code | Quantity | In Stock |
|----------|----------|----------|
| `iphone_13` | 100 | Yes |
| `iphone_13_red` | 0 | No |

---

## Order Placement Flow

```
1. Client ──POST /api/order──▶ API Gateway (port 9000)
           [Bearer JWT token]

2. API Gateway validates JWT with Keycloak

3. API Gateway ──routes──▶ Order Service (port 8082)

4. Order Service:
   ├── Extracts SKU codes from request
   ├── Calls Inventory Service via WebClient (with Circuit Breaker + Retry)
   │   └── GET lb://inventory-service/api/inventory?skuCode=...
   │
   ├── [All items in stock?]
   │   ├── YES:
   │   │   ├── Saves Order to MySQL
   │   │   ├── Publishes OrderPlacedEvent to Kafka (notificationTopic)
   │   │   └── Returns "Order Placed"
   │   │
   │   └── NO:
   │       └── Returns error / fallback message
   │
   └── [Inventory unavailable / timeout?]
       └── Circuit breaker fallback returns error

5. OrderPlacedEventListener ──Kafka──▶ Notification Service (port 8085)

6. Notification Service logs: "Order placed with order number: <orderNumber>"
```
