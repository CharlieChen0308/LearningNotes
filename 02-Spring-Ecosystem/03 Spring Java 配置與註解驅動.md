# 03 Spring Java 配置與註解驅動

> **版本**: Spring Framework 6.x
> **來源**: `_archive/Spring/06 Spring Java 配置與註解驅動（現代方式）.md`

> 本系列第 01 篇介紹了 Spring 3.x 的 XML 配置方式。本篇對照 Spring Framework 6.x 官方文件，補充現代 Java-based 配置方式。兩種方式功能等價，但 Java 配置已成為主流。

## XML 配置 vs Java 配置對照

### Bean 定義

**XML 方式（Spring 3.x）：**

```xml
<bean id="userService" class="com.example.service.UserServiceImpl">
    <property name="userRepository" ref="userRepository"/>
</bean>
<bean id="userRepository" class="com.example.repository.UserRepositoryImpl"/>
```

**Java 方式（Spring 6.x）：**

```java
@Configuration
public class AppConfig {

    @Bean
    public UserService userService(UserRepository userRepository) {
        return new UserServiceImpl(userRepository);
    }

    @Bean
    public UserRepository userRepository() {
        return new UserRepositoryImpl();
    }
}
```

### 元件掃描

**XML 方式：**

```xml
<context:component-scan base-package="com.example"/>
```

**Java 方式：**

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {
}
```

### 啟動容器

**XML 方式：**

```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("spring-config.xml");
```

**Java 方式：**

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
```

## 常用元件註解

Spring 透過元件註解自動偵測並註冊 Bean，無需逐一在配置類中定義：

| 註解 | 適用層 | 說明 |
|------|--------|------|
| `@Component` | 通用 | 通用元件 |
| `@Service` | 業務邏輯層 | 語義化的 `@Component` |
| `@Repository` | 資料存取層 | 自動轉換持久層例外 |
| `@Controller` | 表現層 | 處理 HTTP 請求 |
| `@RestController` | 表現層 | `@Controller` + `@ResponseBody` |

```java
@Service
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;

    // Spring 6.x 推薦建構子注入，單一建構子時 @Autowired 可省略
    public UserServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

## 依賴注入方式對照

### 建構子注入（推薦）

```java
@Service
public class OrderService {

    private final UserService userService;
    private final ProductService productService;

    // Spring 6.x：單一建構子時 @Autowired 可省略
    public OrderService(UserService userService, ProductService productService) {
        this.userService = userService;
        this.productService = productService;
    }
}
```

### Setter 注入

```java
@Service
public class OrderService {

    private UserService userService;

    @Autowired
    public void setUserService(UserService userService) {
        this.userService = userService;
    }
}
```

### 欄位注入（不推薦）

```java
@Service
public class OrderService {

    @Autowired  // 可用但不推薦：無法宣告 final、不利於測試
    private UserService userService;
}
```

### @Autowired vs @Resource

| 特性 | @Autowired | @Resource |
|------|-----------|-----------|
| 來源 | Spring 框架 | Jakarta（標準 Java） |
| 匹配方式 | 先按**型別**匹配 | 先按**名稱**匹配 |
| 配合使用 | `@Qualifier("beanName")` | `@Resource(name="beanName")` |

```java
// 當有多個相同型別的 Bean 時，用 @Qualifier 指定
@Autowired
@Qualifier("mysqlUserRepository")
private UserRepository userRepository;

// 或使用 @Resource 按名稱注入
@Resource(name = "mysqlUserRepository")
private UserRepository userRepository;
```

## @Value — 注入配置值

```java
@Component
public class AppSettings {

    @Value("${app.name:MyApp}")
    private String appName;

    @Value("${server.port:8080}")
    private int port;

    @Value("#{systemProperties['user.home']}")  // SpEL 表達式
    private String userHome;
}
```

## Bean 的作用域

| 作用域 | 說明 |
|--------|------|
| `singleton` | 預設，整個容器只有一個實例 |
| `prototype` | 每次注入或取得都建立新實例 |
| `request` | 每個 HTTP 請求一個實例（Web） |
| `session` | 每個 HTTP Session 一個實例（Web） |

```java
@Component
@Scope("prototype")
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
}
```

## Bean 的生命週期回呼

```java
@Component
public class DatabaseConnectionPool {

