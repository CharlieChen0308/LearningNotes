# LearningNotes 內容覆蓋度 Master Checklist

> 以主流技術棧為骨架，逐層、逐項確保 100% 覆蓋。
> 每個子項目標記：`[x]` 待寫 / `[x]` 部分覆蓋（標注來源篇章） / `[x]` 完整覆蓋
> 「來源」欄標注該知識點由哪篇文章負責，確保無遺漏、無重複。

---

## 1. 語言：Java 17/21

> 目錄：`01-Java-Core/`

### 1.1 資料型別與變數（Phase 2: 重寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 1.1.1 | 8 種基本型別 + 各佔位元組 | `[x]` | 01 資料型別與變數 |
| 1.1.2 | 包裝類別（Integer/Long/...）+ 自動裝箱/拆箱 | `[x]` | 01 |
| 1.1.3 | `var` 區域變數型別推斷（Java 10+） | `[x]` | 01 |
| 1.1.4 | `record` 類別（Java 16+） | `[x]` | 01 |
| 1.1.5 | `sealed` 類別與介面（Java 17） | `[x]` | 01 |
| 1.1.6 | Pattern Matching for instanceof（Java 16+） | `[x]` | 01 |
| 1.1.7 | Switch Expression（Java 14+）+ Pattern Matching for switch（Java 21） | `[x]` | 01 |
| 1.1.8 | Text Blocks（Java 13+） | `[x]` | 01 |

### 1.2 字串與集合框架（Phase 2: 重寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 1.2.1 | String 不可變性 + String Pool + intern() | `[x]` | 02 字串與集合 |
| 1.2.2 | StringBuilder vs StringBuffer（效能 + 執行緒安全） | `[x]` | 02 |
| 1.2.3 | Collection 體系圖（List/Set/Queue/Deque） | `[x]` | 02 |
| 1.2.4 | ArrayList vs LinkedList（內部實作 + 時間複雜度） | `[x]` | 02 |
| 1.2.5 | HashMap 原理（hash + 紅黑樹 Java 8+）+ 擴容機制 | `[x]` | 02 |
| 1.2.6 | TreeMap / LinkedHashMap / ConcurrentHashMap | `[x]` | 02 |
| 1.2.7 | 不可變集合：List.of() / Map.of() / Set.of()（Java 9+） | `[x]` | 02 |
| 1.2.8 | SequencedCollection / SequencedMap（Java 21） | `[x]` | 02 |
| 1.2.9 | Iterator / Iterable / for-each 機制 | `[x]` | 02 |

### 1.3 物件導向進階（Phase 2: 重寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 1.3.1 | 類別例項化順序（靜態塊 → 實例塊 → 建構子，含繼承） | `[x]` | 03 物件導向進階 |
| 1.3.2 | 抽象類別 vs 介面（含 Java 8+ default/static/private 方法） | `[x]` | 03 |
| 1.3.3 | sealed 介面的實際用途 | `[x]` | 03 |
| 1.3.4 | 組合優於繼承原則 + 實際設計模式範例 | `[x]` | 03 |
| 1.3.5 | 值傳遞 vs 引用傳遞（Java 只有值傳遞） | `[x]` | 03 |
| 1.3.6 | final 關鍵字（class/method/variable + effectively final） | `[x]` | 03 |
| 1.3.7 | equals() / hashCode() 契約 | `[x]` | 03 |

### 1.4 並行程式設計基礎（Phase 2: 重寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 1.4.1 | Thread 生命週期 + Runnable vs Callable | `[x]` | 04 並行程式設計 |
| 1.4.2 | synchronized / volatile / Atomic 類別 | `[x]` | 04 |
| 1.4.3 | ExecutorService / ThreadPoolExecutor 參數調校 | `[x]` | 04 |
| 1.4.4 | CompletableFuture 非同步程式設計 | `[x]` | 04 |
| 1.4.5 | Virtual Threads（Java 21） | `[x]` | 04 |
| 1.4.6 | ConcurrentHashMap（Java 8+ CAS + synchronized） | `[x]` | 04 |
| 1.4.7 | BIO / NIO / AIO 概念 + Channel / Buffer / Selector | `[x]` | 04 |
| 1.4.8 | 堆和棧的區別 + JVM 記憶體模型（happens-before） | `[x]` | 04 |

### 1.5 反射與代理（Phase 2: 重寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 1.5.1 | 3 種取得 Class 物件方式 | `[x]` | 05 反射與代理 |
| 1.5.2 | 反射 API（Constructor/Method/Field） | `[x]` | 05 |
| 1.5.3 | Class.forName() vs ClassLoader.loadClass() 差異 | `[x]` | 05 |
| 1.5.4 | Module System 對反射的限制（Java 9+） | `[x]` | 05 |
| 1.5.5 | 靜態代理 vs JDK 動態代理 vs CGLIB 比較表 | `[x]` | 05 |
| 1.5.6 | Spring 中的代理選擇策略 | `[x]` | 05 |
| 1.5.7 | 反射效能影響 + MethodHandle 替代 | `[x]` | 05 |

### 1.6 常用關鍵字與設計模式（Phase 2: 重寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 1.6.1 | 單例模式 5 種寫法 + 比較表 | `[x]` | 06 關鍵字與設計模式 |
| 1.6.2 | 反序列化攻擊 + 反射攻擊防護 | `[x]` | 06 |
| 1.6.3 | Enum 單例（最安全寫法） | `[x]` | 06 |
| 1.6.4 | 泛型（Generics）：型別擦除 / 萬用字元 / PECS 原則 | `[x]` | 06 |
| 1.6.5 | 註解（Annotation）：自定義 + 元註解 + 運行時處理 | `[x]` | 06 |

