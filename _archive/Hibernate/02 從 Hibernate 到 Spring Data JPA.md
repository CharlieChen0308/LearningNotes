# 02 從 Hibernate 到 Spring Data JPA

> 本系列第 01 篇介紹了 Hibernate 的 XML 配置方式。本篇補充現代 Java 持久層的主流方案——Spring Data JPA，它底層仍使用 Hibernate 作為 JPA 實現，但大幅簡化了開發。

## Hibernate vs Spring Data JPA

| 項目 | 傳統 Hibernate（第 01 篇） | Spring Data JPA |
|------|--------------------------|-----------------|
| 配置方式 | hibernate.cfg.xml | application.yml |
| 對映方式 | hbm.xml 對映檔案 | JPA 註解（`@Entity`） |
| 資料存取 | 手寫 Session + HQL | 繼承 `JpaRepository` 介面 |
| 事務管理 | 手動 `session.beginTransaction()` | `@Transactional` 註解 |
| CRUD | 手寫每個方法 | 自動生成（零程式碼） |

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

## 小結

Spring Data JPA 讓 Hibernate 的使用從手寫 XML + Session 操作，進化到只需定義介面就能自動獲得完整的 CRUD 和查詢功能。底層仍然是 Hibernate，但開發效率大幅提升。方法名稱查詢和 `@Query` 覆蓋了絕大多數場景。
