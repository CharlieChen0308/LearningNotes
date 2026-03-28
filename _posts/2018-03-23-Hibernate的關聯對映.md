---
categories: Hibernate
description: 摘要：讓我們一起走進Hibernate的幾種關聯對映關係。
---

首先我們瞭解一個名詞ORM，全稱是（Object Relational Mapping），即物件關係對映。ORM的實現思想就是將關係型資料庫中表的資料對映成物件，以物件的形式展現，這樣開發人員就可以把對資料庫的操作轉化為對這些物件的操作。Hibernate正是實現了這種思想，達到了方便開發人員以物件導向的思想來實現對資料庫的操作。

Hibernate在實現ORM功能的時候主要用到的檔案有：對映類`（*.java）`、對映檔案`（*.hbm.xml）`和資料庫配置檔案`（*.properties/*.cfg.xml）`，它們各自的作用如下：

* 對映類`（*.java）`：它是描述資料庫表的結構，表中的欄位在類中被描述成屬性，將來就可以實現把表中的記錄對映成為該類的物件了。
* 對映檔案`（*.hbm.xml）`：它是指定資料庫表和對映類之間的關係，包括對映類和資料庫表的對應關係、表欄位和類屬性的對應關係。
* 資料庫配置檔案`（*.properties/*.cfg.xml）`：它是指定與資料庫連線時需要的連線資訊，比如連線哪種資料庫、登入資料庫的使用者名稱、密碼以及連線字串等。當然還可以把對映類的地址對映資訊放在這裡。

## 接下來讓我們一起走進Hibernate的幾種關聯對映關係：

### 單向一對一關聯對映（one-to-one）：

兩個物件之間一對的關係，例如：Person（人）-  IdCard（身份證）

有兩種策略可以實現一對一的關聯對映：

* 主鍵關聯：即讓兩個物件具有相同的主鍵值，以表明它們之間的一一對應的關係；資料庫表不會有額外的欄位來維護它們之間的關係，僅透過表的主鍵來關聯。如下圖：

![][1]

注意：需要在Person.hbm.xml對映檔案中配置`one-to-one`標籤，如下：

```xml
<?xml version="1.0"?>  
<!DOCTYPE hibernate-mapping PUBLIC   
    "-//Hibernate/Hibernate Mapping DTD 3.0//EN"  
    "http://hibernate.sourceforge.net/hibernate-mapping-3.0.dtd">  
<hibernate-mapping>  
    <class name="com.nnngu.Person" table="t_person">  
        <id name="id">  
            <!-- 採用foreign生成策略，foreign會取得關聯物件的標識 -->  
            <generator class="foreign">  
                <!-- property只關聯物件 -->  
                <param name="property">idCard</param>  
            </generator>  
        </id>  
        <property name="name"/>  
        <!-- one-to-one指示hibernate如何載入其關聯物件，預設根據主鍵載入  
            也就是拿到關係欄位值，根據對端的主鍵來載入關聯物件  
         -->  
        <one-to-one name="idCard" constrained="true"/>  
    </class>  
</hibernate-mapping>  
```

* 唯一外來鍵關聯：外來鍵關聯，本來是用於多對一的配置，但是加上唯一的限制之後（採用`<many-to-one>`標籤來對映，指定多的一端unique為true，這樣就限制了多的一端的多重性為一），也可以用來表示一對一關聯關係，其實它就是多對一的特殊情況。如下圖：

![][2]

需要在Person.hbm.xml對映檔案中配置`many-to-one`標籤，並指定unique為true，如下：

```xml
<?xml version="1.0"?>  
<!DOCTYPE hibernate-mapping PUBLIC   
    "-//Hibernate/Hibernate Mapping DTD 3.0//EN"  
    "http://hibernate.sourceforge.net/hibernate-mapping-3.0.dtd">  
<hibernate-mapping>  
    <class name="com.nnngu.Person" table="t_person">  
        <id name="id">  
            <generator class="native">  
                <!-- property只關聯物件 -->  
                <param name="property">idCard</param>  
            </generator>  
        </id>  
        <property name="name"/>  
        <many-to-one name="idCard" unique="true"/>  
    </class>  
</hibernate-mapping> 
```

**注意：因為一對一的主鍵關聯對映擴充套件性不好，當我們的需要發生改變想要將其變為一對多的時候變無法操作了，所以我們遇到一對一關聯的時候經常會採用唯一外來鍵關聯來解決問題，而很少使用一對一主鍵關聯。**

### 單向多對一關聯對映（many-to-one）：

多對一關聯對映原理：在多的一端加入一個外來鍵，指向一的一端，如下圖：

