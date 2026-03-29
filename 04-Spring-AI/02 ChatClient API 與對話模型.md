# 02 ChatClient API 與對話模型

> **版本**：Spring AI 1.0+ / Spring Boot 3.x / Java 17+

## ChatClient 簡介

`ChatClient` 是 Spring AI 中與 AI 模型互動的核心 API，設計風格類似於 Spring 的 `WebClient` 和 `RestClient`——採用流暢式 API（Fluent API），支援同步和串流呼叫。

## 建立 ChatClient

### 自動注入

```java
@RestController
public class ChatController {

    private final ChatClient chatClient;

    // Spring Boot 自動配置 ChatClient.Builder
    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
}
```

### 手動建立

```java
ChatClient chatClient = ChatClient.create(chatModel);
```

### 配置預設行為

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder) {
    return builder
        .defaultSystem("你是一位專業的 Java 技術顧問，用繁體中文回答。")
        .defaultAdvisors(new MessageChatMemoryAdvisor(chatMemory))
        .build();
}
```

## 基本對話

### 簡單呼叫

```java
String answer = chatClient.prompt()
    .user("什麼是 Spring Boot？")
    .call()
    .content();
```

### 設定系統提示詞

```java
String answer = chatClient.prompt()
    .system("你是一位資深 Java 工程師")
    .user("請解釋 AOP 的概念")
    .call()
    .content();
```

### 取得完整回應

```java
ChatResponse response = chatClient.prompt()
    .user("什麼是微服務？")
    .call()
    .chatResponse();

// 取得回應文字
String content = response.getResult().getOutput().getText();

// 取得使用量資訊
Usage usage = response.getMetadata().getUsage();
System.out.println("輸入 Token：" + usage.getPromptTokens());
System.out.println("輸出 Token：" + usage.getCompletionTokens());
```

## 串流對話

串流模式讓 AI 回應可以逐字輸出，提升使用者體驗：

```java
Flux<String> stream = chatClient.prompt()
    .user("請寫一篇關於 Spring AI 的介紹")
    .stream()
    .content();
```

在 REST API 中使用 SSE（Server-Sent Events）：

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String message) {
    return chatClient.prompt()
        .user(message)
        .stream()
        .content();
}
```

## Prompt 模板

### 使用 PromptTemplate

```java
@GetMapping("/recommend")
public String recommend(@RequestParam String language) {
    PromptTemplate template = new PromptTemplate(
        "請推薦 3 本學習 {language} 的好書，並簡述推薦理由。");

    Prompt prompt = template.create(Map.of("language", language));

    return chatClient.prompt(prompt)
        .call()
        .content();
}
```

### 從資源檔案載入模板

建立 `src/main/resources/prompts/recommend.st`：

```
請推薦 {count} 本學習 {language} 的好書，並簡述推薦理由。
要求：
1. 按推薦程度排序
2. 包含書名、作者、推薦理由
```

```java
@Value("classpath:prompts/recommend.st")
private Resource promptResource;

@GetMapping("/recommend")
public String recommend(@RequestParam String language) {
    PromptTemplate template = new PromptTemplate(promptResource);
    Prompt prompt = template.create(Map.of(
        "language", language,
        "count", "3"
    ));

    return chatClient.prompt(prompt)
        .call()
        .content();
}
```

## 模型參數調整

### 在請求中調整

```java
import org.springframework.ai.openai.OpenAiChatOptions;

String answer = chatClient.prompt()
    .user("請寫一首詩")
    .options(OpenAiChatOptions.builder()
        .model("gpt-4o")
        .temperature(0.9)       // 創造性（0~1，越高越有創意）
        .maxTokens(500)         // 最大回應長度
        .topP(0.95)             // 核心取樣
        .build())
    .call()
    .content();
```

### 在配置檔案中設定

```yaml
spring:
  ai:
    openai:
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
          max-tokens: 1000
```

## 多輪對話

手動管理對話歷史：

```java
@RestController
public class ConversationController {

    private final ChatClient chatClient;
    private final List<Message> conversationHistory = new ArrayList<>();

    public ConversationController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
        conversationHistory.add(new SystemMessage("你是一位友善的助手"));
    }

    @PostMapping("/conversation")
    public String chat(@RequestBody String userMessage) {
        conversationHistory.add(new UserMessage(userMessage));

        ChatResponse response = chatClient.prompt()
            .messages(conversationHistory)
            .call()
            .chatResponse();

        String assistantMessage = response.getResult().getOutput().getText();
        conversationHistory.add(new AssistantMessage(assistantMessage));

        return assistantMessage;
    }
}
```

> 更推薦使用 Spring AI 提供的 `ChatMemory` 和 `Advisors`，將在後續章節介紹。

## 小結

ChatClient 是 Spring AI 的核心入口，透過流暢的 API 設計，開發者可以輕鬆實現單輪對話、串流輸出、Prompt 模板和模型參數調整。配合 Spring Boot 的自動配置，只需極少的程式碼即可構建 AI 對話功能。

## 延伸閱讀

- [03 結構化輸出](03%20結構化輸出.md) — 將 AI 回應自動對映為 Java 物件
- [07 Advisors API 與對話記憶](07%20Advisors%20API%20與對話記憶.md) — 攔截器模式與多輪對話記憶管理
