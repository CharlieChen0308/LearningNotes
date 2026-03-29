# SOLID 原則與 Clean Code

> **版本**：Java 17+ / Spring Boot 3.x — 涵蓋 SOLID 五大原則、Clean Architecture、Clean Code 實踐

---

## 1、SOLID 原則

SOLID 是 Robert C. Martin（Uncle Bob）在 2000 年代初期彙整的五項物件導向設計原則，目標是讓軟體更容易理解、更容易修改、更容易擴展。這五個字母分別代表一個原則，以下逐一說明。

---

### 1.1 SRP — 單一職責原則（Single Responsibility Principle）

#### 定義

> *"A class should have only one reason to change."*
> — Robert C. Martin,《Agile Software Development》

一個類別應該只有一個被修改的理由，也就是只負責一件事。這裡的「一件事」指的是「一個變更的軸心」——當需求變動時，只有一種原因會導致這個類別需要修改。

#### 為什麼重要

如果一個類別同時負責多件事，任何一件事的變更都可能波及其他功能，導致迴歸錯誤（regression bug）。職責越多，耦合越高，測試越困難。

#### 不遵守會怎樣

修改寄信邏輯時，不小心改壞了訂單計算；修改日誌格式時，影響了核心業務流程。

#### 錯誤示範

```java
@Service
public class OrderService {

    @Autowired
    private JavaMailSender mailSender;

    public Order createOrder(OrderRequest request) {
        // 業務邏輯：建立訂單
        Order order = new Order(request.getItems(), request.getCustomerId());
        orderRepository.save(order);

        // 寄信通知（不該在這裡）
        SimpleMailMessage msg = new SimpleMailMessage();
        msg.setTo(request.getEmail());
        msg.setSubject("訂單建立成功");
        msg.setText("您的訂單 " + order.getId() + " 已建立");
        mailSender.send(msg);

        // 寫入稽核日誌（也不該在這裡）
        log.info("[AUDIT] 使用者 {} 建立訂單 {}", request.getCustomerId(), order.getId());

        return order;
    }
}
```

這個 `OrderService` 同時處理了三件事：訂單業務邏輯、郵件通知、稽核日誌。任何一項的需求變更都會動到這個類別。

#### 正確示範

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final NotificationService notificationService;
    private final AuditLogger auditLogger;

    public Order createOrder(OrderRequest request) {
        Order order = new Order(request.getItems(), request.getCustomerId());
        orderRepository.save(order);

        notificationService.sendOrderConfirmation(order, request.getEmail());
        auditLogger.logOrderCreated(request.getCustomerId(), order.getId());

        return order;
    }
}

@Service
public class NotificationService {
    // 專責處理各種通知（Email、SMS、Push）
    public void sendOrderConfirmation(Order order, String email) { /* ... */ }
}

@Component
public class AuditLogger {
    // 專責處理稽核日誌
    public void logOrderCreated(Long customerId, Long orderId) { /* ... */ }
}
```

#### Spring 實踐要點

在 Spring Boot 中，`@Service` 的職責劃分是 SRP 最直接的體現。一個常見的判斷方式：如果你的 Service 注入了超過 5～7 個依賴，很可能它承擔了過多職責，應該考慮拆分。

#### Trade-off

過度拆分會導致類別爆炸（class explosion），增加理解成本。關鍵在於找到「變更的軸心」——如果兩段邏輯總是因為同一個原因一起改動，它們屬於同一個職責，不需要拆開。

---

### 1.2 OCP — 開放封閉原則（Open/Closed Principle）

#### 定義

> *"Software entities should be open for extension, but closed for modification."*
> — Bertrand Meyer / Robert C. Martin

軟體模組應該對擴展開放、對修改封閉。當需要新增功能時，應該透過擴展（新增程式碼）來實現，而非修改既有程式碼。

#### 為什麼重要

每次修改既有程式碼，都有引入新 bug 的風險。如果架構設計良好，新增功能只需要「加」程式碼，而非「改」程式碼。

#### 不遵守會怎樣

每次新增一種支付方式，就要改一次核心的支付處理方法，改到第十種時，那個方法已經變成一百行的 if-else 地獄。

#### 錯誤示範

```java
@Service
public class PaymentService {

