Вот программа на Spring Boot 3 с Gradle, которая создает веб-сервис для загрузки PDF и его отображения с автоматическим пролистыванием страниц.

## Структура проекта

```
pdf-viewer-service/
├── src/main/java/com/example/pdfviewer/
│   ├── PdfViewerApplication.java
│   ├── controller/
│   │   └── PdfController.java
│   ├── service/
│   │   └── PdfService.java
│   └── config/
│       └── WebConfig.java
├── src/main/resources/
│   ├── static/
│   │   ├── css/
│   │   │   └── style.css
│   │   └── js/
│   │       └── viewer.js
│   └── templates/
│       └── viewer.html
└── build.gradle
```

## 1. build.gradle

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.3'
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
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
    implementation 'org.apache.pdfbox:pdfbox:3.0.2'
    implementation 'commons-io:commons-io:2.15.1'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

## 2. Главный класс приложения

```java
package com.example.pdfviewer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PdfViewerApplication {
    public static void main(String[] args) {
        SpringApplication.run(PdfViewerApplication.class, args);
    }
}
```

## 3. Контроллер

```java
package com.example.pdfviewer.controller;

import com.example.pdfviewer.service.PdfService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.Resource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import java.io.IOException;
import java.util.UUID;

@Controller
public class PdfController {

    @Autowired
    private PdfService pdfService;

    @GetMapping("/")
    public String index() {
        return "upload";
    }

    @PostMapping("/upload")
    public String uploadPdf(@RequestParam("file") MultipartFile file, 
                           RedirectAttributes redirectAttributes) {
        try {
            if (file.isEmpty()) {
                redirectAttributes.addFlashAttribute("error", "Пожалуйста, выберите файл для загрузки");
                return "redirect:/";
            }

            if (!file.getContentType().equals(MediaType.APPLICATION_PDF_VALUE)) {
                redirectAttributes.addFlashAttribute("error", "Пожалуйста, загрузите PDF файл");
                return "redirect:/";
            }

            String fileId = pdfService.savePdf(file);
            return "redirect:/view/" + fileId;

        } catch (IOException e) {
            redirectAttributes.addFlashAttribute("error", "Ошибка при загрузке файла: " + e.getMessage());
            return "redirect:/";
        }
    }

    @GetMapping("/view/{fileId}")
    public String viewPdf(@PathVariable String fileId, Model model) {
        if (!pdfService.fileExists(fileId)) {
            return "redirect:/";
        }
        model.addAttribute("fileId", fileId);
        return "viewer";
    }

    @GetMapping("/pdf/{fileId}")
    @ResponseBody
    public ResponseEntity<Resource> getPdf(@PathVariable String fileId) {
        try {
            Resource resource = pdfService.loadPdfAsResource(fileId);
            
            return ResponseEntity.ok()
                    .header(HttpHeaders.CONTENT_DISPOSITION, "inline; filename=\"" + resource.getFilename() + "\"")
                    .contentType(MediaType.APPLICATION_PDF)
                    .body(resource);
        } catch (IOException e) {
            return ResponseEntity.notFound().build();
        }
    }

    @GetMapping("/pdf/{fileId}/pages")
    @ResponseBody
    public ResponseEntity<Integer> getPageCount(@PathVariable String fileId) {
        try {
            int pageCount = pdfService.getPageCount(fileId);
            return ResponseEntity.ok(pageCount);
        } catch (IOException e) {
            return ResponseEntity.notFound().build();
        }
    }
}
```

## 4. Сервис для работы с PDF

