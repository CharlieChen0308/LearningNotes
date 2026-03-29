# 06 Spring Boot RESTful API 開發

> **版本**: Spring Boot 3.x
> **來源**: `_archive/Spring Boot/04 Spring Boot 開發 RESTful API.md`

## RESTful API 簡介

REST（Representational State Transfer）是一種基於 HTTP 協定的 API 設計風格，核心原則：

| HTTP 方法 | 操作 | 範例 |
|-----------|------|------|
| GET | 查詢資源 | `GET /users/1` |
| POST | 新增資源 | `POST /users` |
| PUT | 完整更新資源 | `PUT /users/1` |
| PATCH | 部分更新資源 | `PATCH /users/1` |
| DELETE | 刪除資源 | `DELETE /users/1` |

## 建立 RESTful Controller

### 基本結構

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    // 查詢全部
    @GetMapping
    public List<UserDto> getAllUsers() {
        return userService.findAll();
    }

    // 根據 ID 查詢
    @GetMapping("/{id}")
    public UserDto getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    // 新增
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDto createUser(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }

    // 更新
    @PutMapping("/{id}")
    public UserDto updateUser(@PathVariable Long id,
                              @Valid @RequestBody UpdateUserRequest request) {
        return userService.update(id, request);
    }

    // 刪除
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### 常用註解說明

| 註解 | 說明 |
|------|------|
| `@RestController` | 等同 `@Controller` + `@ResponseBody`，所有方法返回 JSON |
| `@RequestMapping` | 設定基礎路徑 |
| `@GetMapping` | 處理 GET 請求（等同 `@RequestMapping(method = GET)`） |
| `@PostMapping` | 處理 POST 請求 |
| `@PutMapping` | 處理 PUT 請求 |
| `@DeleteMapping` | 處理 DELETE 請求 |
| `@PatchMapping` | 處理 PATCH 請求 |
| `@PathVariable` | 從 URL 路徑取得參數 |
| `@RequestParam` | 從查詢字串取得參數 |
| `@RequestBody` | 將請求正文反序列化為 Java 物件 |
| `@Valid` | 觸發請求物件的資料驗證 |
| `@ResponseStatus` | 指定回應的 HTTP 狀態碼 |

## 請求參數處理

### @PathVariable — 路徑參數

```java
// GET /api/users/1
@GetMapping("/{id}")
public UserDto getUser(@PathVariable Long id) { ... }

// GET /api/departments/3/users/5
@GetMapping("/departments/{deptId}/users/{userId}")
public UserDto getUser(@PathVariable Long deptId,
                       @PathVariable Long userId) { ... }
```

### @RequestParam — 查詢參數

```java
// GET /api/users?page=0&size=20&sort=name
@GetMapping
public Page<UserDto> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "name") String sort) {
    return userService.findAll(PageRequest.of(page, size, Sort.by(sort)));
}

// GET /api/users/search?keyword=張
@GetMapping("/search")
public List<UserDto> searchUsers(
        @RequestParam String keyword,
        @RequestParam(required = false) String department) {
    return userService.search(keyword, department);
}
```

### @RequestBody — 請求正文

```java
// POST /api/users
// Content-Type: application/json
// { "name": "張三", "email": "zhang@example.com" }
@PostMapping
public UserDto createUser(@Valid @RequestBody CreateUserRequest request) {
    return userService.create(request);
}
```

## ResponseEntity — 完整控制回應

`ResponseEntity` 可以自訂 HTTP 狀態碼、標頭和正文：

```java
@GetMapping("/{id}")
public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
    return userService.findById(id)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
}

@PostMapping
public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserRequest request) {
    UserDto created = userService.create(request);
    URI location = URI.create("/api/users/" + created.getId());
    return ResponseEntity.created(location).body(created);
}

@DeleteMapping("/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
}
```

## 回應方式取捨分析

Spring Boot RESTful API 有三種主流回應方式，各有適用場景：

| 方式 | 優點 | 缺點 | 適用場景 |
|------|------|------|----------|
| 直接返回物件 | 程式碼最簡潔，方法簽名即文件 | 無法自訂 HTTP 狀態碼與標頭 | 簡單 CRUD、狀態碼固定的端點 |
| `ResponseEntity` | 完整控制狀態碼、標頭、正文 | 程式碼較冗長，每個方法都要包裝 | 需動態決定狀態碼（如 200/404）、需設定自訂標頭 |
| 自訂包裝（`ApiResponse`） | 前端解析格式統一，可攜帶業務碼 | HTTP 狀態碼永遠是 200，不利於中間層快取與監控；違反 REST 語意 | 前端團隊偏好統一格式、需要業務級錯誤碼 |

