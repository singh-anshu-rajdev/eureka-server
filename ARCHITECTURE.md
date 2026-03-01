# SmartBank Elite System - Architectural Design Document

## Executive Summary

This document outlines the architectural design of the **SmartBank Elite System**, a microservices-based banking platform. The architecture uses **Netflix Eureka** as the central service registry and discovery mechanism, enabling loose coupling and dynamic service discovery among multiple microservices.

---

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CLIENT APPLICATIONS                              │
│                  (Web, Mobile, Third-party APIs)                     │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │    LOAD BALANCER      │
                    └───────────┬───────────┘
                                │
                ┌───────────────┼───────────────┐
                │                               │
                ▼                               ▼
        ┌──────────────┐              ┌──────────────┐
        │ API GATEWAY  │              │ API GATEWAY  │ (Scaled Instances)
        │ (Primary)    │              │ (Secondary)  │
        │ Port: 8080   │              │ Port: 8080   │
        └──────┬───────┘              └──────┬───────┘
               │                             │
               └──────────────┬──────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
    ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
    │ EUREKA      │      │ EUREKA      │      │ EUREKA      │
    │ SERVER      │◄────►│ SERVER      │◄────►│ SERVER      │
    │ (Primary)   │      │ (Secondary) │      │ (Tertiary)  │
    │ Port: 8761  │      │ Port: 8761  │      │ Port: 8761  │
    └─────────────┘      └─────────────┘      └─────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
        ┌─────────────────────┼──────────────────────────────┬──────────────┐
        │                     │                              │              │
        ▼                     ▼                              ▼              ▼
    ┌──────────────┐    ┌──────────────┐          ┌──────────────┐  ┌──────────────┐
    │ AUTH         │    │ ACCOUNT      │          │ TRANSACTION  │  │ NOTIFICATION │
    │ SERVICE      │    │ SERVICE      │          │ SERVICE      │  │ SERVICE      │
    │ Port: 8081   │    │ Port: 8082   │          │ Port: 8083   │  │ Port: 8084   │
    │              │    │              │          │              │  │              │
    │ ┌──────────┐ │    │ ┌──────────┐ │          │ ┌──────────┐ │  │ ┌──────────┐ │
    │ │ Database │ │    │ │ Database │ │          │ │ Database │ │  │ │ Message  │ │
    │ │ (Auth)   │ │    │ │ (Account)│ │          │ │ (Trans.) │ │  │ │ Queue    │ │
    │ └──────────┘ │    │ └──────────┘ │          │ └──────────┘ │  │ └──────────┘ │
    └──────────────┘    └──────────────┘          └──────────────┘  └──────────────┘
```

### 1.2 Component Overview

| Component | Port | Purpose | Dependencies |
|-----------|------|---------|--------------|
| **Eureka Server** | 8761 | Service Registry & Discovery | - |
| **API Gateway** | 8080 | Request routing, Load balancing | Eureka Server |
| **Auth Service** | 8081 | Authentication & Authorization | Eureka Server |
| **Account Service** | 8082 | Account Management | Eureka Server |
| **Transaction Service** | 8083 | Transaction Processing | Eureka Server, Account Service |
| **Notification Service** | 8084 | Email/SMS Notifications | Eureka Server, Message Queue |

---

## 2. Eureka Server - Service Registry

### 2.1 Role and Responsibilities

The **Eureka Server** is the cornerstone of service discovery in SmartBank Elite System:

- **Service Registration**: All microservices register their availability and network location
- **Service Discovery**: Services query Eureka to find other services
- **Health Monitoring**: Continuous health checks of registered services
- **Load Balancing**: Enables client-side load balancing among service instances
- **Failure Resilience**: Automatic removal of failed services

### 2.2 Configuration Details

```properties
# Eureka Server Identity
spring.application.name=eureka-server
server.port=8761

# Client Configuration (Eureka Server doesn't register itself)
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false

# Server Configuration
eureka.server.enable-self-preservation=false  # Local development
eureka.server.renewal-threshold-update-interval-ms=900000
eureka.server.expected-client-renewal-interval-secs=30

