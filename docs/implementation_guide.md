# AI面试助手实现指南

## 1. 环境准备

### 1.1 开发环境需求

- JDK 17
- Maven 3.6+
- Spring Boot 3.x
- WebFlux
- Project Reactor
- IDE (如IntelliJ IDEA或VS Code)

### 1.2 依赖配置

在`pom.xml`中添加以下依赖：

```xml
<!-- WebFlux -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<!-- Lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>

<!-- Jackson for JSON -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

## 2. 项目结构设置

创建以下包结构：

```
org.lijian.speed2text
├── config         // WebSocket和CORS配置类
├── controller     // REST API控制器
├── handler        // WebSocket流式输出处理器
├── model          // 数据模型类
├── service        // 服务接口和实现
├── util           // 工具类
└── Speed2textApplication.java
```

## 3. 后端服务实现

### 3.1 WebSocket配置

创建`config/WebSocketConfig.java`：

```java
package org.lijian.speed2text.config;

import org.lijian.speed2text.handler.StreamingAnswerHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.Ordered;
import org.springframework.web.reactive.HandlerMapping;
import org.springframework.web.reactive.handler.SimpleUrlHandlerMapping;
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.server.support.WebSocketHandlerAdapter;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class WebSocketConfig {

    @Bean
    public HandlerMapping webSocketMapping() {
        Map<String, WebSocketHandler> map = new HashMap<>();
        map.put("/api/stream/answer", new StreamingAnswerHandler());
        
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setUrlMap(map);
        mapping.setOrder(Ordered.HIGHEST_PRECEDENCE);
        return mapping;
    }
    
    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
}
```

### 3.2 CORS配置

创建`config/CorsConfig.java`：

```java
package org.lijian.speed2text.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsWebFilter;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;

import java.util.Arrays;

@Configuration
public class CorsConfig {
    @Bean
    public CorsWebFilter corsWebFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(Arrays.asList("*"));  // 生产环境应限制来源
        config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(Arrays.asList("*"));
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        
        return new CorsWebFilter(source);
    }
}
```

### 3.3 数据模型

创建以下数据模型类：

`model/QuestionRequest.java`:
```java
package org.lijian.speed2text.model;

public record QuestionRequest(String question, String sessionId) {
}
```

`model/StreamResponse.java`:
```java
package org.lijian.speed2text.model;

public record StreamResponse(String status, String message, String sessionId, String streamUrl) {
}
```

`model/StreamChunk.java`:
```java
package org.lijian.speed2text.model;

public record StreamChunk(String type, String content, String sessionId, boolean isFinal) {
}
```

`model/RecognitionResult.java`:
```java
package org.lijian.speed2text.model;

public record RecognitionResult(String text, long timestamp, boolean isFinal) {
}
```

### 3.4 REST API控制器

创建`controller/InterviewController.java`：

```java
package org.lijian.speed2text.controller;

import org.lijian.speed2text.model.QuestionRequest;
import org.lijian.speed2text.model.StreamResponse;
import org.lijian.speed2text.service.InterviewService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/api/interview")
public class InterviewController {
    
    @Autowired
    private InterviewService interviewService;
    
    @PostMapping("/question")
    public Mono<StreamResponse> processQuestion(@RequestBody QuestionRequest request) {
        // 验证请求
        if (request.question() == null || request.question().trim().isEmpty()) {
            return Mono.just(new StreamResponse(
                "error", 
                "问题内容不能为空", 
                request.sessionId(), 
                null
            ));
        }
        
        // 启动异步处理
        interviewService.processQuestionAsync(request.question(), request.sessionId());
        
        // 返回WebSocket连接信息
        String streamUrl = "/api/stream/answer?sessionId=" + request.sessionId();
        return Mono.just(new StreamResponse(
            "success", 
            "请求已接收，请通过WebSocket连接获取流式回答", 
            request.sessionId(), 
            streamUrl
        ));
    }
}
```

### 3.5 WebSocket流式处理器

创建`handler/StreamingAnswerHandler.java`：

```java
package org.lijian.speed2text.handler;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.lijian.speed2text.model.StreamChunk;
import org.lijian.speed2text.service.StreamingService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.WebSocketSession;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Component
public class StreamingAnswerHandler implements WebSocketHandler {
    
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Autowired
    private StreamingService streamingService;
    
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        // 从URL参数中获取会话ID
        String sessionId = session.getHandshakeInfo()
            .getUri()
            .getQuery()
            .split("=")[1];
            