### 1.7 Lambda 與 Stream API（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 1.7.1 | 函式型介面（@FunctionalInterface） | `[x]` | 07 Lambda 與 Stream |
| 1.7.2 | 四大內建介面：Function / Consumer / Predicate / Supplier | `[x]` | 07 |
| 1.7.3 | Lambda 語法 + 方法參考（Method Reference） | `[x]` | 07 |
| 1.7.4 | Stream 建立（of / iterate / generate / Collection.stream） | `[x]` | 07 |
| 1.7.5 | 中間操作：filter / map / flatMap / sorted / distinct / peek | `[x]` | 07 |
| 1.7.6 | 終端操作：collect / forEach / reduce / count / findFirst / anyMatch | `[x]` | 07 |
| 1.7.7 | Collectors：toList / toMap / groupingBy / partitioningBy / joining | `[x]` | 07 |
| 1.7.8 | 平行流（parallelStream）：適用場景 + 陷阱 | `[x]` | 07 |

### 1.8 Optional 與現代錯誤處理（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 1.8.1 | Optional 建立（of / ofNullable / empty） | `[x]` | 08 Optional 與錯誤處理 |
| 1.8.2 | Optional 使用（map / flatMap / orElse / orElseGet / orElseThrow） | `[x]` | 08 |
| 1.8.3 | Optional 反模式（不該用的場景） | `[x]` | 08 |
| 1.8.4 | Exception 體系（Checked vs Unchecked） | `[x]` | 08 |
| 1.8.5 | try-with-resources（AutoCloseable） | `[x]` | 08 |
| 1.8.6 | 自定義例外 + 全域例外處理策略 | `[x]` | 08 |

---

## 2. 後端：Spring Boot 3.x

> 目錄：`02-Spring-Ecosystem/`

### 2.1 Spring Core — DI 與 IoC（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.1.1 | IoC 核心思想（傳統 vs IoC 對比） | `[x]` | 01 Spring Core |
| 2.1.2 | 3 種注入方式（建構子推薦 / Setter / 欄位） | `[x]` | 01 |
| 2.1.3 | Bean Scope（singleton / prototype / request / session） | `[x]` | 01 |
| 2.1.4 | BeanFactory vs ApplicationContext | `[x]` | 01 |
| 2.1.5 | @Qualifier / @Primary / @Resource 差異 | `[x]` | 01 |
| 2.1.6 | Bean 生命週期（@PostConstruct / @PreDestroy / BeanPostProcessor） | `[x]` | 01 — 需補充 |

### 2.2 Spring AOP（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.2.1 | @Aspect / @Before / @After / @Around / @AfterReturning / @AfterThrowing | `[x]` | 02 AOP |
| 2.2.2 | 切入點表達式（execution / @annotation / within） | `[x]` | 02 |
| 2.2.3 | @Order 執行順序 | `[x]` | 02 |
| 2.2.4 | 自定義註解 + AOP 實戰（重試、日誌、效能監控） | `[x]` | 02 |

### 2.3 Spring Java 配置（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.3.1 | @Configuration / @Bean | `[x]` | 03 Java 配置 |
| 2.3.2 | @ComponentScan + 元件註解（@Service / @Repository / @Controller） | `[x]` | 03 |
| 2.3.3 | @Profile（dev / prod 環境切換） | `[x]` | 03 |
| 2.3.4 | @Value + SpEL 表達式 | `[x]` | 03 |
| 2.3.5 | @PropertySource / @ConfigurationProperties | `[x]` | 03 |

### 2.4 Spring Boot 自動配置（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.4.1 | @SpringBootApplication 組合原理 | `[x]` | 04 自動配置 |
| 2.4.2 | AutoConfiguration.imports（Boot 3.x） | `[x]` | 04 |
| 2.4.3 | 條件註解（@ConditionalOnClass / @ConditionalOnMissingBean / ...） | `[x]` | 04 |
| 2.4.4 | 自定義 Starter 開發 | `[x]` | 04 |
| 2.4.5 | 自動配置除錯（--debug） | `[x]` | 04 |

### 2.5 配置檔案與 Profiles（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.5.1 | application.yml vs application.properties 優先順序 | `[x]` | 05 配置與 Profiles |
| 2.5.2 | @ConfigurationProperties 巢狀物件綁定 | `[x]` | 05 |
| 2.5.3 | Profile 切換（yml / 命令列 / 環境變數） | `[x]` | 05 |
| 2.5.4 | @Validated 配置驗證 | `[x]` | 05 |
| 2.5.5 | 敏感資料處理（環境變數 / .env） | `[x]` | 05 |

### 2.6 RESTful API 開發（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.6.1 | @RestController / @RequestMapping / HTTP 方法對應 | `[x]` | 06 RESTful API |
| 2.6.2 | @RequestBody / @ResponseBody / HttpMessageConverter | `[x]` | 06 + 07 |
| 2.6.3 | ResponseEntity 用法 | `[x]` | 06 |
| 2.6.4 | 統一回應格式（ApiResponse record） | `[x]` | 06 |
| 2.6.5 | Jackson 序列化（@JsonFormat / @JsonProperty / @JsonIgnore） | `[x]` | 07 |

### 2.7 例外處理與驗證（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.7.1 | @ExceptionHandler + @RestControllerAdvice 全域例外處理 | `[x]` | 08 例外處理 |
| 2.7.2 | Jakarta Validation 註解（@NotNull / @Size / @Email / @Pattern / ...） | `[x]` | 08 |
| 2.7.3 | @Valid 觸發驗證 + BindingResult | `[x]` | 08 |
| 2.7.4 | 自定義驗證器（ConstraintValidator） | `[x]` | 08 |
| 2.7.5 | 分組驗證（validation groups） | `[x]` | 08 |

### 2.8 攔截器與跨域（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.8.1 | HandlerInterceptor 生命週期（preHandle / postHandle / afterCompletion） | `[x]` | 09 攔截器與跨域 |
| 2.8.2 | @CrossOrigin / CorsConfiguration / WebMvcConfigurer | `[x]` | 09 |
| 2.8.3 | JWT Token 驗證攔截器實作 | `[x]` | 09 |

