Я создам полный Spring Boot 3 приложение с Gradle, которое принимает REST запросы и сохраняет данные в PostgreSQL. Вот полная структура проекта:

## Структура проекта

```
user-crud-app/
├── src/
│   ├── main/
│   │   ├── java/com/example/usercrud/
│   │   │   ├── UserCrudApplication.java
│   │   │   ├── controller/
│   │   │   │   └── UserController.java
│   │   │   ├── model/
│   │   │   │   └── User.java
│   │   │   ├── repository/
│   │   │   │   └── UserRepository.java
│   │   │   ├── service/
│   │   │   │   ├── UserService.java
│   │   │   │   └── UserServiceImpl.java
│   │   │   └── config/
│   │   │       └── MetricsConfig.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/
│   │           └── V1__create_users_table.sql
├── docker-compose.yml
├── prometheus.yml
├── grafana-datasources.yml
├── grafana-dashboards.yml
├── dashboard.json
├── build.gradle
├── settings.gradle
└── gradle.properties
```

## 1. build.gradle

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.3'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'org.flywaydb.flyway' version '9.22.3'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'

java {
    sourceCompatibility = '17'
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'org.flywaydb:flyway-core'
    
    runtimeOnly 'org.postgresql:postgresql'
    
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.testcontainers:testcontainers'
    testImplementation 'org.testcontainers:junit-jupiter'
}

tasks.named('test') {
    useJUnitPlatform()
}

flyway {
    url = 'jdbc:postgresql://localhost:5432/userdb'
    user = 'user'
    password = 'password'
}
```

## 2. settings.gradle

```gradle
rootProject.name = 'user-crud-app'
```

## 3. src/main/java/com/example/usercrud/UserCrudApplication.java

```java
package com.example.usercrud;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class UserCrudApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserCrudApplication.class, args);
    }
}
```

## 4. src/main/java/com/example/usercrud/model/User.java

```java
package com.example.usercrud.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDateTime;

@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String firstName;
    
    @Column(nullable = false)
    private String lastName;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(nullable = false)
    private Integer age;
    
    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

## 5. src/main/java/com/example/usercrud/repository/UserRepository.java

```java
package com.example.usercrud.repository;

import com.example.usercrud.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
}
```

## 6. src/main/java/com/example/usercrud/service/UserService.java

```java
package com.example.usercrud.service;

import com.example.usercrud.model.User;
import java.util.List;
import java.util.Optional;

public interface UserService {
    User createUser(User user);
    User updateUser(Long id, User user);
    void deleteUser(Long id);
    Optional<User> getUserById(Long id);
    List<User> getAllUsers();
    Optional<User> getUserByEmail(String email);
}
```

## 7. src/main/java/com/example/usercrud/service/UserServiceImpl.java

```java
package com.example.usercrud.service;

import com.example.usercrud.model.User;
import com.example.usercrud.repository.UserRepository;
import jakarta.persistence.EntityNotFoundException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class UserServiceImpl implements UserService {
    
    private final UserRepository userRepository;
    
    @Override
    public User createUser(User user) {
        if (userRepository.existsByEmail(user.getEmail())) {
            throw new IllegalArgumentException("User with email " + user.getEmail() + " already exists");
        }
        log.info("Creating user with email: {}", user.getEmail());
        return userRepository.save(user);
    }
    
    @Override
    public User updateUser(Long id, User userDetails) {
        User existingUser = userRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("User not found with id: " + id));
        
        if (!existingUser.getEmail().equals(userDetails.getEmail()) && 
            userRepository.existsByEmail(userDetails.getEmail())) {
            throw new IllegalArgumentException("Email " + userDetails.getEmail() + " is already taken");
        }
        
        existingUser.setFirstName(userDetails.getFirstName());
        existingUser.setLastName(userDetails.getLastName());
        existingUser.setEmail(userDetails.getEmail());
        existingUser.setAge(userDetails.getAge());
        
        log.info("Updating user with id: {}", id);
        return userRepository.save(existingUser);
    }
    
    @Override
    public void deleteUser(Long id) {
        if (!userRepository.existsById(id)) {
            throw new EntityNotFoundException("User not found with id: " + id);
        }
        log.info("Deleting user with id: {}", id);
        userRepository.deleteById(id);
    }
    
    @Override
    @Transactional(readOnly = true)
    public Optional<User> getUserById(Long id) {
        return userRepository.findById(id);
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }
    
    @Override
    @Transactional(readOnly = true)
    public Optional<User> getUserByEmail(String email) {
        return userRepository.findByEmail(email);
    }
}
```

