напиши java spring boot 3 gradle приложение, которое отправляет и получает сообщения из Kafka. приложение сохраняет сообщение в бд postgres , а потом отправляет его в Kafka, а когда получает сообщение из Kafka, то находит запись в базе данных и обновляет ее. приложение, Kafka и postgres должны запускаться с помощью docker compose 

Вот полный пример Spring Boot 3 приложения с Kafka и PostgreSQL, включая Docker Compose конфигурацию.

## 1. Структура проекта
```
kafka-postgres-app/
├── src/
│   └── main/
│       ├── java/com/example/kafkapostgres/
│       │   ├── controller/
│       │   │   └── MessageController.java
│       │   ├── model/
│       │   │   └── Message.java
│       │   ├── repository/
│       │   │   └── MessageRepository.java
│       │   ├── service/
│       │   │   ├── KafkaConsumerService.java
│       │   │   └── KafkaProducerService.java
│       │   ├── dto/
│       │   │   └── MessageRequest.java
│       │   └── KafkaPostgresApplication.java
│       └── resources/
│           ├── application.yml
│           └── schema.sql
├── build.gradle
├── docker-compose.yml
├── Dockerfile
└── gradlew
```

## 2. Файлы приложения

### `build.gradle`
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
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.kafka:spring-kafka'
    
    runtimeOnly 'org.postgresql:postgresql'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.kafka:spring-kafka-test'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### `src/main/java/com/example/kafkapostgres/KafkaPostgresApplication.java`
```java
package com.example.kafkapostgres;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class KafkaPostgresApplication {
    public static void main(String[] args) {
        SpringApplication.run(KafkaPostgresApplication.class, args);
    }
}
```

### `src/main/java/com/example/kafkapostgres/model/Message.java`
```java
package com.example.kafkapostgres.model;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "messages")
public class Message {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String content;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private MessageStatus status;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "processed_at")
    private LocalDateTime processedAt;
    
    @Column(name = "kafka_offset")
    private Long kafkaOffset;
    
    @Column(name = "kafka_partition")
    private Integer kafkaPartition;
    
    public enum MessageStatus {
        PENDING,
        PROCESSED,
        FAILED
    }
    
    // Конструкторы
    public Message() {}
    
    public Message(String content) {
        this.content = content;
        this.status = MessageStatus.PENDING;
        this.createdAt = LocalDateTime.now();
    }
    
    // Геттеры и сеттеры
    public Long getId() {
        return id;
    }
    
    public void setId(Long id) {
        this.id = id;
    }
    
    public String getContent() {
        return content;
    }
    
    public void setContent(String content) {
        this.content = content;
    }
    
    public MessageStatus getStatus() {
        return status;
    }
    
    public void setStatus(MessageStatus status) {
        this.status = status;
    }
    
    public LocalDateTime getCreatedAt() {
        return createdAt;
    }
    
    public void setCreatedAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
    }
    
    public LocalDateTime getProcessedAt() {
        return processedAt;
    }
    
    public void setProcessedAt(LocalDateTime processedAt) {
        this.processedAt = processedAt;
    }
    
    public Long getKafkaOffset() {
        return kafkaOffset;
    }
    
    public void setKafkaOffset(Long kafkaOffset) {
        this.kafkaOffset = kafkaOffset;
    }
    
    public Integer getKafkaPartition() {
        return kafkaPartition;
    }
    
    public void setKafkaPartition(Integer kafkaPartition) {
        this.kafkaPartition = kafkaPartition;
    }
}
```

### `src/main/java/com/example/kafkapostgres/dto/MessageRequest.java`
```java
package com.example.kafkapostgres.dto;

public class MessageRequest {
    private String content;
    
    public MessageRequest() {}
    
    public MessageRequest(String content) {
        this.content = content;
    }
    
    public String getContent() {
        return content;
    }
    
    public void setContent(String content) {
        this.content = content;
    }
}
```

### `src/main/java/com/example/kafkapostgres/repository/MessageRepository.java`
```java
package com.example.kafkapostgres.repository;

import com.example.kafkapostgres.model.Message;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface MessageRepository extends JpaRepository<Message, Long> {
}
```

### `src/main/java/com/example/kafkapostgres/service/KafkaProducerService.java`
```java
package com.example.kafkapostgres.service;

import com.example.kafkapostgres.model.Message;
import com.example.kafkapostgres.repository.MessageRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Service
public class KafkaProducerService {
    
    @Value("${kafka.topic.name}")
    private String topicName;
    
    @Autowired
    private KafkaTemplate<String, Message> kafkaTemplate;
    
    @Autowired
    private MessageRepository messageRepository;
    
    @Transactional
    public Message sendMessage(String content) {
        // Сохраняем сообщение в базу данных
        Message message = new Message(content);
        message = messageRepository.save(message);
        
        // Отправляем сообщение в Kafka
        kafkaTemplate.send(topicName, message);
        
        return message;
    }
}
```

