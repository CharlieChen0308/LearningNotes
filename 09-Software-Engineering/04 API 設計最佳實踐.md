# 04 API 設計最佳實踐

> **版本**：Java 17+ / Spring Boot 3.x — RESTful API 設計規範、版本策略、錯誤處理

API 是系統之間溝通的契約。設計得好，前後端協作順暢、維護成本低；設計得差，每次改版都像拆炸彈。本文從理論模型到實務落地，整理 RESTful API 設計的核心原則與 Spring Boot 實作方式。

---

## 1、Richardson 成熟度模型

Leonard Richardson 提出的成熟度模型，將 REST API 分為四個層級，幫助我們判斷「目前的 API 設計到了什麼程度」。

### 各層級說明

| 層級 | 名稱 | 特徵 | 範例 |
|------|------|------|------|
| Level 0 | The Swamp of POX | 一個 URL 做所有事 | `POST /api` + body 帶 action |
| Level 1 | Resources | 引入資源概念，不同資源不同 URL | `POST /users`、`POST /orders` |
| Level 2 | HTTP Verbs | 正確使用 HTTP 動詞與狀態碼 | `GET /users/1`、`DELETE /users/1` |
| Level 3 | HATEOAS | 回應中包含超媒體連結，引導客戶端操作 | 回應帶 `_links` 欄位 |

### Level 0：一個 URL 做所有事

最原始的做法，所有請求都發到同一個端點，用 body 裡的欄位區分操作：

```json
POST /api
{
  "action": "getUser",
  "userId": 1
}
```

這本質上是 RPC 風格，完全沒有利用 HTTP 協定本身的語意。

### Level 1：資源（Resources）

開始把不同的「東西」拆成不同的 URL，但動詞還是亂用：

```
POST /users/getById     ← 查詢卻用 POST
POST /users/delete       ← 刪除也用 POST
```

### Level 2：HTTP 動詞

正確搭配 HTTP Method 與狀態碼，這是目前業界主流做法：

```
GET    /users          → 200 OK
POST   /users          → 201 Created
PUT    /users/1        → 200 OK
DELETE /users/1        → 204 No Content
```

### Level 3：HATEOAS

回應中包含可用操作的超連結，客戶端不需要硬編碼 URL：

```json
{
  "id": 1,
  "name": "Charlie",
  "_links": {
    "self": { "href": "/users/1" },
    "orders": { "href": "/users/1/orders" }
  }
}
```

Spring HATEOAS 提供了 `EntityModel`、`CollectionModel` 等工具支援，但實作成本較高。

### 務實建議

**大多數團隊做到 Level 2 就夠了。** Level 3 的 HATEOAS 在理論上很優雅，但實務中前後端通常共同維護 API 文件（如 SpringDoc），超連結帶來的好處有限，反而增加回應體積和實作複雜度。

```mermaid
flowchart LR
    L0["Level 0<br/>單一端點"] --> L1["Level 1<br/>資源拆分"]
    L1 --> L2["Level 2<br/>HTTP 動詞 + 狀態碼"]
    L2 --> L3["Level 3<br/>HATEOAS"]

    style L2 fill:#4CAF50,color:#fff
    style L3 fill:#9E9E9E,color:#fff
```

> 綠色 = 推薦目標，灰色 = 視需求決定。

---

## 2、URL 命名規範

### 核心原則

1. **用名詞不用動詞**：URL 代表資源，動作由 HTTP Method 表達
2. **用複數**：`/users` 代表使用者集合，`/users/{id}` 代表單一使用者
3. **用 kebab-case**：多單字用 `-` 連接，如 `/order-items`
4. **巢狀資源不超過兩層**：`/users/{userId}/orders` 可以，再深就該攤平

### 過濾、排序、分頁用 Query Parameter

```
GET /users?role=admin&sort=createdAt,desc&page=0&size=20
```

不要把過濾條件放進路徑，例如 `/users/role/admin` 是不好的設計。

### 錯誤 vs 正確的 URL 範例對照表

| 錯誤寫法 | 正確寫法 | 說明 |
|----------|----------|------|
| `GET /getUsers` | `GET /users` | 動詞交給 HTTP Method |
| `GET /getUserById?id=1` | `GET /users/1` | 用路徑參數表示唯一資源 |
| `POST /deleteUser` | `DELETE /users/1` | 刪除用 DELETE |
| `GET /user/1` | `GET /users/1` | 集合用複數 |
| `GET /users/1/orders/5/items/3` | `GET /order-items/3` | 巢狀過深應攤平 |
| `POST /users/search` | `GET /users?name=xxx` | 簡單查詢用 Query Parameter |
| `GET /users/1/activate` | `PATCH /users/1` + body | 狀態變更用 PATCH |

