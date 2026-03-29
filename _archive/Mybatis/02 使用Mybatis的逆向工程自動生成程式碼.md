# 02 使用Mybatis的逆向工程自動生成程式碼

> **版本提示**：本篇使用的 MySQL 驅動類 `com.mysql.jdbc.Driver` 已過時，MySQL 8+ 請改用 `com.mysql.cj.jdbc.Driver`。現代 MyBatis 開發方式請參考 [03 MyBatis 與 Spring Boot 整合及註解開發](03%20MyBatis%20%E8%88%87%20Spring%20Boot%20%E6%95%B4%E5%90%88%E5%8F%8A%E8%A8%BB%E8%A7%A3%E9%96%8B%E7%99%BC.md)。

## 1、逆向工程的作用

Mybatis 官方提供了逆向工程，可以針對資料庫表自動生成Mybatis執行所需要的程式碼（包括mapper.xml、Mapper.java、pojo）。

## 2、逆向工程的使用方法

逆向工程需要的jar包如下圖所示：

![][1]

也可以直接下載我Github上面的原始碼（[https://github.com/nnngu/generatorSqlmapCustom](https://github.com/nnngu/generatorSqlmapCustom) ），在 lib 目錄下已經新增了需要的 jar 包。

下載下來的專案目錄如下圖：

![][2]

從上圖中看，①是依賴的jar包。②是配置檔案。③是要執行的Java程式碼，執行它即可生成我們需要的程式碼。

### 2-1、先把配置檔案寫好

`generatorConfig.xml`的程式碼如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <context id="testTables" targetRuntime="MyBatis3">
        <commentGenerator>
            <!-- 是否去除自動生成的註釋 true：是 ： false:否 -->
            <property name="suppressAllComments" value="true" />
        </commentGenerator>
        <!--資料庫連線的資訊：驅動類、連線地址、使用者名稱、密碼 -->
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/taotao_0303" userId="root"
                        password="1">
        </jdbcConnection>
        <!-- 預設false，把JDBC DECIMAL 和 NUMERIC 型別解析為 Integer，為 true時把JDBC DECIMAL 和
            NUMERIC 型別解析為java.math.BigDecimal -->
        <javaTypeResolver>
            <property name="forceBigDecimals" value="false" />
        </javaTypeResolver>

        <!-- targetProject:生成PO類的位置 -->
        <javaModelGenerator targetPackage="com.taotao.pojo"
                            targetProject="./src">
            <!-- enableSubPackages:是否讓schema作為包的字尾 -->
            <property name="enableSubPackages" value="false" />
            <!-- 從資料庫返回的值被清理前後的空格 -->
            <property name="trimStrings" value="true" />
        </javaModelGenerator>
        <!-- targetProject:mapper對映檔案生成的位置 -->
        <sqlMapGenerator targetPackage="com.taotao.mapper"
                         targetProject="./src">
            <!-- enableSubPackages:是否讓schema作為包的字尾 -->
            <property name="enableSubPackages" value="false" />
        </sqlMapGenerator>
        <!-- targetPackage：mapper介面生成的位置 -->
        <javaClientGenerator type="XMLMAPPER"
                             targetPackage="com.taotao.mapper"
                             targetProject="./src">
            <!-- enableSubPackages:是否讓schema作為包的字尾 -->
            <property name="enableSubPackages" value="false" />
        </javaClientGenerator>
        <!-- 指定資料庫表 -->
        <table schema="" tableName="tb_content"></table>
        <table schema="" tableName="tb_content_category"></table>
        <table schema="" tableName="tb_item"></table>
        <table schema="" tableName="tb_item_cat"></table>
        <table schema="" tableName="tb_item_desc"></table>
        <table schema="" tableName="tb_item_param"></table>
        <table schema="" tableName="tb_item_param_item"></table>
        <table schema="" tableName="tb_order"></table>
        <table schema="" tableName="tb_order_item"></table>
        <table schema="" tableName="tb_order_shipping"></table>
        <table schema="" tableName="tb_user"></table>

    </context>
</generatorConfiguration>
```

從上面的配置檔案中可以看出，配置檔案主要做了幾件事：

1、連線資料庫，這是必須的，要不然怎麼根據資料庫的表生成程式碼呢？

2、指定要生成程式碼的位置，要生成的程式碼包括 pojo類、對映檔案mapper.xml、介面Mapper.java。注意：指定生成程式碼的位置時，目錄分隔符：window系統使用`\`，Linux和Mac系統使用`/`。

3、指定資料庫中想要生成哪些表

### 2-2、執行逆向工程，生成程式碼

配置檔案寫好了，然後執行`GeneratorSqlmap.java` 裡面的main方法，即可自動生成程式碼。

`GeneratorSqlmap.java`的程式碼如下：

```java
import java.io.File;
import java.util.ArrayList;
import java.util.List;

import org.mybatis.generator.api.MyBatisGenerator;
import org.mybatis.generator.config.Configuration;
import org.mybatis.generator.config.xml.ConfigurationParser;
import org.mybatis.generator.exception.XMLParserException;
import org.mybatis.generator.internal.DefaultShellCallback;

public class GeneratorSqlmap {

    public void generator() throws Exception {

        List<String> warnings = new ArrayList<String>();
        boolean overwrite = true;
        //指定 逆向工程配置檔案
        File configFile = new File("generatorConfig.xml");
        ConfigurationParser cp = new ConfigurationParser(warnings);
        Configuration config = cp.parseConfiguration(configFile);
        DefaultShellCallback callback = new DefaultShellCallback(overwrite);
        MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config,
                callback, warnings);
        myBatisGenerator.generate(null);

    }

    public static void main(String[] args) throws Exception {
        try {
            GeneratorSqlmap generatorSqlmap = new GeneratorSqlmap();
            generatorSqlmap.generator();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}

```

執行一下即可，然後在src目錄下就可以看到最新生成的程式碼了，如下圖：

![][3]

大功告成！把這些自動生成的程式碼複製到我們真正的專案中即可。














---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/Mybatis/02%20%E4%BD%BF%E7%94%A8Mybatis%E7%9A%84%E9%80%86%E5%90%91%E5%B7%A5%E7%A8%8B%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E4%BB%A3%E7%A0%81.md](https://github.com/nnngu/LearningNotes/blob/master/Mybatis/02%20%E4%BD%BF%E7%94%A8Mybatis%E7%9A%84%E9%80%86%E5%90%91%E5%B7%A5%E7%A8%8B%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E4%BB%A3%E7%A0%81.md)


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/5/1520263736178.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/5/1520264307661.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/6/1520265701449.jpg