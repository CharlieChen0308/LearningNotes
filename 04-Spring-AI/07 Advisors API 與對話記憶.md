# 07 Advisors API 與對話記憶

> **版本**：Spring AI 1.0+ / Spring Boot 3.4.x / Java 17+

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
                ChatResponse safeResponse = ChatResponse.builder()
                    .generations(List.of(new Generation(
                        new AssistantMessage("抱歉，為了安全考量，我無法處理包含敏感資訊的請求。"))))
                    .build();
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

> **API 版本注意**：上述範例中 `new AdvisedResponse(chatResponse, adviseContext)` 的建構方式與
> `ChatResponse.builder()` 的用法，請依實際使用的 Spring AI 版本確認。Spring AI 1.0.x 的 API
> 仍在快速演進中，部分類別的簽章可能在小版本間有所調整，建議參照官方文件或 IDE 自動補全確認。

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
// 方式一：透過 MessageWindowChatMemory 控制視窗大小（推薦）
ChatMemory chatMemory = MessageWindowChatMemory.builder()
    .chatMemoryRepository(chatMemoryRepository)
    .maxMessages(20)  // 最多保留最近 20 條訊息
    .build();
```

> **API 版本注意**：早期版本使用 `chatMemoryRetrieveSize(N)` 方法控制取回筆數，
> Spring AI 1.0 GA 後改為透過 `MessageWindowChatMemory.builder().maxMessages(N)` 設定。
> 請依實際使用版本確認可用的 API。

## ChatMemory 選型指引

### 儲存實現選擇

| 實現 | 適用場景 | 優點 | 限制 |
|------|---------|------|------|
| `InMemoryChatMemoryRepository` | 開發、測試、原型驗證 | 零配置、啟動快 | 重啟即消失，不適合生產 |
| `JdbcChatMemoryRepository` | 生產環境（已有 RDBMS） | 持久化、事務支援、易維運 | 高併發下需注意連線池 |
| `CassandraChatMemoryRepository` | 大規模分散式系統 | 水平擴展、高可用 | 運維複雜度較高 |
| `Neo4jChatMemoryRepository` | 需要對話關係分析 | 圖查詢能力 | 使用場景較特殊 |

### Memory Advisor 選擇

| Advisor | 運作方式 | 適用場景 |
|---------|---------|---------|
| `MessageChatMemoryAdvisor` | 將歷史訊息以完整 Message 列表注入 | 大多數聊天場景，模型能理解對話脈絡 |
| `PromptChatMemoryAdvisor` | 將歷史摘要寫入 System Prompt | 不支援多訊息格式的模型 |
| `VectorStoreChatMemoryAdvisor` | 以語意搜尋取回相關歷史片段 | 超長對話、需跨 Session 召回相關記憶 |

**選擇建議**：一般應用優先使用 `MessageChatMemoryAdvisor`；當對話歷史極長或需要跨對話檢索時，改用 `VectorStoreChatMemoryAdvisor`。

## 生產環境注意事項

### Token 限制管理

對話歷史會持續累積，若未加控制，可能超出模型的 Context Window 上限：

- 使用 `MessageWindowChatMemory` 的 `maxMessages` 限制保留的訊息數量
- 監控每次請求的 Token 使用量，設定告警閾值
- 考慮在超出限制前進行摘要壓縮（目前需自行實作）

### 記憶體清理與過期策略

- **定期清理**：對閒置超過一定時間的 Session 進行清理，避免儲存無限增長
- **TTL 機制**：若使用 Cassandra 或 Redis 儲存，可利用原生 TTL 功能自動過期
- **JDBC 場景**：建議搭配排程任務（如 `@Scheduled`）定期刪除過期對話紀錄

### 成本控制

長對話會顯著增加 Token 消耗，每輪對話都會重送歷史訊息：

- 設定合理的 `maxMessages`（建議 10–30 條）
- 計算公式參考：每輪成本 ≈ 歷史訊息 Token + 新訊息 Token + 回應 Token
- 對於高流量場景，評估是否需要對話摘要機制以降低成本

### Session 管理

- 每個 `conversationId` 對應一組獨立的對話歷史
- 前端應在使用者登入時產生唯一的 `conversationId`，登出或逾時後清除
- 多裝置場景需考慮 Session 同步策略

## 小結

Advisors API 是 Spring AI 的核心設計模式，透過攔截器鏈的方式組合各種 AI 互動能力。對話記憶讓 AI 應用能夠進行連續的多輪對話，而自訂 Advisor 則能靈活地加入日誌、安全過濾等橫切關注點。

## 延伸閱讀

- [02 ChatClient API 與對話模型](02%20ChatClient%20API%20與對話模型.md) — ChatClient 基礎用法與 Prompt 模板
- [05 RAG 檢索增強生成](05%20RAG%20檢索增強生成.md) — 使用 QuestionAnswerAdvisor 實現檢索增強生成
