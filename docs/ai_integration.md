# AI服务集成指南

## 1. 概述

本文档提供了将AI面试助手系统与各种AI服务集成的指南。系统设计为可插拔式架构，允许轻松替换或添加不同的AI服务提供商。本系统使用前端直连FunASR服务和流式AI输出来提供更好的用户体验。

## 2. 支持的AI服务类型

系统可以集成以下类型的AI服务：

1. **文本生成服务** - 用于生成AI回答（支持流式输出）
2. **语音识别服务** - 在前端直接连接FunASR服务
3. **语音合成服务** - 用于将AI回答转换为语音（未来扩展）

## 3. AI服务接口

### 3.1 通用接口设计

系统使用统一的接口设计来与各种AI服务交互：

```java
// 标准响应接口
public interface AIService {
    Mono<String> generateResponse(String prompt, Map<String, Object> options);
}

// 流式响应接口
public interface AIStreamingService {
    Flux<String> generateStreamingResponse(String prompt, Map<String, Object> options);
}
```

### 3.2 抽象实现类

```java
public abstract class AbstractAIStreamingService implements AIStreamingService {
    protected final WebClient webClient;
    protected final String apiEndpoint;
    protected final String apiKey;
    
    protected AbstractAIStreamingService(String apiEndpoint, String apiKey) {
        this.apiEndpoint = apiEndpoint;
        this.apiKey = apiKey;
        this.webClient = WebClient.builder()
            .baseUrl(apiEndpoint)
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
    
    // 为兼容性提供的非流式方法
    public Mono<String> generateResponse(String prompt, Map<String, Object> options) {
        return generateStreamingResponse(prompt, options)
            .collectList()
            .map(chunks -> String.join("", chunks));
    }
}
```

## 4. OpenAI流式服务集成

### 4.1 配置

在`application.properties`或`application.yml`中添加以下配置：

```properties
# OpenAI配置
openai.api.key=your-api-key-here
openai.api.endpoint=https://api.openai.com/v1
openai.api.model=gpt-3.5-turbo
openai.api.temperature=0.7
openai.api.max-tokens=800
```

### 4.2 实现类

```java
package org.lijian.speed2text.service.ai;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Service
public class OpenAIStreamingService extends AbstractAIStreamingService {

    private final String model;
    private final double temperature;
    private final int maxTokens;
    private final ObjectMapper objectMapper;

    public OpenAIStreamingService(
            @Value("${openai.api.endpoint}") String apiEndpoint,
            @Value("${openai.api.key}") String apiKey,
            @Value("${openai.api.model}") String model,
            @Value("${openai.api.temperature}") double temperature,
            @Value("${openai.api.max-tokens}") int maxTokens) {
        super(apiEndpoint, apiKey);
        this.model = model;
        this.temperature = temperature;
        this.maxTokens = maxTokens;
        this.objectMapper = new ObjectMapper();
    }

    @Override
    public Flux<String> generateStreamingResponse(String prompt, Map<String, Object> options) {
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("model", options.getOrDefault("model", model));
        
        List<Map<String, String>> messages = new ArrayList<>();
        messages.add(Map.of("role", "system", "content", "你是一个专业的面试助手，提供有帮助、简洁和有见地的回答。"));
        messages.add(Map.of("role", "user", "content", prompt));
        requestBody.put("messages", messages);
        
        requestBody.put("temperature", options.getOrDefault("temperature", temperature));
        requestBody.put("max_tokens", options.getOrDefault("max_tokens", maxTokens));
        requestBody.put("stream", true);

        return webClient.post()
            .uri("/chat/completions")
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + apiKey)
            .accept(MediaType.TEXT_EVENT_STREAM)
            .bodyValue(requestBody)
            .retrieve()
            .bodyToFlux(String.class)
            .map(this::extractStreamContent)
            .filter(content -> !content.isEmpty())
            .onErrorResume(e -> {
                System.err.println("OpenAI API error: " + e.getMessage());
                return Flux.just("抱歉，我无法生成回答。请稍后再试。");
            });
    }
    
    private String extractStreamContent(String chunk) {
        try {
            if (chunk.startsWith("data: ")) {
                chunk = chunk.substring(6);
            }
            
            if (chunk.equals("[DONE]")) {
                return "";
            }
            
            JsonNode jsonNode = objectMapper.readTree(chunk);
            JsonNode choices = jsonNode.get("choices");
            
            if (choices != null && choices.isArray() && choices.size() > 0) {
                JsonNode delta = choices.get(0).get("delta");
                if (delta != null && delta.has("content")) {
                    return delta.get("content").asText();
                }
            }
            
            return "";
        } catch (JsonProcessingException e) {
            return "";
        }
    }
}
```

## 5. 百度文心一言流式集成

### 5.1 配置

在`application.properties`中添加：

