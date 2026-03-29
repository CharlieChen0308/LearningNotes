# 13 Spring 事務管理

> **版本**：Spring Boot 3.x / Spring Framework 6.x / Java 17+
>
> 事務管理對於企業應用至關重要，當出現異常時，它可以保證資料的一致性。Spring 提供程式設計式與宣告式兩種事務管理方式。

## 1、兩種事務管理方式

| 方式 | 說明 | 適用場景 |
|------|------|---------|
| **宣告式事務** | 基於 AOP，在方法前後自動攔截管理事務 | 絕大多數場景（推薦） |
| **程式設計式事務** | 使用 `TransactionTemplate` 或 `PlatformTransactionManager` | 需要程式碼塊級別控制的少數場景 |

宣告式事務是 Spring 倡導的非侵入式開發方式 — 業務程式碼不需要摻雜事務管理邏輯，只要加上 `@Transactional` 註解即可。

## 2、事務特性

### 2.1 隔離級別（Isolation Level）

| 級別 | 說明 | 防止 |
|------|------|------|
| `DEFAULT` | 使用底層資料庫預設（PostgreSQL/MySQL InnoDB 預設為 READ_COMMITTED / REPEATABLE_READ） | — |
| `READ_UNCOMMITTED` | 可讀取未提交資料 | — |
| `READ_COMMITTED` | 只能讀取已提交資料 | 髒讀 |
| `REPEATABLE_READ` | 同一事務中多次查詢結果相同 | 髒讀、不可重複讀 |
| `SERIALIZABLE` | 事務依次執行，完全隔離 | 髒讀、不可重複讀、幻讀 |

### 2.2 傳播行為（Propagation）

| 傳播行為 | 說明 |
|---------|------|
| `REQUIRED`（預設） | 有事務就加入，沒有就建立新的 |
| `REQUIRES_NEW` | 總是建立新事務，掛起當前事務 |
| `SUPPORTS` | 有事務就加入，沒有就以非事務執行 |
| `NOT_SUPPORTED` | 以非事務執行，掛起當前事務 |
| `NEVER` | 以非事務執行，有事務就拋異常 |
| `MANDATORY` | 必須在事務中執行，沒有就拋異常 |
| `NESTED` | 有事務就建立巢狀事務（Savepoint），沒有就等同 REQUIRED |

### 2.3 回滾規則

- 預設：只有 **unchecked 異常**（`RuntimeException` 及其子類、`Error`）才會觸發回滾
- checked 異常（如 `IOException`）預設**不回滾**
- 可透過 `rollbackFor` / `noRollbackFor` 自訂

## 3、@Transactional 宣告式事務（現代方式）

### 3.1 Spring Boot 自動配置

Spring Boot 會自動配置 `DataSourceTransactionManager`（JDBC/MyBatis）或 `JpaTransactionManager`（JPA），無需手動配置。

### 3.2 基本用法

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;

    // 建構子注入（推薦）
    public OrderService(OrderRepository orderRepository, InventoryService inventoryService) {
        this.orderRepository = orderRepository;
        this.inventoryService = inventoryService;
    }

    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. 建立訂單
        Order order = new Order(request.getCustomerId(), request.getItems());
        orderRepository.save(order);

        // 2. 扣減庫存（如果失敗，整個方法回滾）
        inventoryService.deductStock(request.getItems());

        return order;
    }
}
```

### 3.3 常用屬性

```java
@Transactional(
    readOnly = true,                              // 唯讀事務（查詢最佳化）
    timeout = 30,                                  // 超時秒數
    isolation = Isolation.READ_COMMITTED,          // 隔離級別
    propagation = Propagation.REQUIRED,            // 傳播行為
    rollbackFor = Exception.class,                 // checked 異常也回滾
    noRollbackFor = BusinessWarningException.class // 指定異常不回滾
)
public List<Order> findOrders(Long customerId) {
    return orderRepository.findByCustomerId(customerId);
}
```

### 3.4 類級別 vs 方法級別

```java
@Service
@Transactional(readOnly = true)  // 類級別：所有 public 方法預設唯讀
public class ProductService {
    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    // 繼承類級別的 readOnly = true
    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    @Transactional  // 方法級別覆蓋：readOnly = false（預設）
    public Product create(ProductRequest request) {
        return productRepository.save(new Product(request));
    }
}
```

## 4、@Transactional 常見陷阱

### 4.1 只對 public 方法生效

Spring AOP 基於代理，`@Transactional` 只在 **public** 方法上生效。`protected`、`private`、`default` 方法上的註解會被忽略。

### 4.2 同類內部呼叫不生效

```java
@Service
public class UserService {
    @Transactional
    public void register(User user) {
        saveUser(user);
        sendWelcomeEmail(user);  // 如果這裡拋異常，saveUser 不會回滾！
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendWelcomeEmail(User user) {
        // 同類內部呼叫，不經過代理，REQUIRES_NEW 不生效
    }
}
```

**解決方案**：將 `sendWelcomeEmail` 移到另一個 Service 類中，確保呼叫經過 Spring 代理。

### 4.3 異常被吞掉

```java
@Transactional
public void process() {
    try {
        // 業務邏輯
        riskyOperation();
    } catch (Exception e) {
        log.error("Error", e);
        // 異常被 catch 了，Spring 看不到異常，不會回滾！
    }
}
```

**解決方案**：catch 後重新拋出，或手動標記回滾：

```java
@Transactional
public void process() {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("Error", e);
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

## 5、程式設計式事務

少數需要程式碼塊級別事務控制的場景，使用 `TransactionTemplate`：

```java
@Service
public class BatchService {
    private final TransactionTemplate transactionTemplate;
    private final ItemRepository itemRepository;

    public BatchService(PlatformTransactionManager transactionManager,
                        ItemRepository itemRepository) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
        this.itemRepository = itemRepository;
    }

    public void processBatch(List<Item> items) {
        for (Item item : items) {
            // 每個 item 獨立事務，一個失敗不影響其他
            transactionTemplate.executeWithoutResult(status -> {
                try {
                    itemRepository.save(item);
                } catch (Exception e) {
                    status.setRollbackOnly();
                    log.warn("Failed to process item: {}", item.getId(), e);
                }
            });
        }
    }
}
```

## 6、小結

| 要點 | 說明 |
|------|------|
| 推薦方式 | `@Transactional` 宣告式事務 |
| 預設回滾 | 只有 RuntimeException 和 Error |
| 建議 | 查詢方法加 `readOnly = true`，寫入方法加 `rollbackFor = Exception.class` |
| 注意 | 只對 public 方法生效；避免同類內部呼叫；不要吞掉異常 |

> **延伸閱讀**：
> - [01 Spring Core — DI 與 IoC](01%20Spring%20Core%20—%20DI%20與%20IoC.md) — 理解 Spring 容器與代理機制
> - [02 Spring AOP 註解式開發](02%20Spring%20AOP%20註解式開發.md) — 事務管理的底層就是 AOP
> - [10 Spring Data JPA](10%20Spring%20Data%20JPA.md) — JPA 中的事務整合

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
