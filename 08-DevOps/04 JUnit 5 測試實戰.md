# 04 JUnit 5 測試實戰

> **版本**：JUnit 5.10+ / Spring Boot 3.x / Java 17+

測試是軟體品質的基石。JUnit 5 是 Java 生態系中最主流的測試框架，提供了豐富的註解、彈性的擴展機制，以及與 Spring Boot 的深度整合。本篇將從 JUnit 5 的架構開始，逐步介紹基本用法、參數化測試、Spring Boot 各層級測試，以及 Testcontainers 的整合測試方案。

---

## 1、JUnit 5 架構

JUnit 5 由三個子專案組成，與 JUnit 4 相比是一次完整的架構重新設計：

- **JUnit Platform**：測試的啟動基礎設施，提供 TestEngine API，讓不同測試框架可以在同一平台上執行。
- **JUnit Jupiter**：JUnit 5 的核心，包含新的程式設計模型（註解、斷言）和擴展機制。
- **JUnit Vintage**：向後相容層，讓 JUnit 3 和 JUnit 4 的測試可以在 JUnit 5 平台上執行，方便漸進式遷移。

### JUnit 5 vs JUnit 4 主要差異

| 項目 | JUnit 4 | JUnit 5 |
|------|---------|---------|
| 最低 Java 版本 | Java 5 | Java 8 |
| 架構 | 單一 jar | 模組化（Platform + Jupiter + Vintage） |
| 測試類別 | 必須 public | 可以是 package-private |
| 生命週期註解 | `@Before` / `@After` | `@BeforeEach` / `@AfterEach` |
| 擴展機制 | `@Rule` / `@RunWith` | `@ExtendWith`（統一機制） |
| 巢狀測試 | 不支援 | `@Nested` |
| 參數化測試 | 需要 `@RunWith(Parameterized.class)` | 原生 `@ParameterizedTest` |
| 條件執行 | 需要 `Assume` | `@EnabledOnOs`、`@EnabledIf` 等 |

### Gradle 相依設定

```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    // Spring Boot Starter Test 已包含 JUnit Jupiter、Mockito、AssertJ
}
```

---

## 2、基本註解

### 2.1 生命週期註解

```java
class LifecycleTest {

    @BeforeAll
    static void initAll() {
        // 整個測試類別執行前，只執行一次（必須 static）
        System.out.println("BeforeAll - 初始化共享資源");
    }

    @BeforeEach
    void init() {
        // 每個 @Test 方法執行前都會執行
        System.out.println("BeforeEach - 重置測試狀態");
    }

    @Test
    void testSomething() {
        System.out.println("執行測試");
    }

    @AfterEach
    void tearDown() {
        // 每個 @Test 方法執行後都會執行
        System.out.println("AfterEach - 清理資源");
    }

    @AfterAll
    static void tearDownAll() {
        // 整個測試類別結束後，只執行一次
        System.out.println("AfterAll - 釋放共享資源");
    }
}
```

### 2.2 描述與控制註解

```java
@DisplayName("訂單服務測試")
class OrderServiceTest {

    @Test
    @DisplayName("建立訂單 - 庫存充足時應成功")
    void shouldCreateOrderWhenStockAvailable() {
        // @DisplayName 讓測試報告更易讀
    }

    @Test
    @Disabled("等待 API 規格確認 - JIRA-1234")
    void shouldCalculateShippingFee() {
        // @Disabled 暫時跳過測試，必須附帶原因
    }

    @Test
    @Tag("slow")
    void shouldGenerateMonthlyReport() {
        // @Tag 標記測試分類，可在建置時選擇性執行
    }
}
```

### 2.3 巢狀測試（@Nested）

`@Nested` 讓測試以邏輯群組方式組織，提升可讀性：

```java
@DisplayName("購物車")
class ShoppingCartTest {

    private ShoppingCart cart;

    @BeforeEach
    void setUp() {
        cart = new ShoppingCart();
    }

    @Nested
    @DisplayName("當購物車為空時")
    class WhenEmpty {

        @Test
        @DisplayName("商品數量應為 0")
        void shouldHaveZeroItems() {
            assertEquals(0, cart.getItemCount());
        }

        @Test
        @DisplayName("總金額應為 0")
        void shouldHaveZeroTotal() {
            assertEquals(BigDecimal.ZERO, cart.getTotal());
        }
    }

    @Nested
    @DisplayName("加入商品後")
    class AfterAddingItem {

        @BeforeEach
        void addItem() {
            cart.add(new Product("輪胎", new BigDecimal("2500")));
        }

        @Test
        @DisplayName("商品數量應為 1")
        void shouldHaveOneItem() {
            assertEquals(1, cart.getItemCount());
        }

        @Test
        @DisplayName("總金額應等於商品價格")
        void shouldHaveCorrectTotal() {
            assertEquals(new BigDecimal("2500"), cart.getTotal());
        }
    }
}
```

