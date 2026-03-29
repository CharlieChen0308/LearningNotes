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
    <version>2.8.0</version>
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

## 8、Annotation-First vs Code-First 文件策略

API 文件的產生方式主要分為兩種流派，各有適用場景：

### 8.1 策略比較

| 面向 | Annotation-First（註解優先） | Code-First（程式碼優先） |
|------|---------------------------|------------------------|
| **做法** | 在 Controller/DTO 上加 `@Operation`、`@Schema` 等註解 | 手寫 OpenAPI YAML/JSON，再用工具產生程式碼或僅作為契約 |
| **文件與程式碼一致性** | 高：文件直接從程式碼產生，不易脫節 | 中：需要額外流程確保契約與實作同步 |
| **可讀性** | 註解過多時會降低程式碼可讀性 | 程式碼乾淨，文件集中管理 |
| **適用規模** | 中小型專案、團隊內部 API | 大型專案、跨團隊協作、API-First 設計 |
| **學習成本** | 低：加註解即可 | 中：需熟悉 OpenAPI Spec 語法 |
| **重構友善度** | 高：IDE 重構時註解跟著走 | 低：重構後需手動更新契約檔 |

### 8.2 實務建議

- **內部系統**（如 ERP、後台管理）：推薦 Annotation-First，開發速度快、維護成本低
- **對外公開 API**（如開放平台、第三方串接）：推薦 Code-First（API-First），先定義契約再實作，確保介面穩定
- **混合策略**：核心對外 API 用 Code-First 管理契約，內部輔助 API 用 Annotation-First 快速產出

> **務實選擇**：如果團隊規模在 10 人以下且 API 主要供內部前端使用，Annotation-First 配合 SpringDoc 自動產生文件是最高效的方案。

## 9、Springfox 遷移至 SpringDoc

許多 Spring Boot 2.x 的既有專案仍在使用 Springfox（Swagger 2），但 Springfox 自 2020 年後已停止維護，不支援 Spring Boot 3.x。升級時必須遷移至 SpringDoc。

### 9.1 依賴替換

```xml
<!-- 移除 Springfox -->
<!--
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>3.0.0</version>
</dependency>
-->

<!-- 改用 SpringDoc -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.0</version>
</dependency>
```

### 9.2 註解對照表

| Springfox（舊） | SpringDoc（新） | 說明 |
|-----------------|----------------|------|
| `@Api(tags = "...")` | `@Tag(name = "...")` | Controller 分組 |
| `@ApiOperation(value = "...")` | `@Operation(summary = "...")` | 操作描述 |
| `@ApiParam` | `@Parameter` | 參數描述 |
| `@ApiModel` | `@Schema` | DTO 模型描述 |
| `@ApiModelProperty` | `@Schema` | DTO 欄位描述 |
| `@ApiResponse(code = 200)` | `@ApiResponse(responseCode = "200")` | 回應碼（注意型別從 int 改為 String） |
| `@ApiIgnore` | `@Hidden` | 隱藏端點 |

### 9.3 配置類別遷移

```java
// Springfox 舊寫法（移除）
// @EnableSwagger2
// public class SwaggerConfig {
//     @Bean
//     public Docket api() { ... }
// }

// SpringDoc 新寫法
@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info().title("API 文件").version("1.0.0"));
    }
}
```

### 9.4 URL 變更

| 項目 | Springfox | SpringDoc |
|------|-----------|-----------|
| Swagger UI | `/swagger-ui/` 或 `/swagger-ui.html` | `/swagger-ui.html`（可自訂） |
| API Docs | `/v2/api-docs` | `/v3/api-docs` |

> **遷移步驟摘要**：移除 Springfox 依賴 → 加入 SpringDoc 依賴 → 全域搜尋替換註解 → 更新 Security 放行路徑 → 移除 Docket Bean、改用 OpenAPI Bean → 驗證 Swagger UI 可正常存取。

## 10、生產環境進階配置

### 10.1 自訂錯誤回應文件

預設的 SpringDoc 不會自動記錄全域錯誤格式。透過 `OpenApiCustomizer` 可以統一加入錯誤回應 Schema：

```java
@Bean
public OpenApiCustomizer globalErrorResponseCustomizer() {
    return openApi -> {
        Schema<?> errorSchema = new Schema<>()
            .type("object")
            .addProperty("code", new Schema<>().type("integer").example(400))
            .addProperty("message", new Schema<>().type("string").example("請求參數錯誤"))
            .addProperty("timestamp", new Schema<>().type("string").format("date-time"));

        openApi.getComponents().addSchemas("ErrorResponse", errorSchema);

        // 為所有端點加入 400/500 回應
        openApi.getPaths().values().forEach(pathItem ->
            pathItem.readOperations().forEach(operation -> {
                operation.getResponses().addApiResponse("400",
                    new ApiResponse().description("請求參數錯誤")
                        .content(new Content().addMediaType("application/json",
                            new MediaType().schema(
                                new Schema<>().$ref("#/components/schemas/ErrorResponse")))));
                operation.getResponses().addApiResponse("500",
                    new ApiResponse().description("伺服器內部錯誤"));
            })
        );
    };
}
```

### 10.2 版本化 API 文件

當 API 有多個版本並行時，利用 `GroupedOpenApi` 依版本分組：

