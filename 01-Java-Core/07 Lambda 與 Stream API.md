# 07 Lambda 與 Stream API

> **版本**：Java 17+ — 涵蓋 Function/Consumer/Predicate/Supplier、Stream 操作鏈、Collectors、平行流

## 1、Lambda 表達式

### 1.1 什麼是 Lambda

Lambda 是匿名函式的簡寫，用於實作**函式式介面**（只有一個抽象方法的介面）：

```java
// 傳統寫法
Comparator<String> comp1 = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
};

// Lambda 寫法
Comparator<String> comp2 = (a, b) -> a.length() - b.length();

// 方法引用
Comparator<String> comp3 = Comparator.comparingInt(String::length);
```

### 1.2 語法規則

```java
// 無參數
Runnable r = () -> System.out.println("hello");

// 單參數（可省略括號）
Consumer<String> c = s -> System.out.println(s);

// 多參數
BinaryOperator<Integer> add = (a, b) -> a + b;

// 多行（需要大括號和 return）
Function<String, Integer> parse = s -> {
    s = s.trim();
    return Integer.parseInt(s);
};
```

### 1.3 方法引用（Method Reference）

| 類型 | 語法 | Lambda 等價 |
|------|------|------------|
| 靜態方法 | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| 實例方法（特定物件） | `System.out::println` | `s -> System.out.println(s)` |
| 實例方法（任意物件） | `String::length` | `s -> s.length()` |
| 建構子 | `ArrayList::new` | `() -> new ArrayList<>()` |

## 2、常用函式式介面

`java.util.function` 套件提供的核心介面：

| 介面 | 方法 | 輸入 → 輸出 | 用途 |
|------|------|-------------|------|
| `Function<T, R>` | `apply(T)` | T → R | 轉換 |
| `Consumer<T>` | `accept(T)` | T → void | 消費（如列印） |
| `Supplier<T>` | `get()` | () → T | 產生（如工廠） |
| `Predicate<T>` | `test(T)` | T → boolean | 過濾條件 |
| `UnaryOperator<T>` | `apply(T)` | T → T | 同型轉換 |
| `BinaryOperator<T>` | `apply(T, T)` | (T, T) → T | 二元運算 |
| `BiFunction<T, U, R>` | `apply(T, U)` | (T, U) → R | 雙參數轉換 |

```java
// 組合
Predicate<String> notEmpty = s -> !s.isEmpty();
Predicate<String> startsWithA = s -> s.startsWith("A");
Predicate<String> combined = notEmpty.and(startsWithA);

Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;
Function<String, String> pipeline = trim.andThen(upper);
```

## 3、Stream API

### 3.1 建立 Stream

```java
// 從集合
List<String> list = List.of("a", "b", "c");
Stream<String> s1 = list.stream();

// 從陣列
Stream<int[]> s2 = Arrays.stream(new int[]{1, 2, 3}).boxed();

// 直接建立
Stream<String> s3 = Stream.of("x", "y", "z");

// 無限流
Stream<Integer> s4 = Stream.iterate(0, n -> n + 2);  // 0, 2, 4, ...
Stream<Double> s5 = Stream.generate(Math::random);
```

### 3.2 中間操作（Intermediate）

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Anna", "David");

names.stream()
    .filter(n -> n.startsWith("A"))       // 過濾：["Alice", "Anna"]
    .map(String::toUpperCase)              // 轉換：["ALICE", "ANNA"]
    .sorted()                              // 排序：["ALICE", "ANNA"]
    .distinct()                            // 去重
    .peek(System.out::println)             // 偵錯用（不影響流）
    .limit(10)                             // 取前 N 個
    .skip(1);                              // 跳過前 N 個
```

### 3.3 終端操作（Terminal）

```java
// 收集為 List
List<String> result = names.stream()
    .filter(n -> n.length() > 3)
    .toList();  // Java 16+，不可變 List

// forEach
names.stream().forEach(System.out::println);

