# 02 Spring AOP 註解式開發

> **版本**: Spring Framework 6.x
> **來源**: `_archive/Spring/07 Spring 註解式 AOP 開發.md`

> 本篇聚焦於註解式 AOP 開發，這是 Spring Boot 環境下的主流方式。內容對照 Spring Framework 6.x 官方文件，介紹 `@AspectJ` 註解式 AOP 開發方式。AOP 的核心概念不變，只是配置方式從 XML 轉為註解。

## XML vs 註解方式對照

**XML 方式（第 02 篇）：**

```xml
<aop:config>
    <aop:aspect ref="timeHandler">
        <aop:pointcut id="addTime" expression="execution(* com.example..*(..))" />
        <aop:before method="printTime" pointcut-ref="addTime" />
    </aop:aspect>
</aop:config>
```

**註解方式（本篇）：**

```java
@Aspect
@Component
public class TimeAspect {

    @Before("execution(* com.example..*(..))")
    public void printTime() {
        System.out.println("CurrentTime = " + System.currentTimeMillis());
    }
}
```

## 啟用 @AspectJ 支援

### Spring Boot 環境（自動啟用）

只需引入 AOP Starter：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 純 Spring 環境

```java
@Configuration
@EnableAspectJAutoProxy
public class AopConfig {
}
```

## 定義切面（Aspect）

切面類需要同時標記 `@Aspect` 和 `@Component`：

```java
@Aspect
@Component
public class LoggingAspect {

    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);

    // 定義可重用的切入點
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    // 前置通知
    @Before("serviceLayer()")
    public void logBefore(JoinPoint joinPoint) {
        log.info("→ {}.{}()",
            joinPoint.getTarget().getClass().getSimpleName(),
            joinPoint.getSignature().getName());
    }

    // 後置通知（方法正常返回後）
    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        log.info("← {}.{}() = {}",
            joinPoint.getTarget().getClass().getSimpleName(),
            joinPoint.getSignature().getName(),
            result);
    }

    // 例外通知
    @AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
    public void logException(JoinPoint joinPoint, Exception ex) {
        log.error("✗ {}.{}() 丟出例外: {}",
            joinPoint.getTarget().getClass().getSimpleName(),
            joinPoint.getSignature().getName(),
            ex.getMessage());
    }

    // 最終通知（無論是否例外都會執行）
    @After("serviceLayer()")
    public void logAfter(JoinPoint joinPoint) {
        // 清理資源等
    }
}
```

## 五種通知類型

| 通知類型 | 註解 | 執行時機 |
|---------|------|---------|
| 前置通知 | `@Before` | 方法執行前 |
| 後置返回通知 | `@AfterReturning` | 方法正常返回後 |
| 例外通知 | `@AfterThrowing` | 方法丟出例外後 |
| 最終通知 | `@After` | 方法執行後（無論是否例外） |
| 環繞通知 | `@Around` | 包裹整個方法執行過程 |

## 環繞通知（@Around）

最強大的通知類型，可以完全控制方法的執行：

```java
@Aspect
@Component
public class PerformanceAspect {

    private static final Logger log = LoggerFactory.getLogger(PerformanceAspect.class);

    @Around("execution(* com.example.service.*.*(..))")
    public Object measureTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();

        try {
            Object result = joinPoint.proceed();  // 執行目標方法
            return result;
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            log.info("{}.{}() 耗時 {}ms",
                joinPoint.getTarget().getClass().getSimpleName(),
                joinPoint.getSignature().getName(),
                elapsed);
        }
    }
}
```

## 切入點表達式

### execution — 方法匹配（最常用）

```java
// 匹配 service 套件下所有類的所有方法
@Pointcut("execution(* com.example.service.*.*(..))")

// 匹配 service 套件及其子套件
@Pointcut("execution(* com.example.service..*.*(..))")

// 匹配所有 public 方法
@Pointcut("execution(public * *(..))")

// 匹配返回值為 String 的方法
@Pointcut("execution(String com.example.service.*.*(..))")
```

### @annotation — 自訂註解匹配

```java
// 定義自訂註解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {}

// 切入點：匹配所有標記了 @Loggable 的方法
@Around("@annotation(com.example.annotation.Loggable)")
public Object logMethod(ProceedingJoinPoint joinPoint) throws Throwable {
    // ...
}
```

使用時只需在方法上加註解：

```java
@Service
public class UserService {

    @Loggable  // 自動套用日誌切面
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

### within — 類型匹配

```java
// 匹配 service 套件下的所有類
@Pointcut("within(com.example.service.*)")

// 匹配實現了特定介面的類
@Pointcut("within(com.example.service.UserService+)")
```

### 組合切入點

```java
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceLayer() {}

@Pointcut("execution(* com.example.repository.*.*(..))")
public void repositoryLayer() {}

// 組合：service 或 repository 層
@Before("serviceLayer() || repositoryLayer()")
public void logAll(JoinPoint joinPoint) { ... }

// 組合：service 層但排除特定方法
@Before("serviceLayer() && !execution(* *.toString(..))")
public void logWithExclusion(JoinPoint joinPoint) { ... }
```

## 實用範例：方法重試切面

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Retryable {
    int maxAttempts() default 3;
    long delay() default 1000;
}
```

```java
@Aspect
@Component
public class RetryAspect {

    private static final Logger log = LoggerFactory.getLogger(RetryAspect.class);

    @Around("@annotation(retryable)")
    public Object retry(ProceedingJoinPoint joinPoint, Retryable retryable) throws Throwable {
        int attempts = 0;
        Exception lastException = null;

        while (attempts < retryable.maxAttempts()) {
            try {
                return joinPoint.proceed();
            } catch (Exception e) {
                lastException = e;
                attempts++;
                log.warn("第 {} 次呼叫失敗，準備重試...", attempts);
                if (attempts < retryable.maxAttempts()) {
                    Thread.sleep(retryable.delay());
                }
            }
        }
        throw lastException;
    }
}
```

