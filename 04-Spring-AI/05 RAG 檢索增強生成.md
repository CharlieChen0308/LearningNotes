# 05 RAG 檢索增強生成

> **版本**：Spring AI 1.0+ / Spring Boot 3.4.x / Java 17+

## 什麼是 RAG

RAG（Retrieval Augmented Generation，檢索增強生成）是一種結合資訊檢索和文字生成的技術，解決了 AI 模型的兩大限制：

- **知識過時**：模型訓練資料有截止日期，無法回答最新資訊
- **缺乏專有知識**：模型不了解企業內部文件、產品規格等私有資料

RAG 的核心思想：先從知識庫中檢索相關文件，再將文件作為上下文提供給 AI 模型，讓模型基於這些資料生成回答。

### RAG vs Fine-tuning vs Long-context Window

| 比較項目 | RAG | Fine-tuning | Long-context Window |
|---------|-----|-------------|---------------------|
| 知識更新 | 動態更新，新增文件即時生效 | 靜態，需重新訓練模型 | 動態，每次呼叫時帶入 |
| 成本 | 需建置向量資料庫，推論成本中等 | 訓練成本高，推論成本低 | 無額外基礎設施，但每次呼叫 Token 消耗大 |
| 知識規模 | 可擴展至大量文件 | 受限於訓練資料量 | 受限於模型 Context Window 大小 |
| 實作複雜度 | 中等（需 Embedding + 向量資料庫） | 高（需準備訓練資料集 + 訓練流程） | 低（直接將文件塞入 Prompt） |
| 適用場景 | 企業知識庫、即時更新的文件問答 | 特定領域語言風格、固定知識內化 | 少量文件的快速原型、單次對話 |

> **選擇建議**：大多數企業場景優先考慮 RAG，因其知識可即時更新且不需重新訓練模型。Fine-tuning 適合需要深度內化特定領域風格的場景。Long-context Window 適合文件量小、追求快速驗證的情境。

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

        // 分割長文件（Spring AI 1.0 GA 五參數建構子）
        // TokenTextSplitter(defaultChunkSize, minChunkSizeChars, minChunkLengthToEmbed, maxNumChunks, keepSeparator)
        TokenTextSplitter splitter = new TokenTextSplitter(
            800,    // defaultChunkSize — 每個區塊的目標 Token 數
            200,    // minChunkSizeChars — 區塊最小字元數（過短則合併）
            5,      // minChunkLengthToEmbed — 嵌入的最小區塊長度
            10000,  // maxNumChunks — 單一文件最大區塊數
            true    // keepSeparator — 保留分隔符
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
        // 使用 FilterExpressionBuilder 避免字串拼接注入風險
        FilterExpressionBuilder b = new FilterExpressionBuilder();
        Filter.Expression filterExpression = b.eq("category", category).build();

        SearchRequest searchRequest = SearchRequest.builder()
            .query(question)
            .topK(5)
            .similarityThreshold(0.7)
            .filterExpression(filterExpression)
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

### 生產環境注意事項

#### 文件匯入管線設計

- **批次處理**：大量文件匯入時，使用非同步批次處理避免阻塞，搭配進度追蹤機制
- **增量更新**：透過文件 hash 或修改時間判斷是否需要重新 Embedding，避免全量重建
- **錯誤處理**：匯入失敗時應記錄失敗文件並支援重試，確保知識庫完整性
- **後設資料管理**：統一定義後設資料結構（來源、類別、版本、匯入時間），方便後續過濾與追蹤

#### Chunk Size 最佳化

- **過大的 Chunk**：包含過多無關內容，降低檢索精確度
- **過小的 Chunk**：丟失上下文語義，導致回答片段化
- **建議策略**：從 500~800 Token 開始，根據實際問答品質進行 A/B 測試調整
- **語義分割優先**：在段落、章節邊界處分割，比固定 Token 數分割效果更好

#### Re-ranking 策略

- **兩階段檢索**：先用向量搜尋取回較多候選（如 topK=20），再用 Re-ranker 模型精排取前 5 筆
- **Cross-encoder Re-ranking**：使用交叉編碼器對 query-document pair 重新評分，精確度高於向量相似度
- **多路召回**：同時使用關鍵字搜尋（BM25）和向量搜尋，合併結果後 Re-rank，提升召回率

#### 幻覺偵測與防範

- **引用來源**：要求模型標註回答來自哪份文件，方便使用者驗證
- **信心度門檻**：若最高相似度低於閾值，回覆「找不到相關資料」而非強行生成
- **Grounding 檢查**：比對生成回答與檢索文件的語義重疊度，偵測模型是否「脫離文件」回答
- **System Prompt 強化**：明確指示「僅根據提供的文件回答，若無相關內容則明確告知」

#### 檢索品質監控

- **關鍵指標**：追蹤檢索命中率（Hit Rate）、平均排名倒數（MRR）、回答滿意度
- **日誌記錄**：記錄每次查詢的問題、檢索到的文件 ID、相似度分數、最終回答
- **定期評估**：建立標準問答對（Golden Set），定期執行自動化評估並追蹤品質趨勢
- **使用者回饋**：提供「回答是否有幫助」的回饋機制，作為持續改進的依據

## 小結

RAG 是將 AI 模型與企業知識庫結合的關鍵技術。Spring AI 的 `QuestionAnswerAdvisor` 提供了開箱即用的 RAG 方案，搭配 `VectorStore` 和 ETL Pipeline，可以快速構建基於企業資料的智慧問答系統。

## 延伸閱讀

- [04 Embedding 與向量資料庫](04%20Embedding%20與向量資料庫.md) — Embedding 原理與向量資料庫操作基礎
- [07 Advisors API 與對話記憶](07%20Advisors%20API%20與對話記憶.md) — Advisors 攔截器模式與 QuestionAnswerAdvisor 進階用法

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