    public PaymentResult pay(PaymentRequest request) {
        if ("CREDIT_CARD".equals(request.getMethod())) {
            // 信用卡支付邏輯... 30 行
        } else if ("LINE_PAY".equals(request.getMethod())) {
            // LINE Pay 支付邏輯... 25 行
        } else if ("CASH".equals(request.getMethod())) {
            // 現金支付邏輯... 10 行
        }
        // 每新增一種支付方式，就要改這個方法
        return result;
    }
}
```

#### 正確示範：Strategy Pattern + Spring DI

```java
public interface PaymentStrategy {
    String getMethod();
    PaymentResult pay(PaymentRequest request);
}

@Component
public class CreditCardPayment implements PaymentStrategy {
    @Override
    public String getMethod() { return "CREDIT_CARD"; }

    @Override
    public PaymentResult pay(PaymentRequest request) {
        // 信用卡支付邏輯
        return new PaymentResult(true, "信用卡付款成功");
    }
}

@Component
public class LinePayPayment implements PaymentStrategy {
    @Override
    public String getMethod() { return "LINE_PAY"; }

    @Override
    public PaymentResult pay(PaymentRequest request) {
        // LINE Pay 支付邏輯
        return new PaymentResult(true, "LINE Pay 付款成功");
    }
}

@Service
@RequiredArgsConstructor
public class PaymentService {

    private final Map<String, PaymentStrategy> strategyMap;

    // Spring 會自動注入所有 PaymentStrategy 的實作
    public PaymentService(List<PaymentStrategy> strategies) {
        this.strategyMap = strategies.stream()
                .collect(Collectors.toMap(PaymentStrategy::getMethod, Function.identity()));
    }

    public PaymentResult pay(PaymentRequest request) {
        PaymentStrategy strategy = strategyMap.get(request.getMethod());
        if (strategy == null) {
            throw new UnsupportedPaymentException(request.getMethod());
        }
        return strategy.pay(request);
    }
}
```

新增支付方式時，只需新增一個 `@Component` 類別實作 `PaymentStrategy`，`PaymentService` 完全不用修改。

#### Trade-off

不是所有地方都需要套用 OCP。如果一段邏輯只有兩三個分支且未來不太可能增加，簡單的 if-else 反而更好讀。過早抽象（premature abstraction）是另一種浪費。

---

### 1.3 LSP — 里氏替換原則（Liskov Substitution Principle）

#### 定義

> *"Objects in a program should be replaceable with instances of their subtypes without altering the correctness of that program."*
> — Barbara Liskov, 1987

子類別必須能夠替換父類別使用，且不改變程式的正確性。換句話說，凡是父類別能做的事，子類別都要能做到，而且行為一致。

#### 為什麼重要

如果子類別改變了父類別的行為契約，所有依賴父類別的程式碼都可能出錯。這會讓多型失去意義。

#### 不遵守會怎樣

你以為替換了一個實作就好，結果某些呼叫端因為子類別行為不一致而拋出例外或產生錯誤結果。

#### 經典反例：Square extends Rectangle

```java
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int w)  { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int w) {
        this.width = w;
        this.height = w; // 違反 LSP：改變了 setWidth 的行為契約
    }

    @Override
    public void setHeight(int h) {
        this.width = h;
        this.height = h;
    }
}
```

呼叫端預期 `setWidth` 只改寬度，但 `Square` 連高度一起改了，導致 `area()` 計算結果不符預期。

#### Spring 實例：介面設計符合 LSP

```java
public interface DiscountPolicy {
    /**
     * 計算折扣金額，回傳值必須 >= 0 且 <= originalPrice。
     */
    BigDecimal calculateDiscount(BigDecimal originalPrice, Customer customer);
}

@Component("vipDiscount")
public class VipDiscountPolicy implements DiscountPolicy {
    @Override
    public BigDecimal calculateDiscount(BigDecimal originalPrice, Customer customer) {
        return originalPrice.multiply(new BigDecimal("0.1")); // 九折 → 折扣 10%
    }
}