# Dashboard Access
# URL: http://localhost:8761
```

### 2.3 Eureka Server Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                    EUREKA SERVER                             │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. SERVICE REGISTRATION                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Service A (Auth) sends:                              │  │
│  │ {                                                    │  │
│  │   "instanceId": "AUTH-1",                           │  │
│  │   "appName": "AUTH-SERVICE",                        │  │
│  │   "ipAddr": "192.168.1.100",                        │  │
│  │   "port": 8081,                                      │  │
│  │   "status": "UP"                                     │  │
│  │ }                                                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  2. HEARTBEAT / RENEWAL                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Every 30 seconds (configurable):                     │  │
│  │ Auth Service sends heartbeat to indicate "I'm alive" │  │
│  │ Eureka updates last heartbeat timestamp              │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  3. SERVICE LOOKUP                                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ API Gateway queries: "Find ACCOUNT-SERVICE"          │  │
│  │ Eureka returns list of available instances:         │  │
│  │ - 192.168.1.101:8082 (UP)                           │  │
│  │ - 192.168.1.102:8082 (UP)                           │  │
│  │ Gateway uses load balancer to pick one              │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  4. HEALTH CHECK & EVICTION                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ If no heartbeat received for 90 seconds:            │  │
│  │ - Mark service as SUSPECT                           │  │
│  │ - If continues for 180 seconds: REMOVE from registry│  │
│  └──────────────────────────────────────────────────────┘  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Service Architecture

### 3.1 API Gateway

**Purpose**: Single entry point for all client requests

```
Client Requests
      │
      ▼
┌──────────────────────────────┐
│    API GATEWAY (8080)        │
├──────────────────────────────┤
│ • Request Routing            │
│ • Authentication Check       │
│ • Rate Limiting              │
│ • Load Balancing             │
│ • Request Logging            │
└──────┬──────────────┬────┬───┘
       │              │    │
       ▼              ▼    ▼
    /auth        /account  /transaction
       │              │    │
       ▼              ▼    ▼
 (Routes to)    (Routes to) (Routes to)
  AUTH-8081   ACCOUNT-8082  TRANS-8083
