# 01 Mybatis 的配置和使用

> **版本提示**：本篇基於 MyBatis 3.2.7，使用手動配置方式。注意 MySQL 8+ 的驅動類已改為 `com.mysql.cj.jdbc.Driver`。現代開發推薦使用 Spring Boot Starter 整合，請參考 [03 MyBatis 與 Spring Boot 整合及註解開發](03%20MyBatis%20%E8%88%87%20Spring%20Boot%20%E6%95%B4%E5%90%88%E5%8F%8A%E8%A8%BB%E8%A7%A3%E9%96%8B%E7%99%BC.md)。

## 一、Mybatis 是什麼

![][1]

MyBatis 是一個支援普通SQL查詢、儲存過程和高階對映的優秀持久層框架。MyBatis 消除了幾乎所有的 JDBC 程式碼和引數的手工設定以及對結果集的檢索封裝。MyBatis可以使用簡單的XML或註解用於配置和原始對映，將介面和Java的POJO（Plain Old Java Objects，普通的Java物件）對映成資料庫中的記錄。

## 二、Mybatis 的使用

### 1、導包

Mybatis 需要以下的 jar 包：

```
mybatis.jar
mysql-connector-java.jar
```

使用 Maven 構建的專案，需要在 pom.xml 中新增如下依賴：

```xml
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.2.7</version>
        </dependency>

        <!--資料庫驅動-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.6</version>
        </dependency>
```

### 2、建表

建立資料庫和表，針對MySQL資料庫，SQL指令碼如下：

```sql
create database mybatis_demo;
use mybatis_demo;
CREATE TABLE users(id INT PRIMARY KEY AUTO_INCREMENT, NAME VARCHAR(20), age INT);
insert into users values(null,'郭靖', 27);
insert into users values(null,'黃蓉', 17);
```

![][2]

![][3]

到此，建表工作已經完成。

### 3、建立表所對應的實體類

如下圖所示：

![][4]

User 類的程式碼如下：

```java
package com.nnngu.domain;

public class User {
    // 實體類的屬性和表的欄位名稱一一對應
    private int id;
    private String name;
    private int age;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User [id=" + id + ", name=" + name + ", age=" + age + "]";
    }
}

```

### 4、建立用來操作 users 表的 sql 對映檔案 userMapper.xml

![][5]

`userMapper.xml` 的程式碼如下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!-- 為這個mapper指定一個唯一的namespace，namespace的值習慣上設定成包名+sql對映檔名，這樣就能夠保證namespace的值是唯一的 -->
<mapper namespace="com.nnngu.mapping.userMapper">

    <!-- 在select標籤中編寫查詢的SQL語句， select標籤的id屬性為getUser，id屬性值必須是唯一的，不能夠重複
    使用parameterType屬性指明查詢時使用的引數型別，resultType屬性指明查詢返回的結果集型別
    resultType="com.nnngu.domain.User"就表示將查詢結果封裝成一個User類的物件返回
    User類就是users表所對應的實體類
    -->
    <select id="getUser" parameterType="int"
            resultType="com.nnngu.domain.User">
        select * from users where id=#{id}
    </select>
</mapper>
```

### 5、建立 Mybatis 的配置檔案

建立配置檔案 `mybatis_config.xml` ，在下圖所示的位置

![][6]

`mybatis_config.xml`的程式碼如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <!-- 配置資料庫連線資訊 -->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/mybatis_demo"/>
                <property name="username" value="root"/>
                <property name="password" value="1"/>
            </dataSource>
        </environment>
    </environments>

    <!-- 註冊userMapper.xml -->
    <mappers>
        <mapper resource="com.nnngu.mapping/userMapper.xml"/>
    </mappers>

</configuration>
```

### 6、測試

建立一個Test1類，編寫如下的測試程式碼：

```java
package com.nnngu.test;

import com.nnngu.domain.User;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;
import java.io.InputStream;

public class Test1 {
    public static void main(String[] args) throws IOException {
        // mybatis 的配置檔案
        String resource = "mybatis_config.xml";

        // 使用類載入器載入mybatis的配置檔案（它也載入關聯的對映檔案）
        InputStream is = Test1.class.getClassLoader().getResourceAsStream(resource);

        // 構建sqlSession的工廠
        SqlSessionFactory sessionFactory = new SqlSessionFactoryBuilder().build(is);

        // 建立能執行對映檔案中sql的sqlSession
        SqlSession session = sessionFactory.openSession();

        /**
         * 對映sql的標識字串，
         * com.nnngu.mapping.userMapper是userMapper.xml檔案中mapper標籤的namespace屬性的值，
         * getUser是select標籤的id屬性值，透過select標籤的id屬性值就可以找到要執行的SQL
         */
        String statement = "com.nnngu.mapping.userMapper.getUser"; // 對映sql的標識字串

        // 執行查詢返回一個user物件的sql
        User user = session.selectOne(statement, 1);
        System.out.println(user);
    }
}

```

測試結果：

![][7]















---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/Mybatis/01%20Mybatis%20%E7%9A%84%E9%85%8D%E7%BD%AE%E5%92%8C%E4%BD%BF%E7%94%A8.md](https://github.com/nnngu/LearningNotes/blob/master/Mybatis/01%20Mybatis%20%E7%9A%84%E9%85%8D%E7%BD%AE%E5%92%8C%E4%BD%BF%E7%94%A8.md)


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/22/1519295758420.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/22/1519296834079.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/22/1519296882222.jpg
  [4]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/22/1519297821232.jpg
  [5]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/22/1519298235672.jpg
  [6]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/22/1519297480387.jpg
  [7]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/22/1519299368518.jpg