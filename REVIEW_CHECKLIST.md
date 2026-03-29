# LearningNotes 內容審查 Checklist

> 逐篇檢視每篇文章的內容完整度、正確性、以及與官方最新文件的落差。
> 標記說明：`[ ]` 待檢查 / `[x]` 已確認無誤 / `[!]` 需要更新 / `[-]` 不適用

---

## Algorithm（9 篇 + 1 篇練習） — 已審查

基準：經典演算法，無版本問題，重點在正確性與完整度。

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 氣泡排序和快速排序 | `[x]` 時間複雜度（有：最壞/平均） `[!]` 空間複雜度（無） `[!]` 穩定性（無） `[x]` 程式碼可執行 `[x]` 圖解（4張） `[x]` 含效能對比測試 |
| 02 | 直接選擇排序和堆排序 | `[x]` 時間複雜度（有：最壞/平均） `[!]` 空間複雜度（無） `[!]` 穩定性（無） `[x]` 程式碼可執行 `[x]` 圖解（9張，建堆過程詳細） |
| 03 | 直接插入排序和希爾排序 | `[!]` 時間複雜度（無，僅提及希爾是改進版） `[!]` 空間複雜度（無） `[!]` 穩定性（無） `[x]` 程式碼可執行 `[x]` 含效能對比測試 |
| 04 | 歸併排序 | `[!]` 時間複雜度（無） `[!]` 空間複雜度（無） `[!]` 穩定性（無） `[!]` **程式碼有 BUG**：merge 方法複製資料回原陣列時索引邏輯錯誤 `[x]` 圖解（2張） |
| 05 | 順序查詢 | `[!]` 複雜度分析（無） `[!]` 適用場景（無） `[x]` 程式碼可執行 `[x]` 圖解（2張）— 內容過於簡潔 |
| 06 | 二分查詢 | `[x]` 使用左閉右閉寫法 `[!]` 無邊界條件討論 `[!]` 無變體（第一個/最後一個） `[x]` 有 O(log n) 複雜度 `[x]` 程式碼可執行 |
| 07 | 索引查詢 | `[x]` 分塊查詢原理 `[x]` 索引表設計（IndexItem） `[!]` 無複雜度分析 `[x]` 程式碼可執行 `[x]` 圖解 |
| 08 | 二叉排序樹 | `[x]` 插入/查詢有實作 `[!]` 刪除只有概念說明無程式碼 `[!]` 未提及退化為鏈表 `[!]` 未提及 AVL/紅黑樹 `[x]` 圖解（5張） |
| 09 | 雜湊查詢 | `[x]` 雜湊函數（6種方法） `[x]` 衝突解決（開放定址+鏈結法） `[!]` 未提及載入因子/rehash `[x]` 程式碼可執行 |
| 附 | 整形陣列劃分 | `[!]` 未提及荷蘭國旗問題 `[!]` 未使用三指標法（僅雙指標） `[!]` **演算法錯誤**：遇到零時指標不推進，導致排序結果不正確 `[!]` 無複雜度分析 |

**共通問題（全 9 篇排序/查詢文章）**：
- `[!]` **全部缺少空間複雜度**
- `[!]` **全部缺少穩定性說明**（排序篇）
- `[!]` 04 歸併排序 merge 方法有程式碼 BUG
- `[!]` 附篇整形陣列劃分演算法根本性錯誤

**缺漏評估**：
- `[!]` 缺少：計數排序、基數排序、桶排序
- `[!]` 缺少：七大排序的綜合比較表（時間/空間/穩定性）
- `[!]` 缺少：動態規劃、貪心演算法、回溯等常見演算法分類

---

## Data Structure（10 篇） — 已審查

