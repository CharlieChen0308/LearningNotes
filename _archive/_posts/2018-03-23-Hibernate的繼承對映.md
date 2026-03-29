---
categories: Hibernate
description: 摘要：本文主要總結繼承對映的三種實現方式。
---

## 物件模型示例：

![][1]

## 繼承對映的實現方式有以下三種：

* （一）每棵類繼承樹一張表
* （二）每個類一張表
* （三）每個子類一張表

### （一）每棵類繼承樹一張表

關係模型如下：

![][2]

對映檔案如下：

```xml
<hibernate-mapping package="com.nnngu">  
    <class name="Animal" table="t_animal" lazy="false">  
        <id name="id">  
            <generator class="native"/>  
        </id>  
        <discriminator column="type" type="string"/>  
        <property name="name"/>  
        <property name="sex"/>  
        <subclass name="Pig" discriminator-value="P">  
            <property name="weight"/>  
        </subclass>  
        <subclass name="Bird" discriminator-value="B">  
            <property name="height"/>  
        </subclass>  
    </class>  
</hibernate-mapping>
```

說明：

因為類繼承樹肯定是對應多個類，要把多個類的資訊存放在一張表中，必須有某種機制來區分哪些記錄是屬於哪個類的。這種機制就是，在表中新增一個欄位，用這個欄位的值來進行區分。

用hibernate實現這種策略的時候，有如下步驟：

1、父類用普通的`<class>`標籤定義

2、在父類中定義一個discriminator，即指定這個區分的欄位的名稱和型別
如：`<discriminator column=”XXX” type=”string”/>`

3、子類使用`<subclass>`標籤定義，在定義subclass的時候，需要注意如下幾點：

* Subclass標籤的name屬性是子類的全路徑名
* 在Subclass標籤中，用discriminator-value屬性來標明本子類的discriminator欄位（用來區分不同類的欄位）的值
* Subclass既可以被class標籤所包含（這種包含關係正是表明了類之間的繼承關係），也可以與class標籤平行。當subclass標籤的定義與class標籤平行的時候，需要在subclass標籤中，新增extends屬性，裡面的值是父類的全路徑名稱。
* 子類的其它屬性，像普通類一樣，定義在subclass標籤的內部。
* 關於鑑別值在儲存的時候hibernate會自動儲存，在載入的時候會根據鑑別值取得相關的物件


### （二）每個類一張表

關係模型如下：

![][3]

對映檔案如下：

```xml
<hibernate-mapping package="com.nnngu">  
    <class name="Animal" table="t_animal">  
        <id name="id">  
            <generator class="native"/>  
        </id>  
        <property name="name"/>  
        <property name="sex"/>  
        <joined-subclass name="Pig" table="t_pig">  
            <key column="pid"/>  
            <property name="weight"/>  
        </joined-subclass>  
        <joined-subclass name="Bird" table="t_bird">  
            <key column="bid"/>  
            <property name="height"/>  
        </joined-subclass>  
    </class>  
</hibernate-mapping>  
```

說明：

這種策略是使用`joined-subclass`標籤來定義子類的。父類、子類，每個類都對應一張資料庫表。
在父類對應的資料庫表中，實際上會儲存所有的記錄，包括父類和子類的記錄；在子類對應的資料庫表中，這個表只定義了子類中所特有的屬性對映的欄位。子類與父類，透過相同的主鍵值來關聯。

實現這種策略的時候，有如下步驟：

1、父類用普通的`<class>`標籤定義即可

2、父類不再需要定義`discriminator`欄位

3、子類用`<joined-subclass>`標籤定義，在定義`joined-subclass`的時候，需要注意如下幾點：

* joined-subclass標籤的name屬性是子類的全路徑名
* joined-subclass標籤需要包含一個key標籤，這個標籤指定了子類和父類之間是透過哪個欄位來關聯的。如：`<key column=”PARENT_KEY_ID”/>`，這裡的column，實際上就是父類的主鍵對應的對映欄位名稱。
* joined-subclass標籤，既可以被class標籤所包含（這種包含關係正是表明了類之間的繼承關係），也可以與class標籤平行。 當joined-subclass標籤的定義與class標籤平行的時候，需要在joined-subclass標籤中，新增extends屬性，裡面的值是父類的全路徑名稱。子類的其它屬性，像普通類一樣，定義在joined-subclass標籤的內部。


### （三）每個子類一張表

關係模型如下：

![][4]

對映檔案如下：

```xml
<hibernate-mapping package="com.nnngu">  
    <class name="Animal" table="t_animal" abstract="true">  
        <id name="id">  
            <generator class="assigned"/>  
        </id>  
        <property name="name"/>  
        <property name="sex"/>  
        <union-subclass name="Pig" table="t_pig">  
            <property name="weight"/>  
        </union-subclass>  
        <union-subclass name="Bird" table="t_bird">  
            <property name="height"/>  
        </union-subclass>  
    </class>  
</hibernate-mapping>
```

說明：

這種策略是使用union-subclass標籤來定義子類的。每個子類對應一張表，而且這個表的資訊是完備的，即包含了所有從父類繼承下來的屬性對映的欄位（這就是它跟joined-subclass的不同之處，joined-subclass定義的子類的表，只包含子類特有屬性對映的欄位）。

實現這種策略的時候，有如下步驟：

1、父類用普通`<class>`標籤定義即可

2、子類用`<union-subclass>`標籤定義，在定義union-subclass的時候，需要注意如下幾點：

* union-subclass標籤不再需要包含key標籤（與joined-subclass不同）
* union-subclass標籤，既可以被class標籤所包含（這種包含關係正是表明了類之間的繼承關係），也可以與class標籤平行。 當union-subclass標籤的定義與class標籤平行的時候，需要在union-subclass標籤中，新增extends屬性，裡面的值是父類的全路徑名稱。
* 子類的其它屬性，像普通類一樣，定義在union-subclass標籤的內部。這個時候，雖然在union-subclass裡面定義的只有子類的屬性，但是因為它繼承了父類，所以，不需要定義其它的屬性，在對映到資料庫表的時候，依然包含了父類的所有屬性的對映欄位。

注意：在儲存物件的時候id不能重複（不能使用資料庫的自增方式生成主鍵）














---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-23-Hibernate%E7%9A%84%E7%BB%A7%E6%89%BF%E6%98%A0%E5%B0%84.md](https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-23-Hibernate%E7%9A%84%E7%BB%A7%E6%89%BF%E6%98%A0%E5%B0%84.md)


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/23/1521813220661.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/23/1521813432985.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/23/1521813827293.jpg
  [4]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/23/1521814690190.jpg