        // 获取流式回答数据流
        Flux<StreamChunk> answerStream = streamingService.getAnswerStream(sessionId);
        
        // 将流式回答转换为WebSocket消息并发送
        return session.send(
            answerStream.map(chunk -> {
                try {
                    return session.textMessage(objectMapper.writeValueAsString(chunk));
                } catch (JsonProcessingException e) {
                    return session.textMessage("{\"error\":\"Failed to serialize response\"}");
                }
            })
        );
    }
}
```

## 4. AI服务集成

### 4.1 流式服务接口

创建`service/StreamingService.java`：

```java
package org.lijian.speed2text.service;

import org.lijian.speed2text.model.StreamChunk;
import reactor.core.publisher.Flux;

public interface StreamingService {
    Flux<StreamChunk> getAnswerStream(String sessionId);
}
```

创建`service/StreamingServiceImpl.java`：

```java
package org.lijian.speed2text.service.impl;

import org.lijian.speed2text.model.StreamChunk;
import org.lijian.speed2text.service.AIStreamingService;
import org.lijian.speed2text.service.StreamingService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Sinks;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class StreamingServiceImpl implements StreamingService {
    
    @Autowired
    private AIStreamingService aiStreamingService;
    
    private final Map<String, Sinks.Many<StreamChunk>> sessionSinks = new ConcurrentHashMap<>();
    
    @Override
    public Flux<StreamChunk> getAnswerStream(String sessionId) {
        Sinks.Many<StreamChunk> sink = sessionSinks.computeIfAbsent(
            sessionId, 
            k -> Sinks.many().multicast().onBackpressureBuffer()
        );
        
        return sink.asFlux();
    }
    
    public void processQuestion(String question, String sessionId) {
        Sinks.Many<StreamChunk> sink = sessionSinks.get(sessionId);
        
        if (sink != null) {
            aiStreamingService.generateStreamingResponse(question, Map.of())
                .index()
                .subscribe(
                    tuple -> {
                        boolean isFinal = false;
                        String content = tuple.getT2();
                        
                        // 最后一个数据包标记为最终
                        if (content.contains("[END]")) {
                            content = content.replace("[END]", "");
                            isFinal = true;
                        }
                        
                        StreamChunk chunk = new StreamChunk(
                            isFinal ? "complete" : "chunk", 
                            content, 
                            sessionId, 
                            isFinal
                        );
                        
                        sink.tryEmitNext(chunk);
                        
                        if (isFinal) {
                            sink.tryEmitComplete();
                            sessionSinks.remove(sessionId);
                        }
                    },
                    error -> {
                        sink.tryEmitError(error);
                        sessionSinks.remove(sessionId);
                    }
                );
        }
    }
}
```

### 4.2 AI服务接口

创建`service/AIStreamingService.java`：

```java
package org.lijian.speed2text.service;

import reactor.core.publisher.Flux;

import java.util.Map;

public interface AIStreamingService {
    Flux<String> generateStreamingResponse(String prompt, Map<String, Object> options);
}
```

创建`service/OpenAIStreamingService.java`实现类：

```java
package org.lijian.speed2text.service.impl;

import org.lijian.speed2text.service.AIStreamingService;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Flux;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Service
public class OpenAIStreamingService implements AIStreamingService {
    
    private final WebClient webClient;
    private final String apiKey;
    private final String model;
    
    public OpenAIStreamingService(
            @Value("${openai.api.url:https://api.openai.com/v1}") String apiUrl,
            @Value("${openai.api.key:your-api-key}") String apiKey,
            @Value("${openai.model:gpt-3.5-turbo}") String model) {
        
        this.webClient = WebClient.builder()
            .baseUrl(apiUrl)
            .build();
        this.apiKey = apiKey;
        this.model = model;
    }
    
    @Override
    public Flux<String> generateStreamingResponse(String prompt, Map<String, Object> options) {
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("model", model);
        requestBody.put("messages", List.of(Map.of("role", "user", "content", prompt)));
        requestBody.put("stream", true);
        
        return webClient.post()
            .uri("/chat/completions")
            .header("Authorization", "Bearer " + apiKey)
            .bodyValue(requestBody)
            .retrieve()
            .bodyToFlux(String.class)
            .map(this::extractContentFromChunk)
            .filter(content -> !content.isEmpty());
    }
    