基準：經典資料結構，無版本問題。

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 時間複雜度和空間複雜度 | `[x]` Big-O 表示法（完整） `[x]` 常見複雜度列表 `[!]` 無比較圖表 `[!]` 無主定理 `[x]` 3 個 Java 範例 |
| 02 | 順序表 | `[x]` 完整 SequenceList 實作（5 個核心操作） `[!]` 無動態擴容（固定大小） `[!]` 未提及 ArrayList |
| 03 | 連結串列 | `[x]` **四種變體全覆蓋**（單向/循環單向/雙向/循環雙向） `[!]` 未提及 Java LinkedList `[x]` 9 張圖解 `[x]` 順序 vs 連結比較 |
| 04 | 連結串列面試題 | `[x]` 反轉鏈表 `[x]` 合併有序鏈表 `[!]` **缺少環偵測** `[x]` 5 題完整實作含圖解（15+ 張） |
| 05 | 棧 | `[x]` 陣列實作完整（7 個操作） `[!]` 無鏈式棧實作 `[!]` **無應用場景**（括號匹配、表達式求值都沒有） |
| 06 | 佇列 | `[x]` 基本佇列定義和實作 `[!]` **無循環佇列** `[!]` **無優先佇列** `[!]` **無 Deque** — 僅有最基本的順序佇列 |
| 07 | 二叉樹 | `[x]` 四種遍歷全覆蓋（前/中/後/層序） `[!]` 全部用遞迴，**無迭代實作** `[x]` 6 張圖解 `[x]` 完整/滿二叉樹區分 |
| 08 | 線索二叉樹 | `[x]` 中序線索化過程完整 `[!]` 僅中序，無前序/後序 `[!]` 實際應用場景僅簡要提及 `[x]` 3 張圖解 |
| 09 | 哈夫曼樹 | `[x]` 建構過程（4 步驟） `[x]` 哈夫曼編碼概念 `[!]` **compareTo() 有 BUG**：`that.value - that.value` 應為 `this.value - that.value` `[!]` 無壓縮應用實例 |
| 10 | 圖 | `[x]` 定義和術語完整 `[x]` DFS/BFS 圖解（7 張） `[!]` **完全沒有程式碼**（僅文字描述） `[!]` 無最短路徑 `[!]` 無最小生成樹 |

**嚴重問題**：
- `[!]` **09 哈夫曼樹 compareTo() BUG** — `that.value - that.value` 永遠為 0，排序失效
- `[!]` **10 圖** — 整篇無程式碼實作，建議重寫
- `[!]` **06 佇列** — 內容過於基礎，缺少循環佇列/優先佇列/Deque，建議重寫

**缺漏評估**：
- `[!]` 缺少：紅黑樹 / AVL 樹
- `[!]` 缺少：B 樹 / B+ 樹（資料庫索引相關）
- `[!]` 缺少：Trie（字典樹）
- `[!]` 缺少：堆（Heap）的獨立篇章（目前只在排序中提及）

---

## Java Basis（19 篇 + 2 篇） — 已審查

基準：Java SE 官方文件。注意 Java 8 → 17 → 21 的演進。

| # | 文章 | 審查結果 |
|---|------|---------|
| 001 | 資料型別 | `[x]` 基本/引用分類正確 `[x]` boolean JVM 規範說明 `[!]` 無 Java 版本標注 `[!]` 無 var 關鍵字 `[-]` 無程式碼（資訊性文章可接受） |
| 002 | String 不可繼承 | `[x]` final class 原理正確 `[x]` static vs final 範例 `[!]` 無 String Pool `[!]` 無 Text Blocks |
| 003 | String/StringBuffer/StringBuilder | `[x]` 可變性正確 `[x]` 執行緒安全分析正確 `[x]` synchronized 原始碼 `[!]` 無量化效能基準 |
| 004 | ArrayList vs LinkedList | `[x]` 結構差異 `[x]` 時間複雜度 `[x]` 有基準測試程式碼 `[!]` 無 List.of() `[!]` ArrayList 擴容公式已過時 |
| 005 | 類的例項化順序 | `[x]` 順序正確 `[!]` **內容過於簡陋**（僅列表，無程式碼、無繼承範例、無初始化塊說明）— 建議重寫 |
| 006 | HashMap 多執行緒 | `[x]` 4 種方案 `[!]` **ConcurrentHashMap 架構是 Java 7 版**（Segment） `[!]` 無 Java 8 紅黑樹 `[!]` 無 Java 17+ 變化 |
| 007 | 值傳遞 vs 引用傳遞 | `[x]` 核心答案正確（Java 只有值傳遞） `[x]` 基本/引用型別範例 `[x]` 教學品質良好 |
| 008 | 模擬 C# ref | `[x]` 包裝類方案正確 `[!]` 無 AtomicReference `[!]` 無陣列包裝方案 — 觀點片面 |
| 009 | 堆和棧的區別 | `[x]` 基本概念正確 `[!]` String Pool 描述過時（Java 7+ 已移到 heap） `[!]` 無 JVM 交叉引用 `[!]` 無逃逸分析 |
| 010 | TreeMap/LinkedHashMap | `[x]` 三者區別概念正確 `[!]` **完全沒有程式碼** `[!]` 無 NavigableMap `[!]` 無 SequencedMap `[!]` 無 LRU 快取模式 |
| 011 | 抽象類 vs 介面 | `[x]` 10 點差異列表 `[!]` **完全沒有程式碼** `[!]` **第 3 點過時**（Java 8 後介面可有具體方法） `[!]` 無 sealed 介面 |
| 012 | 繼承 vs 聚合 | `[x]` 基本定義正確 `[!]` **完全沒有程式碼** `[!]` 無「組合優於繼承」原則 `[!]` 無設計模式範例 |
| 013 | BIO/NIO/AIO | `[x]` BIO/NIO 架構概念 `[!]` **完全沒有程式碼** `[!]` AIO 被跳過 `[!]` 無 Channel/Buffer/Selector `[!]` 無 Netty |
| 014 | 反射機制 | `[x]` 3 種取得 Class 方式 `[x]` API 方法列表完整 `[!]` Class.forName 範例語法錯誤（多了 `.class`） `[!]` 無 Module System 限制 |
| 015 | Class.forName vs ClassLoader | `[x]` 初始化差異正確 `[!]` 無 SPI 機制 `[!]` 無 Java 9+ 模組可見性 `[!]` 圖片連結損壞 |
| 016 | 動態代理 | `[x]` JDK/cglib 範例完整可執行 `[x]` InvocationHandler 正確 `[!]` AOP 關聯僅一句話帶過 `[!]` 圖片連結損壞 |
| 017 | 靜態/JDK/cglib 代理 | `[x]` 三種模式完整實作 `[!]` 有全形分號語法錯誤 `[!]` **無比較表** `[!]` 無 Spring 選擇策略 |
| 018 | final 關鍵字 | `[x]` 三種用途 `[!]` **無程式碼** `[!]` 無 effectively final `[!]` 無 JVM 最佳化說明 |
| 019 | 單例模式 5 種寫法 | `[x]` 5 種模式完整程式碼 `[x]` volatile DCL `[x]` enum 寫法 `[!]` 無反序列化攻擊防護 `[!]` 無反射攻擊防護 `[!]` 無比較表 |
| 附 | 許可權管理思路 | `[ ]` 待審查 |
| 附 | 過濾器 vs 攔截器 | `[ ]` 待審查 |