@Component("newCustomerDiscount")
public class NewCustomerDiscountPolicy implements DiscountPolicy {
    @Override
    public BigDecimal calculateDiscount(BigDecimal originalPrice, Customer customer) {
        return originalPrice.multiply(new BigDecimal("0.05")); // 折扣 5%
    }
}
```

兩個實作都遵守介面契約：回傳值 >= 0 且 <= `originalPrice`，呼叫端可以安全替換。

#### Trade-off

嚴格遵守 LSP 有時意味著不能使用繼承，而要改用組合（composition）。這通常是更好的選擇——「組合優於繼承」正是 GoF 的核心建議之一。

---

### 1.4 ISP — 介面隔離原則（Interface Segregation Principle）

#### 定義

> *"Clients should not be forced to depend upon interfaces that they do not use."*
> — Robert C. Martin

不應該強迫客戶端依賴它不需要的方法。大而全的介面應該拆分為多個小而專精的介面。

#### 為什麼重要

當一個介面太大，實作者被迫實作一堆用不到的方法（通常直接 `throw new UnsupportedOperationException()`），呼叫端也被迫依賴不需要的方法。

#### 不遵守會怎樣

修改介面中的一個方法簽章，所有實作者都要跟著改，即便大部分實作者根本沒用到那個方法。

#### 錯誤示範

```java
public interface CustomerRepository {
    Customer findById(Long id);
    List<Customer> findAll();
    Customer save(Customer customer);
    void delete(Long id);
    List<Customer> searchByName(String name);
    List<Customer> findByDateRange(LocalDate start, LocalDate end);
    void exportToCsv(OutputStream out);     // 報表模組才需要
    void importFromExcel(InputStream in);   // 匯入模組才需要
}
```

#### 正確示範

```java
public interface CustomerReadRepository {
    Customer findById(Long id);
    List<Customer> findAll();
    List<Customer> searchByName(String name);
    List<Customer> findByDateRange(LocalDate start, LocalDate end);
}

public interface CustomerWriteRepository {
    Customer save(Customer customer);
    void delete(Long id);
}

public interface CustomerExportRepository {
    void exportToCsv(OutputStream out);
}

public interface CustomerImportRepository {
    void importFromExcel(InputStream in);
}
```

每個模組只依賴自己需要的介面，降低耦合。在 Spring 中，一個實作類別可以同時實作多個介面：

```java
@Repository
public class CustomerRepositoryImpl
        implements CustomerReadRepository, CustomerWriteRepository {
    // 只實作讀寫相關方法，匯出匯入由專門的類別處理
}
```

#### Trade-off

介面拆太細也會增加管理成本。如果一個介面的所有方法總是一起被使用，就不需要拆分。判斷標準：看「客戶端」——不同的客戶端是否只用到介面的一部分。

---

### 1.5 DIP — 依賴反轉原則（Dependency Inversion Principle）

#### 定義

> *"High-level modules should not depend on low-level modules. Both should depend on abstractions."*
> — Robert C. Martin

高層模組不應該依賴低層模組，兩者都應該依賴抽象。抽象不應該依賴細節，細節應該依賴抽象。

#### 為什麼重要

如果高層業務邏輯直接依賴低層實作（例如 Service 直接 `new JdbcCustomerDao()`），更換資料庫實作時就要改業務邏輯。透過依賴抽象（介面），高層模組與低層實作解耦。

#### 不遵守會怎樣

想從 MySQL 換成 PostgreSQL，或從 REST 呼叫改成 gRPC，結果要改幾十個 Service 類別。

#### Spring 的 IoC 就是 DIP 的實現

Spring 的 IoC（Inversion of Control）容器正是 DIP 的最佳實踐。Service 依賴介面，Spring 負責注入具體實作：

```java
// 高層模組依賴抽象
@Service
@RequiredArgsConstructor
public class OrderService {
    private final PaymentGateway paymentGateway; // 介面，非具體實作
}

// 低層模組實作抽象
@Component
public class EcPayGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(PaymentRequest request) { /* ... */ }
}
```

#### 建構子注入 vs @Autowired

```java
// 推薦：建構子注入（不可變、容易測試、明確依賴）
@Service
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}

// 不推薦：欄位注入（難以測試、隱藏依賴、允許 null）
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
}
```

Spring 官方文件明確推薦建構子注入。搭配 Lombok 的 `@RequiredArgsConstructor`，可以省去手寫建構子的樣板程式碼。

#### Trade-off

過度抽象會讓程式碼難以追蹤（IDE 跳轉到介面而非實作）。如果某個模組只有一個實作，且短期內不會有第二個，可以暫時不抽介面——需要時再重構也不遲（YAGNI）。

---

## 2、Clean Code 實踐

Robert C. Martin 在《Clean Code》一書中提出了許多實踐原則，以下是 Java 開發中最重要的三項。

### 2.1 命名（Meaningful Names）

好的命名讓程式碼自我解釋，減少對註解的依賴。

```java
// 壞：含義不明
int d; // elapsed time in days
List<int[]> list1;

