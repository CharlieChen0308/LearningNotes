# 15 API 文件（SpringDoc OpenAPI）

> **版本**：SpringDoc OpenAPI 2.x / Spring Boot 3.x / Java 17+
>
> SpringDoc 是 Spring Boot 3.x 時代的 API 文件工具，取代了舊版 Springfox（已停止維護）。

## 1、快速開始

### 1.1 依賴

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

加入依賴後，啟動應用即可存取：
- Swagger UI：`http://localhost:8080/swagger-ui.html`
- OpenAPI JSON：`http://localhost:8080/v3/api-docs`

### 1.2 基本配置

```yaml
# application.yml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    tags-sorter: alpha
    operations-sorter: method
```

## 2、常用註解

### 2.1 Controller 層

```java
@RestController
@RequestMapping("/api/users")
@Tag(name = "使用者管理", description = "使用者 CRUD 操作")
public class UserController {

    @Operation(
        summary = "查詢使用者",
        description = "根據 ID 查詢使用者詳細資訊"
    )
    @ApiResponse(responseCode = "200", description = "查詢成功")
    @ApiResponse(responseCode = "404", description = "使用者不存在")
    @GetMapping("/{id}")
    public UserResponse getUser(
            @Parameter(description = "使用者 ID", example = "1")
            @PathVariable Long id) {
        return userService.findById(id);
    }

    @Operation(summary = "建立使用者")
    @ApiResponse(responseCode = "201", description = "建立成功")
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@RequestBody @Valid CreateUserRequest request) {
        return userService.create(request);
    }
}
```

### 2.2 DTO 層

```java
@Schema(description = "建立使用者請求")
public record CreateUserRequest(
    @Schema(description = "使用者名稱", example = "Alice", requiredMode = Schema.RequiredMode.REQUIRED)
    @NotBlank
    String name,

    @Schema(description = "電子郵件", example = "alice@example.com")
    @Email
    String email,

    @Schema(description = "年齡", minimum = "0", maximum = "150", example = "25")
    @Min(0) @Max(150)
    Integer age
) {}

@Schema(description = "使用者回應")
public record UserResponse(
    @Schema(description = "使用者 ID", example = "1")
    Long id,
    String name,
    String email,
    Integer age,
    LocalDateTime createdAt
) {}
```

## 3、全局配置

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("使用者管理 API")
                .version("1.0.0")
                .description("Spring Boot 3.x RESTful API 文件")
                .contact(new Contact().name("Dev Team").email("dev@example.com"))
            )
            .addSecurityItem(new SecurityRequirement().addList("Bearer"))
            .components(new Components()
                .addSecuritySchemes("Bearer",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")
                        .description("JWT Token 認證")
                )
            );
    }
}
```

## 4、與 Spring Security 整合

允許未認證存取 Swagger UI：

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(
                "/swagger-ui/**",
                "/v3/api-docs/**",
                "/swagger-ui.html"
            ).permitAll()
            .anyRequest().authenticated()
        )
        .build();
}
```

## 5、API 分組

```java
@Bean
public GroupedOpenApi publicApi() {
    return GroupedOpenApi.builder()
        .group("public")
        .pathsToMatch("/api/public/**")
        .build();
}

@Bean
public GroupedOpenApi adminApi() {
    return GroupedOpenApi.builder()
        .group("admin")
        .pathsToMatch("/api/admin/**")
        .build();
}
```

## 6、生產環境停用

```yaml
# application-prod.yml
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
```

## 7、常用註解速查

| 註解 | 位置 | 用途 |
|------|------|------|
| `@Tag` | Controller 類別 | API 分組名稱 |
| `@Operation` | 方法 | 操作摘要和描述 |
| `@ApiResponse` | 方法 | 回應碼和說明 |
| `@Parameter` | 參數 | 參數描述和範例 |
| `@Schema` | DTO 欄位/類別 | 資料模型描述 |
| `@Hidden` | 類別/方法 | 隱藏不顯示 |

## 8、小結

| 概念 | 重點 |
|------|------|
| SpringDoc | Spring Boot 3.x 的 API 文件標準（取代 Springfox） |
| Swagger UI | `/swagger-ui.html` 互動式測試介面 |
| @Operation + @Schema | 最常用的兩個註解 |
| JWT 整合 | 全局 SecurityScheme + permitAll Swagger 路徑 |
| 生產環境 | 務必停用 |

> **延伸閱讀**：
> - [06 Spring Boot RESTful API 開發](06%20Spring%20Boot%20RESTful%20API%20���發.md) — REST API 基礎
> - [14 Spring Security 與 JWT](14%20Spring%20Security%20與%20JWT.md) — 認證整合
> - [08 Spring MVC 例外處理與驗證](08%20Spring%20MVC%20例外處理與驗證.md) — 驗證註解