**嚴重問題**：
- `[!]` **005 類的例項化順序** — 內容僅一個列表，無程式碼無範例，建議重寫
- `[!]` **006 ConcurrentHashMap** — 架構描述停留在 Java 7（Segment），Java 8+ 已改為 CAS + synchronized
- `[!]` **011 抽象類 vs 介面** — 第 3 點「介面只能宣告方法」在 Java 8 後已不正確
- `[!]` **010、011、012、013、018** — 5 篇完全沒有程式碼範例

**共通問題**：
- `[!]` 全部文章未標注 Java 版本，內容停留在 Java 7 以前
- `[!]` 多篇依賴外部圖片連結（GitHub FigureBed），可能已損壞

**缺漏評估**：
- `[!]` 缺少：Lambda 與 Stream API（Java 8 核心）
- `[!]` 缺少：Optional 類別
- `[!]` 缺少：Record 類別（Java 16+）
- `[!]` 缺少：Pattern Matching / Switch Expression（Java 17+）
- `[!]` 缺少：Virtual Threads（Java 21）
- `[!]` 缺少：集合框架總覽（Collection/Map 體系圖）
- `[!]` 缺少：Exception 處理機制
- `[!]` 缺少：泛型（Generics）
- `[!]` 缺少：多執行緒與並行（Thread/Executor/CompletableFuture）

---

## JavaWeb（3 篇） — 已審查

基準：Jakarta EE（原 Java EE）。注意 javax → jakarta 的遷移。

| # | 文章 | 審查結果 |
|---|------|---------|
| 附 | JSP 和 Servlet | `[!]` **未標注 JSP 已過時** `[!]` 無 Jakarta 命名空間變更 `[!]` 無現代替代方案（Thymeleaf 等） `[x]` 生命週期流程有圖解 |
| 附 | Listener | `[x]` 6 種 Listener 類型完整覆蓋 `[x]` 實用範例（線上人數統計） `[!]` 程式碼缺少結尾大括號 `[!]` 無 Spring Events 替代方案 `[!]` 靜態計數器執行緒安全未處理 |
| 附 | Filter | `[x]` Filter Chain 原理正確 `[x]` 2 個實用範例（登入驗證、編碼設定） `[x]` URL 模式規則 `[!]` 無 Spring Security Filter Chain |

**缺漏評估**：
- `[!]` 缺少：現代 JavaWeb 不再使用 JSP 的說明
- `[!]` 缺少：javax.* → jakarta.* 遷移指引
- `[!]` 全部文章停留在 2018 年，使用 javax.servlet API

---

## JVM（3 篇） — 已審查