> **例外**：複雜搜尋條件（多欄位 + 邏輯運算）確實適合用 `POST /users/search` + Request Body，因為 GET 的 URL 長度有限制。

---

## 3、HTTP 狀態碼正確使用

### 常用狀態碼對照

| 狀態碼 | 意義 | 使用場景 | Spring 對應 |
|--------|------|----------|-------------|
| 200 | OK | 查詢成功、更新成功 | 預設回傳 |
| 201 | Created | 新增資源成功 | `@ResponseStatus(HttpStatus.CREATED)` |
| 204 | No Content | 刪除成功，不需回傳 body | `ResponseEntity.noContent().build()` |
| 400 | Bad Request | 參數驗證失敗、格式錯誤 | `MethodArgumentNotValidException` |
| 401 | Unauthorized | 未認證（沒帶 Token 或 Token 過期） | Spring Security 自動處理 |
| 403 | Forbidden | 已認證但無權限 | Spring Security 自動處理 |
| 404 | Not Found | 資源不存在 | 拋出自訂 Exception |
| 409 | Conflict | 資源衝突（如重複建立） | 拋出自訂 Exception |
| 422 | Unprocessable Entity | 語法正確但語意錯誤 | 業務邏輯驗證失敗 |
| 500 | Internal Server Error | 伺服器未預期錯誤 | 未捕捉的 Exception |

### 常見錯誤：所有回應都回 200

```json
// 錯誤做法：HTTP 200 + 自訂 code
HTTP/1.1 200 OK
{
  "code": 404,
  "message": "User not found"
}
```

這會讓 HTTP 基礎設施（快取、監控、負載均衡器）無法正確判斷請求狀態。**HTTP 狀態碼就是為此設計的，請正確使用。**

```java
// 正確做法：讓 HTTP 狀態碼反映真實狀態
@GetMapping("/{id}")
public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
    return userService.findById(id)
            .map(ResponseEntity::ok)                           // 200
            .orElseThrow(() -> new ResourceNotFoundException(  // 404
                    "User", "id", id));
}
```

---

## 4、錯誤回應格式統一

### 統一錯誤格式

無論什麼錯誤，回應結構應保持一致，讓前端可以統一處理：

```java
public record ErrorResponse(
    String code,           // 業務錯誤碼，如 "USER-001"
    String message,        // 人類可讀的錯誤訊息
    String path,           // 請求路徑
    Instant timestamp,     // 發生時間
    List<FieldError> errors // 欄位驗證錯誤（可選）
) {
    public record FieldError(
        String field,
        String message,
        Object rejectedValue
    ) {}
}
```

### 用 @RestControllerAdvice 統一處理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 業務例外
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(
            BusinessException ex, HttpServletRequest request) {
        var body = new ErrorResponse(
                ex.getCode(),
                ex.getMessage(),
                request.getRequestURI(),
                Instant.now(),
                null
        );
        return ResponseEntity.status(ex.getHttpStatus()).body(body);
    }

    // 參數驗證失敗
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex,
            HttpServletRequest request) {
        var fieldErrors = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> new ErrorResponse.FieldError(
                        e.getField(),
                        e.getDefaultMessage(),
                        e.getRejectedValue()))
                .toList();

        var body = new ErrorResponse(
                "VALIDATION-001",
                "輸入資料驗證失敗",
                request.getRequestURI(),
                Instant.now(),
                fieldErrors
        );
        return ResponseEntity.badRequest().body(body);
    }

    // 未預期的例外（兜底）
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(
            Exception ex, HttpServletRequest request) {
        log.error("未預期錯誤：{}", request.getRequestURI(), ex);
        var body = new ErrorResponse(
                "SYSTEM-001",
                "系統發生錯誤，請稍後再試",
                request.getRequestURI(),
                Instant.now(),
                null
        );
        return ResponseEntity.internalServerError().body(body);
    }
}
```

### 錯誤碼設計（模組前綴 + 編號）

| 模組 | 前綴 | 範例 |
|------|------|------|
| 使用者 | USER | USER-001：使用者不存在 |
| 訂單 | ORDER | ORDER-001：訂單已取消 |
| 認證 | AUTH | AUTH-001：Token 過期 |
| 驗證 | VALIDATION | VALIDATION-001：欄位驗證失敗 |
| 系統 | SYSTEM | SYSTEM-001：未預期錯誤 |

這樣前端可以根據前綴做不同的錯誤處理策略，後端也容易追蹤問題來源。

---

## 5、版本策略

API 上線後，修改介面就可能影響所有客戶端。版本策略決定了「如何在不破壞舊客戶端的前提下演進 API」。

### 常見方案比較

| 策略 | 範例 | 優點 | 缺點 |
|------|------|------|------|
| URL 版本 | `/api/v1/users` | 直覺、易理解、易路由 | URL 會膨脹 |
| Header 版本 | `Accept: application/vnd.api.v1+json` | URL 乾淨 | 不易測試、不易分享 |
| Query Parameter | `/api/users?version=1` | 簡單 | 容易被忽略、快取問題 |

### URL 版本（推薦）

最常見也最務實的做法：

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    // v1 的實作
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    // v2 的實作，回應格式可能不同
}
```

