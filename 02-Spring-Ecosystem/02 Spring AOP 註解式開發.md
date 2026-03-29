# 02 Spring AOP 註解式開發

> **版本**: Spring Framework 6.x
> **來源**: `_archive/Spring/07 Spring 註解式 AOP 開發.md`

> 本系列第 03 篇介紹了 Spring AOP 的 XML 配置方式。本篇對照 Spring Framework 6.x 官方文件，補充 `@AspectJ` 註解式 AOP 開發方式。AOP 的核心概念不變，只是配置方式從 XML 轉為註解。

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

## 小結

Spring 6.x 的 `@AspectJ` 註解式 AOP 相比 XML 配置更加直覺和簡潔。核心概念不變——切面、切入點、通知——但程式碼和配置合併在一起，更易於閱讀和維護。自訂註解 + `@Around` 是最靈活的組合，可以實現日誌、效能監控、重試、權限檢查等各種橫切關注點。