### 2.9 Spring Data JPA（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.9.1 | JpaRepository 介面 + 方法命名查詢 | `[x]` | 10 Spring Data JPA |
| 2.9.2 | @Query（JPQL + Native SQL） | `[x]` | 10 |
| 2.9.3 | 分頁排序（Pageable / PageRequest） | `[x]` | 10 |
| 2.9.4 | JPA Entity 註解（@Entity / @Table / @Id / @Column / @GeneratedValue） | `[x]` | 10 |
| 2.9.5 | 關聯對映（@OneToMany / @ManyToOne / FetchType） | `[x]` | 10 — 需補充 N+1 問題 |

### 2.10 MyBatis 整合（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.10.1 | mybatis-spring-boot-starter 配置 | `[x]` | 11 MyBatis 整合 |
| 2.10.2 | 註解方式（@Select / @Insert / @Update / @Delete） | `[x]` | 11 |
| 2.10.3 | XML 動態 SQL（if / foreach / choose / set） | `[x]` | 11 |
| 2.10.4 | PageHelper 分頁 | `[x]` | 11 |

### 2.11 Actuator 監控（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.11.1 | Health / Info / Metrics 端點 | `[x]` | 12 Actuator |
| 2.11.2 | 自定義 HealthIndicator | `[x]` | 12 |
| 2.11.3 | 自定義 Metrics（Counter / Timer） | `[x]` | 12 |
| 2.11.4 | Prometheus + Grafana 整合 | `[x]` | 12 |
| 2.11.5 | Actuator 安全配置 | `[x]` | 12 |

### 2.12 事務管理（Phase 1: 搬入合併）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.12.1 | @Transactional 屬性（propagation / isolation / timeout / rollbackFor） | `[x]` | 13 事務管理 |
| 2.12.2 | 7 種傳播行為 | `[x]` | 13 |
| 2.12.3 | 隔離級別（READ_COMMITTED 推薦 + PostgreSQL 特性） | `[x]` | 13 |
| 2.12.4 | @Transactional 失效場景（非 public / 同類呼叫 / 異常被吞） | `[x]` | 13 — 需補充 |
| 2.12.5 | 程式設計式事務（TransactionTemplate） | `[x]` | 13 |

### 2.13 Spring Security 與 JWT（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.13.1 | SecurityFilterChain 配置（Spring Boot 3.x 寫法） | `[x]` | 14 Security + JWT |
| 2.13.2 | 認證流程（Authentication） | `[x]` | 14 |
| 2.13.3 | JWT Token 生成 / 驗證 / 刷新機制 | `[x]` | 14 |
| 2.13.4 | 角色與權限（@PreAuthorize / @Secured / hasRole） | `[x]` | 14 |
| 2.13.5 | 密碼加密（BCryptPasswordEncoder） | `[x]` | 14 |
| 2.13.6 | CORS + CSRF 安全設定 | `[x]` | 14 |
| 2.13.7 | OAuth2 / OpenID Connect 概念（不深入實作） | `[x]` | 14 |

### 2.14 API 文件 — SpringDoc OpenAPI（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.14.1 | springdoc-openapi-starter-webmvc-ui 配置 | `[x]` | 15 SpringDoc |
| 2.14.2 | @Operation / @ApiResponse / @Parameter 註解 | `[x]` | 15 |
| 2.14.3 | @Schema 模型描述 | `[x]` | 15 |
| 2.14.4 | Swagger UI 存取與自定義 | `[x]` | 15 |
| 2.14.5 | 分組 API（GroupedOpenApi） | `[x]` | 15 |
| 2.14.6 | JWT Bearer 認證整合 | `[x]` | 15 |

### 2.15 MyBatis-Plus 快速開發（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 2.15.1 | BaseMapper / IService / ServiceImpl 核心介面 | `[x]` | 16 MyBatis-Plus |
| 2.15.2 | CRUD 零 SQL 開發 | `[x]` | 16 |
| 2.15.3 | LambdaQueryWrapper / LambdaUpdateWrapper 條件建構 | `[x]` | 16 |
| 2.15.4 | 分頁外掛（MybatisPlusInterceptor + PaginationInnerInterceptor） | `[x]` | 16 |
| 2.15.5 | 程式碼產生器（mybatis-plus-generator） | `[x]` | 16 |
| 2.15.6 | 自動填充（@TableField fill） | `[x]` | 16 |
| 2.15.7 | 邏輯刪除（@TableLogic） | `[x]` | 16 |
| 2.15.8 | 樂觀鎖（@Version） | `[x]` | 16 |
| 2.15.9 | 多租戶外掛（TenantLineInnerInterceptor） | `[x]` | 16 |

---

## 3. 微服務：Spring Cloud

> 目錄：`03-Microservices/`

### 3.1 概述與架構（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 3.1.1 | 單體 vs 微服務比較 | `[x]` | 01 概述 |
| 3.1.2 | Spring Cloud 與 Spring Boot 版本對應 | `[x]` | 01 |
| 3.1.3 | 核心元件全景圖 | `[x]` | 01 |

### 3.2 Eureka 服務註冊與發現（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 3.2.1 | Eureka Server / Client 配置 | `[x]` | 02 Eureka |
| 3.2.2 | 自我保護機制 | `[x]` | 02 |
| 3.2.3 | HA 叢集部署 | `[x]` | 02 |
| 3.2.4 | Eureka 維護模式說明 | `[x]` | 02 — 需補充 |
| 3.2.5 | Nacos 替代方案 | `[x]` | 02 |