![][3]

關鍵對映程式碼——在多的一端加入如下標籤對映：

```xml
<many-to-one name="group" column="groupid"/>  
```

### 單向一對多關聯對映（one-to-many）：

一對多關聯對映和多對一關聯對映原理是一致的，都是在多的一端加入一個外來鍵，指向一的一端。如下圖（學生和班級）：

![][4]

注意：它與多對一的區別是維護的關係不同

* 多對一維護的關係是：多指向一的關係，有了此關係，載入多的時候可以將一載入上來。
* 一對多維護的關係是：一指向多的關係，有了此關係，在載入一的時候可以將多載入上來。

關鍵對映程式碼——在一的一端加入如下標籤對映：

```xml
<set name="students">  
      <key column="classesid"/>  
      <one-to-many class="com.nnngu.Student"/>  
</set> 
```

### 單向多對多對映（many-to-many）：

多對多關聯對映需要新增加一張表才完成基本對映，如下圖：

![][5]

關鍵對映程式碼——可以在User的一端加入如下標籤對映：

```xml
<set name="roles" table="t_user_role">  
     <key column="user_id"/>  
     <many-to-many class="com.nnngu.Role" column="role_id"/>  
</set>
```

### 雙向一對一關聯對映：

對比單向一對一對映，需要在IdCard也加入`<one-to-one>`標籤。示意圖如下：

![][6]

雙向一對一主鍵對映關鍵對映程式碼——在IdCard端新加入如下標籤對映：

```xml
<one-to-one name="person"/> 
```

雙向一對一唯一外來鍵對映關鍵對映程式碼——在IdCard端新加入如下標籤對映：

```xml
<one-to-one name="person"property-ref="idCard"/>  
```

注意：一對一唯一外來鍵關聯雙向採用`<one-to-one>`標籤對映，必須指定`<one-to-one>`標籤中的property-ref屬性為關係欄位的名稱

### 雙向一對多關聯對映（非常重要）：

採用雙向一對多關聯對映的目的主要是為了解決單向一對多關聯的缺陷。

雙向一對多關聯的對映方式：

* 在一的一端的集合上採用`<key>`標籤，在多的一端加入一個外來鍵
* 在多的一端採用`<many-to-one>`標籤

注意：`<key>`標籤和`<many-to-one>`標籤加入的欄位要保持一致，否則會產生資料混亂。

在Classes的一端加入如下標籤對映：     

```xml
<set name="students"inverse="true">  
      <key column="classesid"/>  
      <one-to-many class="com.nnngu.Student"/>  
</set> 
```

在Student的一端加入如下標籤對映：

```xml
<many-to-one name="classes" column="classesid"/>  
```

>  瞭解inverse屬性：
> * inverse屬性可以用在一對多和多對多雙向關聯上，inverse屬性預設為false，為false表示本端可以維護關係，如果inverse為true，則本端不能維護關係，會交給另一端維護關係，本端失效。所以一對多關聯對映我們通常在多的一端維護關係，讓一的一端失效。
> * inverse是控制方向上的反轉，隻影響儲存。

### 雙向多對多關聯對映：

雙向的目的就是為了兩端都能將對方載入上來，和單向多對多的區別就是雙向需要在兩端都加入標籤對映，需要注意的是：

* 生成的中間表名稱必須一樣
* 生成的中間表中的欄位必須一樣

Role（角色）端關鍵對映程式碼： 

```xml
<set name="users" table="t_user_role">  
       <key column="role_id"/>  
       <many-to-many class="com.nnngu.User" column="user_id"/>  
</set>  
```

User（使用者）端關鍵對映程式碼：

```xml
<set name="roles" table="t_user_role">  
      <key column="user_id"/>  
      <many-to-many class="com. nnngu.Role" column="role_id"/>  
</set> 
```

## 總結

對於上面的七種關聯對映，最重要的就是一對多的對映，因為它更貼近我們的現實生活，比如：教室和學生就可以是典型的一對多的關係，而我們開發軟體的目的之一就是為了解決一些生活中重複性問題，把那些重複的問題交給計算機來幫助我們完成，從而提高我們的工作效率。
















---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-23-Hibernate%E7%9A%84%E5%85%B3%E8%81%94%E6%98%A0%E5%B0%84.md](https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-23-Hibernate%E7%9A%84%E5%85%B3%E8%81%94%E6%98%A0%E5%B0%84.md)


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/23/1521802267620.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/23/1521802632681.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/23/1521803063292.jpg
  [4]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/23/1521803178710.jpg
  [5]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/23/1521803421510.jpg
  [6]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/23/1521803600888.jpg