基準：Oracle JVM 規範 / OpenJDK 文件。注意 G1/ZGC/Shenandoah 等新 GC。

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 記憶體區域 | `[x]` 堆/棧/方法區/PC 計數器基礎正確 `[!]` **無 Metaspace**（仍用永久代術語） `[!]` 無直接記憶體 `[x]` 基礎範例程式碼 `[!]` 5 張圖片連結損壞 |
| 02 | 類載入機制 | `[x]` 雙親委派模型解釋清晰 `[x]` 5 個載入階段正確 `[!]` 無自定義 ClassLoader 範例 `[!]` **無 Module System (Java 9+)** `[!]` 無程式碼 `[!]` 2 張圖片損壞 |
| 03 | 垃圾回收 | `[x]` 3 種演算法說明正確 `[x]` 分代收集概念 `[x]` 實用 GC 最佳化建議 `[!]` **G1 僅列名未說明** `[!]` **無 ZGC** `[!]` **無調優參數** `[!]` CMS 已 Java 14 移除未標注 `[!]` 4 張圖片損壞 |

**嚴重問題**：
- `[!]` 3 篇全部停留在 Java 7 時代，缺少 Java 8+ Metaspace、Java 9+ Module System、Java 15+ ZGC
- `[!]` 共 11 張外部圖片連結全部損壞（GitHub FigureBed CDN）
- `[!]` 03 垃圾回收缺少所有調優參數（-Xms/-Xmx/-Xmn 等）

**缺漏評估**：
- `[!]` 缺少：JVM 調優實戰（jstat/jmap/jstack/Arthas）
- `[!]` 缺少：JIT 編譯與分層編譯
- `[!]` 缺少：GraalVM / Native Image

---

## Struts2（1 篇） — 已審查

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 配置和簡單例子 | `[x]` 完整端到端範例（Maven → web.xml → Action → struts.xml → JSP） `[x]` 2.5 版本差異有標注 `[!]` **未標注 Struts2 已過時** `[!]` **未提及 OGNL 注入等嚴重安全漏洞** `[!]` 無 Spring MVC 遷移建議 |

---

## Hibernate（4 篇） — 已審查

基準：Hibernate ORM 6.x / Jakarta Persistence 3.x。

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 配置和使用 | `[x]` 明確標注 Hibernate 4.x `[x]` 有引導到 Spring Data JPA `[x]` 完整 CRUD 範例 `[x]` 交易管理正確 — 品質優良 |
| 02 | 從 Hibernate 到 Spring Data JPA | `[x]` Repository 介面 `[x]` 12+ 查詢方法命名範例 `[x]` @Query JPQL + Native SQL `[x]` 分頁排序 `[x]` Before/After 比較表 — **品質最佳** |
| 附 | 關聯對映 | `[x]` 4 種關聯全覆蓋（1:1/N:1/1:N/N:N） `[x]` 雙向關聯 `[x]` inverse 屬性 `[!]` **無 FetchType 討論** `[!]` **無 N+1 問題** `[!]` 僅 XML 無 JPA 註解 |
| 附 | 繼承對映 | `[x]` 三種策略完整實作 `[x]` ID 生成限制有標注 `[!]` **無策略選擇建議/比較表** `[!]` 無效能取捨分析 `[!]` 僅 XML 無 JPA 註解 |

---

## Spring（8 篇） — 已審查

基準：**Spring Framework 6.x** 官方文件 https://docs.spring.io/spring-framework/reference/

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 簡單配置和使用 | `[x]` 明確標注 Spring 3.x `[x]` 有連結引導到 06 現代方式 `[x]` 完整範例 — 品質達標（舊版教學） |
| 02 | DI & IoC | `[x]` IoC 原理清晰 `[x]` 三種注入完整 `[x]` 建構子注入標為推薦 `[x]` Bean Scope 4 種 `[x]` Spring 6.x 對齊 — **品質優良** |
| 03 | AOP | `[x]` 標注 Spring 4.x `[x]` 連結引導到 07 註解方式 `[x]` 8 個術語完整 `[x]` 多場景 XML 範例 `[x]` JDK/CGLIB 代理選擇 |
| 04 | 父子容器 | `[x]` WebApplicationContext 架構正確 `[x]` 元件掃描 exclude/include 範例 `[!]` **無版本標注** `[!]` **未提及 Spring Boot 是否仍用父子容器** `[!]` 無交叉引用 |
| 05 | @Autowired/@Resource/@Service | `[x]` @Qualifier 有覆蓋 `[x]` @Resource vs @Autowired 差異 `[!]` 全部範例用欄位注入（未推薦建構子注入） `[!]` 無 @Primary `[!]` 無 @Inject (JSR-330) — 過時 |
| 06 | Java 配置與註解驅動 | `[x]` @Configuration/@Bean `[x]` @ComponentScan `[x]` @Profile（dev/prod 範例） `[x]` 建構子注入推薦 `[x]` @Qualifier vs @Resource 比較表 `[x]` Spring 6.x 對齊 — **品質優良** |
| 07 | 註解式 AOP | `[x]` 5 種 Advice 全覆蓋 `[x]` @Order 執行順序 `[x]` 4 種切入點表達式 `[x]` 自定義註解+重試實例 `[x]` Spring 6.x 對齊 — **品質最佳** |
| 附 | 程式設計式/宣告式事務 | `[x]` 7 種傳播行為完整 `[x]` 5 種隔離級別完整 `[x]` 回滾規則 `[!]` **無 @Transactional 屬性實際範例** `[!]` 無 Spring 6.x 變化 — 2018 年版本 |