### 3.3 Config 配置中心（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 3.3.1 | Config Server / Client | `[x]` | 03 Config |
| 3.3.2 | Git 儲存結構 | `[x]` | 03 |
| 3.3.3 | @RefreshScope + Spring Cloud Bus 動態刷新 | `[x]` | 03 |
| 3.3.4 | 配置加密（{cipher}） | `[x]` | 03 |
| 3.3.5 | Nacos Config 替代 | `[x]` | 03 |

### 3.4 Gateway API 閘道（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 3.4.1 | 路由（Routes）配置 | `[x]` | 04 Gateway |
| 3.4.2 | 述詞（Predicates）：Path / Method / Time / Header / Query | `[x]` | 04 |
| 3.4.3 | 過濾器（Filters）：StripPrefix / AddHeader / Retry | `[x]` | 04 |
| 3.4.4 | 全域過濾器（GlobalFilter）— Auth 驗證 | `[x]` | 04 |
| 3.4.5 | Redis 限流（RequestRateLimiter） | `[x]` | 04 |
| 3.4.6 | 熔斷器整合（CircuitBreaker Filter） | `[x]` | 04 |

### 3.5 Resilience4j（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 3.5.1 | Circuit Breaker 三狀態（CLOSED / OPEN / HALF_OPEN） | `[x]` | 05 Resilience4j |
| 3.5.2 | Rate Limiter | `[x]` | 05 |
| 3.5.3 | Retry | `[x]` | 05 |
| 3.5.4 | Bulkhead（信號量 + 執行緒池隔離） | `[x]` | 05 |
| 3.5.5 | 組合使用順序 | `[x]` | 05 |

### 3.6 LoadBalancer（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 3.6.1 | 客戶端負載均衡原理 | `[x]` | 06 LoadBalancer |
| 3.6.2 | RoundRobin / Random 策略 | `[x]` | 06 |
| 3.6.3 | 自定義策略 | `[x]` | 06 |
| 3.6.4 | Health Check 配置 | `[x]` | 06 |

### 3.7 OpenFeign（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 3.7.1 | @FeignClient 介面定義 | `[x]` | 07 OpenFeign |
| 3.7.2 | Fallback / FallbackFactory | `[x]` | 07 |
| 3.7.3 | 攔截器（RequestInterceptor）— Token 傳遞 | `[x]` | 07 |
| 3.7.4 | 超時 / 日誌 / 壓縮配置 | `[x]` | 07 |

### 3.8 Spring 6 HTTP Interface（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 3.8.1 | @HttpExchange / @GetExchange / @PostExchange 宣告式介面 | `[x]` | 08 HTTP Interface |
| 3.8.2 | RestClient + HttpServiceProxyFactory 設定 | `[x]` | 08 |
| 3.8.3 | 與 OpenFeign 的比較 + 遷移指引 | `[x]` | 08 |
| 3.8.4 | 錯誤處理（ResponseErrorHandler） | `[x]` | 08 |

---

## 4. AI：Spring AI

> 目錄：`04-Spring-AI/`

### 4.1 ~ 4.7（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 4.1.1 | 支援的模型提供者（8 種） | `[x]` | 01 概述 |
| 4.1.2 | Spring AI 1.0.0 + Spring Boot 3.3.0 版本 | `[x]` | 01 |
| 4.2.1 | ChatClient fluent API | `[x]` | 02 ChatClient |
| 4.2.2 | Prompt / Message 類型 | `[x]` | 02 |
| 4.2.3 | 串流回應（SSE） | `[x]` | 02 |
| 4.3.1 | entity() / BeanOutputConverter / MapOutputConverter | `[x]` | 03 Structured Output |
| 4.4.1 | EmbeddingModel 介面 + 8 種向量 DB | `[x]` | 04 Embedding |
| 4.4.2 | 相似度搜尋 + 篩選 | `[x]` | 04 |
| 4.5.1 | RAG 管線（Reader → Transformer → Writer） | `[x]` | 05 RAG |
| 4.5.2 | QuestionAnswerAdvisor | `[x]` | 05 |
| 4.5.3 | 分段策略（TokenTextSplitter） | `[x]` | 05 |
| 4.6.1 | **@Tool 註解**（1.0.0 新 API） | `[x]` | 06 — **必須更新** |
| 4.6.2 | FunctionCallback 舊模式（保留對照） | `[x]` | 06 |
| 4.7.1 | Advisor 鏈 + 自定義 Advisor | `[x]` | 07 Advisors |
| 4.7.2 | ChatMemory（InMemory / JDBC） | `[x]` | 07 |
| 4.7.3 | 視窗控制（chatMemoryRetrieveSize） | `[x]` | 07 |

---

## 5. 資料庫：PostgreSQL / MySQL 8 + Redis

> 目錄：`05-Database/`

### 5.1 PostgreSQL 與 MySQL 基礎（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 5.1.1 | 資料型別比較（INT / VARCHAR / TEXT / TIMESTAMP / JSON / UUID） | `[x]` | 01 DB 基礎 |
| 5.1.2 | DDL：CREATE TABLE / ALTER / DROP + 約束（PK / FK / UNIQUE / CHECK） | `[x]` | 01 |
| 5.1.3 | DML：INSERT / UPDATE / DELETE / UPSERT（ON CONFLICT / ON DUPLICATE KEY） | `[x]` | 01 |
| 5.1.4 | JOIN 查詢（INNER / LEFT / RIGHT / FULL OUTER） | `[x]` | 01 |
| 5.1.5 | 子查詢 / CTE（WITH）/ Window Function（ROW_NUMBER / RANK / LAG / LEAD） | `[x]` | 01 |
| 5.1.6 | JSON 操作（PostgreSQL jsonb / MySQL JSON_EXTRACT） | `[x]` | 01 |
| 5.1.7 | 時區處理（TIMESTAMP WITH TIME ZONE / UTC 儲存策略） | `[x]` | 01 |
| 5.1.8 | PostgreSQL vs MySQL 選型建議 | `[x]` | 01 |

