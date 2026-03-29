# 05 熔斷與限流（Resilience4j）

> **版本**：Spring Cloud 2023.x / Spring Boot 3.x / Java 17+

## 服務雪崩問題

在微服務架構中，服務間存在呼叫鏈。當某個服務出現故障或回應緩慢時，呼叫它的服務也會被拖慢，進而影響更上層的服務，最終導致整個系統崩潰——這就是「服務雪崩」。

```
使用者 → 閘道 → 訂單服務 → 使用者服務（故障）
                        → 商品服務
                        → 支付服務
```

當使用者服務故障時，訂單服務的執行緒都阻塞在等待使用者服務回應上，導致訂單服務也無法處理其他請求。

## Resilience4j 簡介

Resilience4j 是一個輕量級的容錯函式庫，專為 Java 8 和函數式程式設計設計，取代了早期的 Netflix Hystrix。它提供以下核心功能：

| 功能 | 說明 |
|------|------|
| CircuitBreaker（熔斷器）| 當故障率超過閾值時，暫停對下游服務的呼叫 |
| RateLimiter（限流器）| 限制一段時間內的呼叫次數 |
| TimeLimiter（逾時控制）| 限制呼叫的最大等待時間 |
| Retry（重試）| 自動重試失敗的呼叫 |
| Bulkhead（隔離）| 限制並行呼叫數量，防止資源耗盡 |

## 新增依賴

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

## 熔斷器（CircuitBreaker）

熔斷器有三種狀態：

- **CLOSED（關閉）**：正常狀態，請求正常通過
- **OPEN（開啟）**：故障狀態，請求直接失敗，不再呼叫下游
- **HALF_OPEN（半開）**：嘗試放行少量請求，判斷是否恢復

### 配置

```yaml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        # 滑動窗口大小
        sliding-window-size: 10
        # 失敗率閾值（百分比），超過此值觸發熔斷
        failure-rate-threshold: 50
        # OPEN 狀態等待時間，之後切換到 HALF_OPEN
        wait-duration-in-open-state: 10s
        # HALF_OPEN 時允許的請求數
        permitted-number-of-calls-in-half-open-state: 3
        # 最小呼叫次數（低於此數不計算失敗率）
        minimum-number-of-calls: 5
```

### 使用方式

```java
@Service
public class UserServiceClient {

    @Autowired
    private RestTemplate restTemplate;

    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    public Map<String, Object> getUser(Long id) {
        return restTemplate.getForObject(
            "http://service-user/user/" + id, Map.class);
    }

    // 降級方法：參數需與原方法一致，額外加一個 Throwable
    public Map<String, Object> getUserFallback(Long id, Throwable t) {
        Map<String, Object> fallback = new HashMap<>();
        fallback.put("id", id);
        fallback.put("name", "暫時無法獲取");
        fallback.put("error", t.getMessage());
        return fallback;
    }
}
```

## 限流器（RateLimiter）

### 配置

```yaml
resilience4j:
  ratelimiter:
    instances:
      userService:
        # 限流週期
        limit-refresh-period: 1s
        # 每個週期允許的請求數
        limit-for-period: 10
        # 等待許可的最大時間
        timeout-duration: 500ms
```

### 使用方式

```java
@RateLimiter(name = "userService", fallbackMethod = "rateLimitFallback")
@GetMapping("/user/{id}")
public Map<String, Object> getUser(@PathVariable Long id) {
    return userServiceClient.getUser(id);
}

public Map<String, Object> rateLimitFallback(Long id, Throwable t) {
    Map<String, Object> result = new HashMap<>();
    result.put("message", "請求過於頻繁，請稍後重試");
    return result;
}
```

## 重試（Retry）

### 配置

```yaml
resilience4j:
  retry:
    instances:
      userService:
        # 最大重試次數
        max-attempts: 3
        # 重試間隔
        wait-duration: 1s
        # 需要重試的例外
        retry-exceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
```

### 使用方式

```java
@Retry(name = "userService", fallbackMethod = "getUserFallback")
public Map<String, Object> getUser(Long id) {
    return restTemplate.getForObject(
        "http://service-user/user/" + id, Map.class);
}
```

## 隔離（Bulkhead）

### 信號量隔離

```yaml
resilience4j:
  bulkhead:
    instances:
      userService:
        # 最大並行呼叫數
        max-concurrent-calls: 20
        # 等待進入的最大時間
        max-wait-duration: 500ms
```

### 執行緒池隔離

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      userService:
        max-thread-pool-size: 10
        core-thread-pool-size: 5
        queue-capacity: 20
```

### 使用方式

```java
@Bulkhead(name = "userService", fallbackMethod = "getUserFallback")
public Map<String, Object> getUser(Long id) {
    return restTemplate.getForObject(
        "http://service-user/user/" + id, Map.class);
}
```

## 組合使用

多個容錯策略可以組合使用，執行順序為：Retry → CircuitBreaker → RateLimiter → Bulkhead

```java
@CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
@RateLimiter(name = "userService")
@Retry(name = "userService")
@Bulkhead(name = "userService")
public Map<String, Object> getUser(Long id) {
    return restTemplate.getForObject(
        "http://service-user/user/" + id, Map.class);
}
```

## 監控

新增 Actuator 依賴後，可以查看熔斷器狀態：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers
  health:
    circuitbreakers:
      enabled: true
```

存取 `http://localhost:8001/actuator/circuitbreakers` 可查看所有熔斷器的即時狀態。

## 小結

Resilience4j 提供了完整的微服務容錯方案，透過熔斷、限流、重試、隔離等機制，有效防止服務雪崩，提升系統的整體穩定性。

## 延伸閱讀

- [04 API 閘道（Spring Cloud Gateway）](04%20API%20%E9%96%98%E9%81%93%EF%BC%88Spring%20Cloud%20Gateway%EF%BC%89.md) — Gateway 整合熔斷器
- [07 宣告式 HTTP 用戶端（OpenFeign）](07%20%E5%AE%A3%E5%91%8A%E5%BC%8F%20HTTP%20%E7%94%A8%E6%88%B6%E7%AB%AF%EF%BC%88OpenFeign%EF%BC%89.md) — Feign 整合 Resilience4j
