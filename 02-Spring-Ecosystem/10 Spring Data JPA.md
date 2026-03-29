# 10 Spring Data JPA

> **版本**: Spring Boot 3.x / Spring Data JPA 3.x
> **來源**: `_archive/Hibernate/02 從 Hibernate 到 Spring Data JPA.md`

> 本系列第 01 篇介紹了 Hibernate 的 XML 配置方式。本篇補充現代 Java 持久層的主流方案——Spring Data JPA，它底層仍使用 Hibernate 作為 JPA 實現，但大幅簡化了開發。

## Hibernate vs Spring Data JPA

| 項目 | 傳統 Hibernate（第 01 篇） | Spring Data JPA |
|------|--------------------------|-----------------|
| 配置方式 | hibernate.cfg.xml | application.yml |
| 對映方式 | hbm.xml 對映檔案 | JPA 註解（`@Entity`） |
| 資料存取 | 手寫 Session + HQL | 繼承 `JpaRepository` 介面 |
| 事務管理 | 手動 `session.beginTransaction()` | `@Transactional` 註解 |
| CRUD | 手寫每個方法 | 自動生成（零程式碼） |

## JPA vs MyBatis 取捨指南

| 面向 | Spring Data JPA | MyBatis |
|------|----------------|---------|
| 適合場景 | CRUD 為主的業務系統 | 複雜 SQL、報表查詢、DBA 主導的團隊 |
| 開發效率 | 高（自動生成 CRUD） | 中（需手寫 SQL，但可控性強） |
| SQL 控制力 | 低（Hibernate 產生 SQL） | 高（完全手寫，可精細調優） |
| 學習曲線 | 中高（需理解 Lazy Loading、快取、Dirty Checking） | 低（會寫 SQL 即可上手） |
| 跨資料庫 | 佳（JPQL 抽象化） | 差（SQL 可能綁定特定資料庫語法） |

**實務建議**：

- **CRUD 為主** → 選 JPA，開發快、程式碼少
- **複雜 SQL / 報表為主** → 選 MyBatis，SQL 可控性高
- **混合型專案** → 兩者可並存，簡單 CRUD 用 JPA，複雜查詢用 MyBatis

### 輕量替代：Spring Data JDBC

若不需要 JPA 的進階功能（Lazy Loading、Dirty Checking、二級快取），可考慮 **Spring Data JDBC**：

- **無隱式行為**：沒有 Lazy Loading、沒有 Dirty Checking，所見即所得
- **更簡單的對映模型**：Aggregate Root 概念清晰，不需處理 `@ManyToOne` 等複雜關聯
- **效能可預測**：每次操作都是明確的 SQL，不會有意外的額外查詢
- 適合微服務、領域模型簡單的場景

## 快速開始

### 1. 新增依賴

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

### 2. 配置

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
    password: secret
  jpa:
    hibernate:
      ddl-auto: update    # 自動更新表結構（開發用）
    show-sql: true         # 顯示 SQL
    properties:
      hibernate:
        format_sql: true   # 格式化 SQL
```

### 3. 定義實體

**傳統 Hibernate 方式（XML 對映）：**

```xml
<hibernate-mapping>
    <class name="com.example.User" table="users">
        <id name="id" column="id"><generator class="native"/></id>
        <property name="name" column="name"/>
        <property name="email" column="email"/>
    </class>
</hibernate-mapping>
```

**Spring Data JPA 方式（註解對映）：**

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 50)
    private String name;

    @Column(unique = true)
    private String email;

    @Column(name = "create_time")
    private LocalDateTime createTime;

    // 建構子、getter/setter
    protected User() {}  // JPA 需要無參建構子

    public User(String name, String email) {
        this.name = name;
        this.email = email;
        this.createTime = LocalDateTime.now();
    }
}
```

### 4. 定義 Repository

**傳統 Hibernate 方式：**

```java
public class UserDao {
    public User findById(Long id) {
        Session session = sessionFactory.openSession();
        try {
            return session.get(User.class, id);
        } finally {
            session.close();
        }
    }
    // 每個查詢都要手寫...
}
```

**Spring Data JPA 方式：**

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // 什麼都不用寫，自動擁有以下方法：
    // findById(Long id)
    // findAll()
    // save(User user)
    // deleteById(Long id)
    // count()
    // existsById(Long id)
    // ...
}
```

## 方法名稱自動查詢

Spring Data JPA 可以根據方法名稱自動產生查詢 SQL：

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // SELECT * FROM users WHERE email = ?
    Optional<User> findByEmail(String email);

    // SELECT * FROM users WHERE name LIKE ?
    List<User> findByNameContaining(String keyword);

    // SELECT * FROM users WHERE age > ? ORDER BY name ASC
    List<User> findByAgeGreaterThanOrderByNameAsc(int age);

    // SELECT * FROM users WHERE name = ? AND email = ?
    Optional<User> findByNameAndEmail(String name, String email);

    // SELECT COUNT(*) FROM users WHERE age > ?
    long countByAgeGreaterThan(int age);

    // DELETE FROM users WHERE email = ?
    void deleteByEmail(String email);

    // SELECT * FROM users WHERE age BETWEEN ? AND ?
    List<User> findByAgeBetween(int min, int max);
}
```

