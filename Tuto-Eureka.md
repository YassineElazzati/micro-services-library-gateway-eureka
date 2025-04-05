# Tutoriel : Intégration de Eureka et Load Balancing dans une architecture Microservices avec Spring Boot et Spring Cloud Gateway

Ce tutoriel vous guide pas à pas pour mettre en place un projet microservices avec **Eureka** comme service de découverte, et **Spring Cloud Gateway** comme routeur central, avec gestion automatique du **load balancing**.

## ✅ Objectif
Mettre en place une architecture avec :
- **2 microservices** : `auteur-service` et `livre-service`
- **1 API Gateway** : `gateway-service`
- **1 Eureka Server** : `eureka-server`
- **Base de données PostgreSQL pour chaque service**
- **Load Balancing** entre instances du même microservice

---

## 1 Création des projets Spring Boot

### 📌 Cette étape permet de générer l’ossature des 4 projets Spring Boot via [start.spring.io](https://start.spring.io)

### Paramètres communs :
- **Project**: Maven
- **Language**: Java
- **Spring Boot**: 3.2.1
- **Java**: 17
- **Packaging**: Jar

### auteur-service / livre-service
Dépendances :
- `Spring Web` : pour créer des APIs REST
- `Spring Data JPA` : pour la persistance avec une base de données
- `PostgreSQL Driver` : pour communiquer avec PostgreSQL
- `Eureka Discovery Client` : pour s’enregistrer sur Eureka
- `Spring Boot DevTools` : pour le développement
- `Spring Boot Actuator` : pour exposer des points de monitoring

### gateway-service
- `Spring Cloud Gateway` : pour le routing des requêtes
- `Eureka Discovery Client` : pour communiquer avec Eureka
- `Spring Boot Actuator`

### eureka-server
- `Spring Cloud Netflix Eureka Server` : pour lancer le serveur Eureka
- `Spring Boot Actuator`

---

## 2 Mise en place d’Eureka Server

### 📌 Le serveur Eureka permet de centraliser la découverte des services.

### Classe principale `EurekaServerApplication.java`
```java
@EnableEurekaServer // active le serveur Eureka
@SpringBootApplication
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### `application.properties`
```properties
server.port=8761
spring.application.name=eureka-server

eureka.client.register-with-eureka=false  # il ne s’enregistre pas lui-même
eureka.client.fetch-registry=false        # il ne va pas chercher d’autres services
```

---

## 3 Configuration des microservices

### 📌 Ces fichiers indiquent à chaque microservice comment se connecter à Eureka et à sa propre base de données.

### Exemple : `auteur-service/src/main/resources/application-docker.properties`
```properties
server.port=8081
spring.application.name=auteur-service

spring.datasource.url=jdbc:postgresql://auteur-db:5432/auteurdb
spring.datasource.username=postgres
spring.datasource.password=postgres

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka
eureka.instance.prefer-ip-address=true

spring.config.activate.on-profile=docker
```

### 📑 Le fichier pour `livre-service` est identique sauf pour `server.port` (8082) et `spring.datasource.url`.

---

## 4 Configuration de la Gateway

### 📌 Elle route les appels vers les bons microservices en se basant sur Eureka.

### `application.properties`
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

### 📑 L'utilisation de `lb://` permet le load balancing automatique.

---

## 5 Dockerisation

### 📌 Elle permet de lancer l’ensemble des services et BDD dans des containers isolés.

### `Dockerfile` commun
```dockerfile
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

### `docker-compose.yml`
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

---

## 6 Test du Load Balancing

### 📌 Cette étape valide que Eureka et Gateway redistribuent automatiquement les requêtes entre plusieurs instances.

1. Lancez tous les services avec `docker compose up -d`
2. Ouvrez Eureka à [http://localhost:8761](http://localhost:8761)
3. Déployez une **2ème instance** de `livre-service` sur un autre port (ex: 8088) + autre DB (ex: 5435)
4. Elle apparaîtra dans Eureka comme 2ème `LIVRE-SERVICE`
5. Envoyez plusieurs requêtes GET `/api/clients` (via Postman)
6. Observez que les appels alternent entre les deux instances

---

## 7 Conclusion

Eureka + Gateway = 📊 Load balancing automatique, tolérance aux pannes, scalabilité facile

Aucune config supplémentaire n’est nécessaire côté Gateway :
Eureka se charge de la détection des instances et Spring Cloud Gateway distribue les appels avec `lb://` ✔️

