# 🏥 Patient Management System — Microservices

A production-style, cloud-ready **Patient Management System** built with a microservices architecture using **Java Spring Boot**, **gRPC**, **Apache Kafka**, **PostgreSQL**, and **Docker**. Each service is independently deployable, communicates asynchronously via Kafka and synchronously via gRPC, and sits behind a unified API Gateway.

---

## 📐 Architecture Overview

```
                        ┌─────────────────┐
                        │   API Gateway   │
                        └────────┬────────┘
              ┌──────────────────┼──────────────────┐
              │                  │                  │
     ┌────────▼────────┐ ┌───────▼───────┐ ┌───────▼───────┐
     │  Patient Service│ │  Auth Service │ │Billing Service│
     │  (REST + gRPC)  │ │  (JWT / REST) │ │  (gRPC Server)│
     └────────┬────────┘ └───────────────┘ └───────────────┘
              │ Kafka Events
     ┌────────▼────────┐ ┌───────────────┐
     │Analytics Service│ │Notification   │
     │ (Kafka Consumer)│ │Service (Kafka)│
     └─────────────────┘ └───────────────┘
```

| Communication | Mechanism |
|---|---|
| Client → Services | REST via API Gateway |
| Patient ↔ Billing | gRPC |
| Patient → Analytics / Notification | Apache Kafka |
| Auth validation | JWT tokens |

---

## 📦 Services

### 1. `api-gateway`
Central entry point that routes incoming HTTP requests to the appropriate downstream microservice. Handles JWT validation before forwarding requests.

### 2. `auth-service`
Handles user registration, login, and JWT issuance. Backed by its own PostgreSQL database (`auth-service-db`).

- Framework: Spring Boot + Spring Security
- Auth: JWT (`jjwt` 0.12.6)
- DB: PostgreSQL
- Docs: SpringDoc OpenAPI / Swagger UI

### 3. `patient-service`
Core CRUD service for patient records. Publishes events to Kafka when patients are created/updated and calls the Billing Service via gRPC.

- Framework: Spring Boot + Spring Data JPA
- DB: PostgreSQL (`patient-service-db`)
- Messaging: Apache Kafka (producer)
- RPC: gRPC client → Billing Service

### 4. `billing-service`
Manages billing accounts for patients. Exposes a gRPC server consumed by the Patient Service.

- Framework: Spring Boot
- Protocol: gRPC + Protocol Buffers
- DB: PostgreSQL

### 5. `analytics-service`
Consumes Kafka events produced by the Patient Service to generate analytics and insights.

- Framework: Spring Boot
- Messaging: Apache Kafka (consumer)

### 6. `notification-service` *(inferred from README)*
Listens to Kafka events and sends notifications (e.g., email, alerts) when patient events occur.

- Framework: Spring Boot
- Messaging: Apache Kafka (consumer)
- Serialization: Protocol Buffers

### 7. `infrastructure`
Docker Compose configuration and infrastructure definitions for running all services, databases, Kafka, and the API Gateway together.

### 8. `integration-tests`
End-to-end integration tests that validate service interactions across the system.

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Language | Java |
| Framework | Spring Boot |
| Inter-service RPC | gRPC + Protocol Buffers |
| Messaging | Apache Kafka |
| Database | PostgreSQL (per-service) |
| Auth | JWT (JJWT 0.12.6) |
| Serialization | Protobuf 4.29.1 |
| API Docs | SpringDoc OpenAPI |
| Build | Maven |
| Containerization | Docker / Docker Compose |

---

## 🚀 Getting Started

### Prerequisites

- Docker & Docker Compose
- Java 17+
- Maven 3.8+

### Run with Docker Compose

```bash
git clone https://github.com/VEDANT9463/patient-management-system-microservice.git
cd patient-management-system-microservice/infrastructure
docker compose up --build
```

This spins up all services, their databases, Kafka, and the API Gateway.

---

## ⚙️ Environment Variables

### `patient-service`

```env
BILLING_SERVICE_ADDRESS=billing-service
BILLING_SERVICE_GRPC_PORT=9005
JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
SPRING_DATASOURCE_URL=jdbc:postgresql://patient-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_DATASOURCE_PASSWORD=password
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
SPRING_SQL_INIT_MODE=always
```

### `auth-service`

```env
SPRING_DATASOURCE_URL=jdbc:postgresql://auth-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_DATASOURCE_PASSWORD=password
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_SQL_INIT_MODE=always
```

### `auth-service-db`

```env
POSTGRES_DB=db
POSTGRES_USER=admin_user
POSTGRES_PASSWORD=password
```

### `notification-service`

```env
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

### Kafka (Bitnami image)

```env
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT
KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
KAFKA_CFG_NODE_ID=0
KAFKA_CFG_PROCESS_ROLES=controller,broker
```

---

## 🔌 gRPC Setup (Maven — Billing & Patient Services)

Add to `<dependencies>`:

```xml
<!-- gRPC -->
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.69.0</version>
</dependency>
<!-- Required for Java 9+ -->
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>annotations-api</artifactId>
    <version>6.0.53</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>4.29.1</version>
</dependency>
```

Replace `<build>` section:

```xml
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.0</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.25.5:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.68.1:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

---

## 🔑 Default Test User (Auth Service)

A seed user is inserted on startup via `data.sql`:

| Field | Value |
|---|---|
| Email | `testuser@test.com` |
| Password | `password` (bcrypt hashed) |
| Role | `ADMIN` |

---

## 📁 Project Structure

```
patient-management-system-microservice/
├── api-gateway/            # Spring Cloud Gateway
├── auth-service/           # JWT authentication service
├── billing-service/        # Billing gRPC server
├── patient-service/        # Core patient CRUD + Kafka producer + gRPC client
├── analytics-service/      # Kafka consumer for analytics
├── notification-service/   # Kafka consumer for notifications
├── infrastructure/         # Docker Compose & infra config
├── integration-tests/      # End-to-end tests
├── api-requests/           # HTTP request files (REST)
└── grpc-requests/          # gRPC request files
```

---

## 📬 API Requests

Sample HTTP requests are available in the `api-requests/` directory and gRPC requests in `grpc-requests/billing-service/`. These can be used with tools like IntelliJ HTTP Client, Postman, or BloomRPC.

---

## 🧪 Running Integration Tests

```bash
cd integration-tests
mvn test
```

---

## 📄 License

This project is open source. Feel free to fork, use, and contribute.
