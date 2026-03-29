# 04 Embedding 與向量資料庫

## 什麼是 Embedding

Embedding（嵌入向量）是將文字轉換為數值向量的過程。每段文字會被轉換為一個高維度的浮點數陣列，語義相似的文字在向量空間中的距離會比較接近。

例如：
- "Java 程式語言" → [0.12, -0.34, 0.56, ...]
- "Java 開發" → [0.11, -0.33, 0.55, ...]（與上一個很接近）
- "今天天氣很好" → [0.87, 0.23, -0.45, ...]（距離較遠）

Embedding 是 RAG（檢索增強生成）和語義搜尋的基礎。

## Spring AI 的 EmbeddingModel

### 新增依賴（以 OpenAI 為例）

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

### 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small
```

### 使用 EmbeddingModel

```java
@Service
public class EmbeddingService {

    private final EmbeddingModel embeddingModel;

    public EmbeddingService(EmbeddingModel embeddingModel) {
        this.embeddingModel = embeddingModel;
    }

    public float[] getEmbedding(String text) {
        EmbeddingResponse response = embeddingModel.embedForResponse(List.of(text));
        return response.getResult().getOutput();
    }

    public double cosineSimilarity(String text1, String text2) {
        float[] v1 = getEmbedding(text1);
        float[] v2 = getEmbedding(text2);

        double dotProduct = 0, norm1 = 0, norm2 = 0;
        for (int i = 0; i < v1.length; i++) {
            dotProduct += v1[i] * v2[i];
            norm1 += v1[i] * v1[i];
            norm2 += v2[i] * v2[i];
        }
        return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
    }
}
```

## 向量資料庫（Vector Store）

向量資料庫專門用於儲存和查詢向量資料，支援高效的相似度搜尋。Spring AI 提供統一的 `VectorStore` 介面，支援多種向量資料庫：

| 向量資料庫 | 說明 |
|-----------|------|
| PostgreSQL / PGVector | 基於 PostgreSQL 的擴充套件 |
| Redis | 使用 Redis Stack 的向量功能 |
| Chroma | 開源嵌入式向量資料庫 |
| Milvus | 高效能分散式向量資料庫 |
| Elasticsearch | 搜尋引擎的向量搜尋功能 |
| Pinecone | 雲端託管向量資料庫 |
| Qdrant | 高效能向量搜尋引擎 |
| Weaviate | AI 原生向量資料庫 |

### 使用 PGVector 範例

新增依賴：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

配置：

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/vectordb
    username: postgres
    password: password
  ai:
    vectorstore:
      pgvector:
        dimensions: 1536
        index-type: hnsw
```

### 使用 VectorStore

```java
@Service
public class DocumentService {

    private final VectorStore vectorStore;

    public DocumentService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    // 新增文件
    public void addDocuments(List<String> texts) {
        List<Document> documents = texts.stream()
            .map(text -> new Document(text))
            .toList();
        vectorStore.add(documents);
    }

    // 相似度查詢
    public List<Document> search(String query) {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(5)             // 返回最相似的 5 個結果
                .similarityThreshold(0.7)  // 相似度閾值
                .build()
        );
    }
}
```

## Document 與 Metadata

Spring AI 的 `Document` 物件支援後設資料（metadata），可用於過濾查詢：

```java
// 建立帶有後設資料的文件
Document doc = new Document(
    "Spring Boot 3.0 需要 Java 17 以上版本",
    Map.of(
        "source", "official-docs",
        "category", "spring-boot",
        "version", "3.0"
    )
);

vectorStore.add(List.of(doc));
```

### 使用 Metadata Filter 查詢

```java
List<Document> results = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("Spring Boot 版本需求")
        .topK(5)
        .filterExpression("category == 'spring-boot' AND version == '3.0'")
        .build()
);
```

Filter 語法支援：
- 比較：`==`, `!=`, `>`, `>=`, `<`, `<=`
- 邏輯：`AND`, `OR`, `NOT`
- 集合：`IN`, `NIN`

## ETL Pipeline（文件匯入）

Spring AI 提供 ETL 管線，用於讀取、分割和匯入文件：

```java
@Service
public class DocumentIngestionService {

    private final VectorStore vectorStore;

    public void ingestPdf(Resource pdfResource) {
        // 1. 讀取 PDF
        PagePdfDocumentReader reader = new PagePdfDocumentReader(pdfResource);
        List<Document> documents = reader.get();

        // 2. 分割文件（避免超出 Token 限制）
        TokenTextSplitter splitter = new TokenTextSplitter();
        List<Document> chunks = splitter.apply(documents);

        // 3. 儲存到向量資料庫（自動計算 Embedding）
        vectorStore.add(chunks);
    }
}
```

### 支援的文件讀取器

| 讀取器 | 說明 |
|--------|------|
| PagePdfDocumentReader | PDF 文件 |
| TextReader | 純文字檔案 |
| JsonReader | JSON 文件 |
| TikaDocumentReader | 支援多種格式（Word、Excel 等） |

## 小結

Embedding 和向量資料庫是 AI 應用的重要基礎設施。Spring AI 透過統一的 `EmbeddingModel` 和 `VectorStore` 介面，讓開發者可以輕鬆實現語義搜尋和文件管理，為 RAG 應用奠定基礎。
