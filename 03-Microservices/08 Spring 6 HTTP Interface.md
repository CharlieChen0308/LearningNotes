# 08 Spring 6 HTTP Interface

> **版本**：Spring Framework 6.x / Spring Boot 3.x / Java 17+
>
> Spring 6 引入了宣告式 HTTP 用戶端（HTTP Interface），類似 OpenFeign 但無需額外依賴，是 Spring 原生的解決方案。

## 1、HTTP Interface vs OpenFeign

| 特性 | HTTP Interface | OpenFeign |
|------|---------------|-----------|
| 依賴 | Spring 6 內建 | 需要 spring-cloud-starter-openfeign |
| 底層 | RestClient / WebClient | 自有 HTTP 引擎 |
| 響應式 | 支援（WebClient） | 不支援 |
| 攔截器 | Spring 原生 | Feign RequestInterceptor |
| 維護方 | Spring 官方 | Spring Cloud 社群 |
| 推薦 | **新專案首選** | 既有微服務可繼續用 |

## 2、快速開始

### 2.1 定義介面

```java
public interface UserClient {

    @GetExchange("/api/users/{id}")
    UserResponse getUser(@PathVariable Long id);

    @GetExchange("/api/users")
    List<UserResponse> listUsers(@RequestParam(required = false) String name);

    @PostExchange("/api/users")
    UserResponse createUser(@RequestBody CreateUserRequest request);

    @PutExchange("/api/users/{id}")
    UserResponse updateUser(@PathVariable Long id, @RequestBody UpdateUserRequest request);

    @DeleteExchange("/api/users/{id}")
    void deleteUser(@PathVariable Long id);
}
```

### 2.2 建立 Proxy Bean

> ⚠️ 此為教學簡化範例，生產環境應使用服務發現或外部化配置管理服務位址。

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public UserClient userClient() {
        RestClient restClient = RestClient.builder()
            .baseUrl("http://localhost:8081")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();

        RestClientAdapter adapter = RestClientAdapter.create(restClient);
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();

        return factory.createClient(UserClient.class);
    }
}
```

### 2.3 使用

```java
@Service
public class OrderService {
    private final UserClient userClient;

    public OrderService(UserClient userClient) {
        this.userClient = userClient;
    }

    public OrderResponse createOrder(Long userId, OrderRequest request) {
        // 呼叫遠端服務取得使用者資訊
        UserResponse user = userClient.getUser(userId);
        // 建立訂單...
        return new OrderResponse(user.name(), request.items());
    }
}
```

## 3、RestClient（Spring 6.1+）

`RestClient` 是 `RestTemplate` 的現代替代，fluent API 風格：

```java
RestClient client = RestClient.builder()
    .baseUrl("http://localhost:8081")
    .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
    .build();

// GET
UserResponse user = client.get()
    .uri("/api/users/{id}", 1)
    .retrieve()
    .body(UserResponse.class);

// POST
UserResponse created = client.post()
    .uri("/api/users")
    .body(new CreateUserRequest("Alice", "alice@example.com"))
    .retrieve()
    .body(UserResponse.class);

// 錯誤處理
UserResponse user = client.get()
    .uri("/api/users/{id}", 999)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
        throw new UserNotFoundException(999L);
    })
    .body(UserResponse.class);
```

## 4、進階配置

### 4.1 加入 JWT Token

```java
@Bean
public UserClient userClient(TokenProvider tokenProvider) {
    RestClient restClient = RestClient.builder()
        .baseUrl("http://user-service:8081")
        .requestInterceptor((request, body, execution) -> {
            request.getHeaders().setBearerAuth(tokenProvider.getToken());
            return execution.execute(request, body);
        })
        .build();

    return HttpServiceProxyFactory
        .builderFor(RestClientAdapter.create(restClient))
        .build()
        .createClient(UserClient.class);
}
```

### 4.2 逾時設定

```java
RestClient restClient = RestClient.builder()
    .baseUrl("http://user-service:8081")
    .requestFactory(new SimpleClientHttpRequestFactory() {{
        setConnectTimeout(Duration.ofSeconds(5));
        setReadTimeout(Duration.ofSeconds(10));
    }})
    .build();
```

### 4.3 響應式（WebClient）

```java
public interface ReactiveUserClient {

    @GetExchange("/api/users/{id}")
    Mono<UserResponse> getUser(@PathVariable Long id);

    @GetExchange("/api/users")
    Flux<UserResponse> listUsers();
}