```java
@Bean
public GroupedOpenApi v1Api() {
    return GroupedOpenApi.builder()
        .group("v1")
        .displayName("API v1（穩定版）")
        .pathsToMatch("/api/v1/**")
        .build();
}

@Bean
public GroupedOpenApi v2Api() {
    return GroupedOpenApi.builder()
        .group("v2")
        .displayName("API v2（開發中）")
        .pathsToMatch("/api/v2/**")
        .build();
}
```

Swagger UI 上方會出現下拉選單，可切換不同版本的文件。

### 10.3 環境差異化配置

不同環境對文件的需求不同：開發環境需要完整文件方便除錯，生產環境可能要完全關閉或僅暴露部分端點。

```yaml
# application-dev.yml — 開發環境：全開
springdoc:
  api-docs:
    enabled: true
  swagger-ui:
    enabled: true

# application-prod.yml — 生產環境：完全關閉
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
```

若生產環境僅需開放部分端點（例如對外公開 API 需要文件）：

```java
@Bean
@Profile("prod")
public GroupedOpenApi prodPublicApi() {
    return GroupedOpenApi.builder()
        .group("public")
        .pathsToMatch("/api/public/**")
        .addOpenApiCustomizer(openApi -> openApi
            .info(new Info()
                .title("Public API")
                .version("1.0.0")
                .description("僅包含對外公開端點")))
        .build();
}
```

結合 `@Hidden` 註解，可在不改環境配置的情況下隱藏特定內部端點：

```java
@Hidden  // 在所有文件中隱藏此 Controller
@RestController
@RequestMapping("/api/internal")
public class InternalController {
    // 內部監控、健康檢查等端點
}
```

## 11、API 文件測試驗證

API 文件若與實際行為不一致，會比沒有文件更危險。透過測試確保文件的正確性。

### 11.1 依賴

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-api</artifactId>
    <version>2.8.0</version>
    <scope>test</scope>
</dependency>
```

> 注意：這裡用的是 `webmvc-api`（不含 UI），僅用於測試時產生 OpenAPI Spec。

### 11.2 產生 OpenAPI Spec 並驗證

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class OpenApiDocumentationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldGenerateValidOpenApiSpec() throws Exception {
        // 取得 OpenAPI JSON
        String openApiJson = mockMvc.perform(get("/v3/api-docs"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.openapi").value("3.0.1"))
            .andExpect(jsonPath("$.info.title").exists())
            .andReturn()
            .getResponse()
            .getContentAsString();

        // 驗證必要端點存在
        assertThat(openApiJson).contains("/api/users");
        assertThat(openApiJson).contains("/api/auth/login");
    }

    @Test
    void shouldDocumentAllErrorResponses() throws Exception {
        mockMvc.perform(get("/v3/api-docs"))
            .andExpect(status().isOk())
            // 確認錯誤回應 Schema 已定義
            .andExpect(jsonPath("$.components.schemas.ErrorResponse").exists());
    }

    @Test
    void shouldNotExposeInternalEndpoints() throws Exception {
        String openApiJson = mockMvc.perform(get("/v3/api-docs"))
            .andExpect(status().isOk())
            .andReturn()
            .getResponse()
            .getContentAsString();

        // 確認內部端點未出現在文件中
        assertThat(openApiJson).doesNotContain("/api/internal");
    }
}
```

### 11.3 CI 整合建議

在 CI Pipeline 中加入 OpenAPI Spec 驗證步驟，可以在每次提交時自動確認文件完整性：

1. **產生 Spec**：透過 Maven Plugin 在 build 階段自動產生 `openapi.json`
2. **格式驗證**：使用 [swagger-cli](https://github.com/APIDevTools/swagger-cli) 驗證 Spec 語法
3. **破壞性變更偵測**：使用 [openapi-diff](https://github.com/OpenAPITools/openapi-diff) 比較新舊版本，自動偵測不相容變更

```xml
<!-- Maven Plugin：build 時產生 openapi.json -->
<plugin>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-maven-plugin</artifactId>
    <version>1.4</version>
    <executions>
        <execution>
            <id>integration-test</id>
            <goals>
                <goal>generate</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## 12、小結

| 概念 | 重點 |
|------|------|
| SpringDoc | Spring Boot 3.x 的 API 文件標準（取代 Springfox） |
| Swagger UI | `/swagger-ui.html` 互動式測試介面 |
| @Operation + @Schema | 最常用的兩個註解 |
| JWT 整合 | 全局 SecurityScheme + permitAll Swagger 路徑 |
| 文件策略 | 內部系統用 Annotation-First，對外 API 考慮 Code-First |
| Springfox 遷移 | 替換依賴 → 替換註解 → 更新配置類 → 更新 Security 路徑 |
| 生產環境 | 依環境差異化配置，隱藏內部端點，自訂錯誤回應 |
| 版本化文件 | `GroupedOpenApi` 依版本分組，Swagger UI 下拉切換 |
| 文件測試 | 用 `webmvc-api` + MockMvc 驗證 Spec 正確性，CI 自動偵測破壞性變更 |

> **延伸閱讀**：
> - [06 Spring Boot RESTful API 開發](06%20Spring%20Boot%20RESTful%20API%20開發.md) — REST API 基礎
> - [14 Spring Security 與 JWT](14%20Spring%20Security%20與%20JWT.md) — 認證整合
> - [08 Spring MVC 例外處理與驗證](08%20Spring%20MVC%20例外處理與驗證.md) — 驗證註解
