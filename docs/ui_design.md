# AI面试助手UI设计文档

## 1. 设计概述

AI面试助手的UI设计以简洁、直观和专业为核心理念，旨在为用户提供流畅的面试练习体验。界面将分为两个主要部分：语音识别面板和AI回答面板，通过直观的布局帮助用户专注于面试内容。前端应用负责音频捕获、语音识别和用户交互，采用响应式设计适应不同设备的显示需求。

## 2. 布局设计

### 2.1 主布局结构

```
┌─────────────────────────────────────────────────────────────────┐
│ 页面头部 (Header)                                                │
├─────────────────────────────┬─────────────────────────────────┤
│                             │                                 │
│                             │                                 │
│                             │                                 │
│                             │                                 │
│  语音识别面板               │  AI回答面板                     │
│  (Recognition Panel)        │  (AI Response Panel)            │
│                             │                                 │
│                             │                                 │
│                             │                                 │
│                             │                                 │
├─────────────────────────────┴─────────────────────────────────┤
│ 页面底部 (Footer)                                              │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 组件布局

#### 2.2.1 语音识别面板

```
┌─────────────────────────────────────────────────────────────────┐
│ 语音识别 (标题)                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                         │   │
│  │                                                         │   │
│  │                                                         │   │
│  │  识别结果显示区域                                       │   │
│  │  (scrollable)                                           │   │
│  │                                                         │   │
│  │                                                         │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 音频设备选择下拉框                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────┐            ┌─────────────────────────┐   │
│  │  开始录音       │            │  停止录音               │   │
│  └─────────────────┘            └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

#### 2.2.2 AI回答面板

```
┌─────────────────────────────────────────────────────────────────┐
│ AI回答 (标题)                                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                         │   │
│  │                                                         │   │
│  │                                                         │   │
│  │  回答内容显示区域                                       │   │
│  │  (scrollable)                                           │   │
│  │                                                         │   │
│  │                                                         │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 界面组件

### 3.1 页面头部

- **标题**: "AI面试助手"，使用大号字体和醒目颜色
- **版本信息**: 可选的小号文字显示应用版本
- **主题色**: #0066CC（蓝色）

### 3.2 语音识别面板

#### 3.2.1 识别结果显示区域

- **识别结果项**: 展示语音识别的文本结果
  - 样式区分临时结果和最终结果
  - 最终结果项可点击选择，点击后高亮显示
  - 显示时间戳或序号（可选）
- **选中状态**: 当用户点击某个识别结果时，该结果项高亮显示
- **空状态提示**: 当没有识别结果时显示的提示文本
  - "点击"开始录音"按钮开始识别..."

#### 3.2.2 音频设备选择

- **下拉选择框**: 列出所有可用的音频输入设备
  - 显示系统上所有可用的音频输入设备
  - 让用户自行选择合适的虚拟音频设备
  - 默认选择系统默认设备
  - 设备名称前可添加图标区分麦克风和其他音频设备
- **标签文本**: "音频来源："
- **提示信息**: "请选择系统声音或麦克风对应的音频设备"

#### 3.2.3 控制按钮

- **开始录音按钮**:
  - 文字: "开始录音"
  - 颜色: #0066CC（蓝色）
  - 图标: 麦克风图标
  - 状态: 默认启用，录音开始后禁用
- **停止录音按钮**:
  - 文字: "停止录音"
  - 颜色: #DC3545（红色）
  - 图标: 停止图标
  - 状态: 默认禁用，录音开始后启用
- **发送按钮**:
  - 文字: "发送到AI"
  - 颜色: #28A745（绿色）
  - 图标: 发送图标
  - 状态: 默认禁用，选中文本后启用
  - 位置: 放置在识别结果面板底部

### 3.3 AI回答面板

#### 3.3.1 回答内容显示区域

- **回答项**:
  - 显示AI回答的内容
  - 区分不同状态：等待中、流式生成中、已完成、错误
  - 支持Markdown格式（可选）
  - 自动滚动到最新内容
- **空状态提示**:
  - "点击左侧识别结果将问题发送给AI..."
- **加载指示器**:
  - 等待AI回答时显示旋转加载图标或进度条
- **打字机效果**:
  - 流式回答时使用打字机效果逐字显示

### 3.4 状态指示器

- **连接状态**:
  - FunASR服务连接状态
  - 后端服务连接状态
  - 使用小图标和颜色指示（绿色=已连接，红色=未连接）
- **音频捕获指示器**:
  - 显示当前音频输入电平
  - 使用动态条形图或波形显示

## 4. 交互设计

### 4.1 语音识别流程

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│            │    │            │    │            │    │            │
│  选择音频  │───►│  点击开始  │───►│  创建音频  │───►│ 建立FunASR │
│  设备      │    │  录音按钮  │    │  上下文    │    │ WebSocket  │
│            │    │            │    │            │    │            │
└────────────┘    └────────────┘    └────────────┘    └────────────┘
                                                             │
                                                             ▼
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│            │    │            │    │            │    │            │
│  显示识别  │◄───┤  解析识别  │◄───┤  接收识别  │◄───┤  发送音频  │
│  结果      │    │  结果      │    │  结果      │    │  数据      │
│            │    │            │    │            │    │            │
└────────────┘    └────────────┘    └────────────┘    └────────────┘
```