```properties
# 百度文心一言配置
baidu.api.appId=your-app-id-here
baidu.api.apiKey=your-api-key-here
baidu.api.secretKey=your-secret-key-here
baidu.api.endpoint=https://aip.baidubce.com/rpc/2.0/ai_custom/v1/wenxinworkshop/chat
baidu.api.model=ernie-bot
```

### 5.2 实现类

```java
package org.lijian.speed2text.service.ai;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.HashMap;
import java.util.Map;

@Service
public class BaiduAIStreamingService extends AbstractAIStreamingService {

    private final String appId;
    private final String secretKey;
    private final String model;
    private final ObjectMapper objectMapper;

    public BaiduAIStreamingService(
            @Value("${baidu.api.endpoint}") String apiEndpoint,
            @Value("${baidu.api.apiKey}") String apiKey,
            @Value("${baidu.api.appId}") String appId,
            @Value("${baidu.api.secretKey}") String secretKey,
            @Value("${baidu.api.model}") String model) {
        super(apiEndpoint, apiKey);
        this.appId = appId;
        this.secretKey = secretKey;
        this.model = model;
        this.objectMapper = new ObjectMapper();
    }
    
    // 获取访问令牌
    private Mono<String> getAccessToken() {
        return WebClient.create("https://aip.baidubce.com")
            .post()
            .uri(uriBuilder -> uriBuilder
                .path("/oauth/2.0/token")
                .queryParam("grant_type", "client_credentials")
                .queryParam("client_id", apiKey)
                .queryParam("client_secret", secretKey)
                .build())
            .retrieve()
            .bodyToMono(String.class)
            .map(response -> {
                try {
                    JsonNode rootNode = objectMapper.readTree(response);
                    return rootNode.path("access_token").asText();
                } catch (Exception e) {
                    throw new RuntimeException("Failed to parse access token response", e);
                }
            });
    }

    @Override
    public Flux<String> generateStreamingResponse(String prompt, Map<String, Object> options) {
        return getAccessToken()
            .flatMapMany(accessToken -> {
                Map<String, Object> requestBody = new HashMap<>();
                requestBody.put("messages", Map.of(
                    "role", "user", 
                    "content", prompt
                ));
                requestBody.put("stream", true);
                
                String modelName = (String) options.getOrDefault("model", model);
                
                return webClient.post()
                    .uri(uriBuilder -> uriBuilder
                        .path("/" + modelName)
                        .queryParam("access_token", accessToken)
                        .build())
                    .bodyValue(requestBody)
                    .accept(MediaType.TEXT_EVENT_STREAM)
                    .retrieve()
                    .bodyToFlux(String.class)
                    .map(this::extractStreamContent)
                    .filter(content -> !content.isEmpty())
                    .onErrorResume(e -> {
                        System.err.println("Baidu AI API error: " + e.getMessage());
                        return Flux.just("抱歉，我无法生成回答。请稍后再试。");
                    });
            });
    }
    
    private String extractStreamContent(String chunk) {
        try {
            if (chunk.startsWith("data: ")) {
                chunk = chunk.substring(6);
            }
            
            JsonNode jsonNode = objectMapper.readTree(chunk);
            if (jsonNode.has("result")) {
                return jsonNode.get("result").asText();
            }
            
            return "";
        } catch (JsonProcessingException e) {
            return "";
        }
    }
}
```

## 6. 本地大模型流式集成

对于希望在本地部署和使用AI模型的场景，系统也支持集成本地大模型的流式输出。

### 6.1 本地模型服务配置

```properties
# 本地模型配置
local.model.api.endpoint=http://localhost:8000/v1
local.model.api.key=not-needed
local.model.name=llama3
```

### 6.2 实现类