### 何時需要版本？何時不需要？

| 情境 | 需要新版本 | 不需要新版本 |
|------|-----------|-------------|
| 移除欄位 | 是 | - |
| 修改欄位型別 | 是 | - |
| 新增必填參數 | 是 | - |
| 新增選填欄位 | - | 是（向後相容） |
| 新增新端點 | - | 是 |
| 修改內部邏輯（不影響介面） | - | 是 |

**務實建議**：內部系統（前後端同團隊維護）通常不需要版本化，直接同步修改即可。版本化主要用於對外公開 API 或跨團隊協作。

---

## 6、分頁設計

### Offset-based 分頁

Spring Data 預設支援，適合大部分場景：

```
GET /users?page=0&size=20&sort=createdAt,desc
```

```java
@GetMapping
public Page<UserDto> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
    return userService.findAll(PageRequest.of(page, size));
}
```

回應格式（Spring `Page<T>` 序列化後）：

```json
{
  "content": [
    { "id": 1, "name": "Alice" },
    { "id": 2, "name": "Bob" }
  ],
  "totalElements": 150,
  "totalPages": 8,
  "number": 0,
  "size": 20,
  "first": true,
  "last": false
}
```

**缺點**：當資料量大且有頻繁寫入時，`OFFSET` 會越來越慢（資料庫要跳過前 N 筆）。

### Cursor-based 分頁

用上一頁最後一筆的識別值作為游標，適合大資料量或即時動態（如動態消息流）：

```
GET /users?cursor=eyJpZCI6MTAwfQ&size=20
```

```java
@GetMapping
public CursorPage<UserDto> getUsers(
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") int size) {
    Long lastId = decodeCursor(cursor);  // Base64 解碼
    List<UserDto> items = userService.findAfterId(lastId, size + 1);

    boolean hasNext = items.size() > size;
    if (hasNext) items = items.subList(0, size);

    String nextCursor = hasNext
            ? encodeCursor(items.getLast().id())
            : null;

    return new CursorPage<>(items, nextCursor, hasNext);
}
```

### Page vs Slice

| 特性 | `Page<T>` | `Slice<T>` |
|------|-----------|------------|
| 總筆數查詢 | 是（額外 `COUNT` 查詢） | 否 |
| 適用場景 | 需要顯示「共 N 頁」 | 無限捲動、只需「有沒有下一頁」 |
| 效能 | 較慢（大表的 `COUNT` 很貴） | 較快 |

**務實建議**：後台管理系統用 `Page<T>`（使用者需要翻頁導覽）；前台使用者介面用 `Slice<T>` 或 Cursor-based（效能優先）。

---

## 7、冪等性設計

### 什麼是冪等性？

同一個請求執行一次和執行多次，產生的**效果相同**。這在網路不穩定、客戶端重試的場景下非常重要。

| HTTP Method | 天然冪等？ | 說明 |
|-------------|-----------|------|
| GET | 是 | 查詢不改變狀態 |
| PUT | 是 | 完整覆蓋，多次結果相同 |
| DELETE | 是 | 刪除一次和刪除多次結果相同（已刪除） |
| PATCH | 視實作 | 增量更新可能不冪等（如 `count += 1`） |
| POST | 否 | 每次可能建立新資源 |

### POST 的冪等性問題

