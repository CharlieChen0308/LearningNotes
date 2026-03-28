---
categories: Spring
description: 傳統的`Spring`做法是使用`.xml`檔案來對`bean`進行注入或者是配置`aop`、事務，這麼做有兩個缺點：
---

## 什麼是註解

傳統的`Spring`做法是使用`.xml`檔案來對`bean`進行注入或者是配置`aop`、事務，這麼做有兩個缺點：

1、如果所有的內容都配置在`.xml`檔案中，那麼`.xml`檔案將會十分龐大；如果按需求分開`.xml`檔案，那麼`.xml`檔案又會非常多。總之這將導致配置檔案的可讀性與可維護性變得很低。

2、開發中需要在`.java`檔案和`.xml`檔案之間不斷切換，是一件麻煩的事，同時這種思維上的不連貫也會降低開發的效率。

為了解決這兩個問題，`Spring`引入了註解，透過`@XXX`的方式，讓註解與Java Bean 緊密結合，既大大減少了配置檔案的體積，又增加了Java Bean 的可讀性與內聚性。

本篇文章，講講最重要的三個Spring註解，也就是`@Autowired`、`@Resource`和`@Service`。

## 不使用註解

先看一個不使用註解的 Spring 示例，在這個示例的基礎上，再改成註解版本，這樣也能看出使用與不使用註解之間的區別，先定義一個老虎類：

```java
public class Tiger
{
    private String tigerName = "TigerKing";
    
    public String toString()
    {
        return "TigerName:" + tigerName;
    }
}
```

再定義一個猴子類：

```java
public class Monkey
{
    private String monkeyName = "MonkeyKing";
    
    public String toString()
    {
        return "MonkeyName:" + monkeyName;
    }
}
```

再定義一個動物園類：

```java
public class Zoo
{
    private Tiger tiger;
    private Monkey monkey;
    
    public void setTiger(Tiger tiger)
    {
        this.tiger = tiger;
    }
    
    public void setMonkey(Monkey monkey)
    {
        this.monkey = monkey;
    }
    
    public Tiger getTiger()
    {
        return tiger;
    }
    
    public Monkey getMonkey()
    {
        return monkey;
    }
    
    public String toString()
    {
        return tiger + "\n" + monkey;
    }
}
```

在Spring的配置檔案中這麼寫：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"   
    xmlns="http://www.springframework.org/schema/beans"  
    xmlns:context="http://www.springframework.org/schema/context"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans 
        http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-4.2.xsd"
    default-autowire="byType">
    
    <bean id="zoo" class="com.nnngu.bean.Zoo" >
        <property name="tiger" ref="tiger" />
        <property name="monkey" ref="monkey" />
    </bean>
    
    <bean id="tiger" class="com.nnngu.domain.Tiger" />
    <bean id="monkey" class="com.nnngu.domain.Monkey" />
    
</beans>
```

都很熟悉，權當複習一遍了。

## @Autowired

`@Autowired`顧名思義，就是自動裝配，其作用是為了消除Java程式碼裡面的`getter/setter`與bean屬性中的property。當然，`getter`看個人需求，如果私有屬性需要對外提供的話，應當保留該`getter`。

因此，引入`@Autowired`註解，先看一下Spring的配置檔案怎麼寫：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"   
    xmlns="http://www.springframework.org/schema/beans"  
    xmlns:context="http://www.springframework.org/schema/context"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans 
        http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-4.2.xsd">
    
    <context:component-scan base-package="com.nnngu" />
    
    <bean id="zoo" class="com.nnngu.bean.Zoo" />
    <bean id="tiger" class="com.nnngu.domain.Tiger" />
    <bean id="monkey" class="com.nnngu.domain.Monkey" />
    
</beans>
```

注意上面程式碼中的`<context:component-scan base-package="com.nnngu" />` ，作用是告訴Spring我要使用註解了，Spring會自動掃描`com.nnngu`路徑下的註解。

之前在`zoo`這個bean裡面應當注入的兩個屬性`tiger`、`monkey`，現在不需要注入了。再看一下 `Zoo.java` 也很方便，把`getter/setter` 都可以去掉：

