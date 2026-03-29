# 02 Java高併發秒殺專案之Service層

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

專案原始碼：[https://github.com/nnngu/nguSeckill](https://github.com/nnngu/nguSeckill)  

---

首先在編寫`Service`層程式碼前，我們應該首先要知道這一層到底是幹什麼的。

`Service`層主要負責業務模組的邏輯應用設計。同樣是首先設計介面，再設計其實現的類，接著在`Spring`的配置檔案中配置其實現的關聯。這樣我們就可以在應用中呼叫`Service`介面來進行業務處理。`Service`層的業務實現，具體要呼叫到已定義的`dao`層的介面，封裝`Service`層的業務邏輯有利於通用的業務邏輯的獨立性和重複利用性，程式顯得非常簡潔。

在專案中要降低耦合的話，分層是一種很好的概念，就是各層各司其職，儘量不做不相干的事，所以`Service`層的話顧名思義就是業務邏輯，處理程式中的一些業務邏輯，以及呼叫`dao`層的程式碼，這裡我們的`dao`層就是連線資料庫的那一層，呼叫關係可以這樣表達：

View(頁面) > Controller(控制層) > Service(業務邏輯) > Dao(資料訪問) > Database(資料庫)

首先還是介面的設計，設計秒殺商品的介面，在`com.nnngu.service.interfaces`包下建立`SeckillService.java`介面檔案，如下圖：

![][1]

`SeckillService.java`檔案裡面的內容請參照專案的原始碼。

建立好介面之後我們要寫實現類了，在寫實現類的時候我們肯定會碰到一個這樣的問題，你要向前端返回json資料的話，你是返回什麼樣的資料好？直接返回一個數字狀態碼或者文字？這樣設計肯定是不好的，所以我們應該向前端返回一個實體資訊json，裡面包含了一系列的資訊，無論是哪種狀態都應該可以應對，既然是與資料庫欄位無關的類，那就不是PO了，所以我們建立一個DTO資料傳輸類。關於常見的幾種物件我的解釋如下：

* PO：也就是我們為每一張資料庫表寫一個實體類

* VO：對某個頁面或者展現層所需要的資料，封裝成一個實體類

* BO：業務物件

* DTO：跟VO的概念有點混淆，也是相當於頁面需要的資料封裝成一個實體類

* POJO：簡單的無規則java物件

在com.nnngu下建立dto包，然後建立Exposer類，這個類是秒殺時資料庫那邊處理的結果的物件

`Exposer.java`檔案裡面的內容請參照專案的原始碼。

## 定義秒殺中可能會出現的異常

定義一個基礎的異常，所有的子異常繼承這個異常`SeckillException`

```java
package com.nnngu.exception;

/**
 *  秒殺基礎的異常
 * Created by nnngu
 */
public class SeckillException extends RuntimeException {
    // 程式碼省略，請參照專案的原始碼
	... ...
}

```

可能會出現秒殺關閉後被秒殺情況，所以建立秒殺關閉異常`SeckillCloseException`，需要繼承我們前面寫的基礎異常

```java
package com.nnngu.exception;

/**
 * 秒殺已經關閉異常，當秒殺結束就會出現這個異常
 * Created by nnngu
 */
public class SeckillCloseException extends SeckillException{
    // 程式碼省略，請參照專案的原始碼
	... ...
}

```

定義重複秒殺異常`RepeatKillException`

```java
package com.nnngu.exception;

/**
 * 重複秒殺異常，不需要我們手動去try catch
 * Created by nnngu
 */
public class RepeatKillException extends SeckillException{
    // 程式碼省略，請參照專案的原始碼
	... ...
}

```

## 實現Service介面

`com.nnngu.service`包下建立`SeckillServiceImpl.java`類，具體程式碼請參照專案的原始碼。

在這裡我們捕獲了執行時異常，這樣做的原因就是Spring的事務預設發生了RuntimeException才會回滾，可以檢測出來的異常是不會導致事務的回滾的，這樣的目的就是你明知道這裡會發生異常，所以你一定要進行處理。如果只是為了讓編譯透過的話，那捕獲異常也沒意思，所以這裡要注意事務的回滾。

然後我們還發現這裡存在硬編碼的現象，就是返回各種字元常量，例如秒殺成功，秒殺失敗等等，這些字串是可以被重複使用的，而且這樣維護起來也不方便，要到處去類裡面尋找這樣的字串，所有我們使用列舉類來管理這樣狀態，在con.nnngu包下建立enums包，專門放置列舉類，然後再建立SeckillStatEnum列舉類。

列舉類`SeckillStatEnum.java`的程式碼請參照專案的原始碼。

## 注入Service

在`resources/spring`下建立`applicationContext-service.xml`檔案，用來配置`Service`層

`applicationContext-service.xml`的程式碼請參照專案的原始碼。

在這裡開啟了基於註解的事務，常見的事務操作有以下幾種方法：

* 在Spring早期版本中是使用ProxyFactoryBean+XMl方式來配置事務。

* 在Spring配置檔案使用tx:advice+aop名稱空間，好處就是一次配置永久生效，你無須去關心中間出的問題，不過出錯了你很難找出在哪裡出了問題。

* 註解@Transactional的方式，註解可以在方法定義，介面定義，類定義。可以在public方法上，但是不能註解在private、final、static等方法上，因為Spring的事務管理預設是使用cglib動態代理的：
  * private方法因為訪問許可權限制，無法被子類覆蓋
  * final方法無法被子類覆蓋
  * static時類級別的方法，無法被子類覆蓋
  * protected方法可以被子類覆蓋，因此可以被動態位元組碼增強


### 不能被Spring AOP事務增強的方法

序號 |	動態代理策略 |	不能被事務增強的方法
:-: | :-: | :-: 
1	| 基於JDK的動態代理	 | 除了`public`以外的所有方法，並且 `public static` 的方法也不能被增強
2	| 基於cglib的動態代理 |  `private`，`static`，`final` 的方法

## Service層的測試

新增測試類`SeckillServiceImplTest.java`，如下圖：

![][2]

`SeckillServiceImplTest.java`的程式碼請參照專案的原始碼。

### 測試結果：

測試的方法：`public void getSeckillList()`

測試結果如下圖：

![][3]

到此，我們成功完成了Service層開發及測試。

### 下一篇：[03 Java高併發秒殺專案之web層](https://github.com/nnngu/LearningNotes/blob/master/nguSeckill/03%20Java%E9%AB%98%E5%B9%B6%E5%8F%91%E7%A7%92%E6%9D%80%E9%A1%B9%E7%9B%AE%E4%B9%8Bweb%E5%B1%82.md)


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517159588127.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/28/1517104665712.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517162538500.jpg