```java
package org.lijian.speed2text.service.ai;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Service
public class LocalModelStreamingService extends AbstractAIStreamingService {

    private final String modelName;
    private final ObjectMapper objectMapper;

    public LocalModelStreamingService(
            @Value("${local.model.api.endpoint}") String apiEndpoint,
            @Value("${local.model.api.key}") String apiKey,
            @Value("${local.model.name}") String modelName) {
        super(apiEndpoint, apiKey);
        this.modelName = modelName;
        this.objectMapper = new ObjectMapper();
    }

    @Override
    public Flux<String> generateStreamingResponse(String prompt, Map<String, Object> options) {
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("model", options.getOrDefault("model", modelName));
        
        List<Map<String, String>> messages = new ArrayList<>();
        messages.add(Map.of("role", "system", "content", "你是一个专业的面试助手，提供有帮助、简洁和有见地的回答。"));
        messages.add(Map.of("role", "user", "content", prompt));
        requestBody.put("messages", messages);
        
        requestBody.put("temperature", options.getOrDefault("temperature", 0.7));
        requestBody.put("max_tokens", options.getOrDefault("max_tokens", 800));
        requestBody.put("stream", true);

        return webClient.post()
            .uri("/chat/completions")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .bodyValue(requestBody)
            .retrieve()
            .bodyToFlux(String.class)
            .map(this::extractStreamContent)
            .filter(content -> !content.isEmpty())
            .onErrorResume(e -> {
                System.err.println("Local model API error: " + e.getMessage());
                return Flux.just("抱歉，我无法生成回答。请稍后再试。");
            });
    }
    
    private String extractStreamContent(String chunk) {
        try {
            if (chunk.startsWith("data: ")) {
                chunk = chunk.substring(6);
            }
            
            if (chunk.equals("[DONE]")) {
                return "";
            }
            
            JsonNode jsonNode = objectMapper.readTree(chunk);
            JsonNode choices = jsonNode.get("choices");
            
            if (choices != null && choices.isArray() && choices.size() > 0) {
                JsonNode delta = choices.get(0).get("delta");
                if (delta != null && delta.has("content")) {
                    return delta.get("content").asText();
                }
            }
            
            return "";
        } catch (JsonProcessingException e) {
            return "";
        }
    }
}
```

## 7. AI服务工厂

为了支持动态切换AI服务提供商，我们实现一个AI服务工厂类：

```java
package org.lijian.speed2text.service.ai;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

@Component
public class AIStreamingServiceFactory {
    
    private final Map<String, AIStreamingService> services = new HashMap<>();
    
    @Value("${ai.service.default:openai}")
    private String defaultService;
    
    @Autowired
    public AIStreamingServiceFactory(
            OpenAIStreamingService openAiService, 
            BaiduAIStreamingService baiduAIService, 
            LocalModelStreamingService localModelService) {
        services.put("openai", openAiService);
        services.put("baidu", baiduAIService);
        services.put("local", localModelService);
    }
    
    public AIStreamingService getService(String serviceName) {
        return services.getOrDefault(serviceName, services.get(defaultService));
    }
    
    public AIStreamingService getDefaultService() {
        return services.get(defaultService);
    }
}
```

## 8. 流式服务接口

实现流式处理服务接口用于管理流式响应：

```java
package org.lijian.speed2text.service;

import org.lijian.speed2text.model.StreamChunk;
import reactor.core.publisher.Flux;

public interface StreamingService {
    Flux<StreamChunk> getAnswerStream(String sessionId);
    void processQuestion(String question, String sessionId);
}
```

```java
package org.lijian.speed2text.service.impl;

import org.lijian.speed2text.model.StreamChunk;
import org.lijian.speed2text.service.AIStreamingService;
import org.lijian.speed2text.service.AIStreamingServiceFactory;
import org.lijian.speed2text.service.StreamingService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Sinks;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class StreamingServiceImpl implements StreamingService {
    
    @Autowired
    private AIStreamingServiceFactory aiServiceFactory;
    
    @Value("${ai.service.default:openai}")
    private String defaultService;
    
    private final Map<String, Sinks.Many<StreamChunk>> sessionSinks = new ConcurrentHashMap<>();
    
    @Override
    public Flux<StreamChunk> getAnswerStream(String sessionId) {
        Sinks.Many<StreamChunk> sink = sessionSinks.computeIfAbsent(
            sessionId, 
            k -> Sinks.many().multicast().onBackpressureBuffer()
        );
        
        return sink.asFlux();
    }
    
    @Override
    public void processQuestion(String question, String sessionId) {
        AIStreamingService aiService = aiServiceFactory.getDefaultService();
        Sinks.Many<StreamChunk> sink = sessionSinks.get(sessionId);
        
        if (sink != null) {
            aiService.generateStreamingResponse(question, Map.of())
                .doOnNext(content -> {
                    StreamChunk chunk = new StreamChunk("chunk", content, sessionId, false);
                    sink.tryEmitNext(chunk);
                })
                .doOnComplete(() -> {
                    StreamChunk finalChunk = new StreamChunk("complete", "", sessionId, true);
                    sink.tryEmitNext(finalChunk);
                    sink.tryEmitComplete();
                    sessionSinks.remove(sessionId);
                })
                .doOnError(error -> {
                    sink.tryEmitError(error);
                    sessionSinks.remove(sessionId);
                })
                .subscribe();
        }
    }
}
```

## 9. 前端与FunASR集成

前端应用直接与FunASR服务建立WebSocket连接，处理音频捕获和语音识别。

### 9.1 前端FunASR连接

