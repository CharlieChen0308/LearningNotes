# 04 Spring Boot 自動配置與 Starters

> **版本**: Spring Boot 3.x
> **來源**: `_archive/Spring Boot/02 Spring Boot 自動配置與 Starters.md`

## 自動配置（Auto-Configuration）

Spring Boot 最核心的特性就是**自動配置**——根據你引入的依賴，自動幫你配置好 Spring 應用程式所需的各種 Bean，不再需要手寫大量 XML 或 Java 配置。

### @SpringBootApplication 的組成

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

`@SpringBootApplication` 是一個組合註解，等同於：

| 註解 | 功能 |
|------|------|
| `@SpringBootConfiguration` | 標記為配置類（等同 `@Configuration`） |
| `@EnableAutoConfiguration` | 啟用自動配置機制 |
| `@ComponentScan` | 掃描當前套件及子套件中的元件 |

### 自動配置的原理

1. Spring Boot 啟動時，掃描所有 jar 包中的 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 檔案
2. 根據條件註解（如 `@ConditionalOnClass`、`@ConditionalOnMissingBean`）判斷是否載入對應的配置
3. 如果條件滿足，自動建立並註冊 Bean

例如，當 classpath 中存在 `DataSource.class` 和 `spring.datasource.url` 配置時，Spring Boot 會自動建立資料庫連線池。

### 條件註解

| 條件註解 | 判斷條件 |
|---------|---------|
| `@ConditionalOnClass` | classpath 中存在指定類 |
| `@ConditionalOnMissingBean` | 容器中不存在指定 Bean |
| `@ConditionalOnProperty` | 配置屬性滿足指定值 |
| `@ConditionalOnWebApplication` | 當前是 Web 應用程式 |

### 查看自動配置報告

在 `application.properties` 中加入：

```properties
debug=true
```

啟動時會在日誌中輸出：
- **Positive matches**：已啟用的自動配置
- **Negative matches**：未啟用的自動配置（及原因）

## Starters

Starter 是 Spring Boot 提供的**依賴描述符**，一個 Starter 包含了完成某個功能所需的全部依賴，開發者不需要逐一搜尋和新增依賴。

### 常用 Starters

| Starter | 功能 |
|---------|------|
| `spring-boot-starter-web` | Web 應用（內嵌 Tomcat、Spring MVC） |
| `spring-boot-starter-data-jpa` | JPA 資料存取（Hibernate） |
| `spring-boot-starter-data-redis` | Redis 操作 |
| `spring-boot-starter-security` | Spring Security 安全框架 |
| `spring-boot-starter-test` | 測試（JUnit 5、Mockito、AssertJ） |
| `spring-boot-starter-actuator` | 監控與管理端點 |
| `spring-boot-starter-validation` | 資料驗證（Jakarta Validation） |
| `spring-boot-starter-mail` | 郵件發送 |
| `spring-boot-starter-cache` | 快取抽象 |
| `spring-boot-starter-thymeleaf` | Thymeleaf 模板引擎 |

### 使用範例

只需要在 `pom.xml` 中加入一行，就能獲得完整的 Web 開發能力：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

這一個依賴包含了：Spring MVC、內嵌 Tomcat、Jackson JSON 處理、日誌框架等。

### 版本管理

Spring Boot 的父 POM 已經管理了所有 Starter 的版本，不需要手動指定：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.4.1</version>
</parent>
```

如果不使用父 POM，可以用 BOM 方式：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.4.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 自訂配置覆蓋自動配置

自動配置提供的是**預設值**，你隨時可以覆蓋：

### 方式一：在 application.properties 中修改

```properties
# 修改伺服器連接埠（預設 8080）
server.port=9090

# 修改資料來源
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
spring.datasource.password=secret
```

### 方式二：自訂 Bean 覆蓋

```java
@Configuration
public class MyDataSourceConfig {

