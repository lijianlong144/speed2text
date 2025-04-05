# AI面试助手系统架构设计

## 1. 系统概述

AI面试助手是一个基于实时语音识别技术的应用，旨在帮助用户进行面试练习和反馈。系统能够实时捕获用户的语音输入，将其转换为文本，并通过AI服务提供相应的面试问题解答和建议。系统采用前端直连FunASR、用户手动选择文本发送、后端调用兼容OpenAI格式API的架构设计。

## 2. 系统组件设计

### 2.1 整体架构图

```
┌─────────────────────────┐    ┌─────────────────────────┐
│                         │    │                         │
│   前端应用              │    │    FunASR服务           │
│  (音频捕获+WebSocket客户端)│◄───►    (WebSocket服务端)    │
│                         │    │                         │
└──────────┬──────────────┘    └─────────────────────────┘
           │
           │ HTTP REST API（用户选择文本手动发送）
           ▼
┌─────────────────────────┐    ┌─────────────────────────┐
│                         │    │                         │
│   后端服务              │    │   AI服务                │
│   (REST API+WebSocket)  │◄───►   (兼容OpenAI格式API)   │
│                         │    │                         │
└─────────────────────────┘    └─────────────────────────┘
```

### 2.2 组件说明

#### 2.2.1 前端应用
- **功能**: 
  - 枚举并显示所有音频设备供用户选择
  - 捕获Windows系统声音和麦克风输入
  - 与FunASR服务建立WebSocket连接进行实时语音识别
  - 显示识别结果供用户选择
  - 提供按钮让用户将选中文本发送到后端
  - 接收AI流式回答
  - 提供用户交互界面
- **技术实现**:
  - HTML5/CSS3/JavaScript
  - 使用Web Audio API枚举音频设备
  - WebSocket API连接FunASR服务
  - Web Audio API处理音频
  - Fetch API或Axios发送REST请求
- **组件**:
  - UI界面组件
  - 音频设备选择组件
  - 音频捕获组件
  - WebSocket客户端组件
  - HTTP客户端组件

#### 2.2.2 FunASR服务
- **功能**: 提供语音识别服务
- **技术实现**:
  - Docker容器化部署
  - WebSocket协议
- **接口**:
  - WebSocket服务端点：ws://127.0.0.1:10096
- **与前端直接通信**:
  - 接收前端发送的音频数据
  - 返回识别结果给前端

#### 2.2.3 后端服务
- **功能**:
  - 提供REST API接收用户选择的文本
  - 使用兼容OpenAI格式的API发送请求
  - 接收AI流式回答
  - 通过WebSocket将流式回答实时返回给前端
- **技术实现**:
  - Spring Boot 3.x
  - WebFlux
  - Project Reactor
- **组件**:
  - REST控制器
  - WebSocket处理器（用于流式回答）
  - AI服务客户端

#### 2.2.4 AI服务
- **功能**: 提供面试问题的智能回答
- **技术实现**:
  - 兼容OpenAI格式的API
  - 支持流式输出（如SSE或WebSocket）
- **接口**:
  - 兼容OpenAI格式的API端点

## 3. 数据流设计

### 3.1 音频采集与识别流程

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│            │    │            │    │            │    │            │    │            │
│  音频源    │───►│ 前端应用   │───►│ 音频编码   │───►│ WebSocket  │───►│  FunASR    │
│            │    │ (用户选择) │    │            │    │  客户端    │    │  服务      │
└────────────┘    └────────────┘    └────────────┘    └────────────┘    └────────────┘
                                                                               │
                                                                               ▼
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│            │    │            │    │            │    │            │    │            │
│  前端UI    │◄───┤ 文本处理   │◄───┤ 前端应用   │◄───┤ WebSocket  │◄───┤  语音识别  │
│  显示      │    │            │    │            │    │  客户端    │    │  结果      │
└────────────┘    └────────────┘    └────────────┘    └────────────┘    └────────────┘
```

### 3.2 文本处理与AI回答流程

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│            │    │            │    │            │    │            │    │            │
│  用户选择  │───►│ 点击发送   │───►│ REST API   │───►│  后端服务  │───►│ 兼容OpenAI │
│  文本      │    │  按钮      │    │  请求      │    │            │    │  格式API   │
└────────────┘    └────────────┘    └────────────┘    └────────────┘    └────────────┘
                                                                               │
                                                                               ▼
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│            │    │            │    │            │    │            │    │            │
│  显示AI    │◄───┤ 前端应用   │◄───┤ WebSocket  │◄───┤  后端服务  │◄───┤  流式回答  │
│  回答      │    │            │    │  客户端    │    │            │    │  数据      │
└────────────┘    └────────────┘    └────────────┘    └────────────┘    └────────────┘
```