---

## 3、斷言（Assertions）

JUnit Jupiter 提供豐富的靜態斷言方法，位於 `org.junit.jupiter.api.Assertions`。

### 3.1 基礎斷言

```java
import static org.junit.jupiter.api.Assertions.*;

class AssertionExamplesTest {

    @Test
    void basicAssertions() {
        // 相等判斷
        assertEquals(4, 2 + 2, "2 + 2 應等於 4");
        assertNotEquals("hello", "world");

        // 布林判斷
        assertTrue(10 > 5);
        assertFalse(10 < 5);

        // Null 判斷
        assertNull(null);
        assertNotNull("not null");

        // 同一物件判斷
        String s = "test";
        assertSame(s, s);
    }
}
```

### 3.2 例外斷言

```java
@Test
void shouldThrowWhenBalanceInsufficient() {
    Account account = new Account(1000);

    InsufficientBalanceException exception = assertThrows(
        InsufficientBalanceException.class,
        () -> account.withdraw(2000)
    );

    assertEquals("餘額不足，目前餘額：1000", exception.getMessage());
}
```

### 3.3 群組斷言（assertAll）

`assertAll` 會執行所有斷言後統一回報失敗，不會因第一個失敗就中斷：

```java
@Test
void shouldReturnCorrectCustomerInfo() {
    Customer customer = customerService.findById(1L);

    assertAll("客戶資料驗證",
        () -> assertEquals("王大明", customer.getName()),
        () -> assertEquals("0912-345-678", customer.getPhone()),
        () -> assertEquals("台北市", customer.getCity()),
        () -> assertNotNull(customer.getCreatedAt())
    );
}
```

### 3.4 逾時斷言

```java
@Test
void shouldRespondWithinTimeout() {
    assertTimeout(Duration.ofSeconds(3), () -> {
        // 模擬耗時操作
        return externalService.fetchData();
    });
}
```

---

## 4、參數化測試

參數化測試讓同一段測試邏輯可以用不同的輸入值重複執行，大幅減少重複程式碼。

### 4.1 @ValueSource

提供單一參數的簡單來源：

```java
@ParameterizedTest
@DisplayName("手機號碼格式驗證 - 有效號碼")
@ValueSource(strings = {"0912345678", "0923456789", "0934567890"})
void shouldAcceptValidPhoneNumber(String phone) {
    assertTrue(PhoneValidator.isValid(phone));
}

@ParameterizedTest
@DisplayName("正整數判斷")
@ValueSource(ints = {1, 5, 100, Integer.MAX_VALUE})
void shouldBePositive(int number) {
    assertTrue(number > 0);
}
```

### 4.2 @CsvSource

提供多參數的 CSV 格式來源：

```java
@ParameterizedTest
@DisplayName("折扣計算")
@CsvSource({
    "1000, 10, 900",
    "2000, 20, 1600",
    "500,   0, 500",
    "3000, 50, 1500"
})
void shouldCalculateDiscount(int price, int discountPercent, int expected) {
    assertEquals(expected, PriceCalculator.applyDiscount(price, discountPercent));
}
```

### 4.3 @MethodSource

從方法提供複雜的測試參數：

```java
@ParameterizedTest
@DisplayName("訂單狀態轉換驗證")
@MethodSource("provideStatusTransitions")
void shouldAllowValidStatusTransition(OrderStatus from, OrderStatus to, boolean expected) {
    assertEquals(expected, OrderStateMachine.canTransition(from, to));
}

static Stream<Arguments> provideStatusTransitions() {
    return Stream.of(
        Arguments.of(OrderStatus.CREATED, OrderStatus.PAID, true),
        Arguments.of(OrderStatus.PAID, OrderStatus.SHIPPED, true),
        Arguments.of(OrderStatus.SHIPPED, OrderStatus.CREATED, false),
        Arguments.of(OrderStatus.CANCELLED, OrderStatus.PAID, false)
    );
}
```

### 4.4 @EnumSource

遍歷列舉值：

```java
@ParameterizedTest
@DisplayName("所有付款方式都應有對應的處理器")
@EnumSource(PaymentMethod.class)
void shouldHaveProcessorForEveryPaymentMethod(PaymentMethod method) {
    assertNotNull(paymentProcessorFactory.getProcessor(method));
}

@ParameterizedTest
@DisplayName("排除特定列舉值")
@EnumSource(value = OrderStatus.class, names = {"CANCELLED", "REFUNDED"}, mode = EnumSource.Mode.EXCLUDE)
void shouldBeActiveStatus(OrderStatus status) {
    assertTrue(status.isActive());
}
```

---

## 5、Spring Boot 測試