### 方法名稱關鍵字

| 關鍵字 | SQL |
|--------|-----|
| `findBy` | `WHERE` |
| `And` | `AND` |
| `Or` | `OR` |
| `Containing` | `LIKE '%?%'` |
| `StartingWith` | `LIKE '?%'` |
| `GreaterThan` | `> ?` |
| `LessThan` | `< ?` |
| `Between` | `BETWEEN ? AND ?` |
| `IsNull` | `IS NULL` |
| `OrderBy...Asc/Desc` | `ORDER BY ... ASC/DESC` |
| `countBy` | `SELECT COUNT(*)` |
| `existsBy` | `SELECT CASE WHEN COUNT(*) > 0` |

## 自訂查詢（@Query）

當方法名稱無法表達的複雜查詢：

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // JPQL（面向物件的查詢語言）
    @Query("SELECT u FROM User u WHERE u.email LIKE %:keyword% OR u.name LIKE %:keyword%")
    List<User> search(@Param("keyword") String keyword);

    // 原生 SQL
    @Query(value = "SELECT * FROM users WHERE age > :age LIMIT :limit",
           nativeQuery = true)
    List<User> findTopByAge(@Param("age") int age, @Param("limit") int limit);

    // 更新操作
    @Modifying
    @Query("UPDATE User u SET u.email = :email WHERE u.id = :id")
    int updateEmail(@Param("id") Long id, @Param("email") String email);
}
```

## 分頁與排序

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public Page<User> findByPage(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createTime").descending());
        return userRepository.findAll(pageable);
    }

    public Page<User> searchByPage(String keyword, int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return userRepository.findByNameContaining(keyword, pageable);
    }
}
```

Repository 方法加上 `Pageable` 參數即可：

```java
Page<User> findByNameContaining(String keyword, Pageable pageable);
```

## 關聯對映

```java
@Entity
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // 多對一
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    // 一對多
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();
}
```

## 生產環境注意事項

### N+1 查詢問題

當載入一個實體的關聯集合時，JPA 預設會對每一筆關聯資料發送獨立的 SQL 查詢，導致 1 + N 次查詢。解法：

```java
// 方法一：@EntityGraph（宣告式）
@EntityGraph(attributePaths = {"items"})
Optional<Order> findById(Long id);

// 方法二：JOIN FETCH（JPQL）
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);
```

### ddl-auto 禁止用於正式環境

```yaml
spring.jpa.hibernate.ddl-auto: update   # ⚠️ 僅限開發環境
```

正式環境 **必須** 設為 `none` 或 `validate`，並使用資料庫遷移工具（Flyway 或 Liquibase）管理 Schema 變更，確保變更可追蹤、可回滾。

### Lazy Loading 與 Transaction 陷阱

`FetchType.LAZY` 的關聯欄位只能在 Transaction 範圍內存取。若在 Controller 層直接存取 Lazy 屬性，會拋出 `LazyInitializationException`。解法：

- 在 Service 層（`@Transactional` 範圍內）完成所有資料存取，再轉為 DTO 回傳
- 或使用 `@EntityGraph` / `JOIN FETCH` 提前載入所需關聯

### 連線池配置

Spring Boot 預設使用 HikariCP，生產環境建議明確配置：

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # 依據資料庫最大連線數與應用實例數計算
      minimum-idle: 5
      idle-timeout: 300000         # 5 分鐘
      connection-timeout: 30000    # 30 秒
```

## 小結

Spring Data JPA 讓 Hibernate 的使用從手寫 XML + Session 操作，進化到只需定義介面就能自動獲得完整的 CRUD 和查詢功能。底層仍然是 Hibernate，但開發效率大幅提升。方法名稱查詢和 `@Query` 覆蓋了絕大多數場景。選擇 JPA 前，應根據專案的 SQL 複雜度與團隊熟悉度，評估是否搭配 MyBatis 或改用更輕量的 Spring Data JDBC。

## 延伸閱讀

- [11 MyBatis 與 Spring Boot 整合](11%20MyBatis%20與%20Spring%20Boot%20整合.md)——適合複雜 SQL 場景的替代 / 互補方案
- [13 Spring 事務管理](13%20Spring%20事務管理.md)——`@Transactional` 的傳播行為、隔離級別與常見陷阱
- [索引原理與 SQL 優化](../05-Database/02%20索引原理與%20SQL%20優化.md)——理解查詢效能調優的基礎

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