**品質分級**：
- 優良（Spring 6.x 對齊）：02、06、07
- 達標（舊版但有版本標注）：01、03
- 需更新：04（無版本標注/無 Boot）、05（欄位注入為主）、事務篇（缺實際範例）

**缺漏評估**：
- `[!]` 缺少：Spring Bean 生命週期（@PostConstruct / @PreDestroy / BeanPostProcessor）
- `[!]` 缺少：Spring Events（事件機制）
- `[!]` 缺少：SpEL（Spring Expression Language）
- `[!]` 缺少：Spring 6.x 的重大變更（Jakarta EE 9+、AOT、HTTP Interface）

---

## Spring MVC（4 篇） — 已審查

基準：**Spring Framework 6.x** MVC 文件。

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 配置和使用 | `[x]` 明確標注過時（Spring 3.x） `[x]` 引導到 02-04 `[x]` 完整範例 `[!]` MySQL driver 過時（com.mysql.jdbc.Driver） |
| 02 | 註解驅動與 RESTful | `[x]` @RestController `[x]` @RequestBody/@ResponseBody `[x]` HttpMessageConverter + Jackson `[x]` ResponseEntity `[x]` 版本比較表 — **品質優良** |
| 03 | ��外處理與驗證 | `[x]` @ExceptionHandler `[x]` @RestControllerAdvice `[x]` Jakarta Validation 註解完整 `[x]` 自定義驗證器 `[x]` 分組驗證 — **品質優良** |
| 04 | 攔截器與跨域 | `[x]` HandlerInterceptor 生命週期 `[x]` @CrossOrigin 3 種配置 `[x]` CorsConfiguration `[x]` WebMvcConfigurer 6 個方法 `[x]` JWT 驗證範例 — **品質優良** |

**品質分級**：02、03、04 品質優良（Spring Boot 3.x 對齊），01 舊版但標注清楚

**缺漏評估**：
- `[!]` 缺少：WebFlux（響應式 Web 框架）
- `[!]` 缺少：檔案上傳/下載
- `[!]` 缺少：Spring MVC 與 Spring Boot 的整合差異

---

## Mybatis（3 篇） — 已審查

基準：**MyBatis 3.5.x** 官方文件 https://mybatis.org/mybatis-3/

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 配置和使用 | `[x]` 標注過時（MyBatis 3.2.7） `[x]` 引導到 03 `[x]` XML 映射基礎 `[!]` 無動態 SQL `[!]` MySQL driver/connector 過時 |
| 02 | 逆向工程 | `[x]` 標注過時 `[x]` 完整 generatorConfig.xml `[!]` 無 MyBatis Generator 版本號 `[!]` **無 MyBatis-Plus** `[!]` MySQL driver 過時 |
| 03 | 與 Spring Boot 整合 | `[x]` starter 3.0.3（最新） `[x]` 註解 vs XML 雙覆蓋 `[x]` 動態 SQL（if/foreach/choose/set） `[x]` PageHelper 分頁 `[x]` 版本比較表 — **品質優良** |

**品質分級**：03 品質優良，01/02 舊版但有標注

**缺漏評估**：
- `[!]` 缺少：MyBatis-Plus（增強框架）
- `[!]` 缺少：一級/二級快取
- `[!]` 缺少：攔截器（Plugin/Interceptor）

---

## Spring Boot（5 篇） — 已審查