### `src/main/java/com/example/kafkapostgres/service/KafkaConsumerService.java`
```java
package com.example.kafkapostgres.service;

import com.example.kafkapostgres.model.Message;
import com.example.kafkapostgres.repository.MessageRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Service
public class KafkaConsumerService {
    
    private static final Logger logger = LoggerFactory.getLogger(KafkaConsumerService.class);
    
    @Autowired
    private MessageRepository messageRepository;
    
    @KafkaListener(topics = "${kafka.topic.name}", groupId = "${spring.kafka.consumer.group-id}")
    @Transactional
    public void consumeMessage(
            @Payload Message message,
            @Header(KafkaHeaders.RECEIVED_PARTITION) Integer partition,
            @Header(KafkaHeaders.OFFSET) Long offset) {
        
        logger.info("Received message from Kafka: {}", message.getContent());
        
        try {
            // Обновляем сообщение в базе данных
            message.setStatus(Message.MessageStatus.PROCESSED);
            message.setProcessedAt(LocalDateTime.now());
            message.setKafkaPartition(partition);
            message.setKafkaOffset(offset);
            
            messageRepository.save(message);
            
            logger.info("Message processed successfully: {}", message.getId());
        } catch (Exception e) {
            logger.error("Error processing message: {}", e.getMessage());
            
            // Обновляем статус на FAILED в случае ошибки
            message.setStatus(Message.MessageStatus.FAILED);
            messageRepository.save(message);
        }
    }
}
```

### `src/main/java/com/example/kafkapostgres/controller/MessageController.java`
```java
package com.example.kafkapostgres.controller;

import com.example.kafkapostgres.dto.MessageRequest;
import com.example.kafkapostgres.model.Message;
import com.example.kafkapostgres.service.KafkaProducerService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/messages")
public class MessageController {
    
    @Autowired
    private KafkaProducerService kafkaProducerService;
    
    @PostMapping
    public ResponseEntity<Message> sendMessage(@RequestBody MessageRequest request) {
        Message message = kafkaProducerService.sendMessage(request.getContent());
        return ResponseEntity.ok(message);
    }
}
```

### `src/main/resources/application.yml`
```yaml
spring:
  application:
    name: kafka-postgres-app
  
  datasource:
    url: jdbc:postgresql://postgres:5432/message_db
    username: postgres
    password: postgres
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
    show-sql: true
  
  kafka:
    bootstrap-servers: kafka:9092
    consumer:
      group-id: message-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"
        spring.json.type.mapping: "com.example.kafkapostgres.model.Message"
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

kafka:
  topic:
    name: messages-topic

server:
  port: 8080
```

### `src/main/resources/schema.sql` (опционально)
```sql
CREATE TABLE IF NOT EXISTS messages (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP,
    processed_at TIMESTAMP,
    kafka_offset BIGINT,
    kafka_partition INTEGER
);
```

## 3. Docker файлы

### `Dockerfile`
```dockerfile
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY gradlew .
COPY gradle gradle
COPY build.gradle .
COPY src src
RUN chmod +x ./gradlew
RUN ./gradlew build -x test
RUN mv build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### `docker-compose.yml`
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: message_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - kafka-network

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    networks:
      - kafka-network

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    volumes:
      - kafka_data:/var/lib/kafka/data
    networks:
      - kafka-network

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
    networks:
      - kafka-network

  app:
    build: .
    depends_on:
      - postgres
      - kafka
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/message_db
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: postgres
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    networks:
      - kafka-network

volumes:
  postgres_data:
  kafka_data:

networks:
  kafka-network:
    driver: bridge
```

## 4. Инструкция по запуску

1. **Создайте структуру проекта** как указано выше.

2. **Соберите проект:**
```bash
./gradlew build
```

3. **Запустите все сервисы:**
```bash
docker-compose up -d
```

4. **Проверьте, что все сервисы запущены:**
```bash
docker-compose ps
```

5. **Отправьте тестовое сообщение:**
```bash
curl -X POST http://localhost:8080/api/messages \
  -H "Content-Type: application/json" \
  -d '{"content": "Hello Kafka and PostgreSQL!"}'
```

6. **Проверьте результат:**
   - Приложение сохранит сообщение в PostgreSQL
   - Отправит его в Kafka
   - Потребитель получит сообщение из Kafka
   - Обновит запись в базе данных

7. **Мониторинг:**
   - Kafka UI: http://localhost:8081
   - Приложение: http://localhost:8080
   - Проверьте логи: `docker-compose logs -f app`

## 5. Особенности реализации

1. **Транзакционность:** Используется `@Transactional` для обеспечения целостности данных
2. **Обработка ошибок:** Сообщения помечаются как `FAILED` при ошибках обработки
3. **Метаданные Kafka:** Сохраняются offset и partition для отслеживания
4. **Docker Compose:** Все сервисы запускаются одной командой
5. **Spring Boot 3:** Используются актуальные версии зависимостей

Приложение готово к использованию и может быть расширено дополнительной функциональностью.