使用：

```java
@Service
public class ExternalApiService {

    @Retryable(maxAttempts = 3, delay = 2000)
    public String callExternalApi() {
        // 呼叫外部 API，失敗時自動重試
        return restTemplate.getForObject("https://api.example.com/data", String.class);
    }
}
```

## 切面的執行順序

使用 `@Order` 控制多個切面的執行順序：

```java
@Aspect
@Component
@Order(1)  // 數字越小，越先執行
public class SecurityAspect { ... }

@Aspect
@Component
@Order(2)
public class LoggingAspect { ... }
```

## AOP vs HandlerInterceptor vs Servlet Filter

Spring 生態中有三種「攔截」機制，適用場景不同：

| 比較項目 | AOP（`@Aspect`） | HandlerInterceptor | Servlet Filter |
|---------|------------------|---------------------|----------------|
| **作用層級** | 任意 Spring Bean 的方法 | Spring MVC Controller 層 | Servlet 容器層（HTTP 請求/回應） |
| **可取得的資訊** | 方法簽名、參數、回傳值 | `HttpServletRequest`、`Handler` | `ServletRequest`、`ServletResponse` |
| **典型用途** | 交易管理、快取、重試、日誌、權限邏輯 | 登入驗證、請求日誌、Controller 前後處理 | 編碼設定、CORS、壓縮、安全標頭 |
| **是否依賴 Spring MVC** | 否（純 Spring 即可） | 是 | 否（Servlet 規範） |
| **執行順序** | 由 Proxy 機制觸發 | DispatcherServlet 前後 | 在 DispatcherServlet 之前 |

**選擇原則**：

- **業務橫切邏輯**（交易、快取、重試、方法級權限）→ AOP
- **請求級前後處理**（登入檢查、請求計時、Controller 攔截）→ HandlerInterceptor
- **底層 HTTP 處理**（編碼、壓縮、安全標頭、CORS）→ Servlet Filter

> 三者可以同時使用，執行順序為：Filter → Interceptor `preHandle` → AOP → 目標方法 → AOP → Interceptor `postHandle` → Filter。

## 生產注意事項

### CGLIB vs JDK 動態代理

Spring AOP 預設使用兩種代理機制：

| 代理方式 | 條件 | 限制 |
|---------|------|------|
| JDK 動態代理 | 目標類實作了介面 | 只能代理介面方法 |
| CGLIB 代理 | 目標類未實作介面（Spring Boot 預設） | **無法代理 `final` 類和 `final` 方法** |

Spring Boot 2.x 起預設使用 CGLIB（`spring.aop.proxy-target-class=true`）。若類或方法宣告為 `final`，AOP 切面將**靜默失效**，不會拋出錯誤。

### Self-Invocation 陷阱

同一類別內的方法互相呼叫時，**AOP 切面不會生效**，因為呼叫繞過了代理物件：

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(Order order) {
        // 這裡直接呼叫 this.validate()，不經過 Proxy
        // @Transactional 或任何 AOP 切面都不會觸發
        validate(order);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validate(Order order) {
        // 預期開啟新交易，但實際上不會
    }
}
```

**解決方案**：

1. **拆分到不同類別**（推薦）：將 `validate()` 移到獨立的 `OrderValidator` Service
2. **注入自身代理**：透過 `ApplicationContext.getBean()` 或 `@Lazy` 自我注入
3. **使用 `AopContext`**：`((OrderService) AopContext.currentProxy()).validate(order)`（需啟用 `exposeProxy`）

### `@EnableAspectJAutoProxy` 注意事項

| 屬性 | 預設值 | 說明 |
|------|--------|------|
| `proxyTargetClass` | `false`（Spring Boot 預設覆寫為 `true`） | `true` 強制使用 CGLIB |
| `exposeProxy` | `false` | `true` 允許透過 `AopContext.currentProxy()` 取得代理物件 |

### 效能考量

- AOP 代理會增加方法呼叫的額外開銷，**避免在極高頻迴圈內的方法**上使用 AOP
- 切入點表達式越精確，匹配速度越快——避免過於寬泛的 `execution(* *(..))`
- `@Around` 通知比 `@Before` / `@After` 開銷略大，僅在需要控制執行流程時使用

## 小結

Spring 6.x 的 `@AspectJ` 註解式 AOP 相比 XML 配置更加直覺和簡潔。核心概念不變——切面、切入點、通知——但程式碼和配置合併在一起，更易於閱讀和維護。自訂註解 + `@Around` 是最靈活的組合，可以實現日誌、效能監控、重試、權限檢查等各種橫切關注點。

生產環境需特別注意 CGLIB 對 `final` 的限制、同類方法呼叫的 Self-Invocation 陷阱，以及切入點表達式的效能影響。

## 延伸閱讀

- [01 Spring Core — DI 與 IoC](01%20Spring%20Core%20—%20DI%20與%20IoC.md)：AOP 依賴 IoC 容器管理代理物件，理解 Bean 生命週期有助於掌握切面的運作時機
- [09 Spring MVC 攔截器與跨域](09%20Spring%20MVC%20攔截器與跨域.md)：HandlerInterceptor 與 AOP 的差異比較，請求級攔截的實作方式
- [13 Spring 事務管理](13%20Spring%20事務管理.md)：`@Transactional` 是 AOP 最典型的應用，深入了解交易傳播行為與 Self-Invocation 問題

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