基準：**Spring Boot 3.x** 官方文件 https://docs.spring.io/spring-boot/reference/

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 簡單配置和使用 | `[x]` @SpringBootApplication 提及 `[!]` **無版本標注** `[!]` 內嵌伺服器未詳細說明 `[!]` 內容過於簡略 `[!]` 7 張圖片連結損壞 — 2018 年舊版，建議重寫 |
| 02 | 自動配置與 Starters | `[x]` @EnableAutoConfiguration 原理 `[x]` AutoConfiguration.imports（Boot 3.x） `[x]` 條件註解完整 `[x]` 自定義 Starter 範例 `[x]` Boot 3.3.0 — **品質優良** |
| 03 | 配置檔案與 Profiles | `[x]` yml/properties 雙格式 `[x]` @ConfigurationProperties 含巢狀物件 `[x]` 3 種 Profile 切換 `[x]` @Validated 驗證 — **品質優良** |
| 04 | 開發 RESTful API | `[x]` @RestController `[x]` 統一回應格式（Java record） `[x]` 全域例外處理 `[x]` 驗證整合 `[!]` **無 Swagger/SpringDoc** — 重要缺漏 |
| 05 | Actuator 監控 | `[x]` Health/Info/Metrics `[x]` 自定義 HealthIndicator `[x]` Prometheus 整合 `[x]` Spring Security 配置 `[x]` 動態日誌級別 — **品質優良** |

**品質分級**：02、03、05 優良；04 良好但缺 API 文件工具；01 過時需重寫

**缺漏評估**：
- `[!]` 缺少：Spring Boot 3.x 的重大變更（Java 17 最低要求、jakarta 遷移、GraalVM 原生映像）
- `[!]` 缺少：Spring Boot Testing（@SpringBootTest / @WebMvcTest / @DataJpaTest）
- `[!]` 缺少：Spring Security 整合
- `[!]` 缺少：Docker 容器化部署
- `[!]` 04 缺少：Swagger/SpringDoc（OpenAPI 3.0）

---

## Spring Cloud（7 篇） — 已審查

基準：**Spring Cloud 2023.x** 官方文件。注意 Netflix OSS 元件的替代。

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 概述與微服務架構 | `[x]` 微服務 vs 單體完整 `[x]` 版本對應表（Boot 3.3.0 / Cloud 2023.0.2） `[x]` 8 個核心元件 `[x]` Maven 專案結構 — **品質優良** |
| 02 | Eureka | `[x]` Server/Client 完整 `[x]` HA 叢集 `[x]` 自我保護機制 `[x]` Nacos 替代方案含程式碼 `[!]` 未標注 Eureka 維護模式 |
| 03 | Config | `[x]` Server/Client 完整 `[x]` Git 儲存結構 `[x]` @RefreshScope + Bus 動態刷新 `[x]` 加密配置 `[x]` Nacos Config 替代 — **品質優良** |
| 04 | Gateway | `[x]` 路由/述詞/過濾器三核心 `[x]` YAML + Java 配置 `[x]` Redis 限流 `[x]` 熔斷器整合 `[x]` 全域 Auth 過濾器 `[!]` 無 Zuul 比較 |
| 05 | Resilience4j | `[x]` 4 大功能全覆蓋（CB/RL/Retry/Bulkhead） `[x]` 取代 Hystrix `[x]` 組合使用順序 `[x]` 註解方式完整 — **品質優良** |
| 06 | LoadBalancer | `[x]` 客戶端/伺服器端比較 `[x]` 取代 Ribbon `[x]` 內建 + 自定義策略 `[x]` Health check `[x]` WebClient 整合 — **品質優良** |
| 07 | OpenFeign | `[x]` 介面定義 `[x]` LoadBalancer 自動整合 `[x]` Fallback/FallbackFactory `[x]` 攔截器/壓縮/日誌 `[x]` 比較表 `[!]` **無 Spring 6 HTTP Interface** |

**整體品質**：Spring Cloud 系列品質整體優良，是所有分類中最完整的

**缺漏評估**：
- `[!]` 缺少：分散式追蹤（Micrometer Tracing / Zipkin，取代 Sleuth）
- `[!]` 缺少：分散式事務（Seata / Saga 模式）
- `[!]` 缺少：訊息驅動（Spring Cloud Stream）
- `[!]` 缺少：Spring Cloud 與 Kubernetes 原生整合

---

## Spring AI（7 篇） — 已審查