```java
public class Zoo
{
    @Autowired
    private Tiger tiger;
    
    @Autowired
    private Monkey monkey;
    
    public String toString()
    {
        return tiger + "\n" + monkey;
    }
}
```

上面程式碼中`@Autowired`註解的意思是，當Spring發現`@Autowired`註解時，將自動在程式碼上下文中找到和其匹配（預設是型別匹配）的Bean，並自動注入到相應的地方去。

（有一個細節性的問題是，假如配置檔案的bean裡面有兩個property，`Zoo.java`裡面又去掉了屬性的`getter/setter`並使用`@Autowired`註解標註這兩個屬性那會怎麼樣？答案是Spring會按照xml優先的原則去`Zoo.java`中尋找這兩個屬性的`getter/setter`，導致的結果就是初始化bean報錯。）

此時如果我把`.xml`檔案的`<bean id="tiger" class="com.nnngu.domain.Tiger" />`和`<bean id="monkey" class="com.nnngu.domain.Monkey" />` 去掉，再執行，會丟擲如下異常：

```
org.springframework.beans.factory.BeanCreationException: Could not autowire field
```

因為，`@Autowired`註解要去尋找的是一個`bean`，Tiger 和 Monkey 的`bean`定義都給去掉了，自然就不是一個`bean`了，Spring容器找不到也很好理解。那麼，如果屬性的`bean`找不到，我又不想讓Spring容器丟擲異常，而只顯示null，可以嗎？答案是可以的，只要將`@Autowired`註解的required屬性設定為 false 即可，如下：

```java
public class Zoo
{
    @Autowired(required = false)
    private Tiger tiger;
    
    @Autowired(required = false)
    private Monkey monkey;
    
    public String toString()
    {
        return tiger + "\n" + monkey;
    }
}
```

此時，就算找不到 tiger、monkey 這兩個屬性的 `bean`，Spring容器也不再丟擲異常，而是認為這兩個屬性為null。

## @Autowired介面注入

上面的比較簡單，我們只是簡單注入一個Java類，那麼如果有一個介面，有多個實現，Bean裡引用的是介面名，又該怎麼做呢？比如有一個Car介面：

```java
public interface Car
{
    public String carName();
}
```

兩個實現類BMW和Benz：

```java

public class BMW implements Car
{
    public String carName()
    {
        return "BMW car";
    }
}
```

```java

public class Benz implements Car
{
    public String carName()
    {
        return "Benz car";
    }
}
```

寫一個CarFactory，引用Car：

```java

public class CarFactory
{
    @Autowired
    private Car car;
    
    public String toString()
    {
        return car.carName();
    }
}
```

不用說，一定是報錯的，Car介面有兩個實現類，Spring並不知道應當引用哪個實現類。這種情況通常有兩個解決辦法：

1、刪除其中一個實現類，Spring會自動去base-package下尋找Car介面的實現類，發現Car介面只有一個實現類，便會直接引用這個實現類。

2、實現類就是有多個該怎麼辦？此時可以使用`@Qualifier`註解：

```java

public class CarFactory
{
    @Autowired
    @Qualifier("BMW")
    private Car car;
    
    public String toString()
    {
        return car.carName();
    }
}
```

注意`@Qualifier`註解括號裡面的應當是Car介面實現類的類名，我之前試的時候一直以為是bean的名字，所以寫了"bMW"，結果一直報錯。

## @Resource

把`@Resource`註解放在`@Autowired`下面說，是因為它們的作用非常相似。先看一下`@Resource`，直接在`Zoo.java`中寫：

```java

public class Zoo
{
    @Resource(name = "tiger")
    private Tiger tiger;
    
    @Resource(type = Monkey.class)
    private Monkey monkey;
    
    public String toString()
    {
        return tiger + "\n" + monkey;
    }
}
```

說一下`@Resource`的裝配順序：

1、`@Resource`後面如果沒有任何內容，預設是透過name屬性去匹配bean，找不到再按type去匹配

2、指定了name或者type則根據指定的型別去匹配bean

