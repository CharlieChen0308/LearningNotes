# 06 Function Calling 工具呼叫

## 什麼是 Function Calling

AI 模型本身只能處理文字，無法存取即時資料（如天氣、股價）或執行操作（如發送郵件、查詢資料庫）。Function Calling（工具呼叫）讓 AI 模型可以**請求呼叫開發者定義的函式**，從而擴展 AI 的能力。

## 工作流程

```
使用者："台北現在幾度？"
        ↓
AI 模型判斷需要呼叫 getWeather 函式
        ↓
Spring AI 自動呼叫 getWeather("台北")
        ↓
函式返回 {"temp": 28, "unit": "°C"}
        ↓
AI 模型生成最終回答："台北現在的溫度是 28°C。"
```

整個過程對使用者來說是透明的，看起來就像 AI 直接知道答案。

## 基本使用

### 方式一：使用 @Bean 定義函式

```java
@Configuration
public class AiFunctionConfig {

    @Bean
    @Description("根據城市名稱查詢當前天氣")
    public Function<WeatherRequest, WeatherResponse> getWeather() {
        return request -> {
            // 實際應用中會呼叫天氣 API
            double temp = switch (request.city()) {
                case "台北" -> 28.5;
                case "東京" -> 22.0;
                case "紐約" -> 15.3;
                default -> 20.0;
            };
            return new WeatherResponse(request.city(), temp, "°C", "晴天");
        };
    }

    public record WeatherRequest(String city) {}
    public record WeatherResponse(String city, double temperature,
                                  String unit, String condition) {}
}
```

### 在對話中使用

```java
@GetMapping("/weather")
public String askWeather(@RequestParam String question) {
    return chatClient.prompt()
        .user(question)
        .functions("getWeather")  // 指定可用的函式（Bean 名稱）
        .call()
        .content();
}
```

呼叫 `/weather?question=台北和東京哪個比較熱` 時，AI 會自動呼叫 `getWeather` 兩次（分別查詢台北和東京），然後比較結果回答。

### 方式二：使用回呼函式

```java
@GetMapping("/calculate")
public String calculate(@RequestParam String question) {
    return chatClient.prompt()
        .user(question)
        .functions(
            FunctionCallback.builder()
                .function("calculateBMI", (BmiRequest req) -> {
                    double bmi = req.weight() / Math.pow(req.height() / 100.0, 2);
                    String category;
                    if (bmi < 18.5) category = "過輕";
                    else if (bmi < 24) category = "正常";
                    else if (bmi < 27) category = "過重";
                    else category = "肥胖";
                    return new BmiResponse(bmi, category);
                })
                .description("計算 BMI 指數")
                .inputType(BmiRequest.class)
                .build()
        )
        .call()
        .content();
}

record BmiRequest(double height, double weight) {}
record BmiResponse(double bmi, String category) {}
```

## 實用範例：資料庫查詢助手

```java
@Configuration
public class DatabaseFunctionConfig {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Bean
    @Description("根據客戶名稱查詢客戶資訊，包含姓名、電話、地址")
    public Function<CustomerQuery, CustomerInfo> queryCustomer() {
        return query -> {
            Map<String, Object> result = jdbcTemplate.queryForMap(
                "SELECT name, phone, address FROM customers WHERE name LIKE ?",
                "%" + query.name() + "%");
            return new CustomerInfo(
                (String) result.get("name"),
                (String) result.get("phone"),
                (String) result.get("address")
            );
        };
    }

    @Bean
    @Description("查詢指定產品的庫存數量")
    public Function<ProductQuery, StockInfo> queryStock() {
        return query -> {
            Map<String, Object> result = jdbcTemplate.queryForMap(
                "SELECT product_name, quantity FROM inventory WHERE product_name LIKE ?",
                "%" + query.productName() + "%");
            return new StockInfo(
                (String) result.get("product_name"),
                (Integer) result.get("quantity")
            );
        };
    }

    record CustomerQuery(String name) {}
    record CustomerInfo(String name, String phone, String address) {}
    record ProductQuery(String productName) {}
    record StockInfo(String productName, int quantity) {}
}
```

```java
@GetMapping("/assistant")
public String assistant(@RequestParam String question) {
    return chatClient.prompt()
        .system("你是一位客服助手，可以查詢客戶資訊和庫存。用繁體中文回答。")
        .user(question)
        .functions("queryCustomer", "queryStock")
        .call()
        .content();
}
```

使用者可以自然地提問：
- "幫我查一下王先生的電話"
- "205/55R16 的庫存還有多少"

AI 會自動判斷需要呼叫哪個函式，並用自然語言回覆。

## 多函式協作

AI 模型可以在一次對話中呼叫多個函式：

```java
@Bean
@Description("查詢航班資訊")
public Function<FlightQuery, FlightInfo> queryFlight() { ... }

@Bean
@Description("查詢飯店空房")
public Function<HotelQuery, HotelInfo> queryHotel() { ... }

@Bean
@Description("預訂行程")
public Function<BookingRequest, BookingResult> bookTrip() { ... }
```

使用者說："幫我查下週五台北到東京的航班，順便找一間飯店"，AI 會同時呼叫 `queryFlight` 和 `queryHotel`。

## 安全注意事項

- **永遠不要**讓 AI 直接執行 SQL，應該透過參數化查詢
- 對函式的輸入進行驗證和清理
- 限制可用函式的範圍，不要暴露危險操作
- 記錄所有函式呼叫的日誌，便於稽核

```java
@Bean
@Description("查詢訂單狀態")
public Function<OrderQuery, OrderStatus> queryOrder() {
    return query -> {
        // 輸入驗證
        if (query.orderId() == null || query.orderId().isBlank()) {
            return new OrderStatus("error", "訂單編號不能為空");
        }
        // 使用參數化查詢，防止 SQL 注入
        // ...
    };
}
```

## 小結

Function Calling 大幅擴展了 AI 模型的能力邊界，讓 AI 可以存取即時資料並執行操作。Spring AI 透過 `@Bean` 定義和 `FunctionCallback` 兩種方式，讓開發者能輕鬆將既有的業務邏輯暴露給 AI 模型使用。