```javascript
// 连接到FunASR服务
function connectToFunASR() {
    const funasrSocket = new WebSocket('ws://127.0.0.1:10096');
    
    funasrSocket.onopen = () => {
        console.log('已连接到FunASR服务');
    };
    
    funasrSocket.onmessage = (event) => {
        try {
            const result = JSON.parse(event.data);
            // 处理识别结果
            displayRecognitionResult(result);
        } catch (error) {
            console.error('解析FunASR响应失败:', error);
        }
    };
    
    funasrSocket.onclose = () => {
        console.log('FunASR连接已关闭');
    };
    
    funasrSocket.onerror = (error) => {
        console.error('FunASR WebSocket错误:', error);
    };
    
    return funasrSocket;
}

// 发送音频数据到FunASR
function sendAudioToFunASR(audioData, socket) {
    if (socket && socket.readyState === WebSocket.OPEN) {
        socket.send(audioData);
    }
}
```

### 9.2 前端调用后端REST API

```javascript
// 发送识别结果到后端
async function sendRecognitionToBackend(text) {
    try {
        const response = await fetch('/api/interview/question', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                question: text,
                sessionId: generateSessionId()
            })
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const data = await response.json();
        return data.streamUrl; // 返回WebSocket流地址
    } catch (error) {
        console.error('发送到后端失败:', error);
        throw error;
    }
}
```

### 9.3 前端接收流式回答

```javascript
// 连接到流式回答WebSocket
function connectToStreamingResponse(url, responseElement) {
    const socket = new WebSocket(url);
    
    socket.onopen = () => {
        console.log('已连接到流式回答服务');
    };
    
    socket.onmessage = (event) => {
        try {
            const chunk = JSON.parse(event.data);
            
            // 追加内容
            responseElement.textContent += chunk.content;
            
            // 如果是最终消息，更新状态
            if (chunk.isFinal) {
                responseElement.classList.remove('streaming');
                responseElement.classList.add('complete');
                socket.close();
            }
        } catch (error) {
            console.error('解析流式回答失败:', error);
        }
    };
    
    socket.onerror = (error) => {
        console.error('流式回答WebSocket错误:', error);
    };
    
    return socket;
}
```

## 10. 提示词工程

为了获得更好的面试回答质量，需要为不同的AI服务提供适当的提示词模板。

### 10.1 提示词模板服务

```java
package org.lijian.speed2text.service;

import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;

@Service
public class PromptTemplateService {
    
    private final Map<String, String> templates = new HashMap<>();
    
    public PromptTemplateService() {
        // 初始化模板
        templates.put("interview_question", "你是一个专业的面试教练。请针对以下面试问题，提供一个专业、简洁且有深度的回答建议：\n\n{question}");
        templates.put("interview_feedback", "你是一个专业的面试评估专家。请针对以下面试回答，提供专业的反馈和改进建议：\n\n问题：{question}\n回答：{answer}");
    }
    
    public String getTemplate(String templateName) {
        return templates.getOrDefault(templateName, "{input}");
    }
    
    public String formatPrompt(String templateName, Map<String, String> parameters) {
        String template = getTemplate(templateName);
        
        for (Map.Entry<String, String> entry : parameters.entrySet()) {
            template = template.replace("{" + entry.getKey() + "}", entry.getValue());
        }
        
        return template;
    }
}
```

## 11. 错误处理与重试策略

为了提高系统的稳定性，实现错误处理和重试策略：

```java
public Flux<String> generateStreamingResponseWithRetry(String prompt, Map<String, Object> options, int maxRetries) {
    return generateStreamingResponse(prompt, options)
        .retryWhen(Retry.backoff(maxRetries, Duration.ofSeconds(1))
            .filter(e -> !(e instanceof IllegalArgumentException))
            .onRetryExhaustedThrow((retrySpec, retrySignal) -> {
                return new RuntimeException("Failed after max retries", retrySignal.failure());
            }))
        .onErrorResume(e -> Flux.just("抱歉，暂时无法处理您的请求。请稍后再试。"));
}
```

## 12. 安全考虑

实现API密钥的安全管理：

1. 使用环境变量或加密的配置文件存储API密钥
2. 实现API密钥轮换机制
3. 设置调用频率限制和IP限制
4. 在前端和后端之间添加适当的认证机制

```java
@Configuration
@EnableScheduling
public class ApiKeyRotationConfig {
    
    @Autowired
    private OpenAIStreamingService openAIService;
    
    @Scheduled(cron = "0 0 0 * * ?") // 每天午夜执行
    public void rotateApiKeys() {
        // 从安全存储获取新的API密钥
        // 更新服务实例中的API密钥
    }
}
```

## 13. 总结

本文档提供了AI服务流式集成的全面指南，包括：

1. 支持多种AI服务提供商的流式输出
2. 前端直接集成FunASR服务
3. 后端REST API和WebSocket流式输出
4. 提示词工程优化
5. 错误处理与重试策略
6. 安全与监控

通过这些实现，AI面试助手可以提供更流畅的用户体验，实现实时语音识别和流式AI回答。 