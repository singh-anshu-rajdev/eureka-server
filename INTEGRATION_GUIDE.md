# Integration Guide - SmartBank Elite Microservices

This guide provides detailed steps for integrating Auth Service, API Gateway, Account Service, and Transaction Service with the Eureka Server.

---

## Quick Start Integration

### Step 1: Add Eureka Client Dependency to Each Service

Add the following to `pom.xml` of each microservice:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

Ensure your parent POM includes Spring Cloud dependency management:

```xml
<properties>
    <java.version>21</java.version>
    <spring-cloud.version>2025.0.1</spring-cloud.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## Service-Specific Integration

### 1. AUTH SERVICE Integration

#### Configuration (`application.properties`)

```properties
# Application Identity
spring.application.name=auth-service
server.port=8081

# Eureka Client Configuration
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
eureka.instance.hostname=localhost
eureka.instance.instance-id=${spring.application.name}:${server.port}

# Health Check
eureka.client.healthcheck.enabled=true
management.endpoints.web.exposure.include=health,info,metrics

# Prefer IP
eureka.instance.prefer-ip-address=true
```

#### Main Application Class

```java
package com.smartBankElite.authservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class AuthServiceApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(AuthServiceApplication.class, args);
    }
}
```

#### Key Endpoints

```java
@RestController
@RequestMapping("/auth")
public class AuthController {
    
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {
        // Validate credentials
        // Generate JWT Token
        // Return token
    }
    
    @PostMapping("/register")
    public ResponseEntity<?> register(@RequestBody RegisterRequest request) {
        // Validate input
        // Create user
        // Return success
    }
    
    @PostMapping("/validate-token")
    public ResponseEntity<?> validateToken(@RequestHeader("Authorization") String token) {
        // Validate JWT token
        // Return validation result
    }
    
    @PostMapping("/logout")
    public ResponseEntity<?> logout(@RequestHeader("Authorization") String token) {
        // Invalidate token
        // Return success
    }
}
```

---

### 2. API GATEWAY Integration

#### Configuration (`application.properties`)

```properties
# Application Identity
spring.application.name=api-gateway
server.port=8080

# Eureka Client Configuration
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
eureka.instance.hostname=localhost