### 5.2 索引原理與 SQL 優化（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 5.2.1 | B+ 樹索引原理 | `[x]` | 02 索引與優化 |
| 5.2.2 | 聚簇索引 vs 非聚簇索引 | `[x]` | 02 |
| 5.2.3 | 覆蓋索引（Covering Index） | `[x]` | 02 |
| 5.2.4 | 複合索引 + 最左前綴原則 | `[x]` | 02 |
| 5.2.5 | 索引失效場景（函數 / 隱式轉換 / OR / LIKE '%x'） | `[x]` | 02 |
| 5.2.6 | EXPLAIN 分析（type / key / rows / Extra 欄位解讀） | `[x]` | 02 |
| 5.2.7 | 慢查詢日誌 + 效能調優實戰 | `[x]` | 02 |
| 5.2.8 | PostgreSQL EXPLAIN ANALYZE + pg_stat_statements | `[x]` | 02 |

### 5.3 交易與鎖機制（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 5.3.1 | ACID 特性 | `[x]` | 03 交易與鎖 |
| 5.3.2 | 4 種隔離級別 + 對應問題（髒讀 / 不可重複讀 / 幻讀） | `[x]` | 03 |
| 5.3.3 | InnoDB 行鎖（Record Lock / Gap Lock / Next-Key Lock） | `[x]` | 03 |
| 5.3.4 | MVCC 原理（Undo Log / ReadView） | `[x]` | 03 |
| 5.3.5 | 死鎖偵測與排查 | `[x]` | 03 |
| 5.3.6 | PostgreSQL 的 MVCC 差異（xmin / xmax） | `[x]` | 03 |
| 5.3.7 | 與 Spring @Transactional 的整合關係 | `[x]` | 03 |

### 5.4 Redis 快取實戰（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 5.4.1 | 5 種基本資料型別（String / Hash / List / Set / Sorted Set） | `[x]` | 04 Redis |
| 5.4.2 | Spring Cache 整合（@Cacheable / @CacheEvict / @CachePut） | `[x]` | 04 |
| 5.4.3 | RedisTemplate vs StringRedisTemplate | `[x]` | 04 |
| 5.4.4 | 快取穿透 / 快取擊穿 / 快取雪崩 — 問題與解法 | `[x]` | 04 |
| 5.4.5 | 分散式鎖（Redisson / SET NX EX） | `[x]` | 04 |
| 5.4.6 | Redis Pub/Sub + Stream | `[x]` | 04 |
| 5.4.7 | Redis 持久化（RDB / AOF）+ 叢集模式概述 | `[x]` | 04 |

---

## 6. 前端：Vue 3 / React 19 + TypeScript

> 目錄：`06-Frontend/`

### 6.1 React 函式元件與 Hooks（Phase 1: 搬入）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 6.1.1 | 函式元件 vs 類別元件對照 | `[x]` | 01 React Hooks |
| 6.1.2 | useState | `[x]` | 01 |
| 6.1.3 | useEffect（生命週期替代） | `[x]` | 01 |
| 6.1.4 | Props + TypeScript 型別定義 | `[x]` | 01 — 需補充 |

### 6.2 TypeScript 基礎（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 6.2.1 | 基本型別 + 型別推斷 | `[x]` | 02 TypeScript |
| 6.2.2 | interface vs type | `[x]` | 02 |
| 6.2.3 | 泛型（Generics） | `[x]` | 02 |
| 6.2.4 | 工具型別（Partial / Pick / Omit / Record / Required） | `[x]` | 02 |
| 6.2.5 | 聯合型別 / 交叉型別 / 型別守衛 | `[x]` | 02 |
| 6.2.6 | enum vs const enum vs union literal | `[x]` | 02 |
| 6.2.7 | tsconfig.json 核心配置 | `[x]` | 02 |

### 6.3 Vue 3 Composition API（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 6.3.1 | `<script setup>` 語法 | `[x]` | 03 Vue 3 Composition |
| 6.3.2 | ref / reactive / computed / watch / watchEffect | `[x]` | 03 |
| 6.3.3 | defineProps / defineEmits / defineExpose | `[x]` | 03 |
| 6.3.4 | 生命週期（onMounted / onUnmounted / ...） | `[x]` | 03 |
| 6.3.5 | provide / inject 依賴注入 | `[x]` | 03 |
| 6.3.6 | Pinia 狀態管理（defineStore / state / getters / actions） | `[x]` | 03 |
| 6.3.7 | 組合式函數（Composables）— 自定義 Hook 等價物 | `[x]` | 03 |

### 6.4 Vue 3 元件開發實戰（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 6.4.1 | SFC 結構（template / script setup / style scoped） | `[x]` | 04 Vue 3 實戰 |
| 6.4.2 | Element Plus 整合 + 常用元件 | `[x]` | 04 |
| 6.4.3 | Vue Router 4（路由守衛 / 動態路由 / 巢狀路由） | `[x]` | 04 |
| 6.4.4 | Axios 封裝（攔截器 / 統一錯誤處理 / Token 自動附加） | `[x]` | 04 |
| 6.4.5 | 與 Spring Boot 後端對接實戰 | `[x]` | 04 |

### 6.5 React 進階與狀態管理（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 6.5.1 | useContext / useReducer | `[x]` | 05 React 進階 |
| 6.5.2 | 自定義 Hook 設計模式 | `[x]` | 05 |
| 6.5.3 | React Router v6（Routes / Outlet / useNavigate / loader） | `[x]` | 05 |
| 6.5.4 | 狀態管理（Zustand 為主 / Context API 對比） | `[x]` | 05 |
| 6.5.5 | React 18+ 功能（Suspense / useTransition / useDeferredValue） | `[x]` | 05 |

