# 01 Spring的簡單配置和使用

> **版本提示**：本篇基於 Spring 3.x，使用 XML 配置方式。現代 Spring 6.x 已全面改用 Java 配置與註解驅動，請參考 [05 Spring Java 配置與註解驅動（現代方式）](05%20Spring%20Java%20%E9%85%8D%E7%BD%AE%E8%88%87%E8%A8%BB%E8%A7%A3%E9%A9%85%E5%8B%95%EF%BC%88%E7%8F%BE%E4%BB%A3%E6%96%B9%E5%BC%8F%EF%BC%89.md)。

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)

---

## Spring需要的jar包

文章中 Spring 使用的版本是 `3.2.18` ， 需要的 jar 包如下：

```
spring-aop
spring-aspects
spring-beans
spring-context
spring-context-support
spring-core
spring-expression
spring-instrument
spring-instrument-tomcat
spring-jdbc
spring-jms
spring-messaging 4.3.14
spring-orm
spring-oxm
spring-test
spring-tx
spring-web
spring-webmvc
spring-webmvc-portlet
spring-websocket 4.3.14
aopalliance 1.0
aspectjweaver 1.8.13
cglib 3.1
commons-collections 3.2.2
commons-dbcp 1.4
commons-logging 1.1.1
commons-pool 1.6
standard 1.1.2
```

使用 Maven 構建的 Java 專案，需要在 pom.xml 中新增如下依賴：

```xml
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aop</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aspects</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context-support</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-expression</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-instrument</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-instrument-tomcat</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jms</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-messaging</artifactId>
            <version>4.3.14.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-orm</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-oxm</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>3.2.18.RELEASE</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-tx</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc-portlet</artifactId>
            <version>3.2.18.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-websocket</artifactId>
            <version>4.3.14.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>aopalliance</groupId>
            <artifactId>aopalliance</artifactId>
            <version>1.0</version>
        </dependency>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.8.13</version>
        </dependency>
        <dependency>
            <groupId>cglib</groupId>
            <artifactId>cglib</artifactId>
            <version>3.1</version>
        </dependency>
        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.2.2</version>
        </dependency>
        <dependency>
            <groupId>commons-dbcp</groupId>
            <artifactId>commons-dbcp</artifactId>
            <version>1.4</version>
        </dependency>
        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.1.1</version>
        </dependency>
        <dependency>
            <groupId>commons-pool</groupId>
            <artifactId>commons-pool</artifactId>
            <version>1.6</version>
        </dependency>
        <dependency>
            <groupId>taglibs</groupId>
            <artifactId>standard</artifactId>
            <version>1.1.2</version>
        </dependency>
```

## 新建一個 HelloWorld.java

新建一個`HelloWorld.java`，程式碼如下：

```java
package com.nnngu;

public class HelloWorld {
    private String info;

    public String getInfo() {
        return info;
    }

    public void setInfo(String info) {
        this.info = info;
    }
}

```

## 編寫配置檔案 applicationContext.xml

在 resources 目錄下建立 applicationContext.xml 配置檔案

![][1]

程式碼如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans
        xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

    <!-- 配置需要被Spring管理的Bean（建立後放在了Spring IOC容器裡面）-->
    <bean id="hello" class="com.nnngu.HelloWorld">
        <!-- 配置該Bean需要注入的屬性（是透過屬性set方法來注入的） -->
        <property name="info" value="2018，Happy New Year!"/>
    </bean>

</beans>
```

## 使用

建立 `Main.java` ，程式碼如下：

```java
package com.nnngu;

import org.springframework.beans.factory.BeanFactory;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Main {

    public static void main(String[] args) {
        // 獲取Spring的applicationContext配置檔案，注入IOC容器中
        BeanFactory factory = new ClassPathXmlApplicationContext("applicationContext.xml");
        HelloWorld hw = (HelloWorld)factory.getBean("hello");
        System.out.println(hw.getInfo());
        System.out.println(hw);
    }
}

```

執行結果：

![][2]


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/12/1518417902156.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/12/1518418286415.jpg