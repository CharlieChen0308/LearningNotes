# 11 MyBatis 與 Spring Boot 整合

> **版本**: MyBatis 3.x / Spring Boot 3.x
> **來源**: `_archive/Mybatis/03 MyBatis 與 Spring Boot 整合及註解開發.md`

> 本系列前兩篇介紹了 MyBatis 的原始 XML 配置和逆向工程。本篇對照 MyBatis 3 官方文件，補充現代 Spring Boot 整合方式及註解開發。

## 傳統方式 vs Spring Boot 方式

| 項目 | 傳統方式（第 01 篇） | Spring Boot 方式 |
|------|---------------------|-----------------|
| 依賴管理 | 手動下載 jar | Maven / Gradle Starter |
| 配置方式 | mybatis-config.xml | application.yml |
| SqlSessionFactory | 手動建立 | 自動配置 |
| Mapper 註冊 | XML 配置 | `@MapperScan` 自動掃描 |

## 快速整合

### 1. 新增依賴

```xml
<!-- MyBatis Spring Boot Starter -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.3</version>
</dependency>

<!-- 資料庫驅動（以 PostgreSQL 為例）-->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

### 2. 配置資料來源

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
    password: secret

mybatis:
  # Mapper XML 檔案位置
  mapper-locations: classpath:mapper/*.xml
  # 實體類別名套件（XML 中可簡寫類名）
  type-aliases-package: com.example.entity
  configuration:
    # 開啟駝峰命名自動對映（user_name → userName）
    map-underscore-to-camel-case: true
    # 開啟日誌
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

### 3. 啟動類

```java
@SpringBootApplication
@MapperScan("com.example.mapper")  // 自動掃描 Mapper 介面
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

## 註解式開發（無需 XML）

MyBatis 3 支援直接在 Mapper 介面上用註解撰寫 SQL，適合簡單查詢：

### 基本 CRUD

```java
@Mapper
public interface UserMapper {

    @Select("SELECT * FROM users WHERE id = #{id}")
    User findById(Long id);

    @Select("SELECT * FROM users")
    List<User> findAll();

    @Insert("INSERT INTO users(name, email, age) VALUES(#{name}, #{email}, #{age})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    int insert(User user);

    @Update("UPDATE users SET name=#{name}, email=#{email}, age=#{age} WHERE id=#{id}")
    int update(User user);

    @Delete("DELETE FROM users WHERE id = #{id}")
    int deleteById(Long id);
}
```

### 結果對映（@Results）

當資料庫欄位與 Java 屬性名不一致時：

```java
@Results(id = "userResultMap", value = {
    @Result(property = "id", column = "user_id", id = true),
    @Result(property = "name", column = "user_name"),
    @Result(property = "email", column = "user_email"),
    @Result(property = "createTime", column = "create_time")
})
@Select("SELECT user_id, user_name, user_email, create_time FROM users WHERE user_id = #{id}")
User findById(Long id);

// 重用結果對映
@ResultMap("userResultMap")
@Select("SELECT user_id, user_name, user_email, create_time FROM users")
List<User> findAll();
```

> 如果已開啟 `map-underscore-to-camel-case: true`，大部分情況不需要手動定義結果對映。

### 關聯查詢

```java
@Results({
    @Result(property = "id", column = "id", id = true),
    @Result(property = "title", column = "title"),
    @Result(property = "author", column = "author_id",
            one = @One(select = "com.example.mapper.AuthorMapper.findById",
                       fetchType = FetchType.LAZY))
})
@Select("SELECT id, title, author_id FROM blog WHERE id = #{id}")
Blog findBlogWithAuthor(Long id);
```

## 動態 SQL（XML 方式）

對於複雜的動態查詢，XML 仍然比註解更清晰。MyBatis 提供了豐富的動態 SQL 標籤：

### `<if>` — 條件判斷

```xml
<select id="findUsers" resultType="User">
    SELECT * FROM users
    <where>
        <if test="name != null and name != ''">
            AND name LIKE CONCAT('%', #{name}, '%')
        </if>
        <if test="email != null">
            AND email = #{email}
        </if>
        <if test="minAge != null">
            AND age >= #{minAge}
        </if>
    </where>
    ORDER BY id DESC
</select>
```

`<where>` 標籤會自動處理第一個 `AND`/`OR` 前綴。

### `<foreach>` — 迴圈

```xml
<select id="findByIds" resultType="User">
    SELECT * FROM users
    WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

### `<choose>` — 多選一

```xml
<select id="findUser" resultType="User">
    SELECT * FROM users
    <where>
        <choose>
            <when test="id != null">
                id = #{id}
            </when>
            <when test="email != null">
                email = #{email}
            </when>
            <otherwise>
                name = #{name}
            </otherwise>
        </choose>
    </where>
</select>
```

### `<set>` — 動態更新

```xml
<update id="updateUser">
    UPDATE users
    <set>
        <if test="name != null">name = #{name},</if>
        <if test="email != null">email = #{email},</if>
        <if test="age != null">age = #{age},</if>
    </set>
    WHERE id = #{id}
</update>
```

## 註解式動態 SQL

MyBatis 3 的 `@Select` 等註解也支援動態 SQL（使用 `<script>` 標籤）：

```java
@Select("<script>" +
    "SELECT * FROM users" +
    "<where>" +
    "  <if test='name != null'>AND name LIKE CONCAT('%', #{name}, '%')</if>" +
    "  <if test='email != null'>AND email = #{email}</if>" +
    "</where>" +
    "</script>")
List<User> search(@Param("name") String name, @Param("email") String email);
```

> 動態 SQL 較複雜時，建議使用 XML 而非 `<script>` 註解。

## 分頁查詢

搭配 PageHelper 插件實現分頁：

```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>
```

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public PageInfo<User> findByPage(int pageNum, int pageSize) {
        PageHelper.startPage(pageNum, pageSize);
        List<User> users = userMapper.findAll();
        return new PageInfo<>(users);
    }
}
```

## 註解 vs XML 選擇建議

| 場景 | 建議方式 |
|------|---------|
| 簡單 CRUD | 註解（`@Select` 等） |
| 動態 SQL | XML（`<if>`, `<foreach>` 等） |
| 複雜多表關聯 | XML（`<resultMap>`, `<association>`） |
| 專案統一風格 | 選一種統一使用 |

## 小結

MyBatis 與 Spring Boot 的整合透過 Starter 實現了零配置啟動。註解式開發適合簡單場景，XML 的動態 SQL 功能則適合複雜查詢。搭配駝峰命名自動對映和 PageHelper 分頁，可以大幅減少樣板程式碼。

> **延伸閱讀**：
> - [16 MyBatis-Plus 快速開發](16%20MyBatis-Plus%20快速開發.md) — 基於 MyBatis 的增強框架，進一步簡化 CRUD 開發
> - [10 Spring Data JPA](10%20Spring%20Data%20JPA.md) — 另一種 ORM 方案，與 MyBatis 的設計理念對比
