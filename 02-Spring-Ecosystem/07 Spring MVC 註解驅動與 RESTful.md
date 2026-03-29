# 07 Spring MVC 註解驅動與 RESTful

> **版本**: Spring MVC 6.x / Spring Boot 3.x
> **來源**: `_archive/Spring MVC/02 Spring MVC 註解驅動開發與 RESTful API.md`

## 從 XML 到註解驅動

早期的 Spring MVC（如本系列第 01 篇）需要 XML 配置 DispatcherServlet、ViewResolver 等，而現代 Spring MVC（特別是搭配 Spring Boot）已完全採用註解驅動，不再需要任何 XML 配置。

### 傳統方式 vs 現代方式對比

| 項目 | 傳統方式（Spring 3.x） | 現代方式（Spring 6.x） |
|------|----------------------|----------------------|
| 版本 | Spring 3.2.18 | Spring 6.2+ / Spring Boot 3.x |
| Java 版本 | Java 6+ | Java 17+ |
| 配置方式 | web.xml + XML 配置 | 純註解 + Java 配置 |
| 視圖技術 | JSP | Thymeleaf / 前後端分離 |
| 回應格式 | HTML 頁面 | JSON（RESTful API） |
| Controller | `@Controller` + `@RequestMapping` | `@RestController` + `@GetMapping` 等 |
| 依賴管理 | 手動管理每個 jar | Spring Boot Starter 自動管理 |

## DispatcherServlet 請求處理流程

不管使用哪種配置方式，Spring MVC 的核心處理流程不變：

1. 客戶端發送請求到 **DispatcherServlet**（前端控制器）
2. DispatcherServlet 查詢 **HandlerMapping**，找到對應的 Controller 方法
3. 透過 **HandlerAdapter** 呼叫 Controller 方法
4. Controller 返回結果
5. 如果是 `@ResponseBody`，由 **HttpMessageConverter** 將物件轉為 JSON
6. 如果是視圖名稱，由 **ViewResolver** 解析為具體視圖

## @RestController — 建立 REST API

`@RestController` = `@Controller` + `@ResponseBody`，表示所有方法都直接返回資料（通常是 JSON），而非視圖名稱。

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    // 建構子注入（推薦，不需要 @Autowired）
    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public List<ProductDto> list() {
        return productService.findAll();
    }

    @GetMapping("/{id}")
    public ProductDto detail(@PathVariable Long id) {
        return productService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductDto create(@Valid @RequestBody ProductCreateRequest request) {
        return productService.create(request);
    }

    @PutMapping("/{id}")
    public ProductDto update(@PathVariable Long id,
                             @Valid @RequestBody ProductUpdateRequest request) {
        return productService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        productService.delete(id);
    }
}
```

## Handler Method 參數註解

Spring MVC 提供了豐富的註解來自動繫結 HTTP 請求的各種資料：

### @PathVariable — 路徑變數

```java
// GET /api/users/42
@GetMapping("/api/users/{id}")
public UserDto getUser(@PathVariable Long id) { ... }

// 名稱不一致時指定
@GetMapping("/api/users/{userId}/orders/{orderId}")
public OrderDto getOrder(@PathVariable("userId") Long uid,
                         @PathVariable("orderId") Long oid) { ... }
```

### @RequestParam — 查詢參數

```java
// GET /api/products?category=tire&minPrice=500
@GetMapping("/api/products")
public List<ProductDto> search(
    @RequestParam String category,
    @RequestParam(required = false, defaultValue = "0") int minPrice,
    @RequestParam(required = false) Integer maxPrice) { ... }
```

### @RequestHeader — 請求標頭

```java
@GetMapping("/api/profile")
public UserDto profile(@RequestHeader("Authorization") String token,
                       @RequestHeader(value = "Accept-Language",
                                      defaultValue = "zh-TW") String lang) { ... }
```

### @CookieValue — Cookie 值

```java
@GetMapping("/api/session")
public String session(@CookieValue(value = "sessionId",
                                    required = false) String sessionId) { ... }
```

### @RequestBody — 請求正文

```java
// POST /api/users
// Content-Type: application/json
// { "name": "張三", "email": "zhang@example.com" }
@PostMapping("/api/users")
public UserDto create(@Valid @RequestBody CreateUserRequest request) { ... }
```

### @ModelAttribute — 表單資料繫結

```java
// POST /api/users (application/x-www-form-urlencoded)
// name=張三&email=zhang@example.com
@PostMapping("/api/users")
public UserDto create(@ModelAttribute CreateUserRequest request) { ... }
```

## ResponseEntity — 精確控制回應

```java
@GetMapping("/{id}")
public ResponseEntity<ProductDto> getProduct(@PathVariable Long id) {
    return productService.findById(id)
        .map(product -> ResponseEntity.ok()
            .header("X-Custom-Header", "value")
            .body(product))
        .orElse(ResponseEntity.notFound().build());
}

@PostMapping
public ResponseEntity<ProductDto> create(@Valid @RequestBody ProductCreateRequest req) {
    ProductDto created = productService.create(req);
    URI location = ServletUriComponentsBuilder
        .fromCurrentRequest()
        .path("/{id}")
        .buildAndExpand(created.getId())
        .toUri();
    return ResponseEntity.created(location).body(created);
}
```

## JSON 序列化控制

Spring Boot 預設使用 Jackson 處理 JSON。可以透過註解控制序列化行為：

```java
public class OrderDto {

    private Long id;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Taipei")
    private LocalDateTime createTime;

    @JsonProperty("order_status")  // 自訂 JSON 欄位名
    private String status;

    @JsonIgnore  // 不輸出到 JSON
    private String internalNote;

    @JsonInclude(JsonInclude.Include.NON_NULL)  // null 值不輸出
    private String remark;
}
```

全域配置：

```yaml
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Taipei
    default-property-inclusion: non-null
    serialization:
      write-dates-as-timestamps: false
```

## @Controller vs @RestController

| 特性 | @Controller | @RestController |
|------|-------------|-----------------|
| 回應 | 返回**視圖名稱**（HTML） | 返回**資料**（JSON/XML） |
| 需搭配 | ViewResolver（Thymeleaf、JSP） | HttpMessageConverter（Jackson） |
| 等同於 | `@Controller` | `@Controller` + `@ResponseBody` |
| 適用場景 | 伺服器端渲染 | 前後端分離 / REST API |

如果同一個 Controller 中既要返回頁面又要返回 JSON，使用 `@Controller` 並在需要的方法上加 `@ResponseBody`：

```java
@Controller
public class PageController {

    @GetMapping("/")
    public String homePage(Model model) {
        model.addAttribute("title", "首頁");
        return "index";  // 返回 Thymeleaf 視圖
    }

    @GetMapping("/api/data")
    @ResponseBody
    public Map<String, Object> getData() {
        return Map.of("key", "value");  // 返回 JSON
    }
}
```

## 小結

現代 Spring MVC 已經從 XML 配置 + JSP 演進到註解驅動 + RESTful API 的開發模式。搭配 Spring Boot，開發者可以用極少的配置快速建立功能完整的 Web 服務。核心要掌握的就是各種請求參數繫結註解和回應控制方式。
