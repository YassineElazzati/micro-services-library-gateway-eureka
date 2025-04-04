# Mise en place d'un projet microservices avec Spring Boot, Eureka et Gateway

Ce projet démontre comment construire une architecture microservices avec Eureka comme service de découverte et Spring Cloud Gateway pour faire du routing dynamique avec load balancing automatique.

---

## 1. Prérequis

- Java 17  
- Maven  
- Docker / Docker Compose  
- Postman ou Curl pour tester

---

## 2. Structure du projet

```
microservices-library
├── auteur-service
├── livre-service
├── gateway-service
└── eureka-server
```

---

## 3. Initialisation avec start.spring.io

### Pour chaque microservice (auteur, livre, gateway)

- **Project** : Maven
- **Language** : Java
- **Spring Boot** : 3.2.1
- **Group** : com.example
- **Packaging** : Jar
- **Java** : 17

### Auteur / Livre Service - Dépendances :

- Spring Web
- Spring Boot DevTools
- Spring Data JPA
- PostgreSQL Driver
- Spring Boot Actuator
- Eureka Discovery Client

### Gateway - Dépendances :

- Spring Cloud Gateway
- Eureka Discovery Client

### Eureka Server - Dépendances :

- Eureka Server
- Spring Boot Actuator

---

## 4. Configuration Eureka Server

### Classe principale
```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### application.properties
```properties
server.port=8761
spring.application.name=eureka-server

eureka.client.register-with-eureka=false  # Le serveur ne s'enregistre pas lui-même
eureka.client.fetch-registry=false        # Il ne va pas chercher d'autres services
```

---

## 5. Configuration des microservices

### application-docker.properties (exemple pour `auteur-service`)
```properties
server.port=8081
spring.application.name=auteur-service

# BDD PostgreSQL (conteneur Docker)
spring.datasource.url=jdbc:postgresql://auteur-db:5432/auteurdb
spring.datasource.username=postgres
spring.datasource.password=postgres

# JPA
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Eureka
eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka
eureka.instance.prefer-ip-address=true

spring.config.activate.on-profile=docker
```

---

## 6. Configuration Gateway

### application.properties
```properties
server.port=8080
spring.application.name=gateway-service
spring.main.web-application-type=reactive

eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka
eureka.instance.prefer-ip-address=true

spring.cloud.gateway.routes[0].id=auteur-service
spring.cloud.gateway.routes[0].uri=lb://auteur-service
spring.cloud.gateway.routes[0].predicates[0]=Path=/api/auteurs/**

spring.cloud.gateway.routes[1].id=livre-service
spring.cloud.gateway.routes[1].uri=lb://livre-service
spring.cloud.gateway.routes[1].predicates[0]=Path=/api/clients/**
```

---

## 7. Dockerisation

### docker-compose.yml
```yaml
services:
  auteur-db:
    image: postgres:15
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: auteurdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - auteur-data:/var/lib/postgresql/data

  livre-db:
    image: postgres:15
    ports:
      - "5434:5432"
    environment:
      POSTGRES_DB: livredb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - livre-data:/var/lib/postgresql/data

  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  auteur-service:
    build: ./auteur-service
    ports:
      - "8081:8081"
    depends_on:
      - auteur-db
      - eureka-server

  livre-service:
    build: ./livre-service
    ports:
      - "8082:8082"
    depends_on:
      - livre-db
      - eureka-server

  gateway-service:
    build: ./gateway-service
    ports:
      - "8080:8080"
    depends_on:
      - auteur-service
      - livre-service
      - eureka-server

volumes:
  auteur-data:
  livre-data:
```

### Dockerfile (exemple : `auteur-service`)
```Dockerfile
FROM maven:3.9.3-eclipse-temurin-17 as builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar", "--spring.profiles.active=docker"]
```

---

## 8. Test du Load Balancing

- Lancer une **2e instance** de `livre-service` sur un autre port (ex: `8088`) avec un autre port DB (ex: `5435`)
- Vérifier dans Eureka (http://localhost:8761) que `LIVRE-SERVICE` apparaît **2 fois**
- Envoyer plusieurs requêtes GET `/api/clients` depuis Postman ou curl
- Observer dans les logs que les appels alternent entre les deux instances (`8082`, `8088`...)

---

## 9. Conclusion

> Eureka couplé à Spring Cloud Gateway permet de créer une architecture microservices dynamique, résiliente et scalable à chaud. Le **load balancing** se fait automatiquement par détection des instances via `lb://`. Aucune configuration manuelle supplémentaire n'est nécessaire.