    private String extractContentFromChunk(String chunk) {
        // 简化版实现，实际应用需要正确解析SSE格式的响应
        if (chunk.contains("content")) {
            int startIndex = chunk.indexOf("\"content\":\"") + 11;
            int endIndex = chunk.indexOf("\"", startIndex);
            if (startIndex > 10 && endIndex > startIndex) {
                return chunk.substring(startIndex, endIndex);
            }
        }
        return "";
    }
}
```

### 4.3 面试服务

创建`service/InterviewService.java`：

```java
package org.lijian.speed2text.service;

import org.lijian.speed2text.model.QuestionRequest;
import reactor.core.publisher.Mono;

public interface InterviewService {
    Mono<Void> processQuestionAsync(String question, String sessionId);
}
```

创建`service/InterviewServiceImpl.java`：

```java
package org.lijian.speed2text.service.impl;

import org.lijian.speed2text.model.QuestionRequest;
import org.lijian.speed2text.service.InterviewService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

@Service
public class InterviewServiceImpl implements InterviewService {
    
    @Autowired
    private StreamingServiceImpl streamingService;
    
    @Override
    public Mono<Void> processQuestionAsync(String question, String sessionId) {
        return Mono.fromRunnable(() -> 
            streamingService.processQuestion(question, sessionId)
        ).subscribeOn(Schedulers.boundedElastic()).then();
    }
}
```

## 5. 前端实现指南

### 5.1 HTML结构

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI面试助手</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <header>
            <h1>AI面试助手</h1>
        </header>
        
        <main>
            <div class="recognition-panel">
                <h2>语音识别</h2>
                <div id="recognition-results" class="results-container"></div>
                <div class="audio-source">
                    <label for="audio-device">音频来源:</label>
                    <select id="audio-device">
                        <option value="">选择音频设备...</option>
                    </select>
                </div>
                <div class="controls">
                    <button id="start-recognition">开始录音</button>
                    <button id="stop-recognition">停止录音</button>
                </div>
            </div>
            
            <div class="ai-panel">
                <h2>AI回答</h2>
                <div id="ai-responses" class="results-container"></div>
            </div>
        </main>
    </div>
    
    <script src="app.js"></script>
</body>
</html>
```

### 5.2 JavaScript实现

创建一个`app.js`文件：

