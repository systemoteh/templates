Напиши 2 SpringBoot3 Gradle приложения, которые взаимодействуют друг с другом посредством REST-запросов. Первое приложение принимает REST запросы от пользователя, но не имеет собственной базы данных. Все запросы от пользователя к первому приложению перенаправляются на второе приложение, которое имеет собственную базу данных. Приложения могут обрабатывать CRUD операции. Так же есть Docker контейнер для сборки, поиска и отображения логов от этих двух приложений, основанный на EFK (elasticsearch, filebeat, kibana). Все компоненты запускаются через docker compose

Вот полный пример двух Spring Boot 3 приложений с Gradle, Docker Compose и EFK стеком для логирования:

## Структура проекта
```
spring-rest-efk/
├── app1/
├── app2/
├── docker-compose.yml
├── filebeat.yml
├── gradle/
├── gradlew
├── gradlew.bat
├── settings.gradle
└── .env
```

## Корневой settings.gradle
```groovy
rootProject.name = 'spring-rest-efk'
include 'app1', 'app2'
```

## Приложение 1 (App1 - Gateway)

### app1/build.gradle
```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
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
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    
    // Для работы с RestTemplate
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    
    // Для централизованной конфигурации
    implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer:4.1.0'
    
    // Для логирования
    implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### app1/src/main/java/com/example/app1/App1Application.java
```java
package com.example.app1;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
public class App1Application {
    public static void main(String[] args) {
        SpringApplication.run(App1Application.class, args);
    }
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

### app1/src/main/java/com/example/app1/config/WebClientConfig.java
```java
package com.example.app1.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient() {
        return WebClient.builder().build();
    }
}
```

### app1/src/main/java/com/example/app1/controller/ProductProxyController.java
```java
package com.example.app1.controller;

import com.example.app1.dto.ProductRequest;
import com.example.app1.dto.ProductResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

import java.util.List;

@RestController
@RequestMapping("/api/products")
public class ProductProxyController {
    
    private final RestTemplate restTemplate;
    private final WebClient webClient;
    
    @Value("${app2.url:http://app2:8082}")
    private String app2Url;
    
    @Autowired
    public ProductProxyController(RestTemplate restTemplate, WebClient webClient) {
        this.restTemplate = restTemplate;
        this.webClient = webClient;
    }
    
    @GetMapping
    public ResponseEntity<List<ProductResponse>> getAllProducts() {
        String url = app2Url + "/api/products";
        List<ProductResponse> products = webClient.get()
            .uri(url)
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<List<ProductResponse>>() {})
            .block();
        
        return ResponseEntity.ok(products);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProductById(@PathVariable Long id) {
        String url = app2Url + "/api/products/" + id;
        ProductResponse product = webClient.get()
            .uri(url)
            .retrieve()
            .bodyToMono(ProductResponse.class)
            .block();
        
        return product != null ? ResponseEntity.ok(product) : ResponseEntity.notFound().build();
    }
    
    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(@RequestBody ProductRequest productRequest) {
        String url = app2Url + "/api/products";
        ProductResponse product = webClient.post()
            .uri(url)
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(productRequest)
            .retrieve()
            .bodyToMono(ProductResponse.class)
            .block();
        
        return ResponseEntity.status(HttpStatus.CREATED).body(product);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<ProductResponse> updateProduct(
            @PathVariable Long id,
            @RequestBody ProductRequest productRequest) {
        String url = app2Url + "/api/products/" + id;
        ProductResponse product = webClient.put()
            .uri(url)
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(productRequest)
            .retrieve()
            .bodyToMono(ProductResponse.class)
            .block();
        
        return ResponseEntity.ok(product);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        String url = app2Url + "/api/products/" + id;
        webClient.delete()
            .uri(url)
            .retrieve()
            .bodyToMono(Void.class)
            .block();
        
        return ResponseEntity.noContent().build();
    }
}
```

### app1/src/main/java/com/example/app1/dto/ProductRequest.java
```java
package com.example.app1.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;

import java.math.BigDecimal;

public class ProductRequest {
    
    @NotBlank(message = "Name is required")
    private String name;
    
    private String description;
    
    @NotNull(message = "Price is required")
    @Positive(message = "Price must be positive")
    private BigDecimal price;
    
    @NotNull(message = "Quantity is required")
    @Positive(message = "Quantity must be positive")
    private Integer quantity;
    
    // Getters and Setters
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public String getDescription() {
        return description;
    }
    
    public void setDescription(String description) {
        this.description = description;
    }
    
    public BigDecimal getPrice() {
        return price;
    }
    
    public void setPrice(BigDecimal price) {
        this.price = price;
    }
    
    public Integer getQuantity() {
        return quantity;
    }
    
    public void setQuantity(Integer quantity) {
        this.quantity = quantity;
    }
}
```

### app1/src/main/java/com/example/app1/dto/ProductResponse.java
```java
package com.example.app1.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class ProductResponse {
    
    private Long id;
    private String name;
    private String description;
    private BigDecimal price;
    private Integer quantity;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // Getters and Setters
    public Long getId() {
        return id;
    }
    
    public void setId(Long id) {
        this.id = id;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public String getDescription() {
        return description;
    }
    
    public void setDescription(String description) {
        this.description = description;
    }
    
    public BigDecimal getPrice() {
        return price;
    }
    
    public void setPrice(BigDecimal price) {
        this.price = price;
    }
    
    public Integer getQuantity() {
        return quantity;
    }
    
    public void setQuantity(Integer quantity) {
        this.quantity = quantity;
    }
    
    public LocalDateTime getCreatedAt() {
        return createdAt;
    }
    
    public void setCreatedAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
    }
    
    public LocalDateTime getUpdatedAt() {
        return updatedAt;
    }
    
    public void setUpdatedAt(LocalDateTime updatedAt) {
        this.updatedAt = updatedAt;
    }
}
```

### app1/src/main/resources/application.yml
```yaml
server:
  port: 8081

app2:
  url: ${APP2_URL:http://localhost:8082}

spring:
  application:
    name: app1
  webflux:
    client:
      max-in-memory-size: 10MB

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true

logging:
  file:
    name: /var/log/app1/app.log
  level:
    com.example.app1: DEBUG
    org.springframework.web: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

## Приложение 2 (App2 - с базой данных)

### app2/build.gradle
```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
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
    