```java
package com.example.pdfviewer.service;

import org.apache.pdfbox.pdmodel.PDDocument;
import org.springframework.core.io.Resource;
import org.springframework.core.io.UrlResource;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.UUID;

@Service
public class PdfService {

    private final Path uploadDir = Paths.get("uploads");

    public PdfService() throws IOException {
        if (!Files.exists(uploadDir)) {
            Files.createDirectories(uploadDir);
        }
    }

    public String savePdf(MultipartFile file) throws IOException {
        String fileId = UUID.randomUUID().toString();
        String originalFilename = file.getOriginalFilename();
        String extension = originalFilename != null && originalFilename.contains(".") 
            ? originalFilename.substring(originalFilename.lastIndexOf(".")) 
            : ".pdf";
        
        String filename = fileId + extension;
        Path filePath = uploadDir.resolve(filename);
        
        Files.copy(file.getInputStream(), filePath);
        
        return fileId;
    }

    public Resource loadPdfAsResource(String fileId) throws IOException {
        Path filePath = findFilePath(fileId);
        return new UrlResource(filePath.toUri());
    }

    public boolean fileExists(String fileId) {
        try {
            findFilePath(fileId);
            return true;
        } catch (IOException e) {
            return false;
        }
    }

    public int getPageCount(String fileId) throws IOException {
        Path filePath = findFilePath(fileId);
        try (PDDocument document = PDDocument.load(filePath.toFile())) {
            return document.getNumberOfPages();
        }
    }

    private Path findFilePath(String fileId) throws IOException {
        try (var stream = Files.newDirectoryStream(uploadDir, fileId + "*")) {
            for (Path path : stream) {
                if (Files.isRegularFile(path)) {
                    return path;
                }
            }
        }
        throw new IOException("File not found with ID: " + fileId);
    }
}
```

## 5. HTML шаблоны

### upload.html (в папке templates)

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Загрузка PDF</title>
    <link rel="stylesheet" href="/css/style.css">
</head>
<body>
    <div class="container">
        <h1>Загрузите PDF файл</h1>
        
        <div th:if="${error}" class="error">
            <p th:text="${error}"></p>
        </div>
        
        <form action="/upload" method="post" enctype="multipart/form-data">
            <div class="form-group">
                <input type="file" name="file" accept=".pdf,application/pdf" required>
            </div>
            <button type="submit">Загрузить и просмотреть</button>
        </form>
    </div>
</body>
</html>
```

### viewer.html (в папке templates)

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Просмотр PDF с автопролистыванием</title>
    <link rel="stylesheet" href="/css/style.css">
</head>
<body>
    <div class="container">
        <h1>Просмотр PDF</h1>
        
        <div class="controls">
            <label>
                Интервал пролистывания (сек):
                <input type="number" id="interval" value="3" min="1" max="60">
            </label>
            <button id="autoPlayBtn">Старт</button>
            <button id="stopBtn">Стоп</button>
            <button id="prevBtn">Предыдущая</button>
            <button id="nextBtn">Следующая</button>
        </div>
        
        <div class="page-info">
            Страница <span id="currentPage">1</span> из <span id="totalPages">-</span>
        </div>
        
        <div class="pdf-container">
            <canvas id="pdfCanvas"></canvas>
        </div>
    </div>
    
    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>
    <script th:src="@{/js/viewer.js}"></script>
    <script th:inline="javascript">
        /*<![CDATA[*/
        const fileId = [[${fileId}]];
        /*]]>*/
    </script>
</body>
</html>
```

## 6. CSS стили

```css
/* style.css */
body {
    font-family: Arial, sans-serif;
    margin: 0;
    padding: 20px;
    background-color: #f5f5f5;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    background-color: white;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

h1 {
    color: #333;
    text-align: center;
    margin-bottom: 30px;
}

.form-group {
    text-align: center;
    margin: 20px 0;
}

input[type="file"] {
    padding: 10px;
    border: 2px dashed #ddd;
    border-radius: 4px;
    width: 100%;
    max-width: 400px;
}

button {
    background-color: #4CAF50;
    color: white;
    padding: 12px 24px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 16px;
    margin: 5px;
}

button:hover {
    background-color: #45a049;
}

button#stopBtn {
    background-color: #f44336;
}

button#stopBtn:hover {
    background-color: #da190b;
}

.controls {
    text-align: center;
    margin: 20px 0;
    padding: 15px;
    background-color: #f8f9fa;
    border-radius: 4px;
}

.controls label {
    margin-right: 10px;
}

.controls input {
    padding: 8px;
    border: 1px solid #ddd;
    border-radius: 4px;
    width: 70px;
}

.page-info {
    text-align: center;
    font-size: 18px;
    margin: 15px 0;
    font-weight: bold;
}

.pdf-container {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 500px;
    border: 1px solid #ddd;
    border-radius: 4px;
    overflow: auto;
}

#pdfCanvas {
    max-width: 100%;
    height: auto;
    box-shadow: 0 2px 5px rgba(0,0,0,0.2);
}

.error {
    background-color: #ffebee;
    color: #c62828;
    padding: 10px;
    border-radius: 4px;
    margin-bottom: 20px;
    text-align: center;
}
```