# Gateway Routes
spring.cloud.gateway.routes[0].id=auth-service
spring.cloud.gateway.routes[0].uri=lb://AUTH-SERVICE
spring.cloud.gateway.routes[0].predicates[0]=Path=/auth/**

spring.cloud.gateway.routes[1].id=account-service
spring.cloud.gateway.routes[1].uri=lb://ACCOUNT-SERVICE
spring.cloud.gateway.routes[1].predicates[0]=Path=/account/**

spring.cloud.gateway.routes[2].id=transaction-service
spring.cloud.gateway.routes[2].uri=lb://TRANSACTION-SERVICE
spring.cloud.gateway.routes[2].predicates[0]=Path=/transaction/**

# Load Balancer Configuration
spring.cloud.loadbalancer.ribbon.enabled=false
spring.cloud.loadbalancer.cache.ttl=30s
```

#### Configuration Class (YAML Alternative)

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
          filters:
            - StripPrefix=0
        
        - id: account-service
          uri: lb://ACCOUNT-SERVICE
          predicates:
            - Path=/account/**
          filters:
            - StripPrefix=0
        
        - id: transaction-service
          uri: lb://TRANSACTION-SERVICE
          predicates:
            - Path=/transaction/**
          filters:
            - StripPrefix=0

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    prefer-ip-address: true
```

#### Main Application Class

```java
package com.smartBankElite.apigateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

---

### 3. ACCOUNT SERVICE Integration

#### Configuration (`application.properties`)

```properties
# Application Identity
spring.application.name=account-service
server.port=8082

# Eureka Client Configuration
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
eureka.instance.hostname=localhost

# Database Configuration
spring.datasource.url=jdbc:mysql://localhost:3306/smartbank_account
spring.datasource.username=root
spring.datasource.password=password
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

# Health Check
eureka.client.healthcheck.enabled=true
management.endpoints.web.exposure.include=health,info,metrics
```

#### Main Application Class

```java
package com.smartBankElite.accountservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class AccountServiceApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(AccountServiceApplication.class, args);
    }
}
```

#### Key Endpoints

```java
@RestController
@RequestMapping("/account")
public class AccountController {
    
    @Autowired
    private AccountService accountService;
    
    @PostMapping("/create")
    public ResponseEntity<?> createAccount(@RequestBody CreateAccountRequest request) {
        Account account = accountService.createAccount(request);
        return ResponseEntity.ok(account);
    }
    
    @GetMapping("/{accountId}")
    public ResponseEntity<?> getAccount(@PathVariable String accountId) {
        Account account = accountService.getAccount(accountId);
        return ResponseEntity.ok(account);
    }
    
    @GetMapping("/{accountId}/balance")
    public ResponseEntity<?> getBalance(@PathVariable String accountId) {
        BigDecimal balance = accountService.getBalance(accountId);
        return ResponseEntity.ok(balance);
    }
    
    @PutMapping("/{accountId}/balance")
    public ResponseEntity<?> updateBalance(@PathVariable String accountId, 
                                          @RequestBody BalanceUpdate update) {
        accountService.updateBalance(accountId, update.getAmount());
        return ResponseEntity.ok("Balance updated");
    }
    
    @GetMapping("/verify/{accountId}")
    public ResponseEntity<?> verifyAccount(@PathVariable String accountId) {
        boolean exists = accountService.accountExists(accountId);
        return ResponseEntity.ok(new VerifyResponse(exists));
    }
    
    @GetMapping("/user/{userId}")
    public ResponseEntity<?> getUserAccounts(@PathVariable String userId) {
        List<Account> accounts = accountService.getAccountsByUserId(userId);
        return ResponseEntity.ok(accounts);
    }
}
```

#### Entity Class

```java
@Entity
@Table(name = "accounts")
public class Account {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String accountId;
    
    @Column(nullable = false)
    private String userId;
    
    @Column(nullable = false)
    private String accountType; // SAVINGS, CHECKING, etc.
    
    @Column(nullable = false)
    private BigDecimal balance;
    
    @Column(nullable = false)
    private String currency;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    @Column(nullable = false)
    private String status; // ACTIVE, INACTIVE, CLOSED
    
    // Getters and Setters
}
```

---

### 4. TRANSACTION SERVICE Integration

#### Configuration (`application.properties`)

```properties
# Application Identity
spring.application.name=transaction-service
server.port=8083

# Eureka Client Configuration
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
eureka.instance.hostname=localhost

# Database Configuration
spring.datasource.url=jdbc:mysql://localhost:3306/smartbank_transaction
spring.datasource.username=root
spring.datasource.password=password
spring.jpa.hibernate.ddl-auto=update

# Message Queue (RabbitMQ)
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest

# Health Check
eureka.client.healthcheck.enabled=true
management.endpoints.web.exposure.include=health,info,metrics
```

#### Main Application Class

```java
package com.smartBankElite.transactionservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class TransactionServiceApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(TransactionServiceApplication.class, args);
    }
}
```

#### Feign Client for Account Service

```java
@FeignClient(name = "ACCOUNT-SERVICE")
public interface AccountServiceClient {
    
    @GetMapping("/account/{accountId}")
    AccountDTO getAccount(@PathVariable String accountId);
    
    @GetMapping("/account/{accountId}/balance")
    BigDecimal getBalance(@PathVariable String accountId);
    
    @GetMapping("/account/verify/{accountId}")
    VerifyResponse verifyAccount(@PathVariable String accountId);
    
    @PutMapping("/account/{accountId}/balance")
    void updateBalance(@PathVariable String accountId, 
                      @RequestBody BalanceUpdateRequest request);
}
```

#### Transaction Controller

```java
@RestController
@RequestMapping("/transaction")
public class TransactionController {
    
    @Autowired
    private TransactionService transactionService;
    
    @PostMapping("/transfer")
    public ResponseEntity<?> transferFunds(@RequestBody TransferRequest request) {
        Transaction transaction = transactionService.transferFunds(request);
        return ResponseEntity.ok(transaction);
    }
    
    @GetMapping("/{transactionId}")
    public ResponseEntity<?> getTransaction(@PathVariable String transactionId) {
        Transaction transaction = transactionService.getTransaction(transactionId);
        return ResponseEntity.ok(transaction);
    }
    
    @GetMapping("/user/{userId}")
    public ResponseEntity<?> getUserTransactions(@PathVariable String userId) {
        List<Transaction> transactions = transactionService.getUserTransactions(userId);
        return ResponseEntity.ok(transactions);
    }
    
    @PostMapping("/deposit")
    public ResponseEntity<?> deposit(@RequestBody DepositRequest request) {
        Transaction transaction = transactionService.deposit(request);
        return ResponseEntity.ok(transaction);
    }
    
    @PostMapping("/withdraw")
    public ResponseEntity<?> withdraw(@RequestBody WithdrawRequest request) {
        Transaction transaction = transactionService.withdraw(request);
        return ResponseEntity.ok(transaction);
    }
}
```

#### Transaction Service Implementation

```java
@Service
public class TransactionService {
    
    @Autowired
    private TransactionRepository transactionRepository;
    
    @Autowired
    private AccountServiceClient accountServiceClient;
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @Transactional
    public Transaction transferFunds(TransferRequest request) {
        String fromAccountId = request.getFromAccountId();
        String toAccountId = request.getToAccountId();
        BigDecimal amount = request.getAmount();
        
        // 1. Verify both accounts exist (via Eureka discovery)
        AccountDTO fromAccount = accountServiceClient.getAccount(fromAccountId);
        AccountDTO toAccount = accountServiceClient.getAccount(toAccountId);
        
        if (fromAccount == null || toAccount == null) {
            throw new AccountNotFoundException("Invalid account");
        }
        
        // 2. Check sufficient balance
        BigDecimal fromBalance = accountServiceClient.getBalance(fromAccountId);
        if (fromBalance.compareTo(amount) < 0) {
            throw new InsufficientBalanceException("Insufficient funds");
        }
        
        // 3. Deduct from source account
        accountServiceClient.updateBalance(fromAccountId, 
            new BalanceUpdateRequest(amount.negate()));
        
        // 4. Add to destination account
        accountServiceClient.updateBalance(toAccountId, 
            new BalanceUpdateRequest(amount));
        
        // 5. Create transaction record
        Transaction transaction = new Transaction();
        transaction.setTransactionId(UUID.randomUUID().toString());
        transaction.setFromAccountId(fromAccountId);
        transaction.setToAccountId(toAccountId);
        transaction.setAmount(amount);
        transaction.setType(TransactionType.TRANSFER);
        transaction.setStatus(TransactionStatus.SUCCESS);
        transaction.setTimestamp(LocalDateTime.now());
        
        transactionRepository.save(transaction);
        
        // 6. Publish event for notifications
        TransactionEvent event = new TransactionEvent(
            transaction.getTransactionId(),
            "TRANSFER_COMPLETED",
            transaction
        );
        rabbitTemplate.convertAndSend("transaction-events", event);
        
        return transaction;
    }
    
    public Transaction getTransaction(String transactionId) {
        return transactionRepository.findById(transactionId)
            .orElseThrow(() -> new TransactionNotFoundException("Transaction not found"));
    }
    
    public List<Transaction> getUserTransactions(String userId) {
        return transactionRepository.findByUserId(userId);
    }
}
```

---

## Service Registration & Discovery Verification

### Verify Eureka Registration

After starting all services, check the Eureka dashboard:

```
URL: http://localhost:8761
```

You should see:

```
Instances currently registered with Eureka

Application        AMIs        Availability Zones    Status
AUTH-SERVICE       1           192.168.1.100         UP
ACCOUNT-SERVICE    1           192.168.1.101         UP
TRANSACTION-SERVICE 1          192.168.1.102         UP
API-GATEWAY        1           192.168.1.103         UP
```

### Test Service Discovery via REST API

```bash
# Get all registered instances
curl -X GET http://localhost:8761/eureka/apps

# Get specific service instances
curl -X GET http://localhost:8761/eureka/apps/AUTH-SERVICE

# Get instance details
curl -X GET http://localhost:8761/eureka/apps/ACCOUNT-SERVICE/account-service:8082
```

---

## Testing Inter-Service Communication

### Test Flow

```bash
# 1. Create Account via Account Service
curl -X POST http://localhost:8082/account/create \
  -H "Content-Type: application/json" \
  -d '{"userId":"USER-001","accountType":"SAVINGS","currency":"USD"}'

# 2. Transfer Funds via Transaction Service
curl -X POST http://localhost:8083/transaction/transfer \
  -H "Content-Type: application/json" \
  -d '{
    "fromAccountId":"ACC-001",
    "toAccountId":"ACC-002",
    "amount":1000,
    "description":"Payment"
  }'

# 3. Get Transaction Details
curl -X GET http://localhost:8083/transaction/TXN-12345

# 4. Via API Gateway
curl -X POST http://localhost:8080/transaction/transfer \
  -H "Content-Type: application/json" \
  -d '{
    "fromAccountId":"ACC-001",
    "toAccountId":"ACC-002",
    "amount":1000
  }'
```

---

## Production Deployment

### Docker Deployment

#### Dockerfile for Services

```dockerfile
FROM openjdk:21-jdk-slim

WORKDIR /app

COPY target/auth-service.jar ./app.jar

EXPOSE 8081

ENV EUREKA_SERVER=eureka-server:8761

CMD ["java", "-jar", "app.jar"]
```

#### Environment Variables for Production

```properties
# For Docker/Kubernetes
EUREKA_CLIENT_SERVICE_URL_DEFAULT_ZONE=http://eureka-server:8761/eureka
SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/smartbank
SPRING_DATASOURCE_USERNAME=${DB_USERNAME}
SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD}
SPRING_RABBITMQ_HOST=rabbitmq
SPRING_RABBITMQ_PORT=5672
```

### Kubernetes Deployment

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eureka-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: eureka-server
  template:
    metadata:
      labels:
        app: eureka-server
    spec:
      containers:
      - name: eureka-server
        image: smartbank/eureka-server:latest
        ports:
        - containerPort: 8761
        env:
        - name: EUREKA_INSTANCE_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: EUREKA_CLIENT_REGISTER_WITH_EUREKA
          value: "true"
        - name: EUREKA_CLIENT_FETCH_REGISTRY
          value: "true"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        image: smartbank/auth-service:latest
        ports:
        - containerPort: 8081
        env:
        - name: EUREKA_CLIENT_SERVICE_URL_DEFAULT_ZONE
          value: "http://eureka-server-service:8761/eureka"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 10
```

---

## Troubleshooting Integration

### Issue: Service not registering with Eureka

**Solution:**
- Check Eureka Server is running
- Verify `eureka.client.service-url.defaultZone` configuration
- Check logs for connectivity errors
- Ensure `@EnableDiscoveryClient` annotation is present

### Issue: Service discovery returning no instances

**Solution:**
- Verify service is actually running
- Check health endpoint: `http://service-host:port/actuator/health`
- Check Eureka dashboard for service status
- Verify service name matches (case-sensitive)

### Issue: Connection refused between services

**Solution:**
- Ensure all services are running
- Check firewall rules
- Verify correct ports in configuration
- Check DNS resolution of service names

### Issue: Load balancer not distributing traffic

**Solution:**
- Verify multiple instances registered in Eureka
- Check Spring Cloud Load Balancer configuration
- Ensure `@LoadBalanced` annotation on RestTemplate/WebClient
- Check Feign client configuration

---

## Summary

Each microservice needs:

✅ `spring-cloud-starter-netflix-eureka-client` dependency  
✅ `@EnableDiscoveryClient` annotation  
✅ `eureka.client.service-url.defaultZone` configuration  
✅ `spring.application.name` configuration  
✅ Proper endpoint implementation for inter-service communication  
✅ Health check endpoint exposure  

With these integrations, all services will automatically discover and communicate with each other through the Eureka Server!