Spring Boot 提供多種測試切片（Test Slice），針對不同層級提供精確的測試支援。

### 5.1 @SpringBootTest（完整整合測試）

載入完整的 Spring Application Context，適合端對端整合測試：

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    @DisplayName("建立訂單完整流程")
    void shouldCreateOrderEndToEnd() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest("TIRE-001", 4);

        // When
        ResponseEntity<OrderResponse> response = restTemplate.postForEntity(
            "/api/orders", request, OrderResponse.class
        );

        // Then
        assertEquals(HttpStatus.CREATED, response.getStatusCode());
        assertNotNull(response.getBody());
        assertNotNull(response.getBody().getOrderId());
    }
}
```

> 注意：`@SpringBootTest` 會載入所有 Bean，啟動較慢。只有在需要測試跨層級互動時才使用。

### 5.2 @WebMvcTest（Controller 層測試）

只載入 Web 層相關的 Bean，使用 `MockMvc` 模擬 HTTP 請求：

```java
@WebMvcTest(CustomerController.class)
class CustomerControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private CustomerService customerService;

    @Test
    @DisplayName("GET /api/customers/{id} - 客戶存在時回傳 200")
    void shouldReturnCustomerWhenExists() throws Exception {
        // Given
        CustomerDTO customer = new CustomerDTO(1L, "王大明", "0912345678");
        when(customerService.findById(1L)).thenReturn(customer);

        // When & Then
        mockMvc.perform(get("/api/customers/{id}", 1L)
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("王大明"))
            .andExpect(jsonPath("$.phone").value("0912345678"));
    }

    @Test
    @DisplayName("GET /api/customers/{id} - 客戶不存在時回傳 404")
    void shouldReturn404WhenCustomerNotFound() throws Exception {
        // Given
        when(customerService.findById(99L))
            .thenThrow(new CustomerNotFoundException(99L));

        // When & Then
        mockMvc.perform(get("/api/customers/{id}", 99L))
            .andExpect(status().isNotFound());
    }

    @Test
    @DisplayName("POST /api/customers - 缺少必填欄位時回傳 400")
    void shouldReturn400WhenNameMissing() throws Exception {
        String invalidJson = """
            {
                "phone": "0912345678"
            }
            """;

        mockMvc.perform(post("/api/customers")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidJson))
            .andExpect(status().isBadRequest());
    }
}
```

### 5.3 @DataJpaTest（Repository 層測試）

只載入 JPA 相關的 Bean，自動配置嵌入式資料庫（如 H2），每個測試方法預設自動回滾：

```java
@DataJpaTest
class CustomerRepositoryTest {

    @Autowired
    private CustomerRepository customerRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    @DisplayName("依手機號碼查詢客戶")
    void shouldFindByPhone() {
        // Given
        Customer customer = new Customer();
        customer.setName("王大明");
        customer.setPhone("0912345678");
        entityManager.persistAndFlush(customer);

        // When
        Optional<Customer> found = customerRepository.findByPhone("0912345678");

        // Then
        assertTrue(found.isPresent());
        assertEquals("王大明", found.get().getName());
    }

    @Test
    @DisplayName("查詢不存在的手機號碼應回傳空")
    void shouldReturnEmptyWhenPhoneNotFound() {
        Optional<Customer> found = customerRepository.findByPhone("0000000000");
        assertTrue(found.isEmpty());
    }
}
```

### 5.4 Mockito 單元測試

使用 `@MockitoExtension` 進行純單元測試，不載入 Spring Context，速度最快：

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private InventoryService inventoryService;

    @InjectMocks
    private OrderService orderService;

    @Test
    @DisplayName("建立訂單 - 庫存充足時應成功儲存")
    void shouldCreateOrderWhenStockAvailable() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest("TIRE-001", 4);
        when(inventoryService.checkStock("TIRE-001", 4)).thenReturn(true);
        when(orderRepository.save(any(Order.class))).thenAnswer(invocation -> {
            Order order = invocation.getArgument(0);
            order.setId(1L);
            return order;
        });

        // When
        OrderResponse response = orderService.createOrder(request);

        // Then
        assertNotNull(response);
        assertEquals(1L, response.getOrderId());
        verify(inventoryService).checkStock("TIRE-001", 4);
        verify(orderRepository).save(any(Order.class));
    }

    @Test
    @DisplayName("建立訂單 - 庫存不足時應拋出例外")
    void shouldThrowWhenStockInsufficient() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest("TIRE-001", 100);
        when(inventoryService.checkStock("TIRE-001", 100)).thenReturn(false);

        // When & Then
        assertThrows(InsufficientStockException.class,
            () -> orderService.createOrder(request));

        verify(orderRepository, never()).save(any());
    }
}
```

---

## 6、Testcontainers（簡述）

