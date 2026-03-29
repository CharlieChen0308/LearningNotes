# 08 Spring MVC 例外處理與驗證

> **版本**: Spring MVC 6.x / Spring Boot 3.x
> **來源**: `_archive/Spring MVC/03 Spring MVC 例外處理與資料驗證.md`

## 例外處理

在 Web 應用中，例外處理是不可或缺的一環。Spring MVC 提供了多種例外處理機制，從方法級到全域級，層次分明。

### @ExceptionHandler — 方法級例外處理

在 Controller 內定義，只處理**該 Controller** 拋出的例外：

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public UserDto getUser(@PathVariable Long id) {
        return userService.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("使用者不存在：" + id));
    }

    // 只處理本 Controller 的 ResourceNotFoundException
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleNotFound(ResourceNotFoundException ex) {
        return Map.of("error", ex.getMessage());
    }
}
```

### @RestControllerAdvice — 全域例外處理（推薦）

集中處理所有 Controller 的例外，避免重複程式碼：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * 資源不存在（404）
     */
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex,
                                         HttpServletRequest request) {
        return new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            request.getRequestURI()
        );
    }

    /**
     * 請求參數驗證失敗（400）— @Valid @RequestBody
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex,
                                           HttpServletRequest request) {
        Map<String, String> fieldErrors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            fieldErrors.put(error.getField(), error.getDefaultMessage())
        );
        return new ErrorResponse(400, "驗證失敗", request.getRequestURI(), fieldErrors);
    }

    /**
     * 請求參數型別不匹配（400）
     */
    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleTypeMismatch(MethodArgumentTypeMismatchException ex,
                                             HttpServletRequest request) {
        String message = String.format("參數 '%s' 的值 '%s' 型別不正確",
            ex.getName(), ex.getValue());
        return new ErrorResponse(400, message, request.getRequestURI());
    }

    /**
     * 請求方法不支援（405）
     */
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    @ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
    public ErrorResponse handleMethodNotAllowed(HttpRequestMethodNotSupportedException ex,
                                                 HttpServletRequest request) {
        return new ErrorResponse(405,
            "不支援 " + ex.getMethod() + " 方法",
            request.getRequestURI());
    }

    /**
     * 業務邏輯例外（自訂）
     */
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex,
                                                         HttpServletRequest request) {
        ErrorResponse body = new ErrorResponse(
            ex.getCode(), ex.getMessage(), request.getRequestURI());
        return ResponseEntity.status(ex.getCode()).body(body);
    }

    /**
     * 兜底：其他未預期的例外（500）
     */
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex, HttpServletRequest request) {
        // 記錄日誌，但不要把內部細節回傳給前端
        log.error("未預期的錯誤：{}", request.getRequestURI(), ex);
        return new ErrorResponse(500, "伺服器內部錯誤", request.getRequestURI());
    }
}
```

### 統一錯誤回應結構

```java
public record ErrorResponse(
    int status,
    String message,
    String path,
    Map<String, String> fieldErrors
) {
    public ErrorResponse(int status, String message, String path) {
        this(status, message, path, null);
    }
}
```

回應範例：

```json
{
    "status": 400,
    "message": "驗證失敗",
    "path": "/api/users",
    "fieldErrors": {
        "name": "姓名不能為空",
        "email": "Email 格式不正確"
    }
}
```

### 自訂業務例外

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

public class BusinessException extends RuntimeException {
    private final int code;

    public BusinessException(int code, String message) {
        super(message);
        this.code = code;
    }

    public int getCode() { return code; }
}
```

## 資料驗證

### Jakarta Validation 常用註解

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

| 註解 | 說明 | 範例 |
|------|------|------|
| `@NotNull` | 不能為 null | `@NotNull Integer age` |
| `@NotBlank` | 不能為 null 且去除空白後長度 > 0 | `@NotBlank String name` |
| `@NotEmpty` | 不能為 null 且不能為空（字串/集合） | `@NotEmpty List<String> tags` |
| `@Size` | 長度/大小限制 | `@Size(min=2, max=50)` |
| `@Min` / `@Max` | 數值範圍 | `@Min(0) @Max(150) int age` |
| `@Email` | Email 格式 | `@Email String email` |
| `@Pattern` | 正則表達式 | `@Pattern(regexp="^09\\d{8}$")` |
| `@Past` / `@Future` | 日期在過去/未來 | `@Past LocalDate birthday` |
| `@Positive` | 正數 | `@Positive BigDecimal price` |

### 驗證 @RequestBody

```java
public record CreateOrderRequest(
    @NotNull(message = "客戶 ID 不能為空")
    Long customerId,

    @NotEmpty(message = "訂單明細不能為空")
    @Size(max = 50, message = "單筆訂單最多 50 項商品")
    List<@Valid OrderItem> items,

    @Size(max = 200, message = "備註最多 200 字")
    String remark
) {
    public record OrderItem(
        @NotNull(message = "商品 ID 不能為空")
        Long productId,

        @Positive(message = "數量必須為正數")
        int quantity
    ) {}
}
```

```java
@PostMapping("/api/orders")
public OrderDto createOrder(@Valid @RequestBody CreateOrderRequest request) {
    return orderService.create(request);
}
```

### 驗證 @RequestParam 和 @PathVariable

在 Controller 類別上加 `@Validated`：

```java
@RestController
@RequestMapping("/api/users")
@Validated  // 啟用方法參數驗證
public class UserController {

