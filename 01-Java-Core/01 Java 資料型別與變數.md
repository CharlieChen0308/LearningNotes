# 01 Java 資料型別與變數

> **版本**：Java 17+ — 包含 `var` 區域變數型別推斷、`record` 類別等現代特性

## 1、基本資料型別（Primitive Types）

Java 有 8 種基本型別，分為 4 類：

| 類別 | 型別 | 位元組 | 範圍 | 預設值 |
|------|------|--------|------|--------|
| 整數 | `byte` | 1 | -128 ~ 127 | 0 |
| 整數 | `short` | 2 | -32,768 ~ 32,767 | 0 |
| 整數 | `int` | 4 | -2³¹ ~ 2³¹-1（約 ±21 億） | 0 |
| 整數 | `long` | 8 | -2⁶³ ~ 2⁶³-1 | 0L |
| 浮點 | `float` | 4 | IEEE 754 單精度 | 0.0f |
| 浮點 | `double` | 8 | IEEE 754 雙精度 | 0.0d |
| 字元 | `char` | 2 | Unicode 0 ~ 65,535 | '\u0000' |
| 布林 | `boolean` | — | true / false | false |

> `boolean` 的實際佔用取決於 JVM 實作，規範未明確定義。

### 1.1 整數字面量

```java
int decimal = 42;
int hex = 0x2A;           // 16 進位
int octal = 052;          // 8 進位
int binary = 0b0010_1010; // 2 進位（Java 7+）
long big = 10_000_000L;   // 底線分隔（Java 7+），提升可讀性
```

### 1.2 型別轉換

```java
// 自動提升（小 → 大）
int i = 100;
long l = i;        // int → long，安全
double d = l;      // long → double，可能失去精度

// 強制轉換（大 → 小）
double pi = 3.14;
int truncated = (int) pi;  // 3，截斷小數

// 注意：byte/short 運算會自動提升為 int
byte a = 10, b = 20;
// byte c = a + b;  // 編譯錯誤！a + b 結果是 int
byte c = (byte) (a + b);  // 正確
```

## 2、包裝型別（Wrapper Types）

每個基本型別都有對應的包裝類別：

| 基本型別 | 包裝類別 | 快取範圍 |
|---------|---------|---------|
| `int` | `Integer` | -128 ~ 127 |
| `long` | `Long` | -128 ~ 127 |
| `double` | `Double` | 無快取 |
| `boolean` | `Boolean` | TRUE / FALSE |
| `char` | `Character` | 0 ~ 127 |

### 2.1 自動裝箱 / 拆箱

```java
Integer x = 42;       // 自動裝箱：Integer.valueOf(42)
int y = x;            // 自動拆箱：x.intValue()

// 陷阱：快取範圍內 == 為 true，範圍外為 false
Integer a = 127, b = 127;
System.out.println(a == b);   // true（快取）

Integer c = 128, d = 128;
System.out.println(c == d);   // false（不同物件）
System.out.println(c.equals(d)); // true（正確比較方式）
```

### 2.2 拆箱的 NullPointerException

```java
Integer nullable = null;
// int value = nullable;  // NPE！拆箱時呼叫 nullable.intValue()

// 安全做法
int value = (nullable != null) ? nullable : 0;
```

## 3、var 區域變數型別推斷（Java 10+）

```java
var list = new ArrayList<String>();  // 推斷為 ArrayList<String>
var map = Map.of("a", 1, "b", 2);   // 推斷為 Map<String, Integer>

for (var entry : map.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
```

**限制**：
- 只能用於區域變數（不能用於欄位、方法參數、回傳型別）
- 必須有初始化器（不能 `var x;`）
- 不能初始化為 `null`（`var x = null;` 編譯錯誤）

**何時使用**：
- 右邊已經明確表達型別時：`var stream = list.stream();`
- 泛型冗長時：`var map = new HashMap<String, List<Integer>>();`
- **不建議**用在型別不明顯的地方：`var result = service.process();`

## 4、Record 類別（Java 16+）

`record` 是不可變資料載體的簡寫，自動產生 `equals()`、`hashCode()`、`toString()` 及 getter：

```java
// 傳統寫法需要 30+ 行
// record 只需要一行
public record Point(int x, int y) {}

// 使用
var p = new Point(3, 4);
System.out.println(p.x());     // 3（getter 名稱是欄位名，不是 getX）
System.out.println(p);         // Point[x=3, y=4]

// 可加入自訂驗證
public record Range(int min, int max) {
    public Range {  // 緊湊建構子（Compact Constructor）
        if (min > max) {
            throw new IllegalArgumentException("min > max");
        }
    }
}
```

**Record 的限制**：
- 不能繼承其他類別（隱式繼承 `java.lang.Record`）
- 所有欄位都是 `final`，不可變
- 可以實作介面、定義靜態方法和實例方法

**常見用途**：DTO、API 回應物件、Map 的複合 Key

## 5、常量與 final

```java
// 區域常量
final int MAX_RETRY = 3;

// 類別常量
public static final double PI = 3.14159265358979;

// effectively final（Java 8+）：沒有 final 關鍵字但從未被重新賦值
int count = 10;
// Lambda 和匿名類別可以捕獲 effectively final 變數
Runnable r = () -> System.out.println(count);
```

## 6、Sealed Classes（Java 17）

`sealed` 限制哪些類別可以繼承：

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public final class Triangle implements Shape {
    // ...
}

// 搭配 pattern matching（Java 21 正式）
public static double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t -> /* ... */ 0;
    };
}
```

## 7、小結

| 概念 | 重點 |
|------|------|
| 8 種基本型別 | int(4), long(8), double(8), boolean, char(2), byte(1), short(2), float(4) |
| 包裝型別陷阱 | `==` 比較可能有快取問題；拆箱可能 NPE |
| `var` | Java 10+，僅限區域變數，右邊型別要明確 |
| `record` | Java 16+，不可變資料類別，自動產生 equals/hashCode/toString |
| `sealed` | Java 17，限制繼承層次，搭配 switch pattern matching |

> **延伸閱讀**：
> - [02 字串與集合框架](02%20字串與集合框架.md) — String、StringBuilder、List、Map
> - [06 常用關鍵字與設計模式](06%20常用關鍵字與設計模式.md) — final 深入、單例模式

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