@Bean
public ReactiveUserClient reactiveUserClient() {
    WebClient webClient = WebClient.builder()
        .baseUrl("http://user-service:8081")
        .build();

    return HttpServiceProxyFactory
        .builderFor(WebClientAdapter.create(webClient))
        .build()
        .createClient(ReactiveUserClient.class);
}
```

## 5、@HttpExchange 註解一覽

| 註解 | 用途 |
|------|------|
| `@HttpExchange` | 類別層級：設定基礎路徑 |
| `@GetExchange` | GET 請求 |
| `@PostExchange` | POST 請求 |
| `@PutExchange` | PUT 請求 |
| `@DeleteExchange` | DELETE 請求 |
| `@PatchExchange` | PATCH 請求 |

參數註解：`@PathVariable`、`@RequestParam`、`@RequestHeader`、`@RequestBody`、`@CookieValue`

## 6、已知限制

HTTP Interface 作為較新的技術，目前仍有以下限制需注意：

| 限制 | 說明 |
|------|------|
| **無內建服務發現** | 不像 OpenFeign 的 `@FeignClient(name = "user-service")` 可自動整合服務註冊中心，需手動配置服務 URL |
| **無內建斷路器 / 重試** | 不提供 fallback 機制，需自行整合 Resilience4j 等第三方函式庫 |
| **無內建負載均衡** | 需搭配 `@LoadBalanced` WebClient 或 Spring Cloud LoadBalancer 使用 |
| **社群資源較少** | 技術較新（Spring 6 / Spring Boot 3 才引入），網路上的範例與討論相對 OpenFeign 少很多 |
| **攔截器功能較簡單** | Spring 原生的 `ClientHttpRequestInterceptor` 功能不如 Feign 的 `RequestInterceptor` 豐富 |

## 7、生產環境注意事項

### 7.1 連線池配置（WebClient + Reactor Netty）

WebClient 底層使用 Reactor Netty，預設的連線池可能不符合生產環境需求：

```java
@Bean
public ReactiveUserClient reactiveUserClient() {
    // 自訂 Reactor Netty 連線池
    ConnectionProvider provider = ConnectionProvider.builder("custom")
        .maxConnections(200)                         // 最大連線數
        .maxIdleTime(Duration.ofSeconds(30))         // 閒置連線存活時間
        .maxLifeTime(Duration.ofMinutes(5))          // 連線最大存活時間
        .pendingAcquireTimeout(Duration.ofSeconds(10)) // 等待取得連線的逾時
        .build();

    HttpClient httpClient = HttpClient.create(provider);

    WebClient webClient = WebClient.builder()
        .baseUrl("http://user-service:8081")
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();

    return HttpServiceProxyFactory
        .builderFor(WebClientAdapter.create(webClient))
        .build()
        .createClient(ReactiveUserClient.class);
}
```

### 7.2 逾時設定（RestClient）

生產環境建議明確設定連線逾時與讀取逾時，避免無限等待：

```java
@Bean
public UserClient userClient() {
    SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
    requestFactory.setConnectTimeout(Duration.ofSeconds(3));  // 連線逾時
    requestFactory.setReadTimeout(Duration.ofSeconds(10));     // 讀取逾時

    RestClient restClient = RestClient.builder()
        .baseUrl("http://user-service:8081")
        .requestFactory(requestFactory)
        .build();

    return HttpServiceProxyFactory
        .builderFor(RestClientAdapter.create(restClient))
        .build()
        .createClient(UserClient.class);
}
```

### 7.3 搭配 Resilience4j 重試策略

```java
@Configuration
public class ResilienceConfig {

    @Bean
    public RetryRegistry retryRegistry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .retryExceptions(IOException.class, WebClientRequestException.class)
            .ignoreExceptions(UserNotFoundException.class) // 業務例外不重試
            .build();
        return RetryRegistry.of(config);
    }
}

@Service
public class ResilientOrderService {
    private final UserClient userClient;
    private final Retry retry;

    public ResilientOrderService(UserClient userClient, RetryRegistry retryRegistry) {
        this.userClient = userClient;
        this.retry = retryRegistry.retry("userClient");
    }

