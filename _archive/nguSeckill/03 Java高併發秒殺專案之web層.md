# 03 Java高併發秒殺專案之web層

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

專案原始碼：[https://github.com/nnngu/nguSeckill](https://github.com/nnngu/nguSeckill)  

---

## 前端互動流程設計

對於一個系統，需要產品經理、前端工程師和後端工程師的參與，產品經理將使用者的需求做成一個開發文件交給前端工程師和後端工程師，前端工程師為系統完成頁面的開發，後端工程師為系統完成業務邏輯的開發。對於我們這個秒殺系統，它的前端互動流程設計如下圖：

![][1]

這個流程圖就告訴了我們詳情頁的流程邏輯，前端工程師根據這個流程圖設計頁面，而我們後端工程師根據這個流程圖開發我們對應的程式碼。前端互動流程是系統開發中很重要的一部分，接下來進行`Restful`介面設計的學習。

## Restful介面設計學習

什麼是Restful？它就是一種優雅的URL表述方式，用來設計我們資源的訪問URL。透過這個URL的設計，我們就可以很自然的感知到這個URL代表的是哪種業務場景或者什麼樣的資料或資源。基於Restful設計的URL，對於我們介面的使用者、前端、web系統或者搜尋引擎甚至是我們的使用者，都是非常友好的。關於Restful的瞭解大家去網上一搜一大把，我這裡就不再做介紹了。下面看看我們這個秒殺系統的URL設計：

![][2]

接下來基於上述資源介面來開始對Spring MVC框架的使用。

## 配置Spring MVC框架

在`web.xml`檔案裡面引入`DispatcherServlet`：

![][3]

`web.xml`裡面的程式碼請參照專案的原始碼。

## 新增 applicationContext-web.xml

新增 `applicationContext-web.xml`，在下圖所示的位置。

![][11]

 `applicationContext-web.xml`裡面的程式碼請參照專案的原始碼。
 
這樣我們便完成了Spring MVC的相關配置(即將Spring MVC框架整合到了我們的專案中)，接下來就要基於Restful介面進行我們專案的控制器 `SeckillController` 的開發工作了。

## 編寫 SeckillController 

控制器中的每一個方法都對應我們系統中的一個資源URL，其設計應該遵循Restful介面的設計風格。

建立控制器`SeckillController.java`，如下圖：

![][4]

`SeckillController.java`裡面的程式碼請參照專案的原始碼。

`SeckillController.java`中的方法完全是對照Service介面方法進行開發的，第一個方法用於訪問我們商品的列表頁，第二個方法訪問商品的詳情頁，第三個方法用於返回一個json資料，資料中封裝了我們商品的秒殺地址，第四個方法用於封裝使用者是否秒殺成功的資訊，第五個方法用於返回系統當前時間。程式碼中涉及到一個將返回秒殺商品地址封裝為json資料的類，即`SeckillResult`，在dto包中建立它，如下:

## 建立一個全域性ajax請求返回類，返回json

建立`SeckillResult.java`，如下圖：

![][5]

`SeckillResult.java`裡面的程式碼請參照專案的原始碼。

到此，控制器的開發任務完成，接下來進行我們的頁面開發。

## 頁面的編寫

專案的前端頁面是由`Bootstrap`開發的，所以我們要先去下載`Bootstrap`或者是使用線上CDN。

使用線上CDN的方法：

```html
<!-- Bootstrap 核心 CSS 檔案 -->
<link rel="stylesheet" href="https://cdn.bootcss.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">

<!-- 可選的 Bootstrap 主題檔案（一般不用引入） -->
<link rel="stylesheet" href="https://cdn.bootcss.com/bootstrap/3.3.7/css/bootstrap-theme.min.css" integrity="sha384-rHyoN1iRsVXV4nD0JutlnGaslCJuC7uwjduW9SVrLvRYooPp2bWYgmgJQIXwl/Sp" crossorigin="anonymous">

<!-- Bootstrap 核心 JavaScript 檔案 -->
<script src="https://cdn.bootcss.com/bootstrap/3.3.7/js/bootstrap.min.js" integrity="sha384-Tc5IQib027qvyjSMfHjOMaLkfuWVxZxUPnCJA7l2mCWNIpG9mGCD8wGNIcPD7Txa" crossorigin="anonymous"></script>
```

為了方便我們本地除錯，我在專案裡使用的是本地的`Bootstrap`。

### 步驟：

1. 下載`JQuery`，因為`Bootstrap`就是依賴`JQuery`的

2. 下載`Bootstrap`

3. 下載一個倒計時外掛`jquery.countdown.min.js` ，再下載一個操作`Cookie`外掛`jquery.cookie.min.js` 如圖放置：

![][6]

4. 編寫一個公共的頭部`jsp`檔案，位於`WEB-INF/jsp/common`下的`head.jsp`，如下圖：

![][7]

`head.jsp`裡面的程式碼請參照專案的原始碼。

5. 編寫一個公共的`jstl`標籤庫檔案`tag.jsp`，在下圖所示的位置。

![][8]

`tag.jsp`裡面的程式碼請參照專案的原始碼。

6. 編寫列表頁面`list.jsp`，在下圖所示的位置。

![][9]

`list.jsp`裡面的程式碼請參照專案的原始碼。

7. 編寫秒殺詳情頁面`detail.jsp`，在下圖所示的位置。

![][10]

`detail.jsp`裡面的程式碼請參照專案的原始碼。
 
 ## 新增 seckill.js 檔案
 
 新增 `seckill.js` 檔案，在下圖所示的位置。
 
 ![][12]
 
 `seckill.js` 裡面的程式碼請參照專案的原始碼。
 
 ## 執行專案
 
 執行專案，部署到`tomcat`，在瀏覽器位址列輸入 `http://localhost:8080/seckill/list`，敲回車，即可看到如下圖的介面：
 
 ![][13]
 
點選相應商品後面的詳情頁連結即可檢視該商品是否開啟秒殺、以及秒殺該商品等活動。
 
到此，我們成功完成了web層的開發。但一個秒殺系統，往往是會有成千上萬的人進行參與，我們目前的系統是抗不起多少高併發操作的，所以後面我們會對本系統進行高併發的最佳化。請檢視我的下一篇文章。

### 下一篇：[04 Java高併發秒殺專案之高併發最佳化](https://github.com/nnngu/LearningNotes/blob/master/nguSeckill/04%20Java高併發秒殺專案之高併發最佳化.md)
 
 
 
 
 
 


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/30/1517264048018.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/30/1517269670065.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517174370108.jpg
  [4]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517175668689.jpg
  [5]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517175867220.jpg
  [6]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517177850911.jpg
  [7]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517178335741.jpg
  [8]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517179505188.jpg
  [9]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517179663648.jpg
  [10]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517180009140.jpg
  [11]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517182288815.jpg
  [12]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517183946959.jpg
  [13]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/29/1517184316644.jpg