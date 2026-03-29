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

## 6、小結

| 概念 | 重點 |
|------|------|
| HTTP Interface | Spring 6 原生宣告式 HTTP 用戶端 |
| RestClient | Spring 6.1+ 取代 RestTemplate 的 fluent API |
| 底層 | 同步用 RestClient，響應式用 WebClient |
| 與 OpenFeign | 新專案推薦 HTTP Interface，舊專案可繼續用 OpenFeign |

> **延伸閱讀**：
> - [07 宣告式 HTTP 用戶端（OpenFeign）](07%20宣告式%20HTTP%20用戶端（OpenFeign）.md) — 傳統方式對照
> - [01 Spring Cloud 概述與微服務架構](01%20Spring%20Cloud%20概述與微服務架構.md) — 微服務全貌
