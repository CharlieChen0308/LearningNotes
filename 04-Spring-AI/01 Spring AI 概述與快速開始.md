# 01 Spring AI 概述與快速開始

> **版本**：Spring AI 1.0+ / Spring Boot 3.4.x / Java 17+

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

- **OpenAI**（GPT-4o、GPT-4o-mini、GPT-4；GPT-3.5-turbo 已標記為 legacy，建議使用 GPT-4o-mini 作為低成本替代）
- **Anthropic**（Claude 系列，實際可用模型版本取決於 provider SDK 版本與 Anthropic API 支援）
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
    <version>3.4.1</version>
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

## Spring AI vs 其他方案：何時該用哪個？

| 面向 | Spring AI | LangChain4j | 直接使用 SDK |
|------|-----------|-------------|-------------|
| 適合場景 | Java/Spring 生態系、企業應用 | 需要豐富的社群模組、偏好 Python 風格的 chain 組合 | 需要完全掌控、無抽象層開銷 |
| 型別安全 | 強型別，與 Spring Boot 深度整合 | 型別安全，但非 Spring 原生 | 依賴各 SDK 設計 |
| 學習曲線 | Spring 開發者極低 | 熟悉 LangChain 概念者較低 | 需理解各提供者 API 細節 |
| 可攜性 | 統一 API，切換提供者只需改配置 | 有抽象層但整合方式不同 | 綁定單一提供者 |
| 企業整合 | Spring Security、Spring Data 等無縫銜接 | 需自行整合 | 需自行整合 |

**建議**：若專案已在 Spring Boot 生態系內，優先選擇 Spring AI；若需要快速原型驗證且團隊熟悉 Python 風格，可考慮 LangChain4j；若只串接單一提供者且需極致效能控制，直接使用 SDK。

## 生產環境注意事項

在將 Spring AI 應用部署至生產環境前，需注意以下關鍵事項：

### API Key 管理

- **絕對不要**將 API Key 寫死在程式碼或 `application.yml` 中
- 使用環境變數（`${OPENAI_API_KEY}`）或秘密管理工具（如 HashiCorp Vault、AWS Secrets Manager）
- 在 Spring Boot 中可搭配 `spring-cloud-vault` 自動注入

### 速率限制與成本控制

- 各 AI 提供者皆有 API 呼叫速率限制（Rate Limit），需實作重試機制（如 Spring Retry）
- 建議設定每日/每月的 token 用量上限，避免非預期的高額帳單
- 使用 Micrometer 等監控工具追蹤 API 呼叫次數與 token 消耗

### 錯誤處理與降級策略

- AI API 呼叫可能因網路、速率限制或服務中斷而失敗，務必實作 try-catch 與降級邏輯
- 考慮設定備援提供者（fallback），例如主要使用 OpenAI，降級時切換至 Ollama 本地模型
- 使用 Spring AI 的 Advisors 機制可統一攔截與處理錯誤

### 日誌與可觀測性

- 記錄每次 AI 呼叫的輸入摘要、回應時間、token 數量（注意不要記錄敏感的使用者輸入）
- 整合 OpenTelemetry 或 Spring Actuator 進行監控

## 小結

Spring AI 讓 Java 開發者能用熟悉的 Spring 方式開發 AI 應用，統一的 API 抽象使得切換 AI 提供者變得輕而易舉。後續章節將深入介紹 ChatClient、Prompt 模板、Function Calling、RAG 等核心功能。

## 延伸閱讀

- [02 ChatClient API 與對話模型](02%20ChatClient%20API%20與對話模型.md) — 深入了解 ChatClient 的流暢式 API 與對話功能
- [06 Function Calling 工具呼叫](06%20Function%20Calling%20工具呼叫.md) — 讓 AI 模型呼叫自訂函式擴展能力