    // Database
    runtimeOnly 'org.postgresql:postgresql'
    
    // For JSON logging
    implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
    
    // Lombok (optional)
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### app2/src/main/java/com/example/app2/App2Application.java
```java
package com.example.app2;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class App2Application {
    public static void main(String[] args) {
        SpringApplication.run(App2Application.class, args);
    }
}
```

### app2/src/main/java/com/example/app2/model/Product.java
```java
package com.example.app2.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "products")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    private String description;
    
    @Column(nullable = false)
    private BigDecimal price;
    
    @Column(nullable = false)
    private Integer quantity;
    
    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

### app2/src/main/java/com/example/app2/repository/ProductRepository.java
```java
package com.example.app2.repository;

import com.example.app2.model.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
}
```

### app2/src/main/java/com/example/app2/controller/ProductController.java
```java
package com.example.app2.controller;

import com.example.app2.dto.ProductRequest;
import com.example.app2.dto.ProductResponse;
import com.example.app2.model.Product;
import com.example.app2.service.ProductService;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    private final ProductService productService;
    
    @Autowired
    public ProductController(ProductService productService) {
        this.productService = productService;
    }
    
    @GetMapping
    public ResponseEntity<List<ProductResponse>> getAllProducts() {
        List<ProductResponse> products = productService.getAllProducts()
            .stream()
            .map(this::convertToResponse)
            .collect(Collectors.toList());
        return ResponseEntity.ok(products);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProductById(@PathVariable Long id) {
        return productService.getProductById(id)
            .map(product -> ResponseEntity.ok(convertToResponse(product)))
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(@Valid @RequestBody ProductRequest productRequest) {
        Product product = convertToEntity(productRequest);
        Product savedProduct = productService.createProduct(product);
        return ResponseEntity.status(HttpStatus.CREATED).body(convertToResponse(savedProduct));
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<ProductResponse> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody ProductRequest productRequest) {
        return productService.updateProduct(id, convertToEntity(productRequest))
            .map(product -> ResponseEntity.ok(convertToResponse(product)))
            .orElse(ResponseEntity.notFound().build());
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        if (productService.deleteProduct(id)) {
            return ResponseEntity.noContent().build();
        }
        return ResponseEntity.notFound().build();
    }
    
    private Product convertToEntity(ProductRequest request) {
        Product product = new Product();
        product.setName(request.getName());
        product.setDescription(request.getDescription());
        product.setPrice(request.getPrice());
        product.setQuantity(request.getQuantity());
        return product;
    }
    
    private ProductResponse convertToResponse(Product product) {
        ProductResponse response = new ProductResponse();
        response.setId(product.getId());
        response.setName(product.getName());
        response.setDescription(product.getDescription());
        response.setPrice(product.getPrice());
        response.setQuantity(product.getQuantity());
        response.setCreatedAt(product.getCreatedAt());
        response.setUpdatedAt(product.getUpdatedAt());
        return response;
    }
}
```

### app2/src/main/java/com/example/app2/service/ProductService.java
```java
package com.example.app2.service;

import com.example.app2.model.Product;
import com.example.app2.repository.ProductRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Service
@Transactional
public class ProductService {
    
    private final ProductRepository productRepository;
    
    @Autowired
    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }
    
    public List<Product> getAllProducts() {
        return productRepository.findAll();
    }
    
    public Optional<Product> getProductById(Long id) {
        return productRepository.findById(id);
    }
    
    public Product createProduct(Product product) {
        return productRepository.save(product);
    }
    
    public Optional<Product> updateProduct(Long id, Product productDetails) {
        return productRepository.findById(id)
            .map(product -> {
                product.setName(productDetails.getName());
                product.setDescription(productDetails.getDescription());
                product.setPrice(productDetails.getPrice());
                product.setQuantity(productDetails.getQuantity());
                return productRepository.save(product);
            });
    }
    
    public boolean deleteProduct(Long id) {
        if (productRepository.existsById(id)) {
            productRepository.deleteById(id);
            return true;
        }
        return false;
    }
}
```

### app2/src/main/java/com/example/app2/dto/ProductRequest.java
```java
package com.example.app2.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;