### 4.2 AI回答流程

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│            │    │            │    │            │    │            │
│  点击识别  │───►│  高亮显示  │───►│  点击发送  │───►│ 发送HTTP   │
│  结果项    │    │  选中文本  │    │  按钮      │    │ 请求       │
│            │    │            │    │            │    │            │
└────────────┘    └────────────┘    └────────────┘    └────────────┘
                                                             │
                                                             ▼
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│            │    │            │    │            │    │            │
│  接收流URL │───►│ 建立流式   │───►│  接收流式  │───►│  处理回答  │
│  信息      │    │ WebSocket  │    │  数据      │    │  数据块    │
│            │    │            │    │            │    │            │
└────────────┘    └────────────┘    └────────────┘    └────────────┘
                                                             │
                                                             ▼
                                    ┌────────────┐    ┌────────────┐
                                    │            │    │            │
                                    │  逐字显示  │───►│  完成显示  │
                                    │  回答      │    │  回答      │
                                    │            │    │            │
                                    └────────────┘    └────────────┘
```

### 4.3 异常处理流程

- **连接失败**:
  - 显示友好的错误信息
  - 提供重试按钮
  - 提供疑难解答链接

- **权限拒绝**:
  - 解释为什么需要麦克风权限
  - 提供设置浏览器权限的指导

- **设备不可用**:
  - 提供虚拟音频设备安装指南
  - 推荐其他可用的音频设备

## 5. 响应式设计

### 5.1 桌面布局

- 两列布局，左侧语音识别，右侧AI回答
- 最小宽度: 1024px
- 容器最大宽度: 1200px

### 5.2 平板布局

- 屏幕宽度在 768px 到 1023px 之间
- 保持两列布局，但减小边距和内边距
- 调整字体大小和组件间距

### 5.3 移动设备布局

- 屏幕宽度小于 768px
- 切换为单列布局，语音识别面板在上，AI回答面板在下
- 固定控制按钮在屏幕底部或顶部
- 简化某些视觉元素

## 6. 视觉设计

### 6.1 颜色方案

- **主色**: #0066CC（蓝色）
- **强调色**: #28A745（绿色）
- **警告色**: #DC3545（红色）
- **背景色**: #F8F9FA（浅灰色）
- **文本色**: #333333（深灰色）
- **边框色**: #DDDDDD（中灰色）

### 6.2 字体

- **标题**: Segoe UI, Helvetica, sans-serif
- **正文**: Segoe UI, Arial, sans-serif
- **字体大小**:
  - 大标题: 24px
  - 小标题: 18px
  - 正文: 16px
  - 小字体: 14px

### 6.3 阴影与层次

- 卡片阴影: `0 2px 10px rgba(0, 0, 0, 0.1)`
- 结果项阴影: `0 1px 3px rgba(0, 0, 0, 0.1)`
- 按钮阴影: `0 2px 5px rgba(0, 0, 100, 0.1)`

### 6.4 图标

- 使用简洁的线性图标
- 推荐图标集: Font Awesome或Material Icons
- 主要使用的图标:
  - 麦克风
  - 停止
  - 发送
  - 加载
  - 音频波形
  - 设置
  - 错误/警告

## 7. 动画与过渡

### 7.1 UI元素动画

- **按钮悬停**: 轻微的背景色变化，持续时间200ms
- **模块加载**: 淡入效果，持续时间300ms
- **结果项添加**: 顶部滑入并淡入，持续时间250ms

### 7.2 功能性动画

- **加载指示器**: 旋转动画或脉冲动画
- **音频电平可视化**: 实时更新的波形或柱状图
- **打字机效果**: 逐字显示AI回答，速度约每分钟600字

## 8. 状态设计

### 8.1 识别结果状态

- **临时结果**: 浅色背景，左侧蓝色边框
- **最终结果**: 白色背景，左侧绿色边框，悬停效果
- **已选中结果**: 浅蓝色背景，指示当前处理的问题

### 8.2 AI回答状态

- **等待中**: 灰色左边框，显示加载动画
- **生成中**: 橙色左边框，显示打字机效果
- **已完成**: 绿色左边框，显示完整回答
- **错误**: 红色左边框，显示错误信息

### 8.3 连接状态

- **已连接**: 绿色小图标
- **连接中**: 黄色闪烁图标
- **未连接**: 红色图标
- **错误**: 红色图标加感叹号

## 9. 辅助功能设计

### 9.1 无障碍考虑

- 符合WCAG 2.1 AA级标准
- 适当的颜色对比度
- 所有功能可通过键盘操作
- 支持屏幕阅读器

### 9.2 键盘快捷键

- **开始/停止录音**: `Ctrl + R`
- **清除识别结果**: `Ctrl + C`
- **发送选中文本**: `Enter`（选中后）

## 10. 其他UI元素

### 10.1 设置面板

- **音频设置**:
  - 输入设备选择
  - 音量调节
  - 静音控制
- **显示设置**:
  - 主题切换（亮/暗）
  - 字体大小调整
  - 列宽调整

### 10.2 帮助功能

- **工具提示**: 悬停在各个功能按钮上显示简短说明
- **帮助图标**: 可点击打开帮助对话框
- **首次使用引导**: 简短的功能介绍步骤
- **虚拟音频设备设置指南**: 详细的安装和配置说明

## 11. 页面加载与初始化

### 11.1 加载序列

1. 显示应用名称和简洁加载动画
2. 加载必要的CSS和JavaScript
3. 初始化界面结构
4. 枚举可用的音频设备
5. 检查浏览器兼容性
6. 显示就绪状态

### 11.2 初始状态

- 录音按钮启用
- 停止按钮禁用
- 识别结果区域显示引导文本
- AI回答区域显示引导文本
- 自动选择最合适的音频设备（如果可用）

## 12. 示例代码

### 12.1 HTML结构示例

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
        <header class="app-header">
            <h1>AI面试助手</h1>
        </header>
        
        <main class="app-content">
            <section class="recognition-panel">
                <h2>语音识别</h2>
                <div id="recognition-results" class="results-container">
                    <div class="empty-state">点击"开始录音"按钮开始识别...</div>
                </div>
                
                <div class="audio-source">
                    <label for="audio-device">音频来源:</label>
                    <select id="audio-device">
                        <option value="">选择音频设备...</option>
                    </select>
                </div>
                
                <div class="audio-level-indicator">
                    <div class="level-bar"></div>
                </div>
                
                <div class="controls">
                    <button id="start-recognition" class="btn primary">
                        <i class="icon mic"></i>开始录音
                    </button>
                    <button id="stop-recognition" class="btn danger" disabled>
                        <i class="icon stop"></i>停止录音
                    </button>
                </div>
                
                <div class="send-controls">
                    <button id="send-button" class="btn success" disabled>
                        <i class="icon send"></i>发送到AI
                    </button>
                </div>
            </section>
            
            <section class="ai-panel">
                <h2>AI回答</h2>
                <div class="connection-status">
                    <span class="status-item" id="funasr-status">FunASR: <i class="status-icon disconnected"></i></span>
                    <span class="status-item" id="backend-status">后端: <i class="status-icon disconnected"></i></span>
                </div>
                <div id="ai-responses" class="results-container">
                    <div class="empty-state">选择左侧识别结果并点击"发送到AI"按钮...</div>
                </div>
            </section>
        </main>
        
        <footer class="app-footer">
            <p>© 2023 AI面试助手 | <a href="#help" id="help-link">使用帮助</a></p>
        </footer>
    </div>
    
    <div id="help-dialog" class="dialog hidden">
        <div class="dialog-content">
            <h3>使用帮助</h3>
            <div class="help-content">
                <!-- 帮助内容 -->
            </div>
            <button class="btn primary close-dialog">关闭</button>
        </div>
    </div>
    
    <script src="app.js"></script>
</body>
</html>
```