```javascript
// DOM元素
const recognitionResults = document.getElementById('recognition-results');
const aiResponses = document.getElementById('ai-responses');
const startRecognitionBtn = document.getElementById('start-recognition');
const stopRecognitionBtn = document.getElementById('stop-recognition');
const audioDeviceSelect = document.getElementById('audio-device');

// WebSocket连接
let funasrSocket = null;
let backendSocket = null;
let audioContext = null;
let mediaStream = null;
let audioProcessor = null;

// 会话ID
const sessionId = generateSessionId();

// 后端API地址
const BACKEND_API_URL = 'http://localhost:8080/api/interview';

// 初始化
document.addEventListener('DOMContentLoaded', () => {
    // 加载可用的音频设备
    loadAudioDevices();
});

// 加载音频设备
async function loadAudioDevices() {
    try {
        const devices = await navigator.mediaDevices.enumerateDevices();
        const audioInputs = devices.filter(device => device.kind === 'audioinput');
        
        // 清空下拉列表
        audioDeviceSelect.innerHTML = '<option value="">选择音频设备...</option>';
        
        // 添加设备选项
        audioInputs.forEach(device => {
            const option = document.createElement('option');
            option.value = device.deviceId;
            option.text = device.label || `麦克风 ${audioDeviceSelect.length}`;
            
            // 标记虚拟设备
            if (device.label.includes('Line 1') || device.label.includes('CABLE')) {
                option.text += ' (虚拟音频设备)';
                option.selected = true;
            }
            
            audioDeviceSelect.appendChild(option);
        });
    } catch (error) {
        console.error('无法加载音频设备列表:', error);
    }
}

// 连接到FunASR服务
function connectToFunASR() {
    funasrSocket = new WebSocket('ws://127.0.0.1:10096');
    
    funasrSocket.onopen = () => {
        console.log('已连接到FunASR服务');
    };
    
    funasrSocket.onmessage = (event) => {
        const result = JSON.parse(event.data);
        displayRecognitionResult(result);
    };
    
    funasrSocket.onerror = (error) => {
        console.error('FunASR WebSocket错误:', error);
    };
    
    funasrSocket.onclose = () => {
        console.log('FunASR连接已关闭');
    };
    
    return new Promise((resolve, reject) => {
        funasrSocket.onopen = () => resolve();
        funasrSocket.onerror = (error) => reject(error);
    });
}

// 开始音频捕获和处理
async function startAudioCapture() {
    try {
        // 获取选中的音频设备
        const deviceId = audioDeviceSelect.value;
        if (!deviceId) {
            alert('请先选择音频设备');
            return false;
        }
        
        // 配置音频约束
        const constraints = {
            audio: {
                deviceId: { exact: deviceId },
                echoCancellation: false,
                noiseSuppression: false,
                autoGainControl: false
            }
        };
        
        // 获取媒体流
        mediaStream = await navigator.mediaDevices.getUserMedia(constraints);
        
        // 创建音频上下文
        audioContext = new (window.AudioContext || window.webkitAudioContext)();
        const source = audioContext.createMediaStreamSource(mediaStream);
        
        // 创建处理器节点
        audioProcessor = audioContext.createScriptProcessor(4096, 1, 1);
        
        // 处理音频数据
        audioProcessor.onaudioprocess = (e) => {
            const audioData = e.inputBuffer.getChannelData(0);
            if (funasrSocket && funasrSocket.readyState === WebSocket.OPEN) {
                // 将Float32Array转换为Int16Array (适合语音识别)
                const intData = new Int16Array(audioData.length);
                for (let i = 0; i < audioData.length; i++) {
                    intData[i] = audioData[i] * 32767;
                }
                funasrSocket.send(intData.buffer);
            }
        };
        
        // 连接节点
        source.connect(audioProcessor);
        audioProcessor.connect(audioContext.destination);
        
        console.log('音频捕获已开始');
        return true;
    } catch (error) {
        console.error('无法开始音频捕获:', error);
        alert(`音频捕获失败: ${error.message}`);
        return false;
    }
}

// 停止音频捕获
function stopAudioCapture() {
    if (mediaStream) {
        mediaStream.getTracks().forEach(track => track.stop());
    }
    
    if (audioProcessor) {
        audioProcessor.disconnect();
    }
    
    if (audioContext) {
        audioContext.close();
    }
    
    console.log('音频捕获已停止');
}

// 显示识别结果
function displayRecognitionResult(result) {
    // 检查是否有现有元素需要更新（非最终结果）
    const existingElements = document.querySelectorAll('.result-item:not(.final)');
    
    if (!result.isFinal && existingElements.length > 0) {
        // 更新最后一个非最终结果
        const lastElement = existingElements[existingElements.length - 1];
        lastElement.textContent = result.text;
    } else {
        // 创建新元素
        const resultElement = document.createElement('div');
        resultElement.className = result.isFinal ? 'result-item final' : 'result-item';
        resultElement.textContent = result.text;
        
        // 为最终结果添加点击发送功能
        if (result.isFinal) {
            resultElement.addEventListener('click', () => sendToAi(result.text));
        }
        
        recognitionResults.appendChild(resultElement);
        recognitionResults.scrollTop = recognitionResults.scrollHeight;
    }
}

// 将文本发送到AI服务
async function sendToAi(text) {
    try {
        // 创建请求体
        const requestBody = {
            question: text,
            sessionId: sessionId
        };
        
        // 显示等待状态
        const waitingElement = document.createElement('div');
        waitingElement.className = 'response-item waiting';
        waitingElement.textContent = '正在生成回答...';
        aiResponses.appendChild(waitingElement);
        
        // 发送请求
        const response = await fetch(`${BACKEND_API_URL}/question`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(requestBody)
        });
        
        // 检查响应
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        // 解析响应
        const data = await response.json();
        
        // 移除等待元素
        waitingElement.remove();
        
        // 显示初始空回答元素
        const responseElement = document.createElement('div');
        responseElement.className = 'response-item streaming';
        responseElement.textContent = '';
        responseElement.dataset.sessionId = data.sessionId;
        aiResponses.appendChild(responseElement);
        
        // 连接WebSocket获取流式回答
        connectToStreamingResponse(data.streamUrl, responseElement);
    } catch (error) {
        console.error('发送到AI服务失败:', error);
        
        // 显示错误信息
        const errorElement = document.createElement('div');
        errorElement.className = 'response-item error';
        errorElement.textContent = `错误: ${error.message}`;
        aiResponses.appendChild(errorElement);
    }
}

// 连接到流式回答WebSocket
function connectToStreamingResponse(url, responseElement) {
    // 构建完整WebSocket URL
    const wsProtocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
    const host = window.location.host;
    const fullUrl = `${wsProtocol}//${host}${url}`;
    
    // 创建WebSocket连接
    const socket = new WebSocket(fullUrl);
    
    socket.onopen = () => {
        console.log('已连接到流式回答服务');
    };
    
    socket.onmessage = (event) => {
        try {
            const chunk = JSON.parse(event.data);
            
            // 追加内容
            responseElement.textContent += chunk.content;
            
            // 滚动到底部
            aiResponses.scrollTop = aiResponses.scrollHeight;
            
            // 如果是最终消息，更新样式
            if (chunk.isFinal) {
                responseElement.classList.remove('streaming');
                responseElement.classList.add('complete');
                socket.close();
            }
        } catch (error) {
            console.error('解析响应失败:', error);
        }
    };
    
    socket.onerror = (error) => {
        console.error('流式回答WebSocket错误:', error);
        responseElement.classList.add('error');
        responseElement.textContent += '\n[连接错误]';
    };
    
    socket.onclose = () => {
        console.log('流式回答连接已关闭');
        responseElement.classList.remove('streaming');
        
        // 如果没有收到最终消息但连接关闭，标记为错误
        if (responseElement.classList.contains('streaming')) {
            responseElement.classList.add('error');
            responseElement.textContent += '\n[连接已关闭]';
        }
    };
}

