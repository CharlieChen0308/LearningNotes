# 05 Spring Boot 配置檔案與 Profiles

> **版本**: Spring Boot 3.x
> **來源**: `_archive/Spring Boot/03 Spring Boot 配置檔案與 Profiles.md`

## 配置檔案格式

Spring Boot 支援兩種配置檔案格式：

| 格式 | 檔名 | 特點 |
|------|------|------|
| Properties | `application.properties` | 扁平結構，每行一個配置 |
| YAML | `application.yml` | 階層結構，更易閱讀 |

兩者功能完全等價，可以擇一使用。YAML 在配置較多時可讀性更好。

### Properties 格式

```properties
server.port=8080
spring.application.name=my-app
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
```

### YAML 格式

```yaml
server:
  port: 8080

spring:
  application:
    name: my-app
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
```

## 讀取配置值

### 方式一：@Value

```java
@RestController
public class ConfigController {

    @Value("${spring.application.name}")
    private String appName;

    @Value("${server.port}")
    private int serverPort;

    // 提供預設值，避免配置缺失時報錯
    @Value("${app.description:未設定}")
    private String description;

    @GetMapping("/config")
    public Map<String, Object> getConfig() {
        return Map.of(
            "appName", appName,
            "serverPort", serverPort,
            "description", description
        );
    }
}
```

### 方式二：@ConfigurationProperties（推薦）

適合將一組相關配置繫結到 Java 物件，型別安全且支援驗證：

```yaml
app:
  name: TireMaster
  version: 1.0.0
  upload:
    max-size: 10MB
    allowed-types:
      - image/png
      - image/jpeg
      - application/pdf
  mail:
    sender: admin@example.com
    retry-count: 3
```

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {

    private String name;
    private String version;
    private Upload upload = new Upload();
    private Mail mail = new Mail();

    public static class Upload {
        private String maxSize;
        private List<String> allowedTypes;
        // getter / setter
    }

    public static class Mail {
        private String sender;
        private int retryCount;
        // getter / setter
    }

    // getter / setter
}
```

啟用配置屬性：

```java
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class MyApplication { ... }
```

使用：

```java
@RestController
public class AppController {

    private final AppProperties appProperties;

    public AppController(AppProperties appProperties) {
        this.appProperties = appProperties;
    }

    @GetMapping("/app-info")
    public AppProperties getAppInfo() {
        return appProperties;
    }
}
```

### 方式三：Environment

```java
@Service
public class DynamicConfigService {

    private final Environment environment;

    public DynamicConfigService(Environment environment) {
        this.environment = environment;
    }

    public String getProperty(String key) {
        return environment.getProperty(key, "未設定");
    }

    public String[] getActiveProfiles() {
        return environment.getActiveProfiles();
    }
}
```

## Profiles（環境配置）

Profiles 讓你為不同環境（開發、測試、正式）準備不同的配置。

### 建立 Profile 配置檔案

```
src/main/resources/
├── application.yml              # 共用配置
├── application-dev.yml          # 開發環境
├── application-test.yml         # 測試環境
└── application-prod.yml         # 正式環境
```

**application.yml（共用）**：

```yaml
spring:
  application:
    name: my-app
  jpa:
    open-in-view: false
```

**application-dev.yml**：

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb_dev
    username: dev_user
    password: dev_pass

logging:
  level:
    root: DEBUG
    com.example: TRACE
```

**application-prod.yml**：

```yaml
server:
  port: 80

spring:
  datasource:
    url: jdbc:postgresql://db-server:5432/mydb_prod
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

logging:
  level:
    root: WARN
```

### 啟用 Profile

**方式一**：在 application.yml 中指定

```yaml
spring:
  profiles:
    active: dev
```

**方式二**：命令列參數

```bash
java -jar app.jar --spring.profiles.active=prod
```

**方式三**：環境變數

```bash
export SPRING_PROFILES_ACTIVE=prod
```

### Profile 條件化 Bean

```java
@Configuration
public class StorageConfig {

    @Bean
    @Profile("dev")
    public StorageService localStorageService() {
        return new LocalStorageService("/tmp/uploads");
    }

    @Bean
    @Profile("prod")
    public StorageService s3StorageService() {
        return new S3StorageService("my-bucket");
    }
}
```

## 配置的優先順序

Spring Boot 的配置來源優先順序（由高到低）：

1. 命令列參數（`--server.port=9090`）
2. Java 系統屬性（`-Dserver.port=9090`）
3. 環境變數（`SERVER_PORT=9090`）
4. `application-{profile}.yml`
5. `application.yml`
6. `@ConfigurationProperties` 預設值

高優先度的配置會覆蓋低優先度的值。

## 配置驗證

結合 Jakarta Validation 驗證配置值：

```java
@ConfigurationProperties(prefix = "app.mail")
@Validated
public class MailProperties {

    @NotBlank(message = "寄件者不能為空")
    private String sender;

    @Min(value = 1, message = "重試次數至少為 1")
    @Max(value = 10, message = "重試次數最多為 10")
    private int retryCount = 3;

    @Email(message = "管理員信箱格式不正確")
    private String adminEmail;

    // getter / setter
}
```

如果配置值不符合驗證規則，應用程式在啟動時就會報錯，避免帶著錯誤配置運行。

## 敏感配置的處理

**絕對不要**把密碼、API Key 等敏感資訊直接寫在配置檔案中並提交到版本控制。

### 使用環境變數

```yaml
spring:
  datasource:
    password: ${DB_PASSWORD}

app:
  api-key: ${API_KEY}
```

### 使用 .env 檔案（開發環境）

配合 `spring-dotenv` 套件，從 `.env` 檔案讀取：

```
# .env（加入 .gitignore）
DB_PASSWORD=my-secret-password
API_KEY=sk-xxxxxxxxxxxx
```

## 替代方案：外部化配置

當應用程式規模成長到多服務、多環境時，`application.yml` + Profiles 可能不再足夠。以下是常見的外部化配置方案：

| 方案 | 適用場景 | 特點 |
|------|---------|------|
| **Spring Cloud Config** | 微服務架構、需要集中管理配置 | Git-backed、支援加密、即時刷新（搭配 Bus） |
| **Kubernetes ConfigMap / Secret** | K8s 部署環境 | 與容器編排整合、Secret 自動加密儲存 |
| **HashiCorp Vault** | 高安全需求、敏感資訊管理 | 動態密鑰、存取稽核、自動輪換 |

**選擇原則**：

- **單體應用或少量服務**：`application-{profile}.yml` + 環境變數已足夠
- **微服務但已在 K8s 上**：優先用 ConfigMap / Secret，避免額外維護 Config Server
- **需要集中管理 + 動態刷新**：Spring Cloud Config 是 Spring 生態系的原生方案
- **敏感資訊（密碼、金鑰、憑證）**：用 Vault 或雲端廠商的 Secret Manager，不要存在 Git 中

## 延伸閱讀

- [04 Spring Boot 自動配置與 Starters](04%20Spring%20Boot%20自動配置與%20Starters.md)——理解自動配置如何讀取配置值
- [03 配置中心（Spring Cloud Config）](../03-Microservices/03%20配置中心（Spring%20Cloud%20Config）.md)——Spring Cloud Config 的詳細設定與使用

## 小結

Spring Boot 的配置機制既靈活又強大。善用 `@ConfigurationProperties` 實現型別安全的配置繫結，搭配 Profiles 區分多環境配置，再用驗證確保配置正確性，就能構建出可靠的應用程式配置管理。

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