import java.math.BigDecimal;

public class ProductRequest {
    
    @NotBlank(message = "Name is required")
    private String name;
    
    private String description;
    
    @NotNull(message = "Price is required")
    @Positive(message = "Price must be positive")
    private BigDecimal price;
    
    @NotNull(message = "Quantity is required")
    @Positive(message = "Quantity must be positive")
    private Integer quantity;
    
    // Getters and Setters
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public String getDescription() {
        return description;
    }
    
    public void setDescription(String description) {
        this.description = description;
    }
    
    public BigDecimal getPrice() {
        return price;
    }
    
    public void setPrice(BigDecimal price) {
        this.price = price;
    }
    
    public Integer getQuantity() {
        return quantity;
    }
    
    public void setQuantity(Integer quantity) {
        this.quantity = quantity;
    }
}
```

### app2/src/main/java/com/example/app2/dto/ProductResponse.java
```java
package com.example.app2.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class ProductResponse {
    
    private Long id;
    private String name;
    private String description;
    private BigDecimal price;
    private Integer quantity;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // Getters and Setters
    public Long getId() {
        return id;
    }
    
    public void setId(Long id) {
        this.id = id;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public String getDescription() {
        return description;
    }
    
    public void setDescription(String description) {
        this.description = description;
    }
    
    public BigDecimal getPrice() {
        return price;
    }
    
    public void setPrice(BigDecimal price) {
        this.price = price;
    }
    
    public Integer getQuantity() {
        return quantity;
    }
    
    public void setQuantity(Integer quantity) {
        this.quantity = quantity;
    }
    
    public LocalDateTime getCreatedAt() {
        return createdAt;
    }
    
    public void setCreatedAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
    }
    
    public LocalDateTime getUpdatedAt() {
        return updatedAt;
    }
    
    public void setUpdatedAt(LocalDateTime updatedAt) {
        this.updatedAt = updatedAt;
    }
}
```

### app2/src/main/resources/application.yml
```yaml
server:
  port: 8082

spring:
  application:
    name: app2
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://postgres:5432/products_db}
    username: ${DATABASE_USERNAME:postgres}
    password: ${DATABASE_PASSWORD:postgres}
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
    show-sql: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true

logging:
  file:
    name: /var/log/app2/app.log
  level:
    com.example.app2: DEBUG
    org.springframework.web: INFO
    org.hibernate.SQL: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

## Docker Compose конфигурация