// 生成会话ID
function generateSessionId() {
    return 'session-' + Date.now() + '-' + Math.floor(Math.random() * 1000);
}

// 事件监听
startRecognitionBtn.addEventListener('click', async () => {
    try {
        // 禁用按钮，防止重复点击
        startRecognitionBtn.disabled = true;
        
        // 连接FunASR
        await connectToFunASR();
        
        // 开始音频捕获
        const success = await startAudioCapture();
        
        if (success) {
            // 更新UI状态
            startRecognitionBtn.disabled = true;
            stopRecognitionBtn.disabled = false;
            audioDeviceSelect.disabled = true;
        } else {
            // 恢复按钮状态
            startRecognitionBtn.disabled = false;
        }
    } catch (error) {
        console.error('启动失败:', error);
        alert('无法连接到FunASR服务，请确保服务正在运行');
        
        // 恢复按钮状态
        startRecognitionBtn.disabled = false;
    }
});

stopRecognitionBtn.addEventListener('click', () => {
    // 停止音频捕获
    stopAudioCapture();
    
    // 关闭FunASR连接
    if (funasrSocket) {
        funasrSocket.close();
    }
    
    // 更新UI状态
    startRecognitionBtn.disabled = false;
    stopRecognitionBtn.disabled = true;
    audioDeviceSelect.disabled = false;
});

// 页面卸载时关闭连接
window.addEventListener('beforeunload', () => {
    stopAudioCapture();
    
    if (funasrSocket) {
        funasrSocket.close();
    }
});
```

### 5.3 CSS样式

创建一个`styles.css`文件：

```css
* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    line-height: 1.6;
    color: #333;
    background-color: #f8f9fa;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
}

header {
    text-align: center;
    margin-bottom: 30px;
}

h1 {
    color: #0066cc;
}

main {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
}

h2 {
    margin-bottom: 15px;
    color: #444;
    border-bottom: 1px solid #ddd;
    padding-bottom: 5px;
}

.recognition-panel, .ai-panel {
    background-color: white;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    padding: 20px;
    height: 600px;
    display: flex;
    flex-direction: column;
}

.results-container {
    flex: 1;
    overflow-y: auto;
    background-color: #f5f5f5;
    border-radius: 5px;
    padding: 15px;
    margin-bottom: 15px;
}

.audio-source {
    margin-bottom: 15px;
}

.audio-source select {
    padding: 8px;
    border-radius: 5px;
    border: 1px solid #ddd;
    width: 100%;
    font-size: 14px;
}

.controls {
    display: flex;
    gap: 10px;
}

button {
    padding: 10px 15px;
    background-color: #0066cc;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    font-size: 14px;
    flex: 1;
    transition: background-color 0.2s;
}

button:hover {
    background-color: #0055aa;
}

button:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
}

#stop-recognition {
    background-color: #dc3545;
}

#stop-recognition:hover {
    background-color: #c82333;
}

#stop-recognition:disabled {
    background-color: #cccccc;
}

