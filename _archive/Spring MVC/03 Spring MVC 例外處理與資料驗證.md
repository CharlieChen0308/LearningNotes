# 03 Spring MVC 例外處理與資料驗證

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

## 小結

Spring MVC 的例外處理（`@RestControllerAdvice` + `@ExceptionHandler`）和資料驗證（Jakarta Validation + `@Valid`）是構建健壯 API 的基石。全域例外處理統一錯誤回應格式，驗證機制在資料進入業務邏輯前就攔截不合法輸入，兩者搭配使用可以大幅提升 API 的可靠性。
