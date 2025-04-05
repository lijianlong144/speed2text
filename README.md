# AI面试助手 (Speed2Text)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Java Version: 17](https://img.shields.io/badge/Java-17-orange)](https://openjdk.java.net/projects/jdk/17/)
[![Spring Boot: 3.x](https://img.shields.io/badge/Spring%20Boot-3.x-green)](https://spring.io/projects/spring-boot)
[![WebFlux](https://img.shields.io/badge/WebFlux-Reactive-blue)](https://docs.spring.io/spring-framework/reference/web/webflux.html)

## 项目概述

AI面试助手是一款创新的应用程序，利用实时语音识别技术将面试对话转换为文本，并通过强大的AI服务提供面试问题的智能回答。该应用采用前端直连FunASR服务处理音频识别，用户手动选择识别文本并发送，后端通过兼容OpenAI格式的API获取流式回答，为用户提供流畅的面试练习体验。

## 主要功能

- **实时语音识别**：前端直接与FunASR服务连接，支持Windows系统声音和麦克风输入，实时转换为文本
- **文本选择发送**：用户手动选择感兴趣的识别文本，点击发送按钮发送给后端处理
- **AI回答生成**：后端通过兼容OpenAI格式的API获取流式回答，实现打字机效果实时显示
- **响应式架构**：采用Spring WebFlux和Project Reactor构建的响应式后端服务
- **可配置提示词**：所有提示词和API参数通过application.yml配置，无需修改代码

## 系统架构

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

### 主要组件

1. **音频捕获子系统**：在前端实现，枚举所有音频设备供用户选择，支持捕获系统声音和麦克风输入
2. **前端WebSocket客户端**：直接连接FunASR服务，发送音频数据并接收识别结果
3. **FunASR服务**：提供实时语音识别功能
4. **后端服务**：接收用户选择的文本，与兼容OpenAI格式的API通信，通过WebSocket返回流式回答
5. **AI服务**：通过兼容OpenAI格式的API提供智能回答生成，支持流式输出

## 技术栈

### 后端

- Java 17
- Spring Boot 3.x
- Spring WebFlux
- Project Reactor
- WebSocket
- YAML配置

### 前端

- HTML5 Web Audio API
- WebSocket API
- JavaScript ES6+
- REST API
- 响应式设计

## 快速开始

### 前提条件

1. JDK 17 或更高版本
2. Maven 3.6+ 或 Gradle 7.0+
3. Docker（用于运行FunASR服务）
4. 虚拟音频驱动（如VB-Cable或其他）

### 环境设置

#### 1. 安装虚拟音频驱动

下载并安装适合您系统的虚拟音频驱动，例如VB-Cable：
- [VB-Cable](https://vb-audio.com/Cable/)

#### 2. 运行FunASR服务

使用Docker运行FunASR服务：

```bash
docker run -d -p 10095:10095 -p 10096:10096 -p 9988:9988 registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:runtime-sdk-online-en-cpu-0.2.2
```

#### 3. 配置application.yml

编辑`src/main/resources/application.yml`文件，配置API参数和提示词：

```yaml
ai:
  api:
    url: https://api.example.com/v1
    key: your-api-key-here
    model: gpt-3.5-turbo
  system:
    prompt: 你是一个专业的面试助手，提供有帮助、简洁和有见地的回答。
```

#### 4. 构建应用

```bash
mvn clean package
```

#### 5. 运行应用

```bash
java -jar target/speed2text-0.0.1-SNAPSHOT.jar
```

#### 6. 访问应用

在浏览器中打开 http://localhost:8080

### 使用指南

1. **选择音频设备**：从下拉菜单中选择合适的音频设备（如系统声音或麦克风）
2. **开始录音**：点击"开始录音"按钮，系统将开始捕获音频并进行实时识别
3. **查看识别结果**：识别的文本将显示在左侧面板中
4. **选择感兴趣的文本**：点击任何最终识别结果，该结果将被高亮显示
5. **发送到AI**：点击"发送到AI"按钮，将选中的文本发送到后端处理
6. **流式回答**：AI的回答将以打字机效果在右侧面板中实时显示
7. **停止录音**：完成后点击"停止录音"按钮

## 开发文档

详细的开发文档位于`docs`目录：

1. [任务规划](docs/task_planning.md)
2. [系统架构](docs/system_architecture.md)
3. [实现指南](docs/implementation_guide.md)
4. [UI设计](docs/ui_design.md)
5. [AI集成](docs/ai_integration.md)

## 项目结构

```
speed2text/
├── src/
│   ├── main/
│   │   ├── java/org/lijian/speed2text/
│   │   │   ├── config/       # 配置类
│   │   │   ├── controller/   # REST控制器
│   │   │   ├── handler/      # WebSocket处理器
│   │   │   ├── model/        # 数据模型
│   │   │   ├── service/      # 服务接口和实现
│   │   │   ├── util/         # 工具类
│   │   │   └── Speed2textApplication.java
│   │   └── resources/
│   │       ├── static/       # 前端资源
│   │       └── application.properties
│   └── test/                 # 测试代码
├── docs/                     # 文档
│   ├── images/               # 架构图和其他图片
│   ├── task_planning.md
│   ├── system_architecture.md
│   ├── implementation_guide.md
│   ├── ui_design.md
│   └── ai_integration.md
├── pom.xml                   # Maven配置
└── README.md                 # 项目说明
```

## 贡献指南

1. Fork 这个仓库
2. 创建您的功能分支 (`git checkout -b feature/amazing-feature`)
3. 提交您的更改 (`git commit -m 'Add some amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 创建一个Pull Request

## 许可证

该项目采用 MIT 许可证 - 详情请查看 [LICENSE](LICENSE) 文件

## 联系方式

项目维护者：[您的姓名](mailto:your.email@example.com)

项目链接：[https://github.com/yourusername/speed2text](https://github.com/yourusername/speed2text)

## 致谢

- [FunASR](https://github.com/alibaba-damo-academy/FunASR) - 提供语音识别服务
- [Spring Framework](https://spring.io/) - 后端框架
- [Project Reactor](https://projectreactor.io/) - 响应式编程库 