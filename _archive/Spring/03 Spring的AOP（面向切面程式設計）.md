# 03 Spring的AOP（面向切面程式設計）

> **版本提示**：本篇基於 Spring 4.x，使用 XML 配置 AOP。現代 Spring 6.x 推薦使用 `@Aspect` 註解方式，請參考 [07 Spring 註解式 AOP 開發](07%20Spring%20註解式%20AOP%20開發.md)。

## 1、關於AOP

AOP（Aspect Oriented Programming），即面向切面程式設計，可以說是OOP（Object Oriented Programming，物件導向程式設計）的補充和完善。OOP引入封裝、繼承、多型等概念來建立一種物件層次結構，用於模擬公共行為的一個集合。OOP允許開發者定義縱向的關係，但並不適合定義橫向的關係，例如日誌功能。日誌程式碼往往橫向地散佈在所有物件層次中，而與它對應的物件的核心功能毫無關係，對於其他型別的程式碼，如安全性、異常處理等等也是如此，這種散佈在各處的無關的程式碼被稱為橫切（cross cutting），在OOP設計中，它導致了大量程式碼的重複，而不利於各個模組的重用。

AOP技術恰恰相反，它利用一種稱為"橫切"的技術，剖解開封裝的物件內部，並將那些影響了多個類的公共行為封裝到一個可重用模組，並將其命名為"Aspect"，即切面。所謂"切面"，簡單說就是將那些與業務無關，卻為業務模組所共同呼叫的邏輯封裝起來，便於減少系統的重複程式碼，降低模組之間的耦合度，並有利於未來的可操作性和可維護性。

使用"橫切"技術，AOP把軟體系統分為兩個部分：核心關注點和橫切關注點。業務處理的主要流程是核心關注點，與之關係不大的部分是橫切關注點。橫切關注點的一個特點是，他們經常發生在核心關注點的多處，而各處基本相似，比如 許可權認證、日誌、事務等等。AOP的作用在於分離系統中的各種關注點，將核心關注點和橫切關注點分離開來。

## 2、AOP的核心概念

1、橫切關注點

對哪些方法進行攔截，攔截後怎麼處理，這些關注點稱之為橫切關注點

2、切面（aspect）

類是對物體特徵的抽象，切面就是對橫切關注點的抽象

3、連線點（joinpoint）

被攔截到的點，因為Spring只支援方法型別的連線點，所以在Spring中連線點指的就是被攔截到的方法，實際上連線點還可以是欄位或者構造器

4、切入點（pointcut）

對連線點進行攔截的定義

5、通知（advice）

所謂通知指的就是攔截到連線點之後要執行的程式碼，通知分為五類：前置、後置、異常、最終、環繞。

6、目標物件

代理的目標物件

7、織入（weave）

將切面應用到目標物件並導致代理物件建立的過程

8、引入（introduction）

在不修改程式碼的前提下，引入可以在執行期為類動態地新增一些方法或欄位

## 3、Spring對AOP的支援

Spring中AOP代理由Spring的IoC容器負責生成、管理，其依賴關係也由IoC容器負責管理。因此，AOP代理可以直接使用容器中的其它bean例項作為目標，這種關係可由IoC容器的依賴注入提供。Spring建立代理的規則為：

1、預設使用Java動態代理來建立AOP代理，這樣就可以為任何介面例項建立代理了

2、當需要代理的類不是代理介面的時候，Spring會切換為使用CGLIB代理，也可強制使用CGLIB

AOP程式設計其實是很簡單的事情，縱觀AOP程式設計，程式設計師只需要參與三個部分：

1、定義普通業務元件

2、定義切入點，一個切入點可能橫切多個業務元件

3、定義增強處理，增強處理就是在AOP框架為普通業務元件織入的處理動作

所以進行AOP程式設計的關鍵就是定義切入點和定義增強處理，一旦定義了合適的切入點和增強處理，AOP框架將自動生成AOP代理，即：代理物件的方法=增強處理+被代理物件的方法。

下面給出一個Spring AOP的.xml檔案模板，名字叫做aop.xml，之後的內容都在aop.xml上進行擴充套件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop-4.2.xsd">
            