```

**Key Responsibilities**:
- Query Eureka for service locations
- Route requests based on URI patterns
- Apply cross-cutting concerns (logging, metrics)
- Return responses to clients

**Sample Configuration**:
```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: auth-service
          uri: lb://AUTH-SERVICE
          predicates:
            - Path=/auth/**
        - id: account-service
          uri: lb://ACCOUNT-SERVICE
          predicates:
            - Path=/account/**
        - id: transaction-service
          uri: lb://TRANSACTION-SERVICE
          predicates:
            - Path=/transaction/**
```

### 3.2 Auth Service

**Purpose**: Authentication and Authorization

```
Login Request
      │
      ▼
┌─────────────────────────────┐
│   AUTH SERVICE (8081)       │
├─────────────────────────────┤
│ • User Validation           │
│ • Password Hashing          │
│ • JWT Token Generation      │
│ • Token Validation          │
│ • Role-Based Access Control │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────┐
│  Auth Database  │
│ (Users, Roles)  │
└─────────────────┘
```

**Endpoints**:
- `POST /auth/login` - User login
- `POST /auth/register` - User registration
- `POST /auth/validate-token` - Token validation
- `POST /auth/logout` - User logout

### 3.3 Account Service

**Purpose**: Account Management

```
Account Request
      │
      ▼
┌──────────────────────────────┐
│  ACCOUNT SERVICE (8082)      │
├──────────────────────────────┤
│ • Create Account             │
│ • View Account Details       │
│ • Update Account Info        │
│ • Check Account Balance      │
│ • Account Status Management  │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────┐
│ Account Database │
│ (Accounts, ...)  │
└──────────────────┘
```

**Endpoints**:
- `POST /account/create` - Create new account
- `GET /account/{id}` - Get account details
- `PUT /account/{id}` - Update account
- `GET /account/{id}/balance` - Get balance
- `GET /account/user/{userId}` - Get user accounts

### 3.4 Transaction Service

**Purpose**: Transaction Processing

```
Transaction Request
      │
      ▼
┌──────────────────────────────┐
│ TRANSACTION SERVICE (8083)   │
├──────────────────────────────┤
│ • Initiate Transaction       │
│ • Verify Accounts            │
│ • Update Balances            │
│ • Log Transactions           │
│ • Generate Receipts          │
└──────────┬────────────────────┘
           │
      ┌────┴─────┐
      │           │
      ▼           ▼
  ACCOUNT    TRANSACTION
  SERVICE    DATABASE
  (8082)     (via REST)
```

**Key Flow**:
1. Receive transaction request
2. Call Account Service (via Eureka) to verify source account
3. Call Account Service to verify destination account
4. Deduct from source, add to destination
5. Log transaction details
6. Return receipt

**Endpoints**:
- `POST /transaction/transfer` - Transfer funds
- `GET /transaction/{id}` - Get transaction details
- `GET /transaction/user/{userId}` - Get user transactions
- `POST /transaction/deposit` - Deposit funds
- `POST /transaction/withdraw` - Withdraw funds

### 3.5 Notification Service (Optional)

**Purpose**: Send notifications

```
Transaction Complete
      │
      ▼
┌────────────────────────────┐
│ MESSAGE QUEUE (RabbitMQ)   │
├────────────────────────────┤
│ Event Published:           │
│ {                          │
│   "type": "TRANSFER",      │
│   "userId": "USER-123",    │
│   "amount": 100            │
│ }                          │
└────────┬───────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ NOTIFICATION SERVICE (8084)  │
├──────────────────────────────┤
│ • Listen to events           │
│ • Send Email notifications   │
│ • Send SMS notifications     │
│ • Log notifications          │
└──────────────────────────────┘
```

---

## 4. Service-to-Service Communication

### 4.1 Communication Pattern

```
API GATEWAY                      TRANSACTION SERVICE
    │                                  │
    │ 1. Request                       │
    ├─► /transaction/transfer         │
    │                                  │
    │ 2. Query Eureka                  │
    │   "Where is ACCOUNT-SERVICE?"    │
    │                                  │
    │   Eureka Response:               │
    │   "ACCOUNT-SERVICE at 8082"      │
    │                                  │
    │ 3. Call Account Service          │
    │   ├─► Verify From Account        │
    │   │    http://localhost:8082     │
    │   │    /account/verify           │
    │   │                              │
    │   └─► Verify To Account          │
    │        http://localhost:8082     │
    │        /account/verify           │
    │                                  │
    │ 4. Process Transaction           │
    │   ├─► Update From Account        │
    │   │                              │
    │   └─► Update To Account          │
    │                                  │
    │ 5. Response                      │
    │ ◄── Transaction Success          │
    │                                  │
    │ 6. Send Notification Event       │
    │   (to Message Queue/Kafka)       │
    │                                  │
```

### 4.2 Service Discovery in Code

**Using RestTemplate with Eureka**:
```java
@Service
public class TransactionService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    public void verifyAccount(String accountId) {
        // Eureka handles the URL resolution
        String url = "http://ACCOUNT-SERVICE/account/verify/" + accountId;
        AccountResponse response = restTemplate.getForObject(url, AccountResponse.class);
    }
}
```

**Using Feign Client (Recommended)**:
```java
@FeignClient(name = "ACCOUNT-SERVICE")
public interface AccountServiceClient {
    
    @GetMapping("/account/verify/{accountId}")
    AccountResponse verifyAccount(@PathVariable String accountId);
    
    @PutMapping("/account/{accountId}/balance")
    void updateBalance(@PathVariable String accountId, @RequestBody BalanceUpdate update);
}
```

---

## 5. Data Flow Examples

### 5.1 User Registration Flow

```
1. CLIENT requests: POST /auth/register
                    {
                      "email": "user@smartbank.com",
                      "password": "secure123",
                      "name": "John Doe"
                    }

2. API GATEWAY
   - Queries Eureka: "Find AUTH-SERVICE"
   - Gets: AUTH-SERVICE at localhost:8081
   - Forwards request

3. AUTH SERVICE
   - Hash password
   - Validate email format
   - Create user record in database
   - Generate JWT token

4. AUTH SERVICE Response:
   {
     "userId": "USER-001",
     "token": "eyJhbGciOiJIUzI1NiIs...",
     "message": "Registration successful"
   }

5. API GATEWAY returns response to CLIENT
```

### 5.2 Fund Transfer Flow

```
1. CLIENT requests: POST /transaction/transfer
                    {
                      "fromAccountId": "ACC-001",
                      "toAccountId": "ACC-002",
                      "amount": 1000,
                      "description": "Payment"
                    }

2. API GATEWAY
   - Validates JWT token (calls AUTH-SERVICE via Eureka)
   - Routes to TRANSACTION-SERVICE (discovered via Eureka)

3. TRANSACTION SERVICE
   a) Query Eureka: "Find ACCOUNT-SERVICE"
   
   b) Call ACCOUNT-SERVICE to verify From Account:
      GET http://ACCOUNT-SERVICE:8082/account/ACC-001/balance
      Response: { "balance": 5000 }
   
   c) Call ACCOUNT-SERVICE to verify To Account:
      GET http://ACCOUNT-SERVICE:8082/account/ACC-002/exists
      Response: { "exists": true }
   
   d) If balance sufficient:
      - Deduct from From Account
        PUT http://ACCOUNT-SERVICE:8082/account/ACC-001/balance
        { "amount": -1000 }
      
      - Add to To Account
        PUT http://ACCOUNT-SERVICE:8082/account/ACC-002/balance
        { "amount": 1000 }
   
   e) Log transaction
   
   f) Publish event to Message Queue:
      {
        "eventType": "TRANSFER_COMPLETED",
        "fromAccount": "ACC-001",
        "toAccount": "ACC-002",
        "amount": 1000,
        "timestamp": "2026-03-01T10:30:00Z"
      }

4. NOTIFICATION SERVICE (listens to queue)
   - Sends email to user
   - Sends SMS notification

5. Response to CLIENT:
   {
     "transactionId": "TXN-12345",
     "status": "SUCCESS",
     "message": "Transfer completed"
   }
```

---

## 6. Deployment Architecture

### 6.1 Development Environment

```
LOCAL MACHINE
├── Eureka Server (8761)
├── API Gateway (8080)
├── Auth Service (8081)
├── Account Service (8082)
├── Transaction Service (8083)
└── Notification Service (8084)
```

### 6.2 Production Environment (High Availability)

```
┌──────────────────────────────────────────────────────┐
│           CLOUD INFRASTRUCTURE (AWS/GCP)             │
├──────────────────────────────────────────────────────┤
│                                                       │
│  ┌────────────────────────────────────────────────┐ │
│  │         LOAD BALANCER (ALB/NLB)                │ │
│  └────────────┬─────────────────────────────────┘ │
│               │                                     │
│    ┌──────────┼──────────┐                         │
│    │          │          │                         │
│    ▼          ▼          ▼                         │
│  ┌───────┐ ┌───────┐ ┌───────┐                   │
│  │ AZ-1  │ │ AZ-2  │ │ AZ-3  │                   │
│  ├───────┤ ├───────┤ ├───────┤                   │
│  │Eureka │ │Eureka │ │Eureka │                   │
│  │ 8761  │ │ 8761  │ │ 8761  │ (Clustered)      │
│  └──┬────┘ └──┬────┘ └──┬────┘                   │
│     │        │        │                          │
│  ┌──▼──────────▼────────▼──┐                     │
│  │   Service Instances    │                     │
│  ├────────────────────────┤                     │
│  │ Gateway (3 instances)  │                     │
│  │ Auth (3 instances)     │                     │
│  │ Account (2 instances)  │                     │
│  │ Transaction (2 inst.)  │                     │
│  └────────────────────────┘                     │
│         │        │        │                     │
│         └────┬───┴────┬───┘                     │
│              │        │                         │
│              ▼        ▼                         │
│          ┌──────┐ ┌──────┐                     │
│          │  RDS │ │ Cache│                     │
│          │  DB  │ │ Layer│                     │
│          └──────┘ └──────┘                     │
│                                                 │
└──────────────────────────────────────────────────┘
```

### 6.3 Deployment Checklist

- [ ] Eureka Server cluster (3 instances minimum)
- [ ] API Gateway instances (behind load balancer)
- [ ] Auth Service instances (with database replication)
- [ ] Account Service instances (with database replication)
- [ ] Transaction Service instances (with database replication)
- [ ] Message Queue (RabbitMQ/Kafka cluster)
- [ ] Monitoring & Alerting (Prometheus/Grafana)
- [ ] Logging (ELK Stack)
- [ ] Security (API Gateway SSL, OAuth 2.0)

---

## 7. Security Architecture

### 7.1 Security Layers

```
┌─────────────────────────────────────────────┐
│         LAYER 1: API GATEWAY                │
│  - SSL/TLS Encryption (HTTPS)               │
│  - Rate Limiting                            │
│  - DDoS Protection                          │
│  - API Key Management                       │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│         LAYER 2: AUTHENTICATION             │
│  - JWT Token validation                     │
│  - OAuth 2.0 support                        │
│  - User credentials verification           │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│         LAYER 3: AUTHORIZATION              │
│  - Role-Based Access Control (RBAC)        │
│  - Permission verification                 │
│  - Resource-level access checks            │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│       LAYER 4: SERVICE COMMUNICATION        │
│  - Service-to-Service authentication       │
│  - mTLS (mutual TLS)                       │
│  - Request signing                         │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│       LAYER 5: DATA ENCRYPTION              │
│  - Database encryption (at rest)            │
│  - Column-level encryption for sensitive   │
│  - Data masking for PII                    │
└─────────────────────────────────────────────┘
```

### 7.2 Token Flow

```
USER LOGIN
    │
    ▼
AUTH SERVICE validates credentials
    │
    ▼
Generates JWT Token
    │
┌───┴─────────────────────────────────────────┐
│ TOKEN STRUCTURE:                            │
│ ┌─────────────────────────────────────────┐ │
│ │ HEADER                                  │ │
│ │ {                                       │ │
│ │   "alg": "HS256",                      │ │
│ │   "typ": "JWT"                         │ │
│ │ }                                       │ │
│ └─────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────┐ │
│ │ PAYLOAD                                 │ │
│ │ {                                       │ │
│ │   "sub": "USER-123",                   │ │
│ │   "email": "user@smartbank.com",      │ │
│ │   "roles": ["USER", "ACCOUNT_HOLDER"], │ │
│ │   "iat": 1677662400,                   │ │
│ │   "exp": 1677748800                    │ │
│ │ }                                       │ │
│ └─────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────┐ │
│ │ SIGNATURE                               │ │
│ │ HS256(base64(header) + base64(payload))│ │
│ └─────────────────────────────────────────┘ │
└───────────────────────────────────────────────┘
    │
    ▼
Return JWT to CLIENT
    │
    ▼
CLIENT includes in Authorization header:
    GET /transaction/list
    Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
    │
    ▼
API GATEWAY validates token via AUTH-SERVICE
    │
    ├─ Token valid? → Continue
    └─ Token invalid? → 401 Unauthorized
```

---

## 8. Monitoring & Observability

### 8.1 Monitoring Strategy

```
┌──────────────────────────────────────────────┐
│         ALL MICROSERVICES                    │
├──────────────────────────────────────────────┤
│ • Application Metrics (Spring Boot Actuator) │
│ • JVM Metrics (Memory, CPU, GC)              │
│ • Business Metrics (Transactions, Users)     │
│ • Network Metrics (Latency, Throughput)      │
└──────────────┬───────────────────────────────┘
               │
               ▼
        ┌────────────────┐
        │  PROMETHEUS    │
        │  (Scrapes)     │
        │  /actuator/    │
        │  prometheus    │
        └────────┬───────┘
                 │
     ┌───────────┴───────────┐
     │                       │
     ▼                       ▼
┌─────────────┐        ┌──────────────┐
│  GRAFANA    │        │  ALERTMANAGER│
│  Dashboard  │        │  (Alerts)    │
└─────────────┘        └──────────────┘
```

### 8.2 Key Metrics to Monitor

| Component | Metric | Alert Threshold |
|-----------|--------|-----------------|
| **Eureka Server** | Registered instances | < 3 instances down |
| **API Gateway** | Request latency | > 500ms p99 |
| **Services** | Error rate | > 1% |
| **Database** | Connection pool | > 80% usage |
| **JVM** | Heap usage | > 85% |
| **Transactions** | Transaction rate | Anomalies |

### 8.3 Logging Strategy

```
┌──────────────────────────────────────────────┐
│         ALL MICROSERVICES                    │
│ (Structured JSON Logging with correlation ID)
└──────────────┬───────────────────────────────┘
               │
         ┌─────┴──────┐
         │            │
         ▼            ▼
   ┌─────────┐   ┌────────┐
   │ LOGBACK │   │  SLF4J │
   │ + JSON  │   │        │
   └────┬────┘   └────┬───┘
        │             │
        └──────┬──────┘
               │
               ▼
        ┌────────────┐
        │ FILEBEAT / │
        │ LOGSTASH   │
        └─────┬──────┘
              │
              ▼
         ┌────────────┐
         │ ELASTICSEARCH
         └─────┬──────┘
              │
              ▼
         ┌────────────┐
         │   KIBANA   │
         │ (Visualize)│
         └────────────┘
```

---

## 9. Deployment Steps

### 9.1 Local Development Deployment

```bash
# 1. Start Eureka Server
cd eureka-server
mvn spring-boot:run
# Access: http://localhost:8761

# 2. Start Auth Service
cd auth-service
mvn spring-boot:run

# 3. Start Account Service
cd account-service
mvn spring-boot:run

# 4. Start Transaction Service
cd transaction-service
mvn spring-boot:run

# 5. Start API Gateway (Last)
cd api-gateway
mvn spring-boot:run

# All services should appear on Eureka dashboard
```

### 9.2 Production Deployment (Docker)

```dockerfile
# Example Dockerfile for microservice
FROM openjdk:21-jdk-slim
WORKDIR /app
COPY target/app.jar .
EXPOSE 8080
ENV EUREKA_SERVER=eureka-server:8761
CMD ["java", "-jar", "app.jar"]
```

```yaml
# Docker Compose for local environment
version: '3'
services:
  eureka-server:
    image: smartbank/eureka-server:latest
    ports:
      - "8761:8761"
    
  auth-service:
    image: smartbank/auth-service:latest
    ports:
      - "8081:8081"
    environment:
      - EUREKA_SERVER=eureka-server:8761
    depends_on:
      - eureka-server
  
  account-service:
    image: smartbank/account-service:latest
    ports:
      - "8082:8082"
    environment:
      - EUREKA_SERVER=eureka-server:8761
    depends_on:
      - eureka-server
  
  transaction-service:
    image: smartbank/transaction-service:latest
    ports:
      - "8083:8083"
    environment:
      - EUREKA_SERVER=eureka-server:8761
    depends_on:
      - eureka-server
  
  api-gateway:
    image: smartbank/api-gateway:latest
    ports:
      - "8080:8080"
    environment:
      - EUREKA_SERVER=eureka-server:8761
    depends_on:
      - eureka-server
```

---

## 10. Disaster Recovery & High Availability

### 10.1 Eureka Server HA Setup

```
EUREKA CLUSTER CONFIGURATION:

Server 1 (Primary):
  eureka.instance.hostname=eureka-1.smartbank.com
  eureka.client.service-url.defaultZone=\
    http://eureka-2.smartbank.com:8761/eureka,\
    http://eureka-3.smartbank.com:8761/eureka

Server 2 (Secondary):
  eureka.instance.hostname=eureka-2.smartbank.com
  eureka.client.service-url.defaultZone=\
    http://eureka-1.smartbank.com:8761/eureka,\
    http://eureka-3.smartbank.com:8761/eureka

Server 3 (Tertiary):
  eureka.instance.hostname=eureka-3.smartbank.com
  eureka.client.service-url.defaultZone=\
    http://eureka-1.smartbank.com:8761/eureka,\
    http://eureka-2.smartbank.com:8761/eureka
```

### 10.2 Service Replica Configuration

```
Each service should run:
- Minimum 2 instances in production
- Spread across different availability zones
- Behind a load balancer
- With health checks every 10 seconds

Example for Transaction Service:
- Instance 1: AZ-1, 192.168.1.100:8083
- Instance 2: AZ-2, 192.168.1.101:8083
- Instance 3: AZ-3, 192.168.1.102:8083
```

### 10.3 Failover Scenario

```
SCENARIO: Transaction Service Instance 1 fails

Timeline:
T+0s   : Instance 1 crashes
T+30s  : Misses heartbeat (should renew every 30s)
T+60s  : Eureka marks as SUSPECT
T+90s  : Eureka removes from registry

During this time:
- API Gateway queries Eureka: "Find TRANSACTION-SERVICE"
- Eureka returns Instance 2 and Instance 3 only
- New requests route to available instances
- No service interruption for users

Recovery:
- Restart Instance 1
- It re-registers with Eureka
- Included in load balancing again
```

---

## 11. Best Practices & Recommendations

### 11.1 For Developers

1. **Use Feign Client** for inter-service communication (abstraction over REST)
2. **Implement Circuit Breaker** (Resilience4j) for fault tolerance
3. **Add Request Correlation IDs** for distributed tracing
4. **Implement Exponential Backoff** for retries
5. **Add Comprehensive Logging** with structured JSON
6. **Write integration tests** for service communication
7. **Document API contracts** using OpenAPI/Swagger

### 11.2 For Operations

1. **Implement centralized monitoring** (Prometheus + Grafana)
2. **Set up distributed tracing** (Jaeger/Zipkin)
3. **Enable application metrics** (Spring Boot Actuator)
4. **Use container orchestration** (Kubernetes) for production
5. **Implement automated scaling** based on metrics
6. **Regular backup & restore testing** for databases
7. **Security scanning** in CI/CD pipeline

### 11.3 For Architecture

1. **API versioning** for backward compatibility
2. **Database per service** pattern
3. **Event-driven communication** for asynchronous workflows
4. **Service mesh** (Istio/Linkerd) for advanced networking
5. **API gateway rate limiting** and throttling
6. **Implement saga pattern** for distributed transactions

---

## 12. Technology Stack Summary

```
┌────────────────────────────────────────────────────┐
│          SMARTBANK ELITE - TECH STACK              │
├────────────────────────────────────────────────────┤
│                                                     │
│ Language:          Java 21 (LTS)                   │
│ Framework:         Spring Boot 3.5.11              │
│ Cloud Framework:   Spring Cloud 2025.0.1           │
│ Service Registry:  Netflix Eureka                  │
│ API Gateway:       Spring Cloud Gateway            │
│ Authentication:    JWT, OAuth 2.0                  │
│ RPC:               OpenFeign, REST                 │
│ Build Tool:        Maven                           │
│ Containers:        Docker                          │
│ Orchestration:     Kubernetes (recommended)        │
│ Database:          PostgreSQL / MySQL              │
│ Cache:             Redis                           │
│ Message Queue:     RabbitMQ / Apache Kafka         │
│ Monitoring:        Prometheus + Grafana            │
│ Logging:           ELK Stack (Elasticsearch)       │
│ Distributed Trace: Jaeger / Zipkin                 │
│ Testing:           JUnit 5, Mockito                │
│                                                     │
└────────────────────────────────────────────────────┘
```

---

## 13. Conclusion

The **SmartBank Elite System** uses a **microservices architecture** with **Eureka Server** as the central service registry and discovery mechanism. This design provides:

✅ **Scalability**: Services can be scaled independently  
✅ **Resilience**: Failures are isolated and manageable  
✅ **Flexibility**: Easy to add or remove services  
✅ **Maintainability**: Each service has a single responsibility  
✅ **Technology Diversity**: Services can use different tech stacks  
✅ **Organizational Alignment**: Teams can own specific services  

The Eureka Server is essential for enabling dynamic service discovery, load balancing, and health monitoring across all microservices in the platform.

---

**Document Version**: 1.0  
**Last Updated**: March 1, 2026  
**Maintained By**: Anshu Singh