3、指定了name和type則根據指定的name和type去匹配bean，任何一個不匹配都將報錯

然後，區分一下`@Autowired`和`@Resource`兩個註解的區別：

1、`@Autowired`預設按照byType方式進行bean匹配，`@Resource`預設按照byName方式進行bean匹配

2、`@Autowired`是Spring的註解，`@Resource`是J2EE的註解，這個可以看一下匯入註解的時候這兩個註解的包名就一清二楚了

Spring屬於第三方的，J2EE是Java自己的東西，因此，建議使用`@Resource`註解，以減少程式碼和Spring之間的耦合。

## @Service

上面動物園這個例子，還可以繼續簡化，因為Spring的配置檔案裡面還有三個bean，下一步的簡化是把這三個bean也給去掉，使得Spring配置檔案裡面只有一個自動掃描的標籤。這樣做增強了Java程式碼的內聚性並進一步減少配置檔案。

先看一下配置檔案（三個bean都刪除了）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"   
    xmlns="http://www.springframework.org/schema/beans"  
    xmlns:context="http://www.springframework.org/schema/context"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans 
        http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-4.2.xsd">
    
    <context:component-scan base-package="com.nnngu" />
    
</beans>
```

下面以`Zoo.java`為例，其餘的`Monkey.java`和`Tiger.java`都一樣：

```java
@Service
public class Zoo
{
    @Autowired
    private Tiger tiger;
    
    @Autowired
    private Monkey monkey;
    
    public String toString()
    {
        return tiger + "\n" + monkey;
    }
}
```

這樣，`Zoo.java`在Spring容器中存在的形式就是`zoo`，即可以透過			`ApplicationContext`的`getBean("zoo")`方法來得到`Zoo.java`。`@Service`註解，其實做了兩件事情：

1、宣告`Zoo.java`是一個bean，這點很重要，因為`Zoo.java`是一個bean，其他的類才可以使用`@Autowired`將Zoo作為一個成員變數自動注入

2、`Zoo.java`在bean中的id是`zoo`，即類名且首字母小寫

如果我不想用這種形式，而想讓`Zoo.java`在Spring容器中的名字就叫做`Zoo`，可以嗎？答案是可以的：

```java
@Service("Zoo")
@Scope("prototype")
public class Zoo
{
    @Autowired
    private Monkey monkey;
    @Autowired
    private Tiger tiger;
    
    public String toString()
    {
        return tiger + "\n" + monkey;
    }
}
```

這樣，就可以透過`ApplicationContext`的`getBean("Zoo")`方法來得到`Zoo.java`了。

這裡我還多加了一個`@Scope`註解，應該很好理解。因為Spring預設產生的bean是單例的，假如我不想使用單例怎麼辦，xml檔案裡面可以在bean裡面配置scope屬性。註解也一樣，配置`@Scope`即可，預設是"singleton"即單例，"prototype"表示原型即每次都會new一個新的物件出來。

## 補充細節

最後再補充一個細節。假如animal包下有Tiger，domain包下也有Tiger，它們二者都加了`@Service`註解，那麼在`Zoo.java`中即使明確表示我要引用的是domain包下的Tiger，程式執行的時候依然會報錯。

因為兩個Tiger都使用`@Service`註解標註，意味著兩個bean的名字都是"tiger"，那麼我在`Zoo.java`中自動裝配的是哪個tiger呢？不明確，因此，Spring容器會丟擲`ConflictingBeanDefinitionException`異常。





















---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/Spring/04%20Spring%E7%9A%84%40Autowired%E6%B3%A8%E8%A7%A3%E3%80%81%40Resource%E6%B3%A8%E8%A7%A3%E3%80%81%40Service%E6%B3%A8%E8%A7%A3.md](https://github.com/nnngu/LearningNotes/blob/master/Spring/04%20Spring%E7%9A%84%40Autowired%E6%B3%A8%E8%A7%A3%E3%80%81%40Resource%E6%B3%A8%E8%A7%A3%E3%80%81%40Service%E6%B3%A8%E8%A7%A3.md)