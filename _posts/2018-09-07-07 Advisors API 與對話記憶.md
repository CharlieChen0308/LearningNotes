# 07 Advisors API 與對話記憶

## 什麼是 Advisors

Advisors 是 Spring AI 中的攔截器模式，類似於 Spring MVC 的 `HandlerInterceptor` 或 AOP 的概念。Advisors 可以在 AI 請求發送前和回應返回後進行處理，用於封裝常見的 AI 互動模式。

常見應用場景：
- **對話記憶**：自動管理多輪對話的歷史訊息
- **RAG**：自動檢索相關文件並附加到 Prompt
- **日誌記錄**：記錄所有 AI 互動
- **安全過濾**：過濾敏感內容

## 內建 Advisors

| Advisor | 功能 |
|---------|------|
| MessageChatMemoryAdvisor | 將對話歷史以訊息形式附加 |
| VectorStoreChatMemoryAdvisor | 使用向量資料庫儲存和檢索對話歷史 |
| QuestionAnswerAdvisor | RAG 檢索增強生成 |
| SafeGuardAdvisor | 安全過濾，檢查敏感內容 |

## 對話記憶（Chat Memory）

### 為什麼需要對話記憶

AI 模型本身是無狀態的，每次呼叫都是獨立的。如果要實現多輪對話（例如聊天機器人），就需要手動管理對話歷史。`ChatMemory` 自動處理這一切。

### ChatMemory 介面

Spring AI 提供了多種 ChatMemory 實現：

| 實現 | 儲存位置 |
|------|---------|
| InMemoryChatMemory | 記憶體（開發測試用）|
| CassandraChatMemory | Apache Cassandra |
| JdbcChatMemory | 關聯式資料庫（JDBC）|
| Neo4jChatMemory | Neo4j 圖資料庫 |

### 基本配置

```java
@Configuration
public class ChatMemoryConfig {

    @Bean
    public ChatMemory chatMemory() {
        return new InMemoryChatMemory();
    }

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory) {
        return builder
            .defaultSystem("你是一位友善的助手，用繁體中文回答。")
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .build()
            )
            .build();
    }
}
```

### 使用對話記憶

```java
@RestController
@RequestMapping("/chat")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @PostMapping
    public String chat(
            @RequestParam String message,
            @RequestParam(defaultValue = "default") String conversationId) {

        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(
                ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
    }
}
```

對話範例：

```
使用者（conversationId=user1）：我叫小明
AI：你好，小明！很高興認識你。

使用者（conversationId=user1）：我叫什麼名字？
AI：你叫小明呀！你剛才告訴我的。
```

### 使用 JDBC 持久化對話記憶

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-model-chat-memory-jdbc</artifactId>
</dependency>
```

```java
@Bean
public ChatMemory chatMemory(JdbcTemplate jdbcTemplate) {
    return JdbcChatMemory.create(jdbcTemplate);
}
```

## 自訂 Advisor

### 實現日誌記錄 Advisor

```java
@Component
public class LoggingAdvisor implements CallAroundAdvisor {

    private static final Logger log = LoggerFactory.getLogger(LoggingAdvisor.class);

    @Override
    public AdvisedResponse aroundCall(AdvisedRequest advisedRequest,
                                       CallAroundAdvisorChain chain) {
        // 請求前
        log.info("AI 請求 - 使用者訊息: {}",
            advisedRequest.userText());

        long start = System.currentTimeMillis();

        // 執行實際呼叫
        AdvisedResponse response = chain.nextAroundCall(advisedRequest);

        // 回應後
        long elapsed = System.currentTimeMillis() - start;
        log.info("AI 回應 - 耗時: {}ms, 內容長度: {}",
            elapsed,
            response.response().getResult().getOutput().getText().length());

        return response;
    }

    @Override
    public String getName() {
        return "LoggingAdvisor";
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

### 實現安全過濾 Advisor

```java
@Component
public class ContentFilterAdvisor implements CallAroundAdvisor {

    private static final List<String> BLOCKED_KEYWORDS = List.of(
        "密碼", "信用卡號", "身分證字號"
    );

    @Override
    public AdvisedResponse aroundCall(AdvisedRequest advisedRequest,
                                       CallAroundAdvisorChain chain) {
        String userText = advisedRequest.userText();

        // 檢查使用者輸入是否包含敏感關鍵字
        for (String keyword : BLOCKED_KEYWORDS) {
            if (userText.contains(keyword)) {
                // 返回安全提示，不送出給 AI
                ChatResponse safeResponse = createSafeResponse(
                    "抱歉，為了安全考量，我無法處理包含敏感資訊的請求。");
                return new AdvisedResponse(safeResponse,
                    advisedRequest.adviseContext());
            }
        }

        return chain.nextAroundCall(advisedRequest);
    }

    @Override
    public String getName() {
        return "ContentFilterAdvisor";
    }

    @Override
    public int getOrder() {
        return -10;  // 高優先度，最先執行
    }
}
```

## 組合多個 Advisors

```java
@Bean
public ChatClient chatClient(ChatClient.Builder builder,
                              ChatMemory chatMemory,
                              VectorStore vectorStore,
                              LoggingAdvisor loggingAdvisor,
                              ContentFilterAdvisor contentFilterAdvisor) {
    return builder
        .defaultSystem("你是一位專業的客服助手")
        .defaultAdvisors(
            contentFilterAdvisor,       // 先過濾敏感內容
            loggingAdvisor,             // 記錄日誌
            MessageChatMemoryAdvisor.builder(chatMemory).build(),  // 對話記憶
            QuestionAnswerAdvisor.builder(vectorStore).build()     // RAG 檢索
        )
        .build();
}
```

Advisors 按照 `getOrder()` 的順序執行，數字越小越先執行。

## 對話視窗控制

避免對話歷史過長導致 Token 超限：

```java
MessageChatMemoryAdvisor.builder(chatMemory)
    .chatMemoryRetrieveSize(20)  // 最多取回最近 20 條訊息
    .build()
```

## 小結

Advisors API 是 Spring AI 的核心設計模式，透過攔截器鏈的方式組合各種 AI 互動能力。對話記憶讓 AI 應用能夠進行連續的多輪對話，而自訂 Advisor 則能靈活地加入日誌、安全過濾等橫切關注點。