基準：**Spring AI 1.0.x** 官方文件 https://docs.spring.io/spring-ai/reference/ （仍在快速迭代）

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 概述與快速開始 | `[x]` 8 個模型提供者 `[x]` Spring AI 1.0.0 / Boot 3.3.0 `[x]` 同步+串流範例 `[!]` Claude 版本宣稱未驗證 |
| 02 | ChatClient API | `[x]` Fluent API 完整 `[x]` SystemMessage/UserMessage/AssistantMessage `[x]` 串流 SSE `[x]` PromptTemplate — **品質優良** |
| 03 | Structured Output | `[x]` entity() 方法 `[x]` BeanOutputConverter `[x]` ListOutputConverter `[x]` MapOutputConverter `[!]` JSON Schema 細節不足 `[!]` Native Structured Output 說明簡略 |
| 04 | Embedding 與向量 DB | `[x]` EmbeddingModel 介面 `[x]` 8 個向量 DB `[x]` 相似度搜尋 + 篩選 `[x]` PGVector 配置 `[x]` Document Reader 列表 — **品質優良** |
| 05 | RAG | `[x]` Reader/Transformer/Writer 管線 `[x]` QuestionAnswerAdvisor `[x]` 分段策略（800 tokens/200 overlap） `[!]` filterExpression 有 SQL injection 風險 |
| 06 | Function Calling | `[x]` @Bean+@Description 模式 `[x]` FunctionCallback 模式 `[x]` 安全建議 `[!]` **未使用 @Tool 註解**（1.0.0 新 API） — 版本不一致 |
| 07 | Advisors API 與對話記憶 | `[x]` Advisor 鏈組合 `[x]` InMemory/JDBC ChatMemory `[x]` MessageChatMemoryAdvisor `[x]` 自定義 Advisor `[x]` 視窗控制 — **品質優良** |

**嚴重問題**：
- `[!]` **06 Function Calling** — 宣稱 Spring AI 1.0.0 但使用舊版 @Bean+@Description 模式，應更新為 @Tool 註解
- `[!]` **05 RAG** — filterExpression 使用字串拼接，有注入風險

**缺漏評估（Spring AI 變化極快，以下為 1.0.x 新增）**：
- `[!]` 缺少：MCP（Model Context Protocol）整合
- `[!]` 缺少：Multimodal（多模態：圖片/音訊）
- `[!]` 缺少：Evaluation（模型輸出評估）
- `[!]` 缺少：Agent / Agentic Workflow

---

## Maven（1 篇） — 已審查

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | xml 配置新增到編譯目錄 | `[x]` resources 配置語法正確，仍適用 Maven 3.x+ `[!]` 內容極度簡略（39 行） `[!]` 無版��標注 `[!]` 無 filtering/includes/excludes 說明 `[!]` 未解釋 src/main/resources 預設目錄 |

**缺漏評估**：
- `[!]` 缺少：Maven 生命週期與常用指令
- `[!]` 缺少：多模組專案
- `[!]` 缺少：Gradle 替代方案介紹

---

## MySQL（1 篇） — 已審查

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | 儲存過程 | `[x]` 語法範例完整（IN/OUT/INOUT、CRUD） `[x]` 基本相容 MySQL 8.x `[!]` `int(11)` display width 在 8.0+ 已被忽略 `[!]` 使用 `mysql.proc`（8.0 已移除���應用 INFORMATION_SCHEMA.ROUTINES） `[!]` 無錯誤處理 `[!]` 無版本標注 |

**缺漏評估**：
- `[!]` 缺少：索引原理（B+樹 / 覆蓋索引 / 索引失效）
- `[!]` 缺少：SQL 優化與 EXPLAIN
- `[!]` 缺少：交易與鎖機制（InnoDB）
- `[!]` 缺少：MySQL 8.x 新特性（Window Function / CTE / JSON）

---

## React（6 篇） — 已審查

基準：**React 19** 官方文件 https://react.dev

| # | 文章 | 審查結果 |
|---|------|---------|
| 01 | React 元件 | `[x]` 明確標注 React 15.x 已廢棄 `[x]` 連結引導到 06 `[x]` CDN 範例完整 — 舊版標注達標 |
| 02 | React State | `[x]` setState 原理正確 `[x]` 有引導到 useState Hook `[!]` 未提及 setState 非同步/批次特性 |
| 03 | React Props | `[x]` 15+ 種 PropTypes 驗證器 `[x]` 有提及 TypeScript 替代 `[!]` React.PropTypes 在範例中已是廢棄 API `[!]` 無 TypeScript 範例 |
| 04-05 | 音樂相簿專案 | `[!]` **完全無法執行**：React.createClass 已移除、Webpack 2 配���過時、jPlayer 已停止維護 `[!]` 使用 jQuery 直接操作 DOM — 僅供歸檔 |
| 06 | 函式元件與 Hooks | `[x]` useState/useEffect 完整 `[x]` 新舊對照比較 `[x]` Vite 建置 `[x]` 完整 Todo 範例 `[!]` 無自定義 Hook 範例 `[!]` �� React 18+ 功能��Suspense/useTransition） `[!]` useContext 僅表格提及 — **品質良好** |

**品��分級**：06 品質良好；01-03 舊版但標注��楚；04-05 過時無法執行

**缺漏評估**：
- `[!]` 缺少：React Router
- `[!]` 缺少：狀態管理（Context API / Zustand / Redux Toolkit）
- `[!]` 缺少：Next.js / Vite 等現代建置工具
- `[!]` 缺少：Server Components（React 19）
- `[!]` 缺少：TypeScript 整合