.result-item, .response-item {
    background-color: white;
    border-radius: 5px;
    padding: 10px;
    margin-bottom: 10px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.result-item {
    border-left: 3px solid #0066cc;
}

.result-item.final {
    border-left: 3px solid #28a745;
    cursor: pointer;
}

.result-item.final:hover {
    background-color: #f0f0f0;
}

.response-item {
    border-left: 3px solid #6610f2;
    white-space: pre-wrap;
}

.response-item.streaming {
    border-left: 3px solid #fd7e14;
}

.response-item.complete {
    border-left: 3px solid #28a745;
}

.response-item.waiting {
    border-left: 3px solid #6c757d;
    font-style: italic;
}

.response-item.error {
    border-left: 3px solid #dc3545;
    color: #dc3545;
}

/* 响应式设计 */
@media (max-width: 768px) {
    main {
        grid-template-columns: 1fr;
    }
    
    .recognition-panel, .ai-panel {
        height: 500px;
    }
}
```

## 6. 音频驱动配置指南

### 6.1 虚拟音频设备配置

为了捕获Windows系统声音，需要使用虚拟音频设备：

1. **下载并安装虚拟音频设备驱动**
   - 下载和安装 VB-Cable: https://vb-audio.com/Cable/
   - 或者 CABLE Input 和 Line 1: https://vb-audio.com/Cable/VBCABLE_Driver_Pack43.zip

2. **配置Windows声音设置**

   a. **设置虚拟音频设备为录制设备**
   - 右键点击Windows任务栏中的声音图标，选择"声音设置"
   - 点击"声音控制面板"
   - 选择"录制"选项卡
   - 确保"Line 1"或"CABLE Output"已经启用
   - 如果看不到设备，右键单击空白区域，选择"显示已禁用的设备"

   b. **设置音频播放设备**
   - 选择"播放"选项卡
   - 右键点击"CABLE Input"或虚拟音频设备，设置为"默认设备"
   - 同时保留原有的物理扬声器设备，以便同时听到声音

3. **启用立体声混音**
   - 在"录制"选项卡中，找到"立体声混音"
   - 右键点击它，选择"启用"
   - 选择它作为默认设备，这样就可以捕获系统声音

### 6.2 音频设备选择逻辑

在前端应用中，应该实现逻辑来检测和推荐合适的音频捕获设备：

```javascript
async function getRecommendedAudioDevice() {
    const devices = await navigator.mediaDevices.enumerateDevices();
    const audioInputs = devices.filter(device => device.kind === 'audioinput');
    
    // 优先级顺序：CABLE Out > Line 1 > 立体声混音 > 默认麦克风
    const preferredDevices = [
        'CABLE Output',
        'Line 1',
        'Stereo Mix',
        'CABLE'
    ];
    
    // 查找匹配的设备
    for (const preferredName of preferredDevices) {
        const device = audioInputs.find(
            d => d.label && d.label.includes(preferredName)
        );
        if (device) {
            return device.deviceId;
        }
    }
    
    // 如果没有找到首选设备，返回第一个可用的麦克风
    return audioInputs.length > 0 ? audioInputs[0].deviceId : '';
}
```

## 7. 部署与运行

### 7.1 编译项目

```bash
mvn clean package
```

### 7.2 运行FunASR服务

可以使用Docker运行FunASR服务：

```bash
docker run -d -p 10095:10095 -p 10096:10096 -p 9988:9988 registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:runtime-sdk-online-en-gpu-0.2.2
```

或使用CPU版本：

```bash
docker run -d -p 10095:10095 -p 10096:10096 -p 9988:9988 registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:runtime-sdk-online-en-cpu-0.2.2
```

### 7.3 运行后端应用

```bash
java -jar target/speed2text-0.0.1-SNAPSHOT.jar
```

### 7.4 访问应用

在浏览器中访问：http://localhost:8080

## 8. 注意事项与最佳实践

1. **音频捕获**:
   - 确保用户授予浏览器音频访问权限
   - 考虑添加音频电平可视化，以便用户确认音频捕获正常工作
   - 提供详细的虚拟音频设备设置指南

2. **WebSocket连接管理**:
   - 实现断线重连机制
   - 提供连接状态指示器

3. **流式响应**:
   - 显示正在生成的动画或指示器
   - 考虑实现打字机效果，逐字显示回答

4. **安全考虑**:
   - 在生产环境中使用HTTPS和WSS
   - 实现适当的认证和授权机制
   - 限制API请求频率

5. **用户体验**:
   - 提供清晰的错误消息和恢复指导
   - 保存历史记录，便于用户回顾
   - 考虑添加导出功能，允许用户保存会话内容 