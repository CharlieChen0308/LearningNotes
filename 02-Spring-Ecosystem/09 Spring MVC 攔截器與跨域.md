# 09 Spring MVC 攔截器與跨域

> **版本**: Spring MVC 6.x / Spring Boot 3.x
> **來源**: `_archive/Spring MVC/04 Spring MVC 攔截器與跨域設定.md`

## 攔截器（HandlerInterceptor）

攔截器類似於 Servlet Filter，但更緊密地整合在 Spring MVC 框架中，可以存取 Handler 和 ModelAndView 等 Spring MVC 特有的物件。

### 攔截器的執行時機

```
客戶端請求
    ↓
DispatcherServlet
    ↓
preHandle()          ← 攔截器（請求前）
    ↓
Controller 方法執行
    ↓
postHandle()         ← 攔截器（請求後、視圖渲染前）
    ↓
視圖渲染
    ↓
afterCompletion()    ← 攔截器（完成後，可用於清理資源）
    ↓
回應給客戶端
```

### 實作攔截器

```java
@Component
public class RequestLoggingInterceptor implements HandlerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingInterceptor.class);

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        request.setAttribute("startTime", System.currentTimeMillis());
        log.info("→ {} {} (來源: {})",
            request.getMethod(),
            request.getRequestURI(),
            request.getRemoteAddr());
        return true;  // true 繼續執行，false 中斷請求
    }

    @Override
    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler,
                           ModelAndView modelAndView) throws Exception {
        // Controller 執行完畢後（可修改 ModelAndView）
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler,
                                Exception ex) throws Exception {
        long startTime = (long) request.getAttribute("startTime");
        long duration = System.currentTimeMillis() - startTime;
        log.info("← {} {} [{}] {}ms",
            request.getMethod(),
            request.getRequestURI(),
            response.getStatus(),
            duration);
    }
}
```

### 註冊攔截器

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    private final RequestLoggingInterceptor loggingInterceptor;
    private final AuthInterceptor authInterceptor;

    public WebMvcConfig(RequestLoggingInterceptor loggingInterceptor,
                        AuthInterceptor authInterceptor) {
        this.loggingInterceptor = loggingInterceptor;
        this.authInterceptor = authInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 日誌攔截器 — 攔截所有請求
        registry.addInterceptor(loggingInterceptor)
            .addPathPatterns("/**");

        // 認證攔截器 — 攔截 API 請求，排除登入和公開端點
        registry.addInterceptor(authInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns(
                "/api/auth/login",
                "/api/auth/register",
                "/api/public/**"
            );
    }
}
```

### 實用範例：Token 認證攔截器

```java
@Component
public class AuthInterceptor implements HandlerInterceptor {

    private final JwtUtils jwtUtils;

    public AuthInterceptor(JwtUtils jwtUtils) {
        this.jwtUtils = jwtUtils;
    }

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        // OPTIONS 預檢請求直接放行
        if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
            return true;
        }

        String token = request.getHeader("Authorization");
        if (token == null || !token.startsWith("Bearer ")) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.setContentType("application/json;charset=UTF-8");
            response.getWriter().write(
                "{\"status\":401,\"message\":\"未提供認證 Token\"}");
            return false;
        }

        try {
            String jwt = token.substring(7);
            Long userId = jwtUtils.validateAndGetUserId(jwt);
            // 將使用者資訊存入 request，供後續 Controller 使用
            request.setAttribute("currentUserId", userId);
            return true;
        } catch (Exception e) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.setContentType("application/json;charset=UTF-8");
            response.getWriter().write(
                "{\"status\":401,\"message\":\"Token 無效或已過期\"}");
            return false;
        }
    }
}
```

### 認證攔截器的替代方案：Spring Security Filter Chain

上述 `AuthInterceptor` 適合簡單場景，但在正式專案中，**認證與授權建議使用 Spring Security Filter Chain**，原因如下：

- **執行順序更早**：Security Filter 在 `DispatcherServlet` 之前執行，能攔截所有請求（包含靜態資源、Actuator 端點等）
- **功能更完整**：內建 CSRF 防護、Session 管理、Remember Me、OAuth2 整合等
- **宣告式授權**：透過 `@PreAuthorize`、`SecurityFilterChain` 等機制，權限規則更清晰
- **社群標準**：與 JWT、OAuth2、LDAP 等認證方案有成熟整合

> 詳見 [14 Spring Security 與 JWT](14%20Spring%20Security%20與%20JWT.md)

## 攔截器 vs Filter 比較

| 特性 | Filter | HandlerInterceptor |
|------|--------|-------------------|
| 規範 | Servlet 規範 | Spring MVC 框架 |
| 作用範圍 | 所有請求（包含靜態資源） | 僅 DispatcherServlet 處理的請求 |
| 可存取 | HttpServletRequest/Response | + Handler、ModelAndView |
| 執行順序 | Filter → Interceptor → Controller | |
| 適用場景 | 編碼、壓縮、安全 | 認證、日誌、權限、效能監控 |

## 跨域設定（CORS）

### 什麼是 CORS

當前端（如 `http://localhost:3000`）向不同來源的後端（如 `http://localhost:8080`）發送請求時，瀏覽器會基於**同源策略**阻擋請求。CORS（Cross-Origin Resource Sharing）是一種標準機制，讓伺服器告訴瀏覽器允許哪些跨域請求。