使用者點了兩次「送出訂單」，結果建立了兩筆訂單。解法：使用 `Idempotency-Key` Header。

```
POST /orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{ "productId": 1, "quantity": 2 }
```

### 用 Redis 實作防重複提交

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final StringRedisTemplate redis;
    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<OrderDto> createOrder(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @Valid @RequestBody CreateOrderRequest request) {

        String redisKey = "idempotency:" + idempotencyKey;

        // 嘗試設定 key，10 分鐘過期
        Boolean isNew = redis.opsForValue()
                .setIfAbsent(redisKey, "PROCESSING", Duration.ofMinutes(10));

        if (Boolean.FALSE.equals(isNew)) {
            // 已處理過，取得快取的回應
            String cachedResponse = redis.opsForValue().get(redisKey);
            if ("PROCESSING".equals(cachedResponse)) {
                return ResponseEntity.status(HttpStatus.CONFLICT).build();
            }
            return ResponseEntity.ok(deserialize(cachedResponse));
        }

        try {
            OrderDto result = orderService.create(request);
            // 儲存結果供重複請求使用
            redis.opsForValue().set(redisKey, serialize(result),
                    Duration.ofMinutes(10));
            return ResponseEntity.status(HttpStatus.CREATED).body(result);
        } catch (Exception ex) {
            redis.delete(redisKey);  // 失敗時清除，允許重試
            throw ex;
        }
    }
}
```

**重點**：`setIfAbsent`（即 Redis `SETNX`）保證了原子性，即使並發請求也只有一個能成功。

---

## 8、API 安全基礎

### Authentication vs Authorization

| 概念 | 問題 | 常見機制 |
|------|------|----------|
| Authentication（認證） | 你是誰？ | JWT、Session、OAuth 2.0 |
| Authorization（授權） | 你能做什麼？ | RBAC（角色）、ABAC（屬性） |

### Bearer Token（JWT）

```
GET /api/v1/users/me
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Spring Security 設定：

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable())          // REST API 不需要 CSRF
        .sessionManagement(session ->
            session.sessionCreationPolicy(STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/v1/auth/**").permitAll()
            .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 ->
            oauth2.jwt(Customizer.withDefaults()))
        .build();
}
```

### Rate Limiting

防止 API 被濫用。常見做法：

- **固定視窗**：每分鐘最多 100 次請求
- **滑動視窗**：更精確但實作較複雜
- **令牌桶**：允許短暫突發流量

Spring Boot 搭配 Spring Cloud Gateway 或 Bucket4j 可以實現。回應 Header 應包含限流資訊：

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1672531200
```

### 敏感資料不放 URL

URL 會出現在瀏覽器歷史紀錄、伺服器日誌、Proxy 日誌中：

```
# 錯誤：Token 出現在 URL
GET /api/verify?token=abc123secret

# 正確：放在 Header 或 Body
POST /api/verify
Authorization: Bearer abc123secret
```

同理，密碼、身分證字號等敏感資料都不應出現在 URL 或 Query Parameter 中。

---

## 9、小結

### 設計檢查表

| 項目 | 要點 | 優先級 |
|------|------|--------|
| 成熟度 | 做到 Level 2（HTTP 動詞 + 正確狀態碼） | 必要 |
| URL 命名 | 名詞複數、kebab-case、巢狀不超過兩層 | 必要 |
| 狀態碼 | 正確反映請求結果，不要全部 200 | 必要 |
| 錯誤格式 | 統一結構 + `@RestControllerAdvice` | 必要 |
| 版本策略 | 內部系統可省略；對外 API 用 URL 版本 | 視情況 |
| 分頁 | 小資料量用 `Page<T>`；大資料量用 Cursor | 必要 |
| 冪等性 | POST 搭配 `Idempotency-Key` | 涉及金流必要 |
| 安全 | JWT + RBAC + Rate Limiting | 必要 |

### 延伸閱讀

- [06 Spring Boot RESTful API 開發](../02-Spring-Ecosystem/06%20Spring%20Boot%20RESTful%20API%20開發.md)：Spring Boot 實作細節與 Controller 範例
- [15 API 文件（SpringDoc OpenAPI）](../02-Spring-Ecosystem/15%20API%20文件（SpringDoc%20OpenAPI）.md)：用 SpringDoc 自動產生 API 文件
- [14 Spring Security 與 JWT](../02-Spring-Ecosystem/14%20Spring%20Security%20與%20JWT.md)：認證與授權的完整實作