## 4. 接口设计

### 4.1 WebSocket接口

#### 4.1.1 前端与FunASR WebSocket连接
- **连接URL**: ws://127.0.0.1:10096
- **消息格式**:
  - 客户端到服务端: 二进制音频数据
  - 服务端到客户端: JSON格式的识别结果
  ```json
  {
    "text": "识别的文本内容",
    "timestamp": 1635739200,
    "final": true|false
  }
  ```

#### 4.1.2 后端流式回答WebSocket连接
- **连接URL**: ws://{backend-host}/api/stream/answer
- **消息格式**:
  - 服务端到客户端: 
  ```json
  {
    "type": "chunk",
    "content": "AI回答内容片段",
    "sessionId": "会话标识",
    "final": false
  }
  ```
  - 最终完成响应:
  ```json
  {
    "type": "complete",
    "content": "完整AI回答内容",
    "sessionId": "会话标识",
    "final": true
  }
  ```

### 4.2 REST API接口

#### 4.2.1 用户将选中文本发送到后端
- **请求方式**: POST
- **URL**: /api/interview/question
- **请求格式**:
```json
{
  "question": "用户选中的文本内容",
  "sessionId": "会话标识"
}
```
- **响应格式**:
```json
{
  "status": "success",
  "message": "请求已接收，请通过WebSocket连接获取流式回答",
  "sessionId": "会话标识",
  "streamUrl": "ws://{backend-host}/api/stream/answer?sessionId={sessionId}"
}
```

### 4.3 后端与AI服务接口（兼容OpenAI格式）

- **请求方式**: POST
- **URL**: 兼容OpenAI格式API的端点
- **请求格式**:
```json
{
  "model": "配置的模型名称",
  "messages": [
    {
      "role": "system", 
      "content": "配置的系统提示词"
    },
    {
      "role": "user", 
      "content": "用户选择的问题"
    }
  ],
  "stream": true
}
```
- **响应格式**: 流式响应，每个数据块包含:
```json
{
  "choices": [
    {
      "delta": {
        "content": "AI回答内容片段"
      }
    }
  ]
}
```

## 5. 数据模型

### 5.1 消息模型

```java
// REST API请求
public record QuestionRequest(String question, String sessionId) {
}

// REST API响应
public record StreamResponse(String status, String message, String sessionId, String streamUrl) {
}

// WebSocket流式回答消息
public record StreamChunk(String type, String content, String sessionId, boolean isFinal) {
}

// 语音识别结果
public record RecognitionResult(String text, long timestamp, boolean isFinal) {
}
```

### 5.2 会话模型

```java
// 用户会话
public record UserSession(String sessionId, String userId, Instant startTime) {
}
```

## 6. 技术实现细节

### 6.1 前端音频设备选择与捕获

```javascript
// 枚举所有音频设备
async function enumerateAudioDevices() {
    const devices = await navigator.mediaDevices.enumerateDevices();
    const audioInputs = devices.filter(device => device.kind === 'audioinput');
    
    // 将设备添加到下拉列表
    const deviceSelector = document.getElementById('audio-device-selector');
    deviceSelector.innerHTML = '';
    
    audioInputs.forEach(device => {
        const option = document.createElement('option');
        option.value = device.deviceId;
        option.text = device.label || `设备 ${device.deviceId.substring(0, 8)}...`;
        deviceSelector.appendChild(option);
    });
    
    return audioInputs;
}

// 捕获音频
async function captureAudio(deviceId) {
    const constraints = {
        audio: {
            deviceId: deviceId ? { exact: deviceId } : undefined,
            echoCancellation: false,
            noiseSuppression: false,
            autoGainControl: false
        }
    };
    
    return navigator.mediaDevices.getUserMedia(constraints);
}
```

### 6.2 用户选择文本并发送

```javascript
// 用户选择文本
function setupTextSelection() {
    const resultsContainer = document.getElementById('recognition-results');
    
    resultsContainer.addEventListener('click', function(event) {
        const resultItem = event.target.closest('.result-item');
        if (resultItem && resultItem.classList.contains('final')) {
            // 移除其他选中项
            document.querySelectorAll('.result-item.selected').forEach(item => {
                item.classList.remove('selected');
            });
            
            // 选中当前项
            resultItem.classList.add('selected');
            
            // 启用发送按钮
            document.getElementById('send-button').disabled = false;
        }
    });
}

// 发送选中文本到后端
async function sendSelectedText() {
    const selectedItem = document.querySelector('.result-item.selected');
    if (!selectedItem) return;
    
    const text = selectedItem.textContent;
    const sessionId = generateSessionId();
    
    try {
        const response = await fetch('/api/interview/question', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                question: text,
                sessionId: sessionId
            })
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const data = await response.json();
        
        // 连接WebSocket获取流式回答
        connectToStreamingResponse(data.streamUrl, createResponseElement());
    } catch (error) {
        console.error('发送文本失败:', error);
    }
}
```