</beans>
```

## 4、基於Spring的AOP簡單實現

在講解之前，說明一點：使用Spring AOP，要成功執行程式碼，只用Spring提供給開發者的jar包是不夠的，請額外上網下載兩個jar包：

1、aopalliance.jar

2、aspectjweaver.jar

開始講解用Spring AOP的XML實現方式，先定義一個介面：

```java
public interface HelloWorld
{
    void printHelloWorld();
    void doPrint();
}
```

定義兩個介面實現類：

```java
public class HelloWorldImpl1 implements HelloWorld
{
    public void printHelloWorld()
    {
        System.out.println("Enter HelloWorldImpl1.printHelloWorld()");
    }
    
    public void doPrint()
    {
        System.out.println("Enter HelloWorldImpl1.doPrint()");
        return ;
    }
}
```

```java
public class HelloWorldImpl2 implements HelloWorld
{
    public void printHelloWorld()
    {
        System.out.println("Enter HelloWorldImpl2.printHelloWorld()");
    }
    
    public void doPrint()
    {
        System.out.println("Enter HelloWorldImpl2.doPrint()");
        return ;
    }
}
```

橫切關注點，這裡是列印時間：

```java
public class TimeHandler
{
    public void printTime()
    {
        System.out.println("CurrentTime = " + System.currentTimeMillis());
    }
}
```

有這三個類就可以實現一個簡單的Spring AOP了，看一下aop.xml的配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop-4.2.xsd">
        
        <bean id="helloWorldImpl1" class="com.nnngu.aop.HelloWorldImpl1" />
        <bean id="helloWorldImpl2" class="com.nnngu.aop.HelloWorldImpl2" />
        <bean id="timeHandler" class="com.nnngu.aop.TimeHandler" />
        
        <aop:config>
            <aop:aspect id="time" ref="timeHandler">
                <aop:pointcut id="addTime" expression="execution(* com.nnngu.aop.HelloWorld.*(..))" />
                <aop:before method="printTime" pointcut-ref="addTime" />
                <aop:after method="printTime" pointcut-ref="addTime" />
            </aop:aspect>
        </aop:config>
</beans>
```

寫一個main函式呼叫一下：

```java
public static void main(String[] args)
{
    ApplicationContext ctx = 
            new ClassPathXmlApplicationContext("aop.xml");
        
    HelloWorld hw1 = (HelloWorld)ctx.getBean("helloWorldImpl1");
    HelloWorld hw2 = (HelloWorld)ctx.getBean("helloWorldImpl2");
    hw1.printHelloWorld();
    System.out.println(); // 換行
    hw1.doPrint();
    
    System.out.println(); // 換行
    hw2.printHelloWorld();
    System.out.println(); // 換行
    hw2.doPrint();
}
```

執行結果為：

```
CurrentTime = 1446129611993
Enter HelloWorldImpl1.printHelloWorld()
CurrentTime = 1446129611993

CurrentTime = 1446129611994
Enter HelloWorldImpl1.doPrint()
CurrentTime = 1446129611994

CurrentTime = 1446129611994
Enter HelloWorldImpl2.printHelloWorld()
CurrentTime = 1446129611994

CurrentTime = 1446129611994
Enter HelloWorldImpl2.doPrint()
CurrentTime = 1446129611994
```

可以看到給HelloWorld介面的兩個實現類的所有方法都加上了代理，代理內容就是列印時間。

## 5、基於Spring的AOP使用其他細節

### 5-1、增加一個橫切關注點，列印日誌，Java類為：

```java
public class LogHandler
{
    public void LogBefore()
    {
        System.out.println("Log before method");
    }
    
    public void LogAfter()
    {
        System.out.println("Log after method");
    }
}
```

