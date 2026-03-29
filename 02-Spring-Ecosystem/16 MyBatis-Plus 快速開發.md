# 16 MyBatis-Plus 快速開發

> **版本**：MyBatis-Plus 3.5.x / Spring Boot 3.x / Java 17+

---

## 1、MyBatis-Plus 簡介

MyBatis-Plus（簡稱 MP）是 MyBatis 的增強工具，官方口號是「只做增強不做改變」。它在 MyBatis 的基礎上，提供了大量開箱即用的功能，讓開發者不需要撰寫重複的 CRUD SQL，就能完成大部分的資料存取操作。

核心特性：

- **CRUD 自動生成**：繼承 `BaseMapper` 即可獲得完整的增刪改查方法，無需手寫 XML
- **條件構造器（Wrapper）**：以 Lambda 表達式組合查詢條件，型別安全、重構友好
- **內建分頁外掛**：透過攔截器自動改寫 SQL，支援多種資料庫方言
- **程式碼產生器**：根據資料表結構一鍵產生 Entity、Mapper、Service、Controller
- **多種外掛**：樂觀鎖、邏輯刪除、自動填充、多租戶、防全表更新等

與原生 MyBatis 的關係：MP 不會取代 MyBatis，而是在其上層提供便利的封裝。當你需要複雜的 SQL 時，仍然可以使用 MyBatis 的 XML Mapper 或 `@Select` 註解。

---

## 2、快速開始

### 2.1 pom.xml 依賴

Spring Boot 3 需要使用專用的 starter，注意 artifact 名稱帶有 `boot3`：

```xml
<dependencies>
    <!-- MyBatis-Plus Spring Boot 3 Starter -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
        <version>3.5.7</version>
    </dependency>

    <!-- PostgreSQL 驅動 -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

`application.yml` 基本配置：

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: postgres
    password: secret

mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl  # 開發時印出 SQL
  global-config:
    db-config:
      id-type: assign_id        # 預設主鍵策略：雪花算法
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

### 2.2 Entity

```java
package com.example.entity;

import com.baomidou.mybatisplus.annotation.*;
import java.time.LocalDateTime;

@TableName("user")
public class User {

    @TableId(type = IdType.ASSIGN_ID)   // 雪花算法產生 ID
    private Long id;

    private String name;

    private Integer age;

    private String email;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableLogic
    private Integer deleted;

    // 建構子
    public User() {}

    public User(String name, Integer age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }

    // getter / setter 省略（實務上建議搭配 Lombok @Data）

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Integer getAge() { return age; }
    public void setAge(Integer age) { this.age = age; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public LocalDateTime getCreateTime() { return createTime; }
    public void setCreateTime(LocalDateTime createTime) { this.createTime = createTime; }
    public LocalDateTime getUpdateTime() { return updateTime; }
    public void setUpdateTime(LocalDateTime updateTime) { this.updateTime = updateTime; }
    public Integer getDeleted() { return deleted; }
    public void setDeleted(Integer deleted) { this.deleted = deleted; }
}
```

常用註解說明：

| 註解 | 作用 |
|------|------|
| `@TableName` | 指定對應的資料表名稱 |
| `@TableId` | 標記主鍵欄位，`IdType.ASSIGN_ID` 為雪花算法 |
| `@TableField` | 欄位配置，可設定自動填充策略、是否為資料庫欄位等 |
| `@TableLogic` | 標記邏輯刪除欄位 |

### 2.3 Mapper

只需繼承 `BaseMapper<T>`，即可獲得完整的 CRUD 方法：

```java
package com.example.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.example.entity.User;
import org.apache.ibatis.annotations.Mapper;

@Mapper
public interface UserMapper extends BaseMapper<User> {
    // 不需要寫任何方法，BaseMapper 已提供完整的 CRUD
}
```

啟動類別記得加上 `@MapperScan`：

```java
package com.example;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@MapperScan("com.example.mapper")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## 3、BaseMapper CRUD

`BaseMapper<T>` 提供的常用方法一覽：

| 方法 | 說明 |
|------|------|
| `insert(T entity)` | 新增一筆記錄 |
| `deleteById(Serializable id)` | 根據 ID 刪除 |
| `deleteBatchIds(Collection<?> ids)` | 批次刪除 |
| `updateById(T entity)` | 根據 ID 更新（null 欄位不更新） |
| `selectById(Serializable id)` | 根據 ID 查詢 |
| `selectBatchIds(Collection<?> ids)` | 批次查詢 |
| `selectList(Wrapper<T> wrapper)` | 條件查詢列表 |
| `selectPage(Page<T> page, Wrapper<T> wrapper)` | 分頁查詢 |
| `selectCount(Wrapper<T> wrapper)` | 條件計數 |

完整範例：

```java
package com.example.test;

import com.example.entity.User;
import com.example.mapper.UserMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;

@SpringBootTest
class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    void testInsert() {
        var user = new User("張三", 25, "zhang@example.com");
        int rows = userMapper.insert(user);
        // insert 後 user.getId() 會自動回填雪花 ID
        System.out.println("新增筆數: " + rows + ", ID: " + user.getId());
    }

    @Test
    void testSelectById() {
        User user = userMapper.selectById(1L);
        System.out.println(user.getName());
    }

    @Test
    void testUpdateById() {
        var user = new User();
        user.setId(1L);
        user.setAge(30);
        userMapper.updateById(user);  // 只更新 age，其餘欄位不動
    }

    @Test
    void testDeleteById() {
        userMapper.deleteById(1L);  // 若有 @TableLogic，會變成 UPDATE SET deleted=1
    }

    @Test
    void testSelectList() {
        List<User> users = userMapper.selectList(null);  // null 表示無條件，查詢全部
        users.forEach(u -> System.out.println(u.getName()));
    }
}
```

---

## 4、IService / ServiceImpl

MP 在 Mapper 之上還提供了 Service 層封裝，包含批次操作與更豐富的查詢方法。

定義 Service 介面：

```java
package com.example.service;

import com.baomidou.mybatisplus.extension.service.IService;
import com.example.entity.User;

public interface UserService extends IService<User> {
    // 可在此擴充自訂的業務方法
}
```

實作 Service：

```java
package com.example.service.impl;

import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.example.entity.User;
import com.example.mapper.UserMapper;
import com.example.service.UserService;
import org.springframework.stereotype.Service;

@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    // ServiceImpl 已實作 IService 的所有方法
}
```

IService 提供的實用方法：

```java
package com.example.test;

import com.example.entity.User;
import com.example.service.UserService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;

@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;

    @Test
    void testSave() {
        var user = new User("李四", 28, "li@example.com");
        userService.save(user);
    }

    @Test
    void testSaveBatch() {
        List<User> users = List.of(
            new User("王五", 30, "wang@example.com"),
            new User("趙六", 22, "zhao@example.com")
        );
        userService.saveBatch(users);  // 批次新增，預設每次 1000 筆
    }

    @Test
    void testGetById() {
        User user = userService.getById(1L);
        System.out.println(user.getName());
    }

    @Test
    void testList() {
        List<User> all = userService.list();
        System.out.println("總筆數: " + all.size());
    }

    @Test
    void testLambdaQuery() {
        // 鏈式 Lambda 查詢：找出年齡 >= 18 且名字包含「張」的使用者
        List<User> users = userService.lambdaQuery()
                .ge(User::getAge, 18)
                .like(User::getName, "張")
                .orderByDesc(User::getCreateTime)
                .list();

        users.forEach(u -> System.out.println(u.getName() + " - " + u.getAge()));
    }

    @Test
    void testLambdaUpdate() {
        // 鏈式 Lambda 更新：將年齡 < 18 的使用者 email 設為 null
        userService.lambdaUpdate()
                .lt(User::getAge, 18)
                .set(User::getEmail, null)
                .update();
    }
}
```

---

## 5、條件構造器（Wrapper）

條件構造器是 MP 最強大的功能之一，讓你用程式碼動態組合查詢條件，取代手寫 SQL 的 WHERE 子句。

### 5.1 LambdaQueryWrapper（推薦）

使用方法引用（Method Reference），型別安全且支援重構：

```java
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;

// 基本查詢
var wrapper = new LambdaQueryWrapper<User>()
        .eq(User::getAge, 25)                    // age = 25
        .like(User::getName, "張")               // name LIKE '%張%'
        .between(User::getAge, 20, 30)           // age BETWEEN 20 AND 30
        .isNotNull(User::getEmail)               // email IS NOT NULL
        .orderByDesc(User::getCreateTime);       // ORDER BY create_time DESC

List<User> users = userMapper.selectList(wrapper);

// 條件判斷：僅當參數不為空時才加入條件
String nameParam = null;
Integer ageParam = 25;

var conditionalWrapper = new LambdaQueryWrapper<User>()
        .like(nameParam != null, User::getName, nameParam)   // nameParam 為 null，此條件不生效
        .eq(ageParam != null, User::getAge, ageParam);       // ageParam 不為 null，條件生效

// 查詢指定欄位（SELECT name, age FROM user ...）
var selectWrapper = new LambdaQueryWrapper<User>()
        .select(User::getName, User::getAge)
        .gt(User::getAge, 20);
```

### 5.2 LambdaUpdateWrapper

用於組合更新條件與設定更新值：

```java
import com.baomidou.mybatisplus.core.conditions.update.LambdaUpdateWrapper;

var updateWrapper = new LambdaUpdateWrapper<User>()
        .eq(User::getName, "張三")
        .set(User::getAge, 26)
        .set(User::getEmail, "zhang_new@example.com");

userMapper.update(null, updateWrapper);
// 產生 SQL: UPDATE user SET age=26, email='zhang_new@example.com' WHERE name='張三'
```

### 5.3 鏈式查詢

透過 `IService` 提供的鏈式 API，寫法更簡潔：

```java
// 鏈式查詢
List<User> adults = userService.lambdaQuery()
        .ge(User::getAge, 18)
        .like(User::getName, "王")
        .list();

// 鏈式查詢單筆
User one = userService.lambdaQuery()
        .eq(User::getId, 1L)
        .one();

// 鏈式計數
long count = userService.lambdaQuery()
        .gt(User::getAge, 30)
        .count();

// 鏈式更新
userService.lambdaUpdate()
        .eq(User::getId, 1L)
        .set(User::getAge, 35)
        .update();
```

---

## 6、分頁

MP 的分頁功能透過攔截器實現，會自動改寫 SQL 加入 `LIMIT` 和 `OFFSET`。

### 6.1 分頁外掛配置

```java
package com.example.config;

import com.baomidou.mybatisplus.annotation.DbType;
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        var interceptor = new MybatisPlusInterceptor();
        // 指定資料庫類型，以產生正確的分頁 SQL
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.POSTGRE_SQL));
        return interceptor;
    }
}
```

### 6.2 使用

```java
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;

// Mapper 層分頁
var wrapper = new LambdaQueryWrapper<User>()
        .ge(User::getAge, 18)
        .orderByDesc(User::getCreateTime);

Page<User> page = userMapper.selectPage(new Page<>(1, 10), wrapper);
// 第 1 頁，每頁 10 筆

System.out.println("總筆數: " + page.getTotal());
System.out.println("總頁數: " + page.getPages());
System.out.println("當前頁資料: " + page.getRecords());

// Service 層分頁
Page<User> servicePage = userService.page(
        new Page<>(2, 10),     // 第 2 頁
        wrapper
);

// 鏈式分頁
Page<User> chainPage = userService.lambdaQuery()
        .ge(User::getAge, 18)
        .page(new Page<>(1, 10));
```

---

## 7、自動填充

透過實作 `MetaObjectHandler`，可在新增或更新時自動填入 `createTime`、`updateTime` 等欄位。

```java
package com.example.config;

import com.baomidou.mybatisplus.core.handlers.MetaObjectHandler;
import org.apache.ibatis.reflection.MetaObject;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        // 新增時填入 createTime 和 updateTime
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        // 更新時填入 updateTime
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

搭配 Entity 上的 `@TableField(fill = FieldFill.INSERT)` 與 `@TableField(fill = FieldFill.INSERT_UPDATE)` 使用，開發者就不需要在每次 insert 或 update 時手動設定時間欄位。

---

## 8、邏輯刪除

邏輯刪除不會真正從資料庫移除記錄，而是將 `deleted` 欄位標記為已刪除。MP 會自動在所有查詢中加入 `WHERE deleted = 0`。

Entity 標記：

```java
@TableLogic
private Integer deleted;   // 0: 未刪除, 1: 已刪除
```

`application.yml` 配置：

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

配置完成後，行為變化如下：

| 呼叫方法 | 實際執行的 SQL |
|----------|---------------|
| `deleteById(1L)` | `UPDATE user SET deleted=1 WHERE id=1 AND deleted=0` |
| `selectList(null)` | `SELECT * FROM user WHERE deleted=0` |
| `updateById(user)` | `UPDATE user SET ... WHERE id=1 AND deleted=0` |

若需要查詢已刪除的資料，必須透過自訂 SQL 繞過 MP 的自動過濾。

---

## 9、程式碼產生器（簡述）

MP 提供程式碼產生器，可根據資料表結構自動產生 Entity、Mapper、Service、Controller 等檔案。

新增依賴：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.5.7</version>
</dependency>
<dependency>
    <groupId>org.apache.velocity</groupId>
    <artifactId>velocity-engine-core</artifactId>
    <version>2.3</version>
</dependency>
```

產生器範例：

```java
import com.baomidou.mybatisplus.generator.FastAutoGenerator;
import com.baomidou.mybatisplus.generator.config.OutputFile;
import com.baomidou.mybatisplus.generator.engine.VelocityTemplateEngine;

import java.util.Map;

public class CodeGenerator {

    public static void main(String[] args) {
        FastAutoGenerator.create(
                        "jdbc:postgresql://localhost:5432/mydb",
                        "postgres",
                        "secret"
                )
                .globalConfig(builder -> builder
                        .author("developer")
                        .outputDir("src/main/java")
                )
                .packageConfig(builder -> builder
                        .parent("com.example")
                        .entity("entity")
                        .mapper("mapper")
                        .service("service")
                        .controller("controller")
                        .pathInfo(Map.of(
                                OutputFile.xml, "src/main/resources/mapper"
                        ))
                )
                .strategyConfig(builder -> builder
                        .addInclude("user", "order", "product")   // 指定要產生的資料表
                        .entityBuilder()
                        .enableLombok()
                        .logicDeleteColumnName("deleted")
                        .build()
                )
                .templateEngine(new VelocityTemplateEngine())
                .execute();
    }
}
```

執行後會自動產生完整的四層程式碼，大幅減少重複勞動。

---

## 10、多租戶外掛（簡述）

多租戶（Multi-Tenant）是 SaaS 應用常見的架構，MP 提供 `TenantLineInnerInterceptor`，可自動在每條 SQL 加入 `tenant_id` 條件。

```java
package com.example.config;

import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.handler.TenantLineHandler;
import com.baomidou.mybatisplus.extension.plugins.inner.TenantLineInnerInterceptor;
import net.sf.jsqlparser.expression.Expression;
import net.sf.jsqlparser.expression.LongValue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
public class TenantConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        var interceptor = new MybatisPlusInterceptor();

        interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(new TenantLineHandler() {
            @Override
            public Expression getTenantId() {
                // 從 SecurityContext 或 ThreadLocal 取得當前租戶 ID
                return new LongValue(TenantContextHolder.getCurrentTenantId());
            }

            @Override
            public String getTenantIdColumn() {
                return "tenant_id";
            }

            @Override
            public boolean ignoreTable(String tableName) {
                // 不需要租戶過濾的表
                List<String> ignoreTables = List.of("sys_config", "sys_dict");
                return ignoreTables.contains(tableName);
            }
        }));

        return interceptor;
    }
}
```

配置完成後，MP 會自動處理：

- **SELECT**：加入 `WHERE tenant_id = ?`
- **INSERT**：自動補上 `tenant_id` 欄位
- **UPDATE / DELETE**：在 WHERE 條件中加入 `tenant_id = ?`

開發者不需要在業務程式碼中關心租戶隔離，全由外掛統一處理。

---

## 11、小結

MyBatis-Plus 透過「增強不改變」的理念，在保留 MyBatis 靈活性的同時，大幅降低了 CRUD 的開發成本。核心知識點回顧：

| 功能 | 關鍵類別 / 註解 |
|------|----------------|
| 基礎 CRUD | `BaseMapper<T>` |
| Service 封裝 | `IService<T>` / `ServiceImpl` |
| 條件查詢 | `LambdaQueryWrapper` / `LambdaUpdateWrapper` |
| 分頁 | `PaginationInnerInterceptor` / `Page<T>` |
| 自動填充 | `MetaObjectHandler` / `@TableField(fill = ...)` |
| 邏輯刪除 | `@TableLogic` |
| 程式碼產生 | `FastAutoGenerator` |
| 多租戶 | `TenantLineInnerInterceptor` |

實務建議：

1. **優先使用 LambdaWrapper**：避免硬編碼欄位名稱字串，重構時不會遺漏
2. **善用條件判斷參數**：`eq(condition, column, value)` 中的 `condition` 可避免大量 if-else
3. **複雜 SQL 回歸 MyBatis**：MP 不是萬能的，JOIN 多表、子查詢等場景仍建議用 XML Mapper
4. **注意外掛順序**：多個 InnerInterceptor 的添加順序會影響 SQL 改寫結果，建議順序為 Tenant > Pagination > OptimisticLocker

延伸閱讀：

- [11 MyBatis 與 Spring Boot 整合](11%20MyBatis%20與%20Spring%20Boot%20整合.md)
- [13 Spring 事務管理](13%20Spring%20事務管理.md)

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