    public UserResponse getUser(Long id) {
        return Retry.decorateSupplier(retry, () -> userClient.getUser(id)).get();
    }
}
```

### 7.4 錯誤處理最佳實踐

```java
@Bean
public UserClient userClient() {
    RestClient restClient = RestClient.builder()
        .baseUrl("http://user-service:8081")
        .defaultStatusHandler(HttpStatusCode::is4xxClientError, (req, res) -> {
            // 統一處理 4xx 錯誤
            throw new RemoteServiceException("用戶端錯誤: " + res.getStatusCode());
        })
        .defaultStatusHandler(HttpStatusCode::is5xxServerError, (req, res) -> {
            // 統一處理 5xx 錯誤
            throw new RemoteServiceException("遠端服務錯誤: " + res.getStatusCode());
        })
        .build();

    return HttpServiceProxyFactory
        .builderFor(RestClientAdapter.create(restClient))
        .build()
        .createClient(UserClient.class);
}
```

## 8、Spring Cloud 整合指引

### 8.1 搭配 Spring Cloud LoadBalancer

透過 `@LoadBalanced` 讓 WebClient 具備用戶端負載均衡能力：

```java
@Configuration
public class LoadBalancedClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }

    @Bean
    public UserClient userClient(@LoadBalanced WebClient.Builder builder) {
        // 使用服務名稱取代實際 IP:Port
        WebClient webClient = builder
            .baseUrl("http://user-service")
            .build();

        return HttpServiceProxyFactory
            .builderFor(WebClientAdapter.create(webClient))
            .build()
            .createClient(UserClient.class);
    }
}
```

### 8.2 搭配服務發現（Eureka / Nacos）

只要應用已註冊至服務發現中心，`@LoadBalanced` WebClient 會自動將服務名稱解析為實際位址：

```yaml
# application.yml
spring:
  cloud:
    discovery:
      enabled: true

# Eureka 範例
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

```java
// 直接使用服務名稱即可，無需硬編碼 IP
WebClient webClient = loadBalancedBuilder
    .baseUrl("http://user-service")  // user-service 為註冊中心的服務名稱
    .build();
```

> **注意**：使用 `RestClient` 搭配 Spring Cloud LoadBalancer 時，需要額外配置 `RestTemplate` 的 `@LoadBalanced` 支援，目前 WebClient 的整合較為成熟。

## 9、何時不該使用 HTTP Interface

雖然 HTTP Interface 是 Spring 官方推薦的方向，但以下情境可能不適合：

| 情境 | 原因 | 建議方案 |
|------|------|---------|
| **需要豐富的 Feign 攔截器生態** | Feign 有大量成熟的 `RequestInterceptor`（如 OAuth2、日誌、指標），HTTP Interface 的攔截器較為基礎 | 繼續使用 OpenFeign |
| **需要開箱即用的 Spring Cloud 整合** | OpenFeign 的 `@FeignClient(name = "...")` 直接整合服務發現與負載均衡，HTTP Interface 需手動配置 | 繼續使用 OpenFeign |
| **既有大型微服務專案** | 大量 Feign Client 遷移成本高，且 OpenFeign 仍在維護中 | 維持 OpenFeign，新模組可用 HTTP Interface |
| **團隊不熟悉 WebClient** | 響應式場景下需要 WebClient 知識，學習曲線較高 | 先用 RestClient（同步），待團隊熟悉後再引入 WebClient |
| **需要內建的 fallback 機制** | OpenFeign + Sentinel/Resilience4j 提供宣告式 fallback，HTTP Interface 需自行實作 | 使用 OpenFeign 或手動整合 Resilience4j |

> **遷移建議**：既有專案不必急於遷移。可採取漸進策略 — 新模組使用 HTTP Interface，既有模組維持 OpenFeign，兩者可共存於同一專案中。

## 10、小結

| 概念 | 重點 |
|------|------|
| HTTP Interface | Spring 6 原生宣告式 HTTP 用戶端 |
| RestClient | Spring 6.1+ 取代 RestTemplate 的 fluent API |
| 底層 | 同步用 RestClient，響應式用 WebClient |
| 已知限制 | 無內建服務發現、斷路器、負載均衡，需手動整合 |
| 生產環境 | 必須配置連線池、逾時、重試策略與統一錯誤處理 |
| Spring Cloud 整合 | 透過 `@LoadBalanced` WebClient 搭配服務發現使用 |
| 與 OpenFeign | 新專案推薦 HTTP Interface，舊專案可繼續用 OpenFeign，兩者可共存 |

> **延伸閱讀**：
> - [07 宣告式 HTTP 用戶端（OpenFeign）](07%20宣告式%20HTTP%20用戶端（OpenFeign）.md) — 傳統方式對照
> - [01 Spring Cloud 概述與微服務架構](01%20Spring%20Cloud%20概述與微服務架構.md) — 微服務全貌