// reduce
int sum = IntStream.rangeClosed(1, 100).reduce(0, Integer::sum);

// 判斷
boolean anyMatch = names.stream().anyMatch(n -> n.startsWith("A"));
boolean allMatch = names.stream().allMatch(n -> n.length() > 1);
Optional<String> first = names.stream().findFirst();

// 統計
long count = names.stream().count();
```

### 3.4 Collectors 收集器

```java
List<User> users = List.of(
    new User("Alice", 25, "Engineering"),
    new User("Bob", 30, "Engineering"),
    new User("Charlie", 28, "Marketing"),
    new User("Diana", 35, "Marketing")
);

// 轉 Map
Map<String, Integer> nameAge = users.stream()
    .collect(Collectors.toMap(User::name, User::age));

// 分組
Map<String, List<User>> byDept = users.stream()
    .collect(Collectors.groupingBy(User::department));

// 分組 + 統計
Map<String, Long> deptCount = users.stream()
    .collect(Collectors.groupingBy(User::department, Collectors.counting()));

// 分組 + 平均
Map<String, Double> deptAvgAge = users.stream()
    .collect(Collectors.groupingBy(User::department, Collectors.averagingInt(User::age)));

// 分區（true/false 兩組）
Map<Boolean, List<User>> partition = users.stream()
    .collect(Collectors.partitioningBy(u -> u.age() >= 30));

// 連接字串
String names = users.stream()
    .map(User::name)
    .collect(Collectors.joining(", ", "[", "]"));  // [Alice, Bob, Charlie, Diana]

record User(String name, int age, String department) {}
```

### 3.5 flatMap

展平巢狀結構：

```java
List<List<String>> nested = List.of(
    List.of("a", "b"),
    List.of("c", "d"),
    List.of("e")
);

List<String> flat = nested.stream()
    .flatMap(Collection::stream)
    .toList();  // ["a", "b", "c", "d", "e"]

// 實務範例：取出所有訂單的商品
List<OrderItem> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream())
    .toList();
```

## 4、平行流（Parallel Stream）

```java
long sum = IntStream.rangeClosed(1, 10_000_000)
    .parallel()
    .sum();
```

**注意事項**：
- 底層使用 ForkJoinPool.commonPool()
- **不適合** I/O 操作（會佔用共用執行緒池）
- **不適合**小資料集（分割/合併開銷大於收益）
- 確保操作是**無狀態**且**無副作用**的
- `forEachOrdered()` 可保證順序（但會降低並行效率）

**適合場景**：CPU 密集型的大量純計算。

## 5、常見陷阱

### 5.1 Stream 只能消費一次

```java
Stream<String> stream = List.of("a", "b").stream();
stream.forEach(System.out::println);
// stream.forEach(System.out::println);  // IllegalStateException!
```

### 5.2 修改外部狀態

```java
// 錯誤：Lambda 中修改外部集合
var results = new ArrayList<String>();
names.stream()
    .filter(n -> n.length() > 3)
    .forEach(results::add);  // 非執行緒安全，平行流會出問題

// 正確：用 collect
var results = names.stream()
    .filter(n -> n.length() > 3)
    .toList();
```

## 6、小結

| 概念 | 重點 |
|------|------|
| Lambda | 函式式介面的簡寫，搭配方法引用更簡潔 |
| 四大介面 | Function(轉換), Consumer(消費), Supplier(產生), Predicate(過濾) |
| Stream 操作 | 中間操作（惰性）→ 終端操作（觸發執行） |
| Collectors | groupingBy、toMap、joining 最常用 |
| 平行流 | 僅限 CPU 密集 + 大資料集 + 無副作用 |

> **延伸閱讀**：
> - [02 字串與集合框架](02%20字串與集合框架.md) — 集合框架基礎
> - [08 Optional 與現代錯誤處理](08%20Optional%20與現代錯誤處理.md) — Optional 與 Stream 的搭配
> - [04 並行程式設計基礎](04%20並行程式設計基礎.md) — ForkJoinPool 與執行緒池