---

## Spider / nguSeckill / Interview / 其他 — 已審查

| 章節 | 審查結果 |
|------|---------|
| Spider | `[ ]` 待審查（Python 爬蟲，低優先級） |
| nguSeckill | `[x]` 併發概念（交易隔離、行鎖）仍有參考價值 `[!]` SSM 技術棧過時（pre-Spring Boot） `[!]` 無 HikariCP/Redis/非同步方案 `[!]` 無法直接執行 — 僅供學習併發概念 |
| Interview | `[x]` 基礎 Java 答案正確 `[!]` **完全無 Java 8+ 內容**（無 Lambda/Stream/Optional） `[!]` 2018 年版，缺少 50% 現代面試題 — 需大幅補充 |
| 劍指 offer | `[ ]` 待審查（題解類，低優先級） |
| 詩集/排版 | `[-]` 非技術內容，免檢 |

---

## 全局檢查項目 — 已審查

| 項目 | 狀態 |
|------|------|
| `[!]` 所有文章是否標注基於的技術版本 | 新文章（Spring 06/07、Spring MVC 02-04、Spring Boot 02-05、Cloud、AI）有標注；舊文章（Java Basis、JVM、Data Structure、Algorithm）大多無版本標注 |
| `[!]` 過時技術是否有明確警示 | Struts2 **無警示**（最嚴重）；Spring 01/03、React 01-03、Mybatis 01-02 有標注；JSP **無警示** |
| `[x]` 文章間的交叉引用是否正確 | Spring 系列交叉引用完整；React 01-03 都連結到 06 |
| `[!]` 圖片連結是否仍可存取 | JVM 3 篇共 11 張圖片**全部損壞**（GitHub FigureBed CDN）；Java Basis 多篇也損壞 |
| `[!]` 程式碼範例是否可直接執行 | Algorithm 04 歸併排序有 BUG；附篇整形陣列劃分演算法錯誤；Data Structure 09 哈夫曼樹 compareTo BUG；React 04-05 完全無法執行 |
| `[x]` README.md 所有連結是否正確指向本地檔案 | 已在前次對話中完成驗證（110 個連結全部 OK） |

---

## 優先處理建議（已根據審查結果更新）

### 第一優先：修復程式碼錯誤

| 文章 | 問題 | 建議 |
|------|------|------|
| Algorithm 04 歸併排序 | merge 方法索引邏輯 BUG | 重寫 |
| Algorithm 附篇 整形陣列劃分 | 演算法根本性錯誤（雙指標無法處理零） | 重寫（改用三指標荷蘭國旗） |
| Data Structure 09 哈夫曼樹 | compareTo() `that.value - that.value` 永遠為 0 | 修正為 `this.value - that.value` |
| Spring AI 06 Function Calling | 宣稱 1.0.0 但用舊版 API | 更新為 @Tool 註解 |

### 第二優先：重寫過於簡陋/過時的文章

| 文章 | 原因 |
|------|------|
| Java Basis 005 類的例項化順序 | 僅列表，無程式碼無範例 |
| Java Basis 006 HashMap 多執行緒 | ConcurrentHashMap 停留在 Java 7 Segment 架構 |
| Java Basis 011 抽象類 vs 介面 | 核心論點在 Java 8 後已錯誤（介面可有具體方法） |
| Data Structure 06 佇列 | 僅基本順序佇列，無循環/優先/Deque |
| Data Structure 10 圖 | 完全沒有程式碼 |
| Spring Boot 01 簡單配置和使用 | 2018 年舊版，圖片全損 |

### 第三優先：補充安全/版本警示

| 文章 | 需要 |
|------|------|
| Struts2 01 | 加入安全漏洞警示 + Spring MVC 遷移建議 |
| JSP 和 Servlet | 加入 JSP 已過時標注 + Jakarta 命名空間說明 |
| JVM 3 篇 | 加入 Metaspace/Module System/ZGC 說明，修復損壞圖片 |
| Spring 04 父子容器 | 加入版本標注 + Spring Boot 說明 |
| Spring 05 @Autowired | 更新為推薦建構子注入 |

### 第四優先：補充缺漏主題（新文章）

依學習價值排序：
1. Java Basis：Lambda/Stream API、Optional、泛型、多執行緒
2. Algorithm：七大排序綜合比較表
3. Spring Boot：Testing、Spring Security、Swagger/SpringDoc
4. JVM：調優實戰（jstat/jmap/Arthas）
5. MySQL：索引原理、SQL 優化、交易與鎖
6. React：React Router、狀態管理、TypeScript 整合