Testcontainers 讓整合測試能夠使用真實的資料庫、快取等服務，透過 Docker 在測試時自動啟動容器，測試結束後自動銷毀。

### Gradle 相依設定

```kotlin
dependencies {
    testImplementation("org.testcontainers:junit-jupiter:1.19.3")
    testImplementation("org.testcontainers:postgresql:1.19.3")
}
```

### PostgreSQL 整合測試範例

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    @DisplayName("使用真實 PostgreSQL 測試訂單儲存")
    void shouldPersistOrder() {
        // Given
        Order order = new Order();
        order.setProductCode("TIRE-001");
        order.setQuantity(4);
        order.setStatus(OrderStatus.CREATED);

        // When
        Order saved = orderRepository.save(order);

        // Then
        assertNotNull(saved.getId());
        Optional<Order> found = orderRepository.findById(saved.getId());
        assertTrue(found.isPresent());
        assertEquals("TIRE-001", found.get().getProductCode());
    }
}
```

**核心概念**：

- `@Testcontainers`：啟用 Testcontainers 的 JUnit 5 擴展
- `@Container`：標記要管理的容器，`static` 表示整個測試類別共享同一個容器
- `@DynamicPropertySource`：在容器啟動後動態注入連線資訊到 Spring 環境

Testcontainers 的優勢在於使用與正式環境相同的資料庫，避免 H2 與 PostgreSQL 之間的 SQL 方言差異導致測試通過但上線失敗。

---

## 7、測試最佳實踐

### 7.1 Given-When-Then 結構

每個測試方法遵循三段式結構，提升可讀性：

```java
@Test
void shouldApplyDiscount() {
    // Given（前置條件）
    Product product = new Product("輪胎", new BigDecimal("3000"));
    DiscountPolicy policy = new PercentageDiscount(10);

    // When（執行目標行為）
    BigDecimal finalPrice = policy.apply(product.getPrice());

    // Then（驗證結果）
    assertEquals(new BigDecimal("2700"), finalPrice);
}
```

### 7.2 測試命名規範

好的測試名稱應描述「在什麼條件下，做什麼事，預期什麼結果」：

```java
// 推薦：should + 預期行為 + when + 條件
void shouldReturnEmptyList_whenNoOrdersExist()
void shouldThrowException_whenPasswordTooShort()
void shouldSendEmail_whenOrderConfirmed()

// 或搭配 @DisplayName 使用中文描述
@DisplayName("密碼長度不足 8 位時應拋出驗證例外")
void shouldThrowWhenPasswordTooShort()
```

### 7.3 測試金字塔

測試金字塔是測試策略的指導原則，從底部到頂部：

| 層級 | 佔比 | 特性 | 工具 |
|------|------|------|------|
| **Unit Test** | 約 70% | 快速、隔離、數量多 | JUnit + Mockito |
| **Integration Test** | 約 20% | 驗證元件互動、較慢 | @SpringBootTest、Testcontainers |
| **E2E Test** | 約 10% | 模擬真實使用者、最慢 | Selenium、Playwright |

**原則**：

- 單元測試應佔最大比例，執行速度快，能覆蓋大部分邏輯分支
- 整合測試驗證元件之間的互動是否正確
- E2E 測試只覆蓋關鍵業務流程，避免過多導致測試套件緩慢
- 越靠近金字塔底部的測試，維護成本越低，回饋速度越快

### 7.4 其他實務建議

- **一個測試方法只驗證一件事**：避免在同一個測試中驗證多個不相關的行為
- **測試之間互相獨立**：不依賴執行順序，不共享可變狀態
- **避免測試中的邏輯**：不在測試中使用 `if`、`for`、`switch`，測試本身應該是直觀的
- **使用有意義的測試資料**：避免 `"aaa"`、`123` 等無意義值，用符合業務語意的資料
- **測試失敗訊息要有價值**：善用斷言的 message 參數，讓失敗時能快速定位問題

---

## 8、小結

本篇涵蓋了 JUnit 5 測試實戰的完整知識體系：

- **JUnit 5 架構**：Platform + Jupiter + Vintage 的模組化設計
- **基本用法**：生命週期註解、`@Nested` 群組測試、`@DisplayName` 增強可讀性
- **斷言技巧**：`assertAll` 群組驗證、`assertThrows` 例外捕捉
- **參數化測試**：四種參數來源，減少重複測試程式碼
- **Spring Boot 測試切片**：依層級選擇最適合的測試方式
- **Testcontainers**：用真實資料庫做整合測試

延伸學習方向：

- **AssertJ**：流式斷言庫，語法更直覺（`assertThat(list).hasSize(3).contains("item")`）
- **ArchUnit**：架構測試，驗證套件相依規則
- **JaCoCo**：測試覆蓋率報告，整合到 CI/CD 流程中

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
