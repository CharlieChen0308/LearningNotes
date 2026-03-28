---
categories: JavaWeb
description: 本文總結 JSP 和 Servlet 的工作原理和生命週期。
---

JSP的英文名叫Java Server Pages，翻譯為中文是Java伺服器頁面的意思，其底層就是一個簡化的Servlet設計，是由sum公司主導參與建立的一種動態網頁技術標準。Servlet 就是 Java 程式語言中的一個類，它被用來擴充套件伺服器的效能。

## JSP的執行過程和生命週期

JSP的執行過程和生命週期，如下圖：

![][1]

## Servlet的生命週期

Servlet的生命週期主要分為以下三個階段：一是容器初始化。即`init()`，二是呼叫`service()`方法，判斷客戶端請求的方式。最後是銷燬，呼叫`destroy()`方法。

詳細的 Servlet 生命週期示意圖如下：

![][2]

## JSP與Servlet的優缺點比較

* JSP優點：提高程式碼的可複用性、將HTML程式碼進行分離、程式利於開發維護。
* JSP缺點：不容易跟蹤與排錯。不能處理流程和業務邏輯。
* Servlet優點是響應客戶端的請求，根據請求動態響應，最大的優點是作為一個服務，控制程式的流向，過濾等。MVC中的C就是servlet。
* Servlet缺點：Servlet在表示邏輯上對於檢視的表示相對於JSP麻煩太多，在負責顯示工作完成並生成頁面上，JSP更優。

## 編寫第一個JSP檔案

編寫第一個JSP檔案，為解決跳轉路徑問題，可在頭部加上

```jsp
<%    
String path = request.getContextPath();    
String basePath = request.getScheme() + "://" + request.getServerName() + ":" + request.getServerPort() + path + "/";
%>
```

如下圖：

![][3]

## 編寫第一個Servlet程式

編寫第一個Servlet程式，這裡使用Servlet3.0，不需在web.xml中配置，可自己設定名稱，但必須要與頁面中form表單中的action對應。如下圖：

![][4]


















---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-22-JSP%20%E5%92%8C%20Servlet%20%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%92%8C%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.md](https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-22-JSP%20%E5%92%8C%20Servlet%20%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%92%8C%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.md)


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/22/1521695280116.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/22/1521696361730.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/22/1521696030375.jpg
  [4]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/22/1521696228346.jpg