**實務建議**：優先使用「直接返回 + `@ResponseStatus`」處理固定狀態碼，需要動態決定時改用 `ResponseEntity`。自訂包裝（`ApiResponse`）除非團隊有明確共識，否則容易造成 HTTP 語意與實際狀態不一致的問題。

## 統一回應格式

定義統一的 API 回應結構：

```java
public record ApiResponse<T>(
    int code,
    String message,
    T data
) {
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(200, "成功", data);
    }

    public static <T> ApiResponse<T> error(int code, String message) {
        return new ApiResponse<>(code, message, null);
    }
}
```

```java
@GetMapping("/{id}")
public ApiResponse<UserDto> getUser(@PathVariable Long id) {
    UserDto user = userService.findById(id);
    return ApiResponse.success(user);
}
```

回應範例：

```json
{
    "code": 200,
    "message": "成功",
    "data": {
        "id": 1,
        "name": "張三",
        "email": "zhang@example.com"
    }
}
```

## 資料驗證

### 新增依賴

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### 定義驗證規則

```java
public record CreateUserRequest(
    @NotBlank(message = "姓名不能為空")
    @Size(max = 50, message = "姓名不能超過 50 個字")
    String name,

    @NotBlank(message = "Email 不能為空")
    @Email(message = "Email 格式不正確")
    String email,

    @NotBlank(message = "密碼不能為空")
    @Size(min = 8, max = 20, message = "密碼長度必須在 8~20 之間")
    String password,

    @Min(value = 0, message = "年齡不能為負數")
    @Max(value = 150, message = "年齡不合理")
    Integer age
) {}
```

### 在 Controller 中使用 @Valid

```java
@PostMapping
public UserDto createUser(@Valid @RequestBody CreateUserRequest request) {
    return userService.create(request);
}
```

驗證失敗時會拋出 `MethodArgumentNotValidException`，配合全域例外處理可統一回傳錯誤訊息（見 [08 Spring MVC 例外處理與驗證](08%20Spring%20MVC%20例外處理與驗證.md)）。

## 全域例外處理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 資料驗證失敗
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<Map<String, String>> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage()));
        return ApiResponse.error(400, "驗證失敗");
    }

    // 資源不存在
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiResponse<Void> handleNotFound(ResourceNotFoundException ex) {
        return ApiResponse.error(404, ex.getMessage());
    }

    // 其他未預期的例外
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiResponse<Void> handleGeneral(Exception ex) {
        return ApiResponse.error(500, "伺服器內部錯誤");
    }
}
```

## CORS 跨域設定

### 全域設定

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

### 補充：YAML 配置方式

> **注意**：`spring.web.cors` 並非 Spring Boot 的有效屬性路徑，無法透過 `application.yml` 直接設定 CORS。請使用上方的 `WebMvcConfigurer` 程式化設定方式。

## 生產注意事項

### API 版本策略

常見做法是 URL 路徑版本（`/api/v1/users`），簡單直觀且方便路由層控制。Header 版本（`Accept: application/vnd.myapp.v1+json`）語意更正確但偵錯較困難。建議初期採用 URL 路徑版本，待 API 穩定後再評估是否遷移。

### Rate Limiting（限流）

生產環境務必對外部 API 加上限流。Spring Boot 可搭配 Bucket4j 或 API Gateway（如 Spring Cloud Gateway）的內建限流功能。建議在 Gateway 層統一處理，而非每個微服務各自實作。

### 分頁最佳實踐

- 永遠為列表端點提供分頁，避免一次返回全部資料
- 設定 `size` 上限（如最大 100），防止惡意請求拖垮資料庫
- 回應中包含 `totalElements`、`totalPages`、`currentPage` 等中繼資訊
- 大資料量場景考慮改用 Cursor-based 分頁取代 Offset-based

### 回應大小限制

- 避免嵌套過深的物件結構，善用 DTO 投影只返回前端需要的欄位
- 大量資料匯出場景使用串流回應（`StreamingResponseBody`），而非一次載入記憶體
- 可在 Nginx 或 Gateway 層設定 response body size 上限

## 小結

Spring Boot 結合 Spring MVC 的註解式開發，讓建立 RESTful API 變得非常簡潔。善用 `@RestController`、`ResponseEntity`、`@Valid` 和 `@RestControllerAdvice`，就能構建出結構清晰、驗證完整、錯誤處理統一的 API 服務。

## 延伸閱讀

- [07 Spring MVC 註解驅動與 RESTful](07%20Spring%20MVC%20註解驅動與%20RESTful.md) — 深入了解 Handler Method 參數繫結與 JSON 序列化控制
- [08 Spring MVC 例外處理與驗證](08%20Spring%20MVC%20例外處理與驗證.md) — 全域例外處理、資料驗證、自訂驗證註解
- [API 設計最佳實踐](../09-Software-Engineering/04%20API%20設計最佳實踐.md) — 跨框架的 API 設計原則與命名規範

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