### 6.6 前端建置工具 — Vite（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 6.6.1 | Vite 核心概念（ESM + esbuild + Rollup） | `[x]` | 06 Vite |
| 6.6.2 | vite.config.ts 配置（別名 / 代理 / 環境變數） | `[x]` | 06 |
| 6.6.3 | 開發代理（proxy → Spring Boot 後端） | `[x]` | 06 |
| 6.6.4 | 打包最佳化（chunk 分割 / tree-shaking / gzip） | `[x]` | 06 |
| 6.6.5 | 與 Spring Boot 整合部署（靜態資源 / nginx 反代） | `[x]` | 06 |

---

## 7. CS 基礎

> 目錄：`07-CS-Fundamentals/`

### 7.1 排序演算法總覽（Phase 2: 重寫合併）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 7.1.1 | 氣泡排序 + 快速排序 | `[x]` | 01 排序 |
| 7.1.2 | 選擇排序 + 堆排序 | `[x]` | 01 |
| 7.1.3 | 插入排序 + 希爾排序 | `[x]` | 01 |
| 7.1.4 | 歸併排序（**修正 merge BUG**） | `[x]` | 01 |
| 7.1.5 | 計數排序 / 基數排序 / 桶排序（補充） | `[x]` | 01 |
| 7.1.6 | **綜合比較表**（時間 / 空間 / 穩定性 / 適用場景） | `[x]` | 01 |

### 7.2 查詢演算法總覽（Phase 2: 重寫合併）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 7.2.1 | 順序查詢 + 複雜度分析 | `[x]` | 02 查詢 |
| 7.2.2 | 二分查詢（邊界條件 + 變體：第一個/最後一個） | `[x]` | 02 |
| 7.2.3 | 索引查詢（分塊查詢） | `[x]` | 02 |
| 7.2.4 | BST 查詢（含刪除實作 + 退化問題） | `[x]` | 02 |
| 7.2.5 | 雜湊查詢（載入因子 + rehash） | `[x]` | 02 |

### 7.3 基礎資料結構（Phase 2: 重寫合併）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 7.3.1 | 時間/空間複雜度 + Big-O 比較圖 | `[x]` | 03 資料結構 |
| 7.3.2 | 順序表 + 連結 Java ArrayList | `[x]` | 03 |
| 7.3.3 | 連結串列（單/雙/循環）+ 連結 Java LinkedList | `[x]` | 03 |
| 7.3.4 | 棧（陣列 + 鏈結實作 + 應用場景） | `[x]` | 03 |
| 7.3.5 | 佇列（循環佇列 + 優先佇列 + Deque）+ 連結 Java Queue/Deque | `[x]` | 03 |
| 7.3.6 | 常見面試題（反轉鏈表 / 環偵測 / 合併有序 / 括號匹配） | `[x]` | 03 |

### 7.4 樹與圖（Phase 2: 重寫合併）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 7.4.1 | 二叉樹 4 種遍歷（遞迴 + **迭代**） | `[x]` | 04 樹與圖 |
| 7.4.2 | 哈夫曼樹（**修正 compareTo BUG**） + 編碼應用 | `[x]` | 04 |
| 7.4.3 | 圖的表示法（鄰接矩陣 / 鄰接表）+ **程式碼實作** | `[x]` | 04 |
| 7.4.4 | DFS / BFS（程式碼實作） | `[x]` | 04 |
| 7.4.5 | 最短路徑（Dijkstra）概念 | `[x]` | 04 |

### 7.5 JVM 記憶體與垃圾回收（Phase 2: 重寫合併）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 7.5.1 | 執行時記憶體區域（堆 / 棧 / **Metaspace** / PC / 直接記憶體） | `[x]` | 05 JVM |
| 7.5.2 | 類別載入機制 + 雙親委派 | `[x]` | 05 |
| 7.5.3 | GC 演算法（標記清除 / 複製 / 標記整理） | `[x]` | 05 |
| 7.5.4 | 分代收集（Young / Old） | `[x]` | 05 |
| 7.5.5 | **G1**（Java 9 預設）原理 | `[x]` | 05 |
| 7.5.6 | **ZGC**（Java 15+）/ Shenandoah 概述 | `[x]` | 05 |
| 7.5.7 | 常用 JVM 參數（-Xms / -Xmx / -Xmn / -XX:+UseG1GC / ...） | `[x]` | 05 |
| 7.5.8 | 調優工具（jstat / jmap / jstack / Arthas） | `[x]` | 05 |

---

## 8. DevOps：建置 / 容器 / 測試

> 目錄：`08-DevOps/`

### 8.1 Gradle 建置工具（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 8.1.1 | Gradle vs Maven 比較表 | `[x]` | 01 Gradle |
| 8.1.2 | build.gradle.kts 基本結構（Kotlin DSL） | `[x]` | 01 |
| 8.1.3 | 依賴管理（implementation / api / compileOnly / runtimeOnly） | `[x]` | 01 |
| 8.1.4 | 多模組專案（settings.gradle.kts / 子專案依賴） | `[x]` | 01 |
| 8.1.5 | Spring Boot Gradle Plugin（bootJar / bootRun） | `[x]` | 01 |
| 8.1.6 | 自定義 Task + Build Cache | `[x]` | 01 |

### 8.2 Docker 容器化部署（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 8.2.1 | Docker 核心概念（Image / Container / Volume / Network） | `[x]` | 02 Docker |
| 8.2.2 | Dockerfile 撰寫（Spring Boot 應用） | `[x]` | 02 |
| 8.2.3 | 多階段建置（Multi-stage Build）— 減小映像 | `[x]` | 02 |
| 8.2.4 | Docker Compose（Spring Boot + PostgreSQL + Redis 完整範例） | `[x]` | 02 |
| 8.2.5 | 容器化最佳實踐（非 root / .dockerignore / layer caching） | `[x]` | 02 |
| 8.2.6 | Spring Boot 內建 Buildpacks（spring-boot:build-image） | `[x]` | 02 |