## 7. JavaScript для просмотрщика

```javascript
// viewer.js
let pdfDoc = null;
let currentPage = 1;
let totalPages = 0;
let autoPlayInterval = null;
let scale = 1.5;

// Инициализация после загрузки страницы
document.addEventListener('DOMContentLoaded', async function() {
    if (typeof fileId !== 'undefined') {
        await loadPdf();
    }
    
    // Элементы управления
    document.getElementById('autoPlayBtn').addEventListener('click', startAutoPlay);
    document.getElementById('stopBtn').addEventListener('click', stopAutoPlay);
    document.getElementById('prevBtn').addEventListener('click', prevPage);
    document.getElementById('nextBtn').addEventListener('click', nextPage);
});

async function loadPdf() {
    try {
        // Получаем количество страниц
        const pageCountResponse = await fetch(`/pdf/${fileId}/pages`);
        totalPages = await pageCountResponse.json();
        document.getElementById('totalPages').textContent = totalPages;
        
        // Загружаем PDF
        const loadingTask = pdfjsLib.getDocument(`/pdf/${fileId}`);
        pdfDoc = await loadingTask.promise;
        
        // Отображаем первую страницу
        await renderPage(1);
        
    } catch (error) {
        console.error('Ошибка загрузки PDF:', error);
        alert('Ошибка загрузки PDF файла');
    }
}

async function renderPage(pageNumber) {
    try {
        const page = await pdfDoc.getPage(pageNumber);
        const viewport = page.getViewport({ scale: scale });
        
        const canvas = document.getElementById('pdfCanvas');
        const context = canvas.getContext('2d');
        
        canvas.height = viewport.height;
        canvas.width = viewport.width;
        
        const renderContext = {
            canvasContext: context,
            viewport: viewport
        };
        
        await page.render(renderContext).promise;
        
        currentPage = pageNumber;
        document.getElementById('currentPage').textContent = currentPage;
        
    } catch (error) {
        console.error('Ошибка отображения страницы:', error);
    }
}

async function nextPage() {
    if (currentPage < totalPages) {
        await renderPage(currentPage + 1);
    } else {
        // Если достигнут конец, останавливаем автопролистывание
        stopAutoPlay();
    }
}

async function prevPage() {
    if (currentPage > 1) {
        await renderPage(currentPage - 1);
    }
}

function startAutoPlay() {
    stopAutoPlay(); // Останавливаем текущий интервал если есть
    
    const intervalSeconds = parseInt(document.getElementById('interval').value) * 1000;
    
    autoPlayInterval = setInterval(async () => {
        if (currentPage < totalPages) {
            await nextPage();
        } else {
            stopAutoPlay();
        }
    }, intervalSeconds);
    
    document.getElementById('autoPlayBtn').disabled = true;
}

function stopAutoPlay() {
    if (autoPlayInterval) {
        clearInterval(autoPlayInterval);
        autoPlayInterval = null;
    }
    document.getElementById('autoPlayBtn').disabled = false;
}
```

## 8. Конфигурация Web

```java
package com.example.pdfviewer.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/css/**")
                .addResourceLocations("classpath:/static/css/");
        registry.addResourceHandler("/js/**")
                .addResourceLocations("classpath:/static/js/");
    }
}
```

## Запуск приложения

1. Соберите проект:
```bash
./gradlew build
```

2. Запустите приложение:
```bash
./gradlew bootRun
```

3. Откройте браузер и перейдите по адресу: `http://localhost:8080`

## Функциональность

- **Загрузка PDF**: пользователь может загрузить PDF файл через веб-интерфейс
- **Просмотр PDF**: файл отображается с помощью библиотеки pdf.js
- **Автоматическое пролистывание**: можно запустить автоматическое перелистывание страниц с настраиваемым интервалом
- **Ручное управление**: кнопки для перехода между страницами
- **Информация о страницах**: отображается текущая страница и общее количество

Приложение сохраняет загруженные файлы в папку `uploads` и автоматически удаляет их при перезапуске сервера (для продакшена нужно добавить механизм очистки).