    @GetMapping("/{id}")
    public UserDto getUser(@PathVariable @Min(1) Long id) { ... }

    @GetMapping
    public List<UserDto> search(
        @RequestParam @Size(min = 1, max = 50) String keyword) { ... }
}
```

### 分組驗證

同一個欄位在不同場景有不同的驗證規則：

```java
public class UserRequest {

    public interface Create {}
    public interface Update {}

    @Null(groups = Create.class, message = "新增時不需提供 ID")
    @NotNull(groups = Update.class, message = "更新時必須提供 ID")
    private Long id;

    @NotBlank(groups = {Create.class, Update.class})
    private String name;

    @NotBlank(groups = Create.class, message = "新增時密碼不能為空")
    private String password;
}
```

```java
@PostMapping
public UserDto create(@Validated(UserRequest.Create.class)
                      @RequestBody UserRequest request) { ... }

@PutMapping("/{id}")
public UserDto update(@Validated(UserRequest.Update.class)
                      @RequestBody UserRequest request) { ... }
```

### 自訂驗證註解

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = TaiwanPhoneValidator.class)
public @interface TaiwanPhone {
    String message() default "手機號碼格式不正確（應為 09 開頭的 10 位數字）";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class TaiwanPhoneValidator implements ConstraintValidator<TaiwanPhone, String> {

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true;  // null 由 @NotBlank 處理
        return value.matches("^09\\d{8}$");
    }
}
```

使用：

```java
public record CreateCustomerRequest(
    @NotBlank String name,
    @TaiwanPhone String phone
) {}
```

## 替代方案：Spring Boot 3.x ProblemDetail（RFC 7807）

Spring Boot 3.x 內建支援 RFC 7807 Problem Details 標準錯誤格式，不需要自訂 `ErrorResponse`：

```java
// 啟用 RFC 7807 支援
// application.yml
spring:
  mvc:
    problemdetails:
      enabled: true
```

啟用後，Spring MVC 內建的例外（如 `MethodArgumentNotValidException`、`HttpRequestMethodNotSupportedException`）會自動返回標準格式：

```json
{
    "type": "about:blank",
    "title": "Bad Request",
    "status": 400,
    "detail": "驗證失敗",
    "instance": "/api/users"
}
```

也可以在 `@ExceptionHandler` 中手動建構 `ProblemDetail`：

```java
@ExceptionHandler(ResourceNotFoundException.class)
public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(
        HttpStatus.NOT_FOUND, ex.getMessage());
    problem.setTitle("資源不存在");
    problem.setProperty("timestamp", Instant.now());
    return problem;
}
```

## 取捨分析：例外處理方式比較

| 方式 | 優點 | 缺點 | 適用場景 |
|------|------|------|----------|
| `@ExceptionHandler`（Controller 內） | 簡單直觀，可針對特定 Controller 客製 | 無法跨 Controller 共用，容易產生重複程式碼 | 單一 Controller 有獨特的錯誤處理邏輯 |
| `@RestControllerAdvice` | 全域統一，一處維護 | 所有 Controller 共用同一套邏輯，彈性較低 | 大多數專案的標準做法 |
| `ResponseStatusException` | 不需定義自訂例外類別，一行程式碼即可拋出 | 錯誤處理邏輯散落在業務程式碼中，難以統一格式 | 快速原型、小型專案 |
| `ProblemDetail`（RFC 7807） | 業界標準格式，跨語言/跨團隊互通性佳 | 需要 Spring Boot 3.x+，自訂欄位需額外處理 | 新專案、對外公開 API、微服務架構 |

**實務建議**：新專案建議採用 `@RestControllerAdvice` + `ProblemDetail` 組合——用 `@RestControllerAdvice` 集中管理，用 `ProblemDetail` 作為標準回應格式。既有專案若已有自訂 `ErrorResponse`，不必強行遷移，但對外 API 可考慮逐步採用 RFC 7807。

## 小結

Spring MVC 的例外處理（`@RestControllerAdvice` + `@ExceptionHandler`）和資料驗證（Jakarta Validation + `@Valid`）是構建健壯 API 的基石。全域例外處理統一錯誤回應格式，驗證機制在資料進入業務邏輯前就攔截不合法輸入，兩者搭配使用可以大幅提升 API 的可靠性。

## 延伸閱讀

- [06 Spring Boot RESTful API 開發](06%20Spring%20Boot%20RESTful%20API%20開發.md) — RESTful API 建立、回應方式取捨、統一回應格式
- [07 Spring MVC 註解驅動與 RESTful](07%20Spring%20MVC%20註解驅動與%20RESTful.md) — 註解驅動開發、JSON 序列化控制、@Controller vs @RestController

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