aop.xml配置為：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop-4.2.xsd">
        
        <bean id="helloWorldImpl1" class="com.nnngu.aop.HelloWorldImpl1" />
        <bean id="helloWorldImpl2" class="com.nnngu.aop.HelloWorldImpl2" />
        <bean id="timeHandler" class="com.nnngu.aop.TimeHandler" />
        <bean id="logHandler" class="com.nnngu.aop.LogHandler" />
        
        <aop:config>
            <aop:aspect id="time" ref="timeHandler" order="1">
                <aop:pointcut id="addTime" expression="execution(* com.nnngu.aop.HelloWorld.*(..))" />
                <aop:before method="printTime" pointcut-ref="addTime" />
                <aop:after method="printTime" pointcut-ref="addTime" />
            </aop:aspect>
            <aop:aspect id="log" ref="logHandler" order="2">
                <aop:pointcut id="printLog" expression="execution(* com.nnngu.aop.HelloWorld.*(..))" />
                <aop:before method="LogBefore" pointcut-ref="printLog" />
                <aop:after method="LogAfter" pointcut-ref="printLog" />
            </aop:aspect>
        </aop:config>
</beans>
```

測試類不變，列印結果為：

```
CurrentTime = 1446130273734
Log before method
Enter HelloWorldImpl1.printHelloWorld()
Log after method
CurrentTime = 1446130273735

CurrentTime = 1446130273736
Log before method
Enter HelloWorldImpl1.doPrint()
Log after method
CurrentTime = 1446130273736

CurrentTime = 1446130273736
Log before method
Enter HelloWorldImpl2.printHelloWorld()
Log after method
CurrentTime = 1446130273736

CurrentTime = 1446130273737
Log before method
Enter HelloWorldImpl2.doPrint()
Log after method
CurrentTime = 1446130273737
```

要想讓logHandler在timeHandler前使用有兩個辦法：

（1）aspect裡面有一個order屬性，order屬性的數字就是橫切關注點的順序

（2）在aop.xml裡，把logHandler定義在timeHandler前面，Spring預設以aspect的定義順序作為織入順序

### 5-2、我只想織入介面中的某些方法

修改一下pointcut的expression就好了：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop-4.2.xsd">
        
        <bean id="helloWorldImpl1" class="com.nnngu.aop.HelloWorldImpl1" />
        <bean id="helloWorldImpl2" class="com.nnngu.aop.HelloWorldImpl2" />
        <bean id="timeHandler" class="com.nnngu.aop.TimeHandler" />
        <bean id="logHandler" class="com.nnngu.aop.LogHandler" />
        
        <aop:config>
            <aop:aspect id="time" ref="timeHandler" order="1">
                <aop:pointcut id="addTime" expression="execution(* com.nnngu.aop.HelloWorld.print*(..))" />
                <aop:before method="printTime" pointcut-ref="addTime" />
                <aop:after method="printTime" pointcut-ref="addTime" />
            </aop:aspect>
            <aop:aspect id="log" ref="logHandler" order="2">
                <aop:pointcut id="printLog" expression="execution(* com.nnngu.aop.HelloWorld.do*(..))" />
                <aop:before method="LogBefore" pointcut-ref="printLog" />
                <aop:after method="LogAfter" pointcut-ref="printLog" />
            </aop:aspect>
        </aop:config>
</beans>
```

以上的修改，表示timeHandler只會織入HelloWorld介面print開頭的方法，logHandler只會織入HelloWorld介面do開頭的方法

### 5-3、強制使用CGLIB生成代理

前面說過Spring使用JDK動態代理或是CGLIB生成代理是有規則的，高版本的Spring會自動選擇是使用JDK動態代理還是CGLIB生成代理內容，當然我們也可以強制使用CGLIB生成代理，那就是<aop:config>裡面有一個"proxy-target-class"屬性，這個屬性值如果被設定為true，那麼基於類的代理將起作用，如果proxy-target-class被設定為false或者這個屬性被省略，那麼基於介面的代理將起作用。
















---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/Spring/02%20Spring%E7%9A%84AOP%EF%BC%88%E9%9D%A2%E5%90%91%E5%88%87%E9%9D%A2%E7%BC%96%E7%A8%8B%EF%BC%89.md](https://github.com/nnngu/LearningNotes/blob/master/Spring/02%20Spring%E7%9A%84AOP%EF%BC%88%E9%9D%A2%E5%90%91%E5%88%87%E9%9D%A2%E7%BC%96%E7%A8%8B%EF%BC%89.md)