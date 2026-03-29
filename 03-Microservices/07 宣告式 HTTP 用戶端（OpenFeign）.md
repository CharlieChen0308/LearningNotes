# 07 宣告式 HTTP 用戶端（OpenFeign）

> **版本**：Spring Cloud 2023.x / Spring Boot 3.x / Java 17+

## 為什麼需要 OpenFeign

在微服務架構中，服務間需要頻繁地進行 HTTP 呼叫。使用 RestTemplate 雖然可行，但程式碼較為繁瑣，需要手動拼接 URL、處理參數序列化等。

OpenFeign 是一個**宣告式的 HTTP 用戶端**，只需要建立一個介面並加上註解，就能實現服務間的呼叫，像呼叫本地方法一樣簡單。

## 新增依賴

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

## 啟用 OpenFeign

在啟動類上新增 `@EnableFeignClients` 註解：

```java
@SpringBootApplication
@EnableFeignClients
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

## 基本使用

### 定義 Feign Client 介面

```java
@FeignClient(name = "service-user")
public interface UserClient {

    @GetMapping("/user/{id}")
    Map<String, Object> getUser(@PathVariable("id") Long id);

    @PostMapping("/user")
    Map<String, Object> createUser(@RequestBody Map<String, Object> user);

    @GetMapping("/user/list")
    List<Map<String, Object>> getUserList(
        @RequestParam("page") int page,
        @RequestParam("size") int size);
}
```

### 呼叫服務

```java
@RestController
@RequestMapping("/order")
public class OrderController {

    @Autowired
    private UserClient userClient;

    @GetMapping("/{userId}")
    public Map<String, Object> getOrder(@PathVariable Long userId) {
        // 像呼叫本地方法一樣呼叫遠端服務
        Map<String, Object> user = userClient.getUser(userId);

        Map<String, Object> order = new HashMap<>();
        order.put("orderId", 1001L);
        order.put("user", user);
        return order;
    }
}
```

## 進階配置

### 逾時設定

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          # 全域設定
          default:
            connect-timeout: 5000
            read-timeout: 5000
          # 針對特定服務
          service-user:
            connect-timeout: 3000
            read-timeout: 3000
```

### 日誌設定

Feign 提供四種日誌等級：

| 等級 | 說明 |
|------|------|
| NONE | 不記錄（預設）|
| BASIC | 記錄請求方法、URL、回應狀態碼和執行時間 |
| HEADERS | 在 BASIC 基礎上加上請求和回應標頭 |
| FULL | 記錄所有內容（標頭、正文、後設資料）|

```java
@Configuration
public class FeignConfig {

    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

```yaml
logging:
  level:
    com.example.client.UserClient: DEBUG
```

### 請求攔截器

在所有 Feign 請求中自動帶上認證 Token：

```java
@Configuration
public class FeignInterceptorConfig {

    @Bean
    public RequestInterceptor requestInterceptor() {
        return requestTemplate -> {
            // 從當前請求上下文取得 Token
            ServletRequestAttributes attributes = (ServletRequestAttributes)
                RequestContextHolder.getRequestAttributes();
            if (attributes != null) {
                String token = attributes.getRequest().getHeader("Authorization");
                if (token != null) {
                    requestTemplate.header("Authorization", token);
                }
            }
        };
    }
}
```

## Feign 整合 Resilience4j

### 啟用熔斷

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true
```

### 定義降級類

```java
@FeignClient(name = "service-user", fallback = UserClientFallback.class)
public interface UserClient {

    @GetMapping("/user/{id}")
    Map<String, Object> getUser(@PathVariable("id") Long id);
}
```

```java
@Component
public class UserClientFallback implements UserClient {

    @Override
    public Map<String, Object> getUser(Long id) {
        Map<String, Object> fallback = new HashMap<>();
        fallback.put("id", id);
        fallback.put("name", "服務暫時不可用");
        return fallback;
    }
}
```

### 使用 FallbackFactory 獲取異常資訊

```java
@FeignClient(name = "service-user", fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient {

    @GetMapping("/user/{id}")
    Map<String, Object> getUser(@PathVariable("id") Long id);
}
```

```java
@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        return new UserClient() {
            @Override
            public Map<String, Object> getUser(Long id) {
                Map<String, Object> fallback = new HashMap<>();
                fallback.put("id", id);
                fallback.put("error", "呼叫失敗：" + cause.getMessage());
                return fallback;
            }
        };
    }
}
```

## Feign 的請求壓縮

```yaml
spring:
  cloud:
    openfeign:
      compression:
        request:
          enabled: true
          mime-types: text/xml,application/xml,application/json
          min-request-size: 2048
        response:
          enabled: true
```

## RestTemplate vs OpenFeign 比較

| 特性 | RestTemplate | OpenFeign |
|------|-------------|-----------|
| 呼叫方式 | 程式碼編寫 | 宣告式介面 |
| 可讀性 | 一般 | 優秀，類似本地呼叫 |
| 內建負載均衡 | 需手動加 @LoadBalanced | 自動整合 |
| 熔斷整合 | 手動整合 | 原生支援 |
| 請求攔截 | 需自訂 | 提供 RequestInterceptor |

## 小結

OpenFeign 透過宣告式的方式大幅簡化了微服務間的 HTTP 呼叫，搭配 LoadBalancer 和 Resilience4j，可以輕鬆實現負載均衡和容錯處理。它是 Spring Cloud 微服務開發中最常用的元件之一。

## 延伸閱讀

- [08 Spring 6 HTTP Interface](08%20Spring%206%20HTTP%20Interface.md) — Spring 6 原生 HTTP 用戶端
- [06 負載均衡（Spring Cloud LoadBalancer）](06%20%E8%B2%A0%E8%BC%89%E5%9D%87%E8%A1%A1%EF%BC%88Spring%20Cloud%20LoadBalancer%EF%BC%89.md) — 用戶端負載均衡