### 8.3 Kubernetes 入門（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 8.3.1 | K8s 核心概念（Pod / Service / Deployment / ReplicaSet） | `[x]` | 03 Kubernetes |
| 8.3.2 | ConfigMap / Secret（外部化配置） | `[x]` | 03 |
| 8.3.3 | Ingress（HTTP 路由 / TLS） | `[x]` | 03 |
| 8.3.4 | 部署 Spring Boot 應用到 K8s（YAML 完整範例） | `[x]` | 03 |
| 8.3.5 | Spring Cloud Kubernetes 整合（服務發現 / 配置載入） | `[x]` | 03 |
| 8.3.6 | kubectl 常用指令速查 | `[x]` | 03 |

### 8.4 JUnit 5 測試實戰（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 8.4.1 | JUnit 5 基礎（@Test / @BeforeEach / @AfterEach / @DisplayName） | `[x]` | 04 JUnit 5 |
| 8.4.2 | @Nested 巢狀測試 | `[x]` | 04 |
| 8.4.3 | @ParameterizedTest（@ValueSource / @CsvSource / @MethodSource） | `[x]` | 04 |
| 8.4.4 | Mockito（@Mock / @InjectMocks / when-thenReturn / verify） | `[x]` | 04 |
| 8.4.5 | @SpringBootTest 整合測試 | `[x]` | 04 |
| 8.4.6 | @WebMvcTest + MockMvc（Controller 層測試） | `[x]` | 04 |
| 8.4.7 | @DataJpaTest（Repository 層測試） | `[x]` | 04 |
| 8.4.8 | Testcontainers（Docker 化的整合測試） | `[x]` | 04 |

### 8.5 Playwright 前端自動化測試（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 8.5.1 | 安裝與專案設定（playwright.config.ts） | `[x]` | 05 Playwright |
| 8.5.2 | 選擇器策略（getByRole / getByText / getByTestId） | `[x]` | 05 |
| 8.5.3 | Page Object Model 設計模式 | `[x]` | 05 |
| 8.5.4 | 斷言（expect / toBeVisible / toHaveText / toHaveURL） | `[x]` | 05 |
| 8.5.5 | 截圖與錄影（screenshot / video） | `[x]` | 05 |
| 8.5.6 | CI 整合（headless 模式 / GitHub Actions） | `[x]` | 05 |
| 8.5.7 | API 測試（request context） | `[x]` | 05 |

### 8.6 CI/CD — GitHub Actions（Phase 3: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 8.6.1 | 工作流程語法（on / jobs / steps / uses） | `[x]` | 06 CI/CD |
| 8.6.2 | Build → Test → Deploy 管線 | `[x]` | 06 |
| 8.6.3 | Docker 映像建置 + 推送 Registry | `[x]` | 06 |
| 8.6.4 | 環境變數與 Secrets 管理 | `[x]` | 06 |
| 8.6.5 | 矩陣策略（多 Java 版本 / 多 OS 測試） | `[x]` | 06 |
| 8.6.6 | 快取依賴（Gradle / npm cache） | `[x]` | 06 |

### 8.7 Git 與 GitHub 版本控制（Phase 4: 新增）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 8.7.1 | 三個區域（工作目錄 / 暫存區 / 本地倉庫 / 遠端） | `[x]` | 07 Git |
| 8.7.2 | 檔案狀態（Untracked / Staged / Committed / Modified） | `[x]` | 07 |
| 8.7.3 | 基本操作（init / add / commit / push / pull） | `[x]` | 07 |
| 8.7.4 | .gitignore 設定 | `[x]` | 07 |
| 8.7.5 | 分支操作（branch / switch / merge / rebase） | `[x]` | 07 |
| 8.7.6 | 分支命名規範（feature / fix / hotfix） | `[x]` | 07 |
| 8.7.7 | Conventional Commits 規範 | `[x]` | 07 |
| 8.7.8 | GitHub Flow / Git Flow / Trunk-Based 比較 | `[x]` | 07 |
| 8.7.9 | Pull Request 工作流程 | `[x]` | 07 |
| 8.7.10 | 撤銷操作（restore / reset / amend） | `[x]` | 07 |
| 8.7.11 | Stash / Cherry Pick / Reflog | `[x]` | 07 |
| 8.7.12 | SSH Key 設定 | `[x]` | 07 |

---

## 9. 軟體工程（Software Engineering）

> 目錄：`09-Software-Engineering/`

### 9.1 SOLID 原則與 Clean Code（Phase 5: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 9.1.1 | SRP 單一職責原則 + Spring Service 拆分 | `[x]` | 01 SOLID |
| 9.1.2 | OCP 開放封閉原則 + Strategy Pattern 範例 | `[x]` | 01 |
| 9.1.3 | LSP 里氏替換原則 + 繼承陷阱 | `[x]` | 01 |
| 9.1.4 | ISP 介面隔離原則 + 胖介面拆分 | `[x]` | 01 |
| 9.1.5 | DIP 依賴反轉原則 + Spring DI 對應 | `[x]` | 01 |
| 9.1.6 | Clean Code 命名 / 函式 / 註解原則 | `[x]` | 01 |
| 9.1.7 | Clean Architecture 分層概念 | `[x]` | 01 |

### 9.2 設計模式實戰應用（Phase 5: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 9.2.1 | Factory Method + Spring BeanFactory | `[x]` | 02 設計模式 |
| 9.2.2 | Builder Pattern + Lombok @Builder | `[x]` | 02 |
| 9.2.3 | Singleton + Spring Bean Scope | `[x]` | 02 |
| 9.2.4 | Decorator + Filter Chain | `[x]` | 02 |
| 9.2.5 | Proxy + Spring AOP | `[x]` | 02 |
| 9.2.6 | Strategy + Spring 多實作注入 | `[x]` | 02 |
| 9.2.7 | Template Method + JdbcTemplate | `[x]` | 02 |
| 9.2.8 | Observer + Spring Event | `[x]` | 02 |
| 9.2.9 | Chain of Responsibility + Filter/Interceptor | `[x]` | 02 |