### 12.2 CSS样式示例（部分）

```css
:root {
    --primary-color: #0066cc;
    --success-color: #28a745;
    --danger-color: #dc3545;
    --warning-color: #fd7e14;
    --background-color: #f8f9fa;
    --text-color: #333333;
    --border-color: #dddddd;
    --shadow-color: rgba(0, 0, 0, 0.1);
    --card-shadow: 0 2px 10px var(--shadow-color);
    --item-shadow: 0 1px 3px var(--shadow-color);
}

* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    font-family: 'Segoe UI', Arial, sans-serif;
    line-height: 1.6;
    color: var(--text-color);
    background-color: var(--background-color);
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
}

.app-content {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
}

.recognition-panel, .ai-panel {
    background-color: white;
    border-radius: 8px;
    box-shadow: var(--card-shadow);
    padding: 20px;
    display: flex;
    flex-direction: column;
    height: 600px;
}

.results-container {
    flex: 1;
    overflow-y: auto;
    background-color: #f5f5f5;
    border-radius: 5px;
    padding: 15px;
    margin-bottom: 15px;
}

.empty-state {
    color: #6c757d;
    text-align: center;
    padding: 20px;
    font-style: italic;
}

.audio-source {
    margin-bottom: 15px;
}

.audio-source select {
    width: 100%;
    padding: 8px;
    border-radius: 5px;
    border: 1px solid var(--border-color);
}

.controls {
    display: flex;
    gap: 10px;
}

.btn {
    padding: 10px 15px;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    font-size: 14px;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    flex: 1;
    transition: background-color 0.2s;
}

.btn.primary {
    background-color: var(--primary-color);
    color: white;
}

.btn.danger {
    background-color: var(--danger-color);
    color: white;
}

.btn:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
}

.result-item {
    background-color: white;
    border-radius: 5px;
    padding: 10px;
    margin-bottom: 10px;
    box-shadow: var(--item-shadow);
    border-left: 3px solid var(--primary-color);
}

.result-item.final {
    border-left: 3px solid var(--success-color);
    cursor: pointer;
}

.result-item.selected {
    background-color: #e6f7ff;
    border: 1px solid var(--primary-color);
    border-left: 3px solid var(--primary-color);
}

.response-item {
    background-color: white;
    border-radius: 5px;
    padding: 10px;
    margin-bottom: 10px;
    box-shadow: var(--item-shadow);
    border-left: 3px solid #6610f2;
    white-space: pre-wrap;
}

.response-item.streaming {
    border-left: 3px solid var(--warning-color);
}

.response-item.complete {
    border-left: 3px solid var(--success-color);
}

.send-controls {
    margin-top: 10px;
    display: flex;
    justify-content: center;
}

.btn.success {
    background-color: var(--success-color);
    color: white;
}

/* 响应式设计 */
@media (max-width: 768px) {
    .app-content {
        grid-template-columns: 1fr;
    }
    
    .recognition-panel, .ai-panel {
        height: 500px;
    }
}
```

