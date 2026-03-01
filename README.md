# Eureka Server - SmartBank Elite System

## Overview

The **Eureka Server** is the central **Service Registry** and **Service Discovery** component of the SmartBank Elite System. It acts as a registry where all microservices (Auth Service, API Gateway, Account Service, and Transaction Service) register themselves and discover other services dynamically.

## Project Information

- **Project Name**: eureka-server
- **Version**: 0.0.1-SNAPSHOT
- **Description**: Service Registry Server for SmartBank Elite System
- **Java Version**: Java 21
- **Spring Boot Version**: 3.5.11
- **Spring Cloud Version**: 2025.0.1

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                       SMARTBANK ELITE SYSTEM                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    EUREKA SERVER (Port 8761)                 │  │
│  │              Service Registry & Discovery Center             │  │
│  │              - Maintains Service Registry                    │  │
│  │              - Health Checks                                 │  │
│  │              - Load Balancing Support                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              ▲                                       │
│          ┌───────────────────┼───────────────────┬──────────────┐   │
│          │                   │                   │              │   │
│          ▼                   ▼                   ▼              ▼   │
│   ┌─────────────┐    ┌─────────────┐   ┌─────────────┐  ┌──────────┐
│   │   API       │    │    AUTH     │   │  ACCOUNT    │  │TRANSACTION
│   │  GATEWAY    │    │   SERVICE   │   │  SERVICE    │  │ SERVICE  │
│   │ (Port 8080) │    │ (Port 8081) │   │ (Port 8082) │  │(Port 8083)
│   └─────────────┘    └─────────────┘   └─────────────┘  └──────────┘
│          │                   │                   │              │   │
│          └───────────────────┼───────────────────┴──────────────┘   │
│                        (Register & Discover)                        │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

## Service Connectivity

### Connected Services

1. **API Gateway** (Port 8080)
   - Entry point for all client requests
   - Registers with Eureka for service discovery
   - Routes requests to backend services
   - Uses Eureka to discover other microservices

2. **Auth Service** (Port 8081)
   - Handles authentication and authorization
   - Registers with Eureka
   - Provides JWT token generation and validation
   - Discovered by API Gateway and other services

3. **Account Service** (Port 8082)
   - Manages customer accounts and account operations
   - Registers with Eureka
   - Handles account creation, updates, and queries
   - Discovered by API Gateway and Transaction Service

4. **Transaction Service** (Port 8083)
   - Handles financial transactions
   - Registers with Eureka
   - Communicates with Account Service for account verification
   - Discovered by API Gateway and other services

### Service Communication Flow

```
Client Request
    │
    ▼
API Gateway (discovers services via Eureka)
    │
    ├─► Auth Service (validates credentials)
    │
    └─► Account Service (fetches account details)
    │
    └─► Transaction Service (processes transactions)
         │
         └─► Account Service (updates balances)
```

## Key Features

✅ **Service Registration**: All microservices automatically register themselves  
✅ **Service Discovery**: Services can discover each other dynamically  
✅ **Health Monitoring**: Automatic health checks for registered services  
✅ **Load Balancing Support**: Integration with Spring Cloud Load Balancer  
✅ **High Availability**: Scalable registry for growing microservices  
✅ **Self-Preservation Disabled (Dev)**: For local development clarity  

## Getting Started

### Prerequisites

- Java 21 or higher
- Maven 3.6+
- Git

### Installation & Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/singh-anshu-rajdev/eureka-server.git
   cd eureka-server
   ```

2. **Build the project**
   ```bash
   mvn clean install
   ```

3. **Run the Eureka Server**
   ```bash
   mvn spring-boot:run
   ```

4. **Access Eureka Dashboard**
   ```
   http://localhost:8761
   ```

### Configuration

The Eureka Server is configured via `application.properties`:

```properties
# Application Name
spring.application.name=eureka-server

# Server Port
server.port=8761

# Do not register itself as a client
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false

# Disable self-preservation for local development
eureka.server.enable-self-preservation=false
```

## Usage Guide

### For Microservices Integration

Each microservice (Auth, Gateway, Account, Transaction) needs to:

1. **Add Eureka Client Dependency**
   ```xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
   </dependency>
   ```

2. **Enable Eureka Client**
   ```java
   @SpringBootApplication
   @EnableEurekaClient
   public class YourServiceApplication {
       public static void main(String[] args) {
           SpringApplication.run(YourServiceApplication.class, args);
       }
   }
   ```

3. **Configure Eureka Server URL**
   ```properties
   eureka.client.service-url.defaultZone=http://localhost:8761/eureka
   spring.application.name=your-service-name
   server.port=<your-port>
   ```

### Service Discovery Example (in Gateway/Services)

```java
@Autowired
private DiscoveryClient discoveryClient;

public void discoverService(String serviceName) {
    List<ServiceInstance> instances = discoveryClient.getInstances(serviceName);
    instances.forEach(instance -> {
        System.out.println("Service: " + instance.getServiceId());
        System.out.println("URL: " + instance.getUri());
    });
}
```

## Eureka Dashboard

Once the Eureka Server is running, access the web dashboard at:

```
http://localhost:8761
```

### Dashboard Features:
- **Registered Instances**: View all registered microservices
- **Instance Status**: Monitor UP/DOWN status of services
- **Availability Zones**: View service distribution
- **General Info**: Server stats and metrics
- **Application Stats**: Registered instances count

## Development Notes

### Local Development Setup

1. Start Eureka Server first (port 8761)
2. Then start other services in any order:
   - Auth Service (8081)
   - Account Service (8082)
   - Transaction Service (8083)
   - API Gateway (8080)

3. Monitor service registration at `http://localhost:8761`

### Production Considerations

- **Enable Self-Preservation**: Set `eureka.server.enable-self-preservation=true` in production
- **Clustered Setup**: Configure multiple Eureka instances for high availability
- **Security**: Add authentication and encryption for Eureka endpoints
- **Health Checks**: Configure custom health check intervals
- **Metrics**: Enable monitoring and alerting

## Technologies Used

- **Spring Boot 3.5.11**: Application framework
- **Spring Cloud 2025.0.1**: Cloud-native features
- **Netflix Eureka**: Service registry and discovery
- **Java 21**: Latest Java LTS version
- **Maven**: Build automation tool

## Directory Structure

```
eureka-server/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/smartBankElite/eurekaserver/
│   │   │       └── EurekaServerApplication.java
│   │   └── resources/
│   │       └── application.properties
│   └── test/
│       └── java/
│           └── com/smartBankElite/eurekaserver/
│               └── EurekaServerApplicationTests.java
├── pom.xml
├── mvnw
├── mvnw.cmd
├── HELP.md
└── README.md
```

## Troubleshooting

### Services not registering?
- Ensure Eureka Server is running on `http://localhost:8761`
- Check `eureka.client.service-url.defaultZone` in service configuration
- Verify network connectivity between services

### Dashboard showing red/DOWN status?
- Check service health endpoints
- Verify service is actually running
- Check logs for exceptions

### Port conflicts?
- Ensure port 8761 is available
- Modify `server.port` in `application.properties` if needed

## Contributing

1. Fork the repository
2. Create your feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

## License

This project is part of SmartBank Elite System.

## Contact & Support

For issues, questions, or contributions related to the Eureka Server, please reach out to the development team.

---

**SmartBank Elite System** - Microservices Architecture with Service Registry & Discovery