### docker-compose.yml
```yaml
version: '3.8'

services:
  # Приложение 1
  app1:
    build:
      context: ./app1
      dockerfile: Dockerfile
    container_name: app1
    ports:
      - "8081:8081"
    environment:
      - APP2_URL=http://app2:8082
    networks:
      - app-network
    depends_on:
      - app2
    volumes:
      - app1-logs:/var/log/app1
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # Приложение 2
  app2:
    build:
      context: ./app2
      dockerfile: Dockerfile
    container_name: app2
    ports:
      - "8082:8082"
    environment:
      - DATABASE_URL=jdbc:postgresql://postgres:5432/products_db
      - DATABASE_USERNAME=postgres
      - DATABASE_PASSWORD=postgres
    networks:
      - app-network
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - app2-logs:/var/log/app2
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # База данных PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    environment:
      - POSTGRES_DB=products_db
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    ports:
      - "5432:5432"
    networks:
      - app-network
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Elasticsearch
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    networks:
      - app-network
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Kibana
  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - app-network
    depends_on:
      elasticsearch:
        condition: service_healthy

  # Filebeat
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    container_name: filebeat
    user: root
    command: filebeat -e -strict.perms=false
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - app1-logs:/var/log/app1:ro
      - app2-logs:/var/log/app2:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - app-network
    depends_on:
      - elasticsearch
      - kibana

networks:
  app-network:
    driver: bridge

volumes:
  app1-logs:
  app2-logs:
  postgres-data:
  elasticsearch-data:
```

### filebeat.yml
```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app1/*.log
    fields:
      app: app1
      environment: production
    fields_under_root: true
    json.keys_under_root: true
    json.add_error_key: true

  - type: log
    enabled: true
    paths:
      - /var/log/app2/*.log
    fields:
      app: app2
      environment: production
    fields_under_root: true
    json.keys_under_root: true
    json.add_error_key: true

  - type: container
    enabled: true
    paths:
      - /var/lib/docker/containers/*/*.log
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - decode_json_fields:
      fields: ["message"]
      target: ""
      overwrite_keys: true

setup.template.settings:
  index.number_of_shards: 1
  index.codec: best_compression

setup.kibana:
  host: "kibana:5601"

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  indices:
    - index: "filebeat-app1-%{+yyyy.MM.dd}"
      when.equals:
        app: "app1"
    - index: "filebeat-app2-%{+yyyy.MM.dd}"
      when.equals:
        app: "app2"
    - index: "filebeat-containers-%{+yyyy.MM.dd}"
      when:
        not:
          or:
            - equals:
                app: "app1"
            - equals:
                app: "app2"

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

### Dockerfile для app1
```dockerfile
# app1/Dockerfile
FROM openjdk:17-jdk-slim
VOLUME /tmp
ARG JAR_FILE=build/libs/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Dockerfile для app2
```dockerfile
# app2/Dockerfile
FROM openjdk:17-jdk-slim
VOLUME /tmp
ARG JAR_FILE=build/libs/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### .env файл
```env
# .env
COMPOSE_PROJECT_NAME=spring-rest-efk
```

## Инструкция по запуску

1. **Сборка приложений:**
```bash
# Из корневой директории
./gradlew :app1:build
./gradlew :app2:build
```

2. **Запуск всех сервисов:**
```bash
docker-compose up --build -d
```

3. **Проверка статуса:**
```bash
docker-compose ps
```

4. **Остановка всех сервисов:**
```bash
docker-compose down
```

## Доступ к сервисам

- **App1:** http://localhost:8081/api/products
- **App2:** http://localhost:8082/api/products
- **PostgreSQL:** localhost:5432
- **Elasticsearch:** http://localhost:9200
- **Kibana:** http://localhost:5601

## Примеры запросов

### Через App1 (прокси):
```bash
# Создание продукта
curl -X POST http://localhost:8081/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Product",
    "description": "Test Description",
    "price": 99.99,
    "quantity": 10
  }'

# Получение всех продуктов
curl http://localhost:8081/api/products

# Получение продукта по ID
curl http://localhost:8081/api/products/1

# Обновление продукта
curl -X PUT http://localhost:8081/api/products/1 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Product",
    "description": "Updated Description",
    "price": 129.99,
    "quantity": 5
  }'

# Удаление продукта
curl -X DELETE http://localhost:8081/api/products/1
```

## Настройка Kibana для просмотра логов

1. Откройте Kibana: http://localhost:5601
2. Перейдите в Stack Management → Index Patterns
3. Создайте index pattern: `filebeat-*`
4. Используйте `@timestamp` как поле времени
5. Перейдите в Discover для просмотра логов

Эта конфигурация предоставляет полный стек для двух взаимодействующих Spring Boot приложений с централизованным логированием через EFK.
