# 05 RAG 檢索增強生成

## 什麼是 RAG

RAG（Retrieval Augmented Generation，檢索增強生成）是一種結合資訊檢索和文字生成的技術，解決了 AI 模型的兩大限制：

- **知識過時**：模型訓練資料有截止日期，無法回答最新資訊
- **缺乏專有知識**：模型不了解企業內部文件、產品規格等私有資料

RAG 的核心思想：先從知識庫中檢索相關文件，再將文件作為上下文提供給 AI 模型，讓模型基於這些資料生成回答。

## RAG 的工作流程

```
使用者提問 → 向量檢索（從知識庫找到相關文件）
                ↓
         將文件 + 問題一起送給 AI 模型
                ↓
         AI 基於文件內容生成精確回答
```

分為兩個階段：

### 1. 匯入階段（Ingestion）

```
原始文件 → 文件分割 → 計算 Embedding → 儲存到向量資料庫
```

### 2. 查詢階段（Query）

```
使用者問題 → 計算 Embedding → 相似度搜尋 → 取得相關文件
         → 組合 Prompt → AI 生成回答
```

## Spring AI 中的 RAG 實現

### 使用 QuestionAnswerAdvisor

Spring AI 提供了 `QuestionAnswerAdvisor`，開箱即用的 RAG 方案：

```java
@RestController
public class RagController {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public RagController(ChatClient.Builder builder, VectorStore vectorStore) {
        this.vectorStore = vectorStore;
        this.chatClient = builder.build();
    }

    @GetMapping("/rag")
    public String askWithRag(@RequestParam String question) {
        return chatClient.prompt()
            .advisors(QuestionAnswerAdvisor.builder(vectorStore).build())
            .user(question)
            .call()
            .content();
    }
}
```

`QuestionAnswerAdvisor` 會自動：
1. 將使用者問題轉換為 Embedding
2. 從 VectorStore 搜尋相關文件
3. 將文件內容附加到 Prompt 中
4. 讓 AI 模型基於文件生成回答

### 自訂搜尋參數

```java
QuestionAnswerAdvisor advisor = QuestionAnswerAdvisor.builder(vectorStore)
    .searchRequest(SearchRequest.builder()
        .topK(5)                        // 檢索前 5 個相關文件
        .similarityThreshold(0.75)      // 相似度閾值
        .filterExpression("category == 'product'")  // 後設資料過濾
        .build())
    .build();
```

## 完整 RAG 應用範例

### 1. 文件匯入服務

```java
@Service
public class KnowledgeBaseService {

    private final VectorStore vectorStore;

    public KnowledgeBaseService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    /**
     * 匯入文字文件到知識庫
     */
    public void importTextDocuments(List<String> texts, String category) {
        List<Document> documents = texts.stream()
            .map(text -> new Document(text, Map.of("category", category)))
            .toList();

        // 分割長文件
        TokenTextSplitter splitter = new TokenTextSplitter(
            800,    // 每個區塊的最大 Token 數
            200,    // 區塊間重疊的 Token 數
            5,      // 最小區塊大小
            10000,  // 最大字元數
            true    // 保留分隔符
        );

        List<Document> chunks = splitter.apply(documents);
        vectorStore.add(chunks);
    }

    /**
     * 匯入 PDF 文件
     */
    public void importPdf(Resource pdfResource, String category) {
        PagePdfDocumentReader reader = new PagePdfDocumentReader(pdfResource);
        List<Document> documents = reader.get();

        // 為每個文件加上後設資料
        documents.forEach(doc ->
            doc.getMetadata().put("category", category));

        TokenTextSplitter splitter = new TokenTextSplitter();
        List<Document> chunks = splitter.apply(documents);
        vectorStore.add(chunks);
    }
}
```

### 2. RAG 問答服務

```java
@Service
public class RagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public RagService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.vectorStore = vectorStore;
        this.chatClient = builder
            .defaultSystem("""
                你是一位專業的客服助手。請根據提供的文件內容回答問題。
                如果文件中沒有相關資訊，請明確告知使用者你不確定，
                不要編造答案。
                """)
            .build();
    }

    public String ask(String question, String category) {
        SearchRequest searchRequest = SearchRequest.builder()
            .query(question)
            .topK(5)
            .similarityThreshold(0.7)
            .filterExpression("category == '" + category + "'")
            .build();

        return chatClient.prompt()
            .advisors(QuestionAnswerAdvisor.builder(vectorStore)
                .searchRequest(searchRequest)
                .build())
            .user(question)
            .call()
            .content();
    }
}
```

### 3. REST API

```java
@RestController
@RequestMapping("/knowledge")
public class KnowledgeController {

    private final KnowledgeBaseService knowledgeBaseService;
    private final RagService ragService;

    // 匯入文件
    @PostMapping("/import")
    public String importDocuments(@RequestBody ImportRequest request) {
        knowledgeBaseService.importTextDocuments(
            request.texts(), request.category());
        return "已匯入 " + request.texts().size() + " 筆文件";
    }

    // RAG 問答
    @GetMapping("/ask")
    public String ask(
            @RequestParam String question,
            @RequestParam(defaultValue = "general") String category) {
        return ragService.ask(question, category);
    }
}

record ImportRequest(List<String> texts, String category) {}
```

## RAG 最佳實踐

### 文件分割策略

- **區塊大小**：太大會包含太多無關內容，太小會丟失上下文。建議 500~1000 Token
- **重疊區域**：相鄰區塊應有 10~20% 的重疊，避免在邊界處丟失資訊
- **按語義分割**：優先在段落、章節邊界處分割

### 檢索最佳化

- 調整 `topK`：太少可能遺漏重要資訊，太多會引入雜訊
- 設定合理的 `similarityThreshold`：過濾掉相關度低的結果
- 使用 Metadata Filter 縮小搜尋範圍

### Prompt 工程

- 明確告知 AI 只根據提供的文件回答
- 要求 AI 在不確定時說明
- 指定回答的格式和語言

## 小結

RAG 是將 AI 模型與企業知識庫結合的關鍵技術。Spring AI 的 `QuestionAnswerAdvisor` 提供了開箱即用的 RAG 方案，搭配 `VectorStore` 和 ETL Pipeline，可以快速構建基於企業資料的智慧問答系統。
