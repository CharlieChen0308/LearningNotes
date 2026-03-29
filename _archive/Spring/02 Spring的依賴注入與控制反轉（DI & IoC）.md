# 02 Spring的依賴注入與控制反轉（DI & IoC）

> 依賴注入（Dependency Injection）與控制反轉（Inversion of Control）是 Spring 框架的核心設計思想。理解這兩個概念，才能真正掌握 Spring 為什麼要這樣設計。

## 1、什麼是控制反轉（IoC）

### 傳統方式的問題

在傳統的 Java 開發中，物件自己負責建立和管理依賴物件：

```java
public class OrderService {
    // OrderService 自己建立 OrderDao
    private OrderDao orderDao = new OrderDaoImpl();

    public void createOrder(Order order) {
        orderDao.save(order);
    }
}
```

這種方式有幾個問題：

1. **緊耦合**：`OrderService` 直接依賴 `OrderDaoImpl` 這個具體實作，如果要換成 `OrderDaoMongoImpl`，就必須修改 `OrderService` 的原始碼。
2. **難以測試**：無法輕易將 `OrderDao` 替換為 Mock 物件來做單元測試。
3. **難以管理**：當物件之間的依賴關係複雜時，手動管理建立順序和生命週期非常困難。

### IoC 的核心思想

**控制反轉**就是把「物件建立和依賴管理」的控制權，從物件自身**反轉**交給外部容器（Spring IoC Container）來負責。

- **傳統方式**：物件主動建立依賴 → 控制權在物件自己手上
- **IoC 方式**：容器負責建立依賴並注入 → 控制權反轉到容器

```java
public class OrderService {
    // 不再自己 new，由外部注入
    private OrderDao orderDao;

    public OrderService(OrderDao orderDao) {
        this.orderDao = orderDao;
    }

    public void createOrder(Order order) {
        orderDao.save(order);
    }
}
```

## 2、什麼是依賴注入（DI）

**依賴注入**是實現 IoC 的具體手段。簡單來說：

> **IoC 是設計思想，DI 是實現方式。**

Spring 提供三種注入方式：

### 2.1 建構子注入（推薦）

透過建構子的參數來注入依賴，Spring 6.x 官方推薦的方式：

```java
@Service
public class OrderService {
    private final OrderDao orderDao;
    private final NotificationService notificationService;

    // 單一建構子時，@Autowired 可省略
    public OrderService(OrderDao orderDao, NotificationService notificationService) {
        this.orderDao = orderDao;
        this.notificationService = notificationService;
    }
}
```

**優點**：
- 依賴可宣告為 `final`，保證不可變性
- 物件建立後就完全初始化，不會出現半初始化狀態
- 容易寫單元測試（直接 new 並傳入 Mock）

### 2.2 Setter 注入

透過 setter 方法來注入，適合可選依賴：

```java
@Service
public class ReportService {
    private ReportFormatter formatter;

    @Autowired(required = false)
    public void setFormatter(ReportFormatter formatter) {
        this.formatter = formatter;
    }
}
```

### 2.3 欄位注入（不推薦）

直接在欄位上加 `@Autowired`，雖然簡潔但有缺點：

```java
@Service
public class OrderService {
    @Autowired  // 不推薦：無法宣告 final，難以做單元測試
    private OrderDao orderDao;
}
```

## 3、Spring IoC 容器

Spring IoC 容器負責管理所有 Bean 的生命週期。核心介面有兩個：

| 介面 | 說明 |
|------|------|
| `BeanFactory` | 最基本的容器，提供懶載入 |
| `ApplicationContext` | 擴充 BeanFactory，提供事件機制、國際化、自動掃描等 |

實際開發中幾乎都使用 `ApplicationContext`。

### 3.1 XML 配置方式（傳統）

```xml
<beans>
    <bean id="orderDao" class="com.example.OrderDaoImpl"/>

    <bean id="orderService" class="com.example.OrderService">
        <!-- 建構子注入 -->
        <constructor-arg ref="orderDao"/>
    </bean>
</beans>
```

```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
OrderService service = ctx.getBean(OrderService.class);
```

> 更多 XML 配置範例請參考 [01 Spring的簡單配置和使用](01%20Spring的簡單配置和使用.md)。

### 3.2 Java 配置方式（現代）

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}
```

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
OrderService service = ctx.getBean(OrderService.class);
```

> 完整的 Java 配置方式請參考 [06 Spring Java 配置與註解驅動（現代方式）](06%20Spring%20Java%20配置與註解驅動（現代方式）.md)。

## 4、Bean 的作用域（Scope）

Spring 管理的 Bean 預設是單例的，可以透過 `@Scope` 改變：

| Scope | 說明 |
|-------|------|
| `singleton` | 預設值。整個容器中只有一個實例 |
| `prototype` | 每次取用都建立新實例 |
| `request` | 每個 HTTP 請求一個實例（Web 環境） |
| `session` | 每個 HTTP Session 一個實例（Web 環境） |

```java
@Service
@Scope("prototype")
public class ShoppingCart {
    // 每次注入都會是新的實例
}
```

## 5、實際範例：理解 IoC 的好處

假設一個通知服務需要發送訊息，傳統做法與 IoC 做法的對比：

### 傳統做法（緊耦合）

```java
public class NotificationService {
    // 寫死了 Email，改用 SMS 就要改程式碼
    private EmailSender sender = new EmailSender();

    public void notify(String message) {
        sender.send(message);
    }
}
```

### IoC 做法（鬆耦合）

```java
// 定義介面
public interface MessageSender {
    void send(String message);
}

// Email 實作
@Component
public class EmailSender implements MessageSender {
    public void send(String message) { /* 發送 Email */ }
}

// SMS 實作
@Component
public class SmsSender implements MessageSender {
    public void send(String message) { /* 發送簡訊 */ }
}

// 服務類：依賴介面，不依賴具體實作
@Service
public class NotificationService {
    private final MessageSender sender;

    public NotificationService(MessageSender sender) {
        this.sender = sender;
    }

    public void notify(String message) {
        sender.send(message);
    }
}
```

透過 IoC，切換 Email 到 SMS 只需要調整配置（或使用 `@Primary`、`@Qualifier`），完全不用改 `NotificationService` 的程式碼。

## 6、小結

| 概念 | 一句話解釋 |
|------|-----------|
| IoC（控制反轉） | 物件不再自己管理依賴，交給容器來管 |
| DI（依賴注入） | 容器把依賴物件「注入」到需要它的物件中 |
| IoC 容器 | Spring 用來管理所有 Bean 生命週期的核心元件 |
| 建構子注入 | 官方推薦的 DI 方式，依賴不可變、容易測試 |

> **延伸閱讀**：
> - [05 Spring的@Autowired註解、@Resource註解、@Service註解](05%20Spring的@Autowired註解、@Resource註解、@Service註解.md) — DI 常用註解的詳細用法
> - [06 Spring Java 配置與註解驅動（現代方式）](06%20Spring%20Java%20配置與註解驅動（現代方式）.md) — 取代 XML 的現代配置方式