### 方式一：全域設定（推薦）

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "http://localhost:3000",
                "http://localhost:3001"
            )
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            .allowedHeaders("*")
            .exposedHeaders("Authorization", "X-Total-Count")
            .allowCredentials(true)
            .maxAge(3600);  // 預檢請求快取 1 小時
    }
}
```

### 方式二：Controller 級別

```java
@RestController
@RequestMapping("/api/public")
@CrossOrigin(origins = "*", maxAge = 3600)
public class PublicController {

    @GetMapping("/info")
    public Map<String, String> info() {
        return Map.of("version", "1.0");
    }
}
```

### 方式三：方法級別

```java
@CrossOrigin(origins = "http://localhost:3000")
@GetMapping("/api/data")
public List<DataDto> getData() { ... }
```

### CORS 設定參數說明

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `allowedOrigins` | 允許的來源 | 無（必須設定） |
| `allowedMethods` | 允許的 HTTP 方法 | GET, HEAD, POST |
| `allowedHeaders` | 允許的請求標頭 | 無 |
| `exposedHeaders` | 前端可讀取的回應標頭 | 無 |
| `allowCredentials` | 是否允許攜帶 Cookie | false |
| `maxAge` | 預檢請求快取時間（秒） | 1800 |

> 注意：`allowCredentials=true` 時，`allowedOrigins` 不能設為 `"*"`，必須指定具體來源。

### 生產環境 CORS 注意事項

- **禁止 `allowedOrigins("*")`**：允許所有來源等於關閉同源策略保護，正式環境必須明確列出允許的域名
- **預檢快取（`maxAge`）**：瀏覽器對 CORS 預檢請求（OPTIONS）會依 `maxAge` 快取結果。建議設為 3600（1 小時）以減少不必要的 OPTIONS 請求，提升效能
- **Credential 處理**：當 `allowCredentials=true` 時，前端 `fetch` 必須設定 `credentials: 'include'`，且後端不可使用萬用字元 `"*"` 作為 `allowedOrigins`，否則瀏覽器會拒絕回應
- **搭配 Spring Security**：若專案使用 Spring Security，CORS 設定建議透過 `SecurityFilterChain` 的 `.cors()` 配置，避免 `WebMvcConfigurer` 的設定被 Security Filter 覆蓋

### 透過 CorsFilter 設定

如果需要在 Filter 層面處理 CORS（例如搭配 Spring Security）：

```java
@Bean
public CorsFilter corsFilter() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("http://localhost:3000"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);

    return new CorsFilter(source);
}
```

## 靜態資源處理

```java
@Configuration
public class StaticResourceConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // 上傳檔案目錄
        registry.addResourceHandler("/uploads/**")
            .addResourceLocations("file:C:/uploads/")
            .setCachePeriod(3600);

        // Swagger UI（如有使用）
        // ⚠️ Springfox 已停止維護，建議改用 SpringDoc OpenAPI（參見第 15 篇）
        // SpringDoc 不需要手動配置靜態資源，加入依賴即可自動註冊
        registry.addResourceHandler("/swagger-ui/**")
            .addResourceLocations("classpath:/META-INF/resources/webjars/springfox-swagger-ui/");
    }
}
```

## WebMvcConfigurer 常用配置彙總

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    // 攔截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) { ... }

    // CORS
    @Override
    public void addCorsMappings(CorsRegistry registry) { ... }

    // 靜態資源
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) { ... }

    // 訊息轉換器（自訂 JSON 序列化等）
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) { ... }

    // 格式化器（日期、數字等）
    @Override
    public void addFormatters(FormatterRegistry registry) { ... }

    // 視圖控制器（簡單的 URL → 視圖對映）
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("index");
    }
}
```

## 小結

攔截器和 CORS 是 Spring MVC 應用中常見的橫切關注點。攔截器適合用於日誌、效能監控等場景（認證授權建議改用 Spring Security），而 CORS 設定則是前後端分離架構中的必備配置。透過 `WebMvcConfigurer` 介面，可以集中管理這些配置，保持程式碼整潔。

## 延伸閱讀

- [14 Spring Security 與 JWT](14%20Spring%20Security%20與%20JWT.md)——認證授權的正式方案，涵蓋 Filter Chain、JWT 整合
- [15 API 文件（SpringDoc OpenAPI）](15%20API%20文件（SpringDoc%20OpenAPI）.md)——取代 Springfox 的 API 文件方案

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