// 好：意圖明確
int elapsedTimeInDays;
List<Cell> flaggedCells;

// 壞：名稱不一致
public Customer getCustomer(Long id);     // 有時用 get
public Customer fetchCustomerById(Long id); // 有時用 fetch
public Customer retrieveCustomer(Long id);  // 有時用 retrieve

// 好：團隊統一用同一個詞彙
public Customer findById(Long id);
public Order findById(Long id);
```

**Spring Boot 命名慣例**：

| 層級 | 命名格式 | 範例 |
|------|---------|------|
| Controller | `XxxController` | `OrderController` |
| Service | `XxxService` | `OrderService` |
| Repository | `XxxRepository` | `OrderRepository` |
| DTO | `XxxRequest` / `XxxResponse` | `CreateOrderRequest` |
| Entity | 單數名詞 | `Order`、`Customer` |

### 2.2 函式設計

#### 小函式、單一職責

> *"The first rule of functions is that they should be small. The second rule is that they should be smaller than that."*
> — Robert C. Martin,《Clean Code》

```java
// 壞：一個方法做太多事（驗證 + 計算 + 儲存 + 通知）
public Order processOrder(OrderRequest request) {
    // 驗證... 20 行
    // 計算金額... 15 行
    // 儲存... 10 行
    // 發送通知... 10 行
    return order;
}

// 好：每個方法只做一件事
public Order processOrder(OrderRequest request) {
    validateRequest(request);
    Order order = createOrderFromRequest(request);
    BigDecimal total = calculateTotal(order);
    order.setTotal(total);
    orderRepository.save(order);
    notificationService.notifyOrderCreated(order);
    return order;
}
```

#### 避免 Side Effect

```java
// 壞：名為 check 卻偷偷做了初始化（side effect）
public boolean checkPassword(String username, String password) {
    User user = userRepository.findByUsername(username);
    if (passwordEncoder.matches(password, user.getPassword())) {
        Session.initialize(); // 隱藏的副作用！
        return true;
    }
    return false;
}

// 好：名稱反映實際行為
public boolean verifyPassword(String username, String password) {
    User user = userRepository.findByUsername(username);
    return passwordEncoder.matches(password, user.getPassword());
}

public void authenticateAndInitSession(String username, String password) {
    if (verifyPassword(username, password)) {
        Session.initialize();
    }
}
```

### 2.3 註解的正確使用

#### 好的註解

```java
// 解釋「為什麼」而非「做什麼」
// 因為第三方 API 有速率限制（每秒 5 次），所以需要加入延遲
Thread.sleep(200);

// 警告後果
// WARNING: 此方法非 thread-safe，不可在多執行緒環境下呼叫

// TODO 標記
// TODO: 效能優化 — 大量資料時改為批次查詢（#TIRE-234）

// 法律資訊
/* Copyright (c) 2025 TireMaster Inc. All rights reserved. */
```

#### 壞的註解

```java
// 壞：重複程式碼已經表達的資訊
// 設定名稱
customer.setName(name);

// 壞：過時的註解（比沒有註解更糟）
// 回傳客戶列表（實際上已改為回傳 Page<Customer>）
public List<Customer> findAll() { /* ... */ }

// 壞：被註解掉的程式碼（用版控管理，別留著）
// order.setDiscount(0.1);
// order.setVipLevel(3);
```

---

## 3、Clean Architecture

### 3.1 同心圓架構

Robert C. Martin 在《Clean Architecture》中提出了同心圓架構，核心原則是**依賴規則（Dependency Rule）**：原始碼的依賴方向只能從外層指向內層，內層不應該知道外層的存在。

```mermaid
flowchart TB
    subgraph 外層 — Frameworks & Drivers
        A[Web / Controller]
        B[Database / JPA]
        C[External API]
    end

    subgraph 中外層 — Interface Adapters
        D[Controller / Gateway]
        E[Presenter / DTO]
        F[Repository Impl]
    end

    subgraph 中內層 — Application Business Rules
        G[Use Cases / Service]
    end

    subgraph 核心 — Enterprise Business Rules
        H[Entities / Domain Model]
    end

    A --> D
    B --> F
    D --> G
    E --> G
    F --> G
    G --> H