    @PostConstruct
    public void init() {
        // 初始化連線池（等同 XML 的 init-method）
    }

    @PreDestroy
    public void cleanup() {
        // 釋放連線（等同 XML 的 destroy-method）
    }
}
```

## 多配置類組合

```java
@Configuration
@Import({DataSourceConfig.class, SecurityConfig.class})
@ComponentScan("com.example")
@PropertySource("classpath:application.properties")
public class AppConfig {
}
```

## 條件化配置

Spring 6.x 提供 `@Conditional` 和 `@Profile` 來根據條件載入 Bean：

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        // H2 記憶體資料庫
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2).build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://db:5432/mydb");
        return ds;
    }
}
```

## 取捨分析

### XML 配置仍有價值的場景

雖然 Java 配置已成為主流，但 XML 在以下情境仍然實用：

- **整合 Legacy 系統**：舊專案已有大量 XML 配置，逐步遷移比一次重寫更安全
- **第三方函式庫無原始碼**：無法加上 `@Component` 等註解時，XML 的 `<bean>` 定義是最直接的方式
- **非開發人員需要調整配置**：XML 不需要編譯，維運人員可直接修改（但現代做法更傾向用 `application.yml`）

### Java 配置 vs 純註解掃描

| 面向 | `@Configuration` + `@Bean` | `@ComponentScan` + 元件註解 |
|------|---------------------------|----------------------------|
| 控制粒度 | 高——每個 Bean 的建立邏輯清晰可見 | 低——依賴框架自動掃描 |
| 適用場景 | 第三方類別、需要複雜初始化邏輯 | 自己撰寫的業務類別 |
| 可讀性 | 集中管理，一目瞭然 | 分散在各類別上，需靠 IDE 輔助 |

**實務建議**：兩者混用最常見——業務類別用元件註解，基礎設施（DataSource、RestTemplate、ObjectMapper 等）用 `@Bean` 方法明確定義。

## 生產注意事項

### @ComponentScan 掃描範圍控制

`@ComponentScan` 預設掃描當前套件及所有子套件。**不要**把啟動類放在根套件（如 `com`），否則會掃描整個 classpath，導致：

- 啟動時間大幅增加
- 意外載入不相關的 Bean（尤其在多模組專案中）

```java
// 明確指定掃描範圍，避免過度掃描
@ComponentScan(basePackages = {
    "com.example.service",
    "com.example.repository"
})

// 或用 excludeFilters 排除特定類別
@ComponentScan(
    basePackages = "com.example",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.REGEX,
        pattern = "com\\.example\\.legacy\\..*"
    )
)
```

### @Configuration 的 proxyBeanMethods

`@Configuration` 預設會透過 CGLIB 代理確保 `@Bean` 方法之間互相呼叫時回傳同一個 singleton 實例。但在以下場景可以關閉：

```java
// GraalVM Native Image 或不需要 Bean 方法互相呼叫時
@Configuration(proxyBeanMethods = false)
public class LightweightConfig {

    @Bean
    public MyService myService() {
        return new MyService();
    }
}
```

- `proxyBeanMethods = true`（預設）：產生 CGLIB 代理，保證 singleton 語義
- `proxyBeanMethods = false`：不產生代理，啟動更快、記憶體更少，**GraalVM Native Image 編譯必須設為 false**

## 延伸閱讀

- [01 Spring Core — DI 與 IoC](01%20Spring%20Core%20—%20DI%20與%20IoC.md)——理解依賴注入的基礎原理
- [04 Spring Boot 自動配置與 Starters](04%20Spring%20Boot%20自動配置與%20Starters.md)——Java 配置如何被 Spring Boot 進一步自動化

## 小結

Spring Framework 6.x 的 Java-based 配置完全取代了 XML，提供了型別安全、重構友好的配置體驗。核心要點：使用 `@Configuration` + `@Bean` 定義配置，使用元件註解 + `@ComponentScan` 自動偵測，優先使用建構子注入。

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
