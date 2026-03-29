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

Resilience4j 是一個輕量級的容錯函式庫，專為 Java 8 和函數式程式設計設計。它取代了 Netflix Hystrix——Hystrix 已於 2018 年進入維護模式，官方明確宣布**停止開發（deprecated）**，不再接受新功能或修復，Spring Cloud 2020.x 起也已移除對 Hystrix 的官方支援。Resilience4j 提供以下核心功能：

| 功能 | 說明 |
|------|------|
| CircuitBreaker（熔斷器）| 當故障率超過閾值時，暫停對下游服務的呼叫 |
| RateLimiter（限流器）| 限制一段時間內的呼叫次數 |
| TimeLimiter（逾時控制）| 限制呼叫的最大等待時間 |
| Retry（重試）| 自動重試失敗的呼叫 |
| Bulkhead（隔離）| 限制並行呼叫數量，防止資源耗盡 |

## Resilience4j vs Alibaba Sentinel 比較

| 比較維度 | Resilience4j | Alibaba Sentinel |
|---------|-------------|-----------------|
| 定位與重量 | 輕量級函式庫，僅提供核心容錯原語，無額外基礎設施依賴 | 功能豐富的流量治理框架，內建控制台、規則管理、叢集限流 |
| Spring 生態整合 | Spring Cloud 官方推薦，`spring-cloud-starter-circuitbreaker` 預設實作 | 透過 `spring-cloud-alibaba` 整合，與 Nacos、Dubbo 等阿里生態深度綁定 |
| 社群與維護 | 國際社群活躍，GitHub 持續更新，文件完善 | 阿里巴巴主導，中文社群龐大，適合國內微服務技術棧 |
| 監控能力 | 依賴 Micrometer 整合 Prometheus/Grafana，需自行搭建 | 自帶 Dashboard 控制台，支援即時規則推送與監控 |
| 規則配置方式 | 程式碼或 YAML 靜態配置為主，動態調整需額外整合 | 支援動態規則推送（Nacos、ZooKeeper），可在運行時即時調整 |

**選型建議**：若專案使用標準 Spring Cloud 技術棧且偏好輕量方案，選 Resilience4j；若使用阿里雲或 Spring Cloud Alibaba 生態，且需要開箱即用的控制台與動態規則管理，選 Sentinel。

## 策略選型指引

不同的容錯模式適用於不同場景，選擇時應根據故障特性決定：

| 模式 | 適用場景 | 典型範例 |
|------|---------|---------|
| **CircuitBreaker（熔斷器）** | 下游服務不穩定，持續失敗 | 使用者服務頻繁超時，暫停呼叫等其自動恢復 |
| **RateLimiter（限流器）** | 保護自身服務不被過量請求壓垮 | API 對外開放，限制每秒最多 100 次呼叫 |
| **Retry（重試）** | 暫時性故障，重試後大機率成功 | 網路抖動、DNS 解析偶發逾時 |
| **Bulkhead（隔離）** | 隔離慢速消費者，避免拖垮整體資源 | 報表查詢很慢，但不應影響即時交易 |
| **TimeLimiter（逾時控制）** | 防止無限期等待，確保回應時間可控 | 第三方支付 API 偶爾無回應，需設定上限 |

**組合建議**：實務上常見組合為 `Retry + CircuitBreaker`（先重試暫時性故障，累積失敗後熔斷）或 `Bulkhead + CircuitBreaker`（隔離資源同時熔斷不穩定服務）。

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

> 此為教學簡化範例，生產環境需額外考慮：滑動窗口大小應根據實際流量調整（見下方「生產環境參數調校」章節）、失敗率閾值需配合業務容忍度設定。

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

> 此為教學簡化範例，生產環境需額外考慮：限流閾值應根據壓測結果與服務承載能力設定，並搭配監控告警。

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

> 此為教學簡化範例，生產環境需額外考慮：重試次數過多會放大下游壓力，建議搭配指數退避（exponential backoff）策略，並排除不可重試的例外（如 4xx 錯誤）。

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

> 此為教學簡化範例，生產環境需額外考慮：各策略的參數需協調一致（例如 Retry 的總耗時不應超過 TimeLimiter 的上限），並透過監控驗證組合效果。

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

## 生產環境參數調校

### 滑動窗口大小（sliding-window-size）

窗口太小會導致誤判（幾次偶發失敗就觸發熔斷），窗口太大則反應遲鈍。建議依據服務的 QPS 設定：

| 服務 QPS | 建議窗口大小 | 說明 |
|---------|------------|------|
| < 10 | 20～50 | 低流量服務需要較大窗口累積足夠樣本 |
| 10～100 | 50～100 | 中等流量，平衡靈敏度與穩定性 |
| > 100 | 100～200 | 高流量服務樣本充足，可使用較大窗口減少抖動 |

也可使用 `sliding-window-type: TIME_BASED`，以時間（如最近 60 秒）取代次數作為窗口。

### 失敗率閾值（failure-rate-threshold）

- **50%**（預設值）：適用於大多數場景，平衡容錯與可用性
- **30%～40%**：對可用性要求高的核心服務（如支付），提早熔斷保護
- **60%～80%**：對偶發錯誤容忍度高的非核心服務（如推薦、日誌）

**調校流程**：先以預設值上線 → 觀察 1～2 週監控數據 → 根據實際失敗分佈微調。

### 整合 Prometheus + Grafana 監控

Resilience4j 原生支援 Micrometer，只需新增依賴即可自動匯出指標：

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
```

常用 Grafana 監控指標：

| 指標名稱 | 用途 |
|---------|------|
| `resilience4j_circuitbreaker_state` | 熔斷器當前狀態（0=CLOSED, 1=OPEN, 2=HALF_OPEN） |
| `resilience4j_circuitbreaker_failure_rate` | 即時失敗率 |
| `resilience4j_circuitbreaker_calls_seconds` | 呼叫耗時分佈（可設定 P99 告警） |
| `resilience4j_ratelimiter_available_permissions` | 限流器剩餘許可數 |
| `resilience4j_bulkhead_available_concurrent_calls` | 隔離器剩餘並行數 |

**建議告警規則**：熔斷器進入 OPEN 狀態時立即告警；失敗率持續高於閾值 5 分鐘時告警。

## 小結

Resilience4j 提供了完整的微服務容錯方案，透過熔斷、限流、重試、隔離等機制，有效防止服務雪崩，提升系統的整體穩定性。

## 延伸閱讀

- [04 API 閘道（Spring Cloud Gateway）](04%20API%20%E9%96%98%E9%81%93%EF%BC%88Spring%20Cloud%20Gateway%EF%BC%89.md) — Gateway 整合熔斷器
- [07 宣告式 HTTP 用戶端（OpenFeign）](07%20%E5%AE%A3%E5%91%8A%E5%BC%8F%20HTTP%20%E7%94%A8%E6%88%B6%E7%AB%AF%EF%BC%88OpenFeign%EF%BC%89.md) — Feign 整合 Resilience4j
- [09 系統設計入門](../09-Software-Engineering/09%20系統設計入門.md) — 高可用設計與消除單點故障