    @Bean
    public DataSource dataSource() {
        // 自訂的 DataSource 會取代自動配置的
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        ds.setUsername("admin");
        ds.setPassword("secret");
        ds.setMaximumPoolSize(20);
        return ds;
    }
}
```

由於 `@ConditionalOnMissingBean` 的存在，自動配置發現你已經定義了 `DataSource` Bean，就不會再建立預設的。

## 建立自訂 Starter

如果你有一組在多個專案中重複使用的配置，可以封裝成 Starter：

### 1. 命名規範

- 官方 Starter：`spring-boot-starter-{name}`
- 第三方 Starter：`{name}-spring-boot-starter`

### 2. 自訂自動配置類

```java
@AutoConfiguration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties.getEndpoint(), properties.getTimeout());
    }
}
```

### 3. 配置屬性類

```java
@ConfigurationProperties(prefix = "my.service")
public class MyProperties {
    private String endpoint = "http://localhost:8080";
    private int timeout = 5000;

    // getter / setter
}
```

### 4. 註冊自動配置

在 `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`：

```
com.example.MyAutoConfiguration
```

## 取捨分析：何時排除自動配置

自動配置雖然方便，但並非總是適合。以下情境應考慮排除：

| 情境 | 說明 | 範例 |
|------|------|------|
| Bean 衝突 | 自動配置的 Bean 與自訂 Bean 產生衝突 | 自訂 `ObjectMapper` 但 Jackson 自動配置仍介入 |
| 效能考量 | 不需要的自動配置拖慢啟動時間 | 不用 JMX 卻載入 `JmxAutoConfiguration` |
| 不必要的功能 | 引入的 Starter 包含用不到的配置 | 只需 JDBC 但 `DataSourceAutoConfiguration` 嘗試建立連線池 |

## 生產注意事項

### 排除自動配置的策略

```java
// 方式一：在 @SpringBootApplication 上排除
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    JmxAutoConfiguration.class
})
public class MyApplication { ... }

// 方式二：在 application.yml 中排除（適合環境差異化）
// spring:
//   autoconfigure:
//     exclude:
//       - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

**選擇原則**：固定不需要的用 `exclude` 參數；依環境而定的用 `application.yml` 配置。

### 偵錯自動配置

當自動配置行為不如預期時，使用以下方式排查：

```bash
# 方式一：啟動時加 --debug 旗標
java -jar app.jar --debug

# 方式二：在配置檔中設定
# debug=true
```

啟動日誌會輸出 `ConditionEvaluationReport`，包含每個自動配置類的匹配結果與原因。重點關注：

- **Positive matches**：哪些配置被啟用、為什麼
- **Negative matches**：哪些配置被跳過、缺少什麼條件
- **Unconditional classes**：無條件載入的配置類

### 自訂 Starter 最佳實務

建立自訂 Starter 時，遵循以下原則：

1. **拆分模組**：`{name}-spring-boot-starter`（只含依賴）+ `{name}-spring-boot-autoconfigure`（含自動配置邏輯）
2. **不要使用 `@ComponentScan`**：自動配置模組應只靠 `@Bean` 方法註冊 Bean，避免掃描到使用者的類別
3. **一律加 `@ConditionalOnMissingBean`**：讓使用者能覆蓋任何預設 Bean
4. **提供 `@ConfigurationProperties`**：讓使用者透過配置檔案調整行為，而非寫程式碼

## 延伸閱讀

- [03 Spring Java 配置與註解驅動](03%20Spring%20Java%20配置與註解驅動.md)——理解 `@Configuration` 與 `@Bean` 的基礎
- [05 Spring Boot 配置檔案與 Profiles](05%20Spring%20Boot%20配置檔案與%20Profiles.md)——如何透過配置檔案覆蓋自動配置的預設值

## 小結

自動配置和 Starters 是 Spring Boot 能夠「開箱即用」的核心機制。理解自動配置的原理（條件註解 + 預設值），就能在享受便利的同時，靈活地覆蓋任何預設行為。

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