## 8. src/main/java/com/example/usercrud/controller/UserController.java

```java
package com.example.usercrud.controller;

import com.example.usercrud.model.User;
import com.example.usercrud.service.UserService;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import jakarta.persistence.EntityNotFoundException;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
@Slf4j
public class UserController {
    
    private final UserService userService;
    private final MeterRegistry meterRegistry;
    
    private Counter successCounter;
    private Counter errorCounter;
    
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        try {
            User createdUser = userService.createUser(user);
            incrementSuccessCounter("create");
            return new ResponseEntity<>(createdUser, HttpStatus.CREATED);
        } catch (Exception e) {
            incrementErrorCounter("create");
            log.error("Error creating user: {}", e.getMessage());
            throw e;
        }
    }
    
    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        try {
            List<User> users = userService.getAllUsers();
            incrementSuccessCounter("getAll");
            return ResponseEntity.ok(users);
        } catch (Exception e) {
            incrementErrorCounter("getAll");
            log.error("Error getting all users: {}", e.getMessage());
            throw e;
        }
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable Long id) {
        try {
            User user = userService.getUserById(id)
                    .orElseThrow(() -> new EntityNotFoundException("User not found with id: " + id));
            incrementSuccessCounter("getById");
            return ResponseEntity.ok(user);
        } catch (Exception e) {
            incrementErrorCounter("getById");
            log.error("Error getting user by id {}: {}", id, e.getMessage());
            throw e;
        }
    }
    
    @GetMapping("/email/{email}")
    public ResponseEntity<User> getUserByEmail(@PathVariable String email) {
        try {
            User user = userService.getUserByEmail(email)
                    .orElseThrow(() -> new EntityNotFoundException("User not found with email: " + email));
            incrementSuccessCounter("getByEmail");
            return ResponseEntity.ok(user);
        } catch (Exception e) {
            incrementErrorCounter("getByEmail");
            log.error("Error getting user by email {}: {}", email, e.getMessage());
            throw e;
        }
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(@PathVariable Long id, @Valid @RequestBody User user) {
        try {
            User updatedUser = userService.updateUser(id, user);
            incrementSuccessCounter("update");
            return ResponseEntity.ok(updatedUser);
        } catch (Exception e) {
            incrementErrorCounter("update");
            log.error("Error updating user {}: {}", id, e.getMessage());
            throw e;
        }
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        try {
            userService.deleteUser(id);
            incrementSuccessCounter("delete");
            return ResponseEntity.noContent().build();
        } catch (Exception e) {
            incrementErrorCounter("delete");
            log.error("Error deleting user {}: {}", id, e.getMessage());
            throw e;
        }
    }
    
    private void incrementSuccessCounter(String operation) {
        if (successCounter == null) {
            successCounter = Counter.builder("api.requests.success")
                    .description("Successful API requests")
                    .tag("operation", operation)
                    .register(meterRegistry);
        }
        successCounter.increment();
    }
    
    private void incrementErrorCounter(String operation) {
        if (errorCounter == null) {
            errorCounter = Counter.builder("api.requests.error")
                    .description("Failed API requests")
                    .tag("operation", operation)
                    .register(meterRegistry);
        }
        errorCounter.increment();
    }
}
```

## 9. src/main/java/com/example/usercrud/config/MetricsConfig.java

```java
package com.example.usercrud.config;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config().commonTags("application", "user-crud-app");
    }
}
```

## 10. src/main/resources/application.yml

```yaml
spring:
  application:
    name: user-crud-app
  
  datasource:
    url: jdbc:postgresql://postgres:5432/userdb
    username: user
    password: password
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
    show-sql: true
  
  flyway:
    enabled: true
    baseline-on-migrate: true

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
  endpoint:
    prometheus:
      enabled: true
    metrics:
      enabled: true

logging:
  level:
    com.example.usercrud: DEBUG
    org.springframework.web: INFO
    org.hibernate: INFO
```

## 11. src/main/resources/db/migration/V1__create_users_table.sql