### 6.3 后端REST API控制器

```java
@RestController
@RequestMapping("/api/interview")
public class InterviewController {
    
    @Autowired
    private InterviewService interviewService;
    
    @PostMapping("/question")
    public Mono<StreamResponse> processQuestion(@RequestBody QuestionRequest request) {
        // 启动AI流式处理
        interviewService.processQuestionAsync(request);
        
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

### 6.4 兼容OpenAI格式的API调用

```java
@Service
public class OpenAICompatibleService implements AIStreamingService {
    
    private final WebClient webClient;
    private final String apiUrl;
    private final String apiKey;
    private final String model;
    private final String systemPrompt;
    
    public OpenAICompatibleService(
            @Value("${ai.api.url}") String apiUrl,
            @Value("${ai.api.key}") String apiKey,
            @Value("${ai.api.model}") String model,
            @Value("${ai.system.prompt}") String systemPrompt) {
        this.apiUrl = apiUrl;
        this.apiKey = apiKey;
        this.model = model;
        this.systemPrompt = systemPrompt;
        
        this.webClient = WebClient.builder()
            .baseUrl(apiUrl)
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + apiKey)
            .build();
    }
    
    @Override
    public Flux<String> generateStreamingResponse(String prompt) {
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("model", model);
        
        List<Map<String, String>> messages = new ArrayList<>();
        messages.add(Map.of("role", "system", "content", systemPrompt));
        messages.add(Map.of("role", "user", "content", prompt));
        
        requestBody.put("messages", messages);
        requestBody.put("stream", true);
        
        return webClient.post()
            .uri("/chat/completions")
            .bodyValue(requestBody)
            .retrieve()
            .bodyToFlux(String.class)
            .map(this::extractContent)
            .filter(content -> !content.isEmpty());
    }
    
    private String extractContent(String chunk) {
        // 解析OpenAI兼容格式的响应
        // 实现略
        return content;
    }
}
```

## 7. 配置管理

### 7.1 application.yml配置文件

```yaml
server:
  port: 8080

spring:
  webflux:
    base-path: /
  jackson:
    default-property-inclusion: non_null

ai:
  api:
    url: https://api.example.com/v1
    key: your-api-key-here
    model: gpt-3.5-turbo
  system:
    prompt: 你是一个专业的面试助手，提供有帮助、简洁和有见地的回答。请根据问题提供相关的面试答案建议。

websocket:
  stream:
    path: /api/stream/answer

funasr:
  websocket:
    url: ws://127.0.0.1:10096
```

## 8. 安全考虑

1. **前端安全**:
   - 音频数据本地处理，不经过后端
   - WebSocket连接使用安全检查

2. **API安全**:
   - REST API使用适当的认证和授权
   - 防止CSRF和XSS攻击

3. **AI服务访问控制**:
   - API密钥管理
   - 请求频率限制

## 9. 扩展性考虑

1. **多语言支持**:
   - 支持不同语言的语音识别和AI回答

2. **多AI服务提供商**:
   - 设计可插拔的AI服务接口

3. **用户个性化**:
   - 支持用户自定义提示词和上下文

## 10. 完整工作流程

1. **前端启动**:
   - 枚举所有音频设备并在UI中显示
   - 用户选择合适的音频设备（如Line 1虚拟驱动）

2. **开始识别**:
   - 用户点击"开始录音"按钮
   - 前端捕获选定设备的音频
   - 通过WebSocket发送到FunASR服务
   - FunASR返回识别结果
   - 前端显示识别结果

3. **用户操作**:
   - 用户查看识别结果
   - 用户点击选择感兴趣的文本（问题）
   - 用户点击"发送"按钮

4. **后端处理**:
   - 后端接收用户选择的文本
   - 后端应用预先配置的提示词模板
   - 后端通过兼容OpenAI格式的API发送请求
   - 接收流式回答

5. **返回结果**:
   - 后端通过WebSocket将流式回答实时发送给前端
   - 前端使用打字机效果显示AI回答
   - 当流式回答完成时，前端更新状态 