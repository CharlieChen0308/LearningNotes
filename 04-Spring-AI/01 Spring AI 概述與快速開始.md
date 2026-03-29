# 01 Spring AI 概述與快速開始

## 什麼是 Spring AI

Spring AI 是 Spring 官方推出的 AI 應用開發框架，目標是將 Spring 生態系統的設計原則（如可移植性、模組化設計）帶入 AI 領域，讓 Java 開發者能輕鬆構建 AI 驅動的應用程式。

Spring AI 提供了跨 AI 提供者的可攜式 API 抽象，開發者可以用統一的程式碼介面切換不同的 AI 模型，而無需修改業務邏輯。

## 主要功能

| 功能 | 說明 |
|------|------|
| Chat Completion | 與 AI 模型進行對話（支援同步與串流） |
| Embedding | 文字嵌入向量生成 |
| Image Generation | 文字生成圖片 |
| Audio Transcription | 語音轉文字 |
| Structured Output | AI 回應自動對映到 Java 物件 |
| Function Calling | 讓 AI 模型呼叫自訂函式取得即時資料 |
| RAG | 檢索增強生成，結合企業知識庫 |
| Vector Store | 向量資料庫操作 |
| Chat Memory | 對話記憶管理 |
| Advisors | 可組合的 AI 互動攔截器 |

## 支援的 AI 提供者

Spring AI 支援所有主流 AI 模型供應商：

- **OpenAI**（GPT-4o、GPT-4、GPT-3.5）
- **Anthropic**（Claude 4.5/4.6 系列）
- **Google**（Gemini）
- **Amazon Bedrock**（多種模型）
- **Microsoft Azure OpenAI**
- **Ollama**（本地開源模型）
- **Mistral AI**
- **NVIDIA**

## 快速開始

### 1. 建立專案

使用 Spring Initializr 或手動新增依賴。以 OpenAI 為例：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 2. 配置 API Key

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
```

### 3. 使用 ChatClient

```java
@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }
}
```

### 4. 串流回應

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String message) {
    return chatClient.prompt()
        .user(message)
        .stream()
        .content();
}
```

## 使用 Ollama（本地模型）

如果想在本地運行 AI 模型，可以使用 Ollama：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3
```

程式碼無需任何修改，ChatClient 的介面完全一致。

## 切換 AI 提供者

這就是 Spring AI 可攜式 API 的優勢——只需更換依賴和配置，業務程式碼完全不變：

```java
// 這段程式碼在任何 AI 提供者下都能正常運作
String response = chatClient.prompt()
    .user("請用一句話解釋什麼是微服務")
    .call()
    .content();
```

## 小結

Spring AI 讓 Java 開發者能用熟悉的 Spring 方式開發 AI 應用，統一的 API 抽象使得切換 AI 提供者變得輕而易舉。後續章節將深入介紹 ChatClient、Prompt 模板、Function Calling、RAG 等核心功能。