```sql
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    age INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

## 12. docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: user-crud-postgres
    environment:
      POSTGRES_DB: userdb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - user-crud-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d userdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: user-crud-app
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/userdb
      SPRING_DATASOURCE_USERNAME: user
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_PROFILES_ACTIVE: docker
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - user-crud-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  prometheus:
    image: prom/prometheus:latest
    container_name: user-crud-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - user-crud-network
    depends_on:
      app:
        condition: service_healthy

  grafana:
    image: grafana/grafana:latest
    container_name: user-crud-grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=
    volumes:
      - ./grafana-datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml
      - ./grafana-dashboards.yml:/etc/grafana/provisioning/dashboards/dashboards.yml
      - ./dashboard.json:/var/lib/grafana/dashboards/dashboard.json
      - grafana_data:/var/lib/grafana
    networks:
      - user-crud-network
    depends_on:
      - prometheus

networks:
  user-crud-network:
    driver: bridge

volumes:
  postgres_data:
  prometheus_data:
  grafana_data:
```

## 13. Dockerfile

```dockerfile
FROM openjdk:17-jdk-slim

WORKDIR /app

COPY build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 14. prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app:8080']
        labels:
          application: 'user-crud-app'
```

## 15. grafana-datasources.yml

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
    jsonData:
      timeInterval: 15s
```

## 16. grafana-dashboards.yml

```yaml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
```

## 17. dashboard.json (сокращенная версия с метриками)

```json
{
  "dashboard": {
    "title": "User CRUD Application Metrics",
    "panels": [
      {
        "title": "API Request Success Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(api_requests_success_total[5m])",
            "legendFormat": "Success"
          },
          {
            "expr": "rate(api_requests_error_total[5m])",
            "legendFormat": "Error"
          }
        ]
      },
      {
        "title": "API Request Count by Operation",
        "type": "barchart",
        "targets": [
          {
            "expr": "sum(api_requests_success_total) by (operation)",
            "legendFormat": "Success: {{operation}}"
          },
          {
            "expr": "sum(api_requests_error_total) by (operation)",
            "legendFormat": "Error: {{operation}}"
          }
        ]
      },
      {
        "title": "JVM Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{area=\"heap\"}",
            "legendFormat": "Heap Memory"
          },
          {
            "expr": "jvm_memory_used_bytes{area=\"nonheap\"}",
            "legendFormat": "Non-Heap Memory"
          }
        ]
      }
    ]
  }
}
```

## 18. gradle.properties

```properties
org.gradle.jvmargs=-Xmx2048m -XX:MaxMetaspaceSize=512m
org.gradle.parallel=true
org.gradle.daemon=true
```

## Инструкция по запуску

1. **Соберите приложение:**
```bash
./gradlew clean build
```

2. **Запустите все сервисы через Docker Compose:**
```bash
docker-compose up -d
```

3. **Проверьте статус сервисов:**
```bash
docker-compose ps
```

4. **Доступ к сервисам:**
   - Приложение: http://localhost:8080
   - Prometheus: http://localhost:9090
   - Grafana: http://localhost:3000 (логин: admin, пароль: admin)

5. **Тестирование API:**

Создать пользователя:
```bash
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{"firstName":"John","lastName":"Doe","email":"john@example.com","age":30}'
```

Получить всех пользователей:
```bash
curl http://localhost:8080/api/users
```

Получить пользователя по ID:
```bash
curl http://localhost:8080/api/users/1
```

Обновить пользователя:
```bash
curl -X PUT http://localhost:8080/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Jane","lastName":"Doe","email":"jane@example.com","age":28}'
```

Удалить пользователя:
```bash
curl -X DELETE http://localhost:8080/api/users/1
```

6. **Просмотр метрик:**
   - Метрики в формате Prometheus: http://localhost:8080/actuator/prometheus
   - В Grafana можно создать дашборд для визуализации метрик успешных и ошибочных запросов

## Метрики, которые собираются:

- `api_requests_success_total` - количество успешных запросов (с тегом operation)
- `api_requests_error_total` - количество запросов с ошибками (с тегом operation)
- Стандартные метрики Spring Boot Actuator и JVM

Приложение полностью готово к работе и предоставляет все необходимые CRUD операции с метриками и мониторингом через Grafana.