```

### 3.2 依賴規則

- **內層**（Entity、Domain Model）：純粹的業務規則，不依賴任何框架。
- **中內層**（Use Case / Service）：協調 Entity 完成業務流程，只依賴 Domain。
- **中外層**（Adapter）：轉換資料格式（DTO ↔ Entity），實作 Repository 介面。
- **外層**（Framework）：Spring MVC、JPA、外部 API 呼叫。

依賴方向永遠是：**外 → 內**。如果內層需要呼叫外層（例如 Service 需要資料庫），透過介面反轉依賴（DIP）。

### 3.3 Spring Boot 專案的分層對應

```
src/main/java/com/tiremaster/store/
├── controller/          ← 外層：接收 HTTP 請求
│   └── OrderController.java
├── dto/                 ← 中外層：資料傳輸物件
│   ├── CreateOrderRequest.java
│   └── OrderResponse.java
├── service/             ← 中內層：業務邏輯（Use Case）
│   ├── OrderService.java
│   └── impl/
│       └── OrderServiceImpl.java
├── domain/              ← 核心：領域模型（Entity）
│   └── Order.java
└── repository/          ← 中外層：資料存取介面 + 實作
    └── OrderRepository.java
```

### 3.4 DRY / KISS / YAGNI

| 原則 | 全稱 | 說明 | 常見違反情境 |
|------|------|------|------------|
| **DRY** | Don't Repeat Yourself | 每個知識在系統中只有一個明確的表述 | 複製貼上程式碼、多處硬編碼相同常數 |
| **KISS** | Keep It Simple, Stupid | 用最簡單的方案解決問題 | 為了「未來可能」而引入複雜框架 |
| **YAGNI** | You Aren't Gonna Need It | 不要為尚未確認的需求寫程式碼 | 預先設計用不到的抽象層 |

**注意**：DRY 不是指「程式碼長得一樣就要合併」。如果兩段程式碼只是碰巧相似，但代表不同的業務概念，合併反而增加耦合。判斷標準是：它們是否因為相同的原因變更？

---

## 4、小結

### SOLID 原則速查表

| 原則 | 核心問題 | 一句話 | 違反的代價 |
|------|---------|-------|-----------|
| **SRP** | 這個類別有幾個被修改的理由？ | 一個類別只負責一件事 | 改 A 壞 B，測試困難 |
| **OCP** | 新增功能要改幾個檔案？ | 對擴展開放，對修改封閉 | if-else 地獄，改核心高風險 |
| **LSP** | 子類別能安全替換父類別嗎？ | 子類別不改變父類別的行為契約 | 多型失效，替換即爆炸 |
| **ISP** | 實作者被迫實作不需要的方法嗎？ | 介面小而專精 | 改介面連鎖反應，空實作滿天飛 |
| **DIP** | 高層模組直接依賴低層實作嗎？ | 依賴抽象，不依賴具體 | 換實作要改業務邏輯 |

### 關鍵心得

1. **SOLID 不是教條**——它們是指導方針，需要根據情境判斷適用程度。
2. **過度設計和設計不足一樣糟**——在「足夠好」和「完美」之間，選擇前者。
3. **先讓它運作，再讓它正確，最後讓它快速**（Make it work, make it right, make it fast）。
4. **Clean Code 的最終目標**：讓下一個讀程式碼的人（可能是三個月後的你自己）能快速理解。

### 延伸閱讀

- [02 設計模式實戰](02%20設計模式實戰應用.md) — Strategy、Factory、Observer 等模式在 Spring Boot 中的應用
- [03 架構模式](03%20軟體架構模式.md) — Layered、Hexagonal、Event-Driven 架構比較
- [08 程式碼審查](08%20程式碼審查與重構.md) — 如何在 Code Review 中檢查 SOLID 與 Clean Code 實踐

### 參考書目

- Robert C. Martin,《Clean Code》, Prentice Hall, 2008
- Robert C. Martin,《Clean Architecture》, Prentice Hall, 2017
- Robert C. Martin,《Agile Software Development, Principles, Patterns, and Practices》, Prentice Hall, 2002

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