### 12.3 JavaScript交互示例（部分）

```javascript
// 音频电平可视化
function visualizeAudioLevel(audioContext, mediaStreamSource) {
    const analyser = audioContext.createAnalyser();
    analyser.fftSize = 256;
    
    mediaStreamSource.connect(analyser);
    
    const bufferLength = analyser.frequencyBinCount;
    const dataArray = new Uint8Array(bufferLength);
    
    const levelBar = document.querySelector('.level-bar');
    
    function updateLevel() {
        if (!audioContext || audioContext.state !== 'running') return;
        
        analyser.getByteFrequencyData(dataArray);
        
        // 计算平均音量
        let sum = 0;
        for (let i = 0; i < bufferLength; i++) {
            sum += dataArray[i];
        }
        const average = sum / bufferLength;
        
        // 更新音量条
        const percent = Math.min(100, average * 2); // 放大效果
        levelBar.style.width = percent + '%';
        levelBar.style.backgroundColor = getColorForLevel(percent);
        
        requestAnimationFrame(updateLevel);
    }
    
    updateLevel();
}

// 根据音量级别获取颜色
function getColorForLevel(level) {
    if (level < 30) return '#28a745'; // 低音量 - 绿色
    if (level < 70) return '#fd7e14'; // 中音量 - 橙色
    return '#dc3545'; // 高音量 - 红色
}

// 打字机效果
function typewriterEffect(element, text, speed = 30) {
    let i = 0;
    element.textContent = '';
    
    function type() {
        if (i < text.length) {
            element.textContent += text.charAt(i);
            i++;
            setTimeout(type, speed);
        }
    }
    
    type();
}

// 设置文本选择功能
function setupTextSelection() {
    const resultsContainer = document.getElementById('recognition-results');
    const sendButton = document.getElementById('send-button');
    
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
            sendButton.disabled = false;
        }
    });
    
    // 发送按钮点击事件
    sendButton.addEventListener('click', function() {
        sendSelectedText();
    });
}

// 发送选中文本到后端
async function sendSelectedText() {
    const selectedItem = document.querySelector('.result-item.selected');
    if (!selectedItem) return;
    
    const text = selectedItem.textContent;
    const sessionId = generateSessionId();
    
    try {
        // 显示正在处理的状态
        const responseElement = createResponseElement();
        responseElement.classList.add('processing');
        responseElement.textContent = "正在处理问题...";
        
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
        
        // 更新UI状态
        responseElement.textContent = "";
        responseElement.classList.remove('processing');
        responseElement.classList.add('streaming');
        
        // 连接WebSocket获取流式回答
        connectToStreamingResponse(data.streamUrl, responseElement);
    } catch (error) {
        console.error('发送文本失败:', error);
        
        // 显示错误信息
        const responsesContainer = document.getElementById('ai-responses');
        const errorElement = document.createElement('div');
        errorElement.className = 'response-item error';
        errorElement.textContent = `请求失败: ${error.message}`;
        responsesContainer.appendChild(errorElement);
    }
}
```

## 13. 用户体验注意事项

### 13.1 首次使用体验

- 提供音频设备选择指南，解释如何识别和选择适合的音频设备
- 显示简短的功能演示或引导
- 解释文本选择和发送机制
- 提供音频设备设置的帮助链接

### 13.2 错误处理与反馈

- 所有用户操作提供即时反馈
- 错误消息应具体且提供解决方案
- 连接问题应提供故障排除建议

### 13.3 性能优化

- 限制历史记录数量，避免页面过载
- 识别结果和AI回答按需加载
- 考虑添加导出/清除历史记录功能

## 14. 未来增强

- 暗黑模式主题切换
- 多语言支持
- 面试场景选择
- 标记和收藏重要回答
- 导出会话记录为PDF或文本
- 移动端应用适配 