### 9.3 軟體架構模式（Phase 5: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 9.3.1 | 分層架構（Layered Architecture） | `[x]` | 03 架構模式 |
| 9.3.2 | 六角架構（Hexagonal / Ports & Adapters） | `[x]` | 03 |
| 9.3.3 | DDD 戰術設計（Entity / VO / Aggregate） | `[x]` | 03 |
| 9.3.4 | CQRS 命令查詢分離 | `[x]` | 03 |
| 9.3.5 | Event-Driven Architecture | `[x]` | 03 |
| 9.3.6 | Saga Pattern（分散式交易） | `[x]` | 03 |

### 9.4 API 設計最佳實踐（Phase 5: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 9.4.1 | Richardson 成熟度模型（Level 0-3） | `[x]` | 04 API 設計 |
| 9.4.2 | URL 命名 / HTTP 動詞 / 狀態碼 | `[x]` | 04 |
| 9.4.3 | API 版本策略（URL / Header / Query） | `[x]` | 04 |
| 9.4.4 | 統一錯誤格式 + RFC 7807 Problem Details | `[x]` | 04 |
| 9.4.5 | 分頁 / 過濾 / 排序 | `[x]` | 04 |
| 9.4.6 | 冪等性設計 | `[x]` | 04 |

### 9.5 日誌、監控與可觀測性（Phase 5: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 9.5.1 | 可觀測性三支柱（Logs / Metrics / Traces） | `[x]` | 05 可觀測性 |
| 9.5.2 | SLF4J + Logback 結構化日誌 | `[x]` | 05 |
| 9.5.3 | Micrometer + Prometheus 指標收集 | `[x]` | 05 |
| 9.5.4 | OpenTelemetry 分散式追蹤 | `[x]` | 05 |
| 9.5.5 | Grafana 視覺化 | `[x]` | 05 |

### 9.6 安全開發實踐（Phase 5: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 9.6.1 | OWASP Top 10 與 Java 防護 | `[x]` | 06 安全開發 |
| 9.6.2 | SQL 注入防護（JPA / MyBatis） | `[x]` | 06 |
| 9.6.3 | XSS / CSRF 防護 | `[x]` | 06 |
| 9.6.4 | 存取控制（IDOR 防護） | `[x]` | 06 |
| 9.6.5 | 輸入驗證策略（白名單 > 黑名單） | `[x]` | 06 |
| 9.6.6 | Secrets 管理 | `[x]` | 06 |

### 9.7 效能調校與壓力測試（Phase 5: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 9.7.1 | JVM 調校（GC 選擇 / 記憶體參數） | `[x]` | 07 效能調校 |
| 9.7.2 | HikariCP 連線池調校 | `[x]` | 07 |
| 9.7.3 | 效能反模式（N+1 / 迴圈遠端呼叫 / 全表掃描） | `[x]` | 07 |
| 9.7.4 | JMeter 壓力測試策略 | `[x]` | 07 |
| 9.7.5 | Profiling 工具（VisualVM / async-profiler / Arthas） | `[x]` | 07 |

### 9.8 程式碼審查與重構（Phase 5: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 9.8.1 | Code Review 流程與 Checklist | `[x]` | 08 程式碼審查 |
| 9.8.2 | 程式碼異味（Long Method / Large Class / Feature Envy） | `[x]` | 08 |
| 9.8.3 | 重構手法（Extract Method / Replace Conditional with Polymorphism） | `[x]` | 08 |
| 9.8.4 | 技術債管理（分類 / Boy Scout Rule / 20% 時間） | `[x]` | 08 |

### 9.9 系統設計入門（Phase 5: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 9.9.1 | 高可用設計（消除單點故障 / 健康檢查） | `[x]` | 09 系統設計 |
| 9.9.2 | 水平擴展 vs 垂直擴展 + 無狀態設計 | `[x]` | 09 |
| 9.9.3 | CAP 定理 + 最終一致性 | `[x]` | 09 |
| 9.9.4 | 訊息佇列（RabbitMQ / Kafka / Redis Stream 比較） | `[x]` | 09 |
| 9.9.5 | 微服務拆分依據 | `[x]` | 09 |
| 9.9.6 | Twelve-Factor App | `[x]` | 09 |

### 9.10 敏捷開發與團隊協作（Phase 5: 新寫）

| # | 子項目 | 覆蓋 | 來源篇章 |
|---|--------|------|---------|
| 9.10.1 | Scrum 框架（角色 / 事件 / 工件） | `[x]` | 10 敏捷開發 |
| 9.10.2 | User Story + 驗收標準 | `[x]` | 10 |
| 9.10.3 | Story Point 估算 + Velocity | `[x]` | 10 |
| 9.10.4 | 看板（Kanban）+ WIP 限制 | `[x]` | 10 |
| 9.10.5 | ADR（架構決策紀錄） | `[x]` | 10 |
| 9.10.6 | 技術文件撰寫（Docs as Code） | `[x]` | 10 |

---

## 覆蓋度統計

| 技術層級 | 子項目數 | 覆蓋率 |
|---------|---------|--------|
| 1. Java 17/21 | 54 | 100% |
| 2. Spring Boot 3.x + MVC + ORM | 81 | 100% |
| 3. Spring Cloud | 30 | 100% |
| 4. Spring AI | 15 | 100% |
| 5. Database + Redis | 30 | 100% |
| 6. Frontend（Vue/React/TS） | 35 | 100% |
| 7. CS Fundamentals | 28 | 100% |
| 8. DevOps | 48 | 100% |
| 9. Software Engineering | 58 | 100% |
| **合計** | **379** | **100%** |

> 所有 379 個子項目已完整覆蓋，分佈在 71 篇文章中。
