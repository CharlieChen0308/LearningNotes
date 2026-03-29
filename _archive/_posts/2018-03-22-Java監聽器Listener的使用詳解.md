---
categories: JavaWeb
description: 本文主要介紹各種Java監聽器以及他們的使用。
---

監聽器用於監聽Web應用中某些物件的建立、銷燬、增加，修改，刪除等動作的發生，然後作出相應的響應處理。當監聽範圍的物件的狀態發生變化的時候，伺服器自動呼叫監聽器物件中的方法。常用於統計網站線上人數、系統載入時進行資訊初始化、統計網站的訪問量等等。

## 分類：

### 按監聽的物件劃分

可以分為：

* ServletContext物件的監聽器
* HttpSession物件的監聽器
* ServletRequest物件的監聽器

### 按監聽的事件劃分

可以分為：

* 物件自身的建立和銷燬的監聽器
* 物件中屬性的建立和消除的監聽器
* session中的某個物件的狀態變化的監聽器

## 示例：用監聽器統計網站的線上人數

**原理：每當有一個訪問連線到伺服器時，伺服器就會建立一個session來管理會話。那麼我們就可以透過統計session的數量來獲得當前線上人數。所以這裡用到的是HttpSessionListener。**

1、建立監聽器類，實現HttpSessionListener介面，並重寫監聽器類中的方法。程式碼如下：

```java
public class onLineCount implements HttpSessionListener {

    public int count=0;//記錄session的數量
    public void sessionCreated(HttpSessionEvent arg0) {//監聽session的建立
        count++;
        arg0.getSession().getServletContext().setAttribute("Count", count);

    }

    @Override
    public void sessionDestroyed(HttpSessionEvent arg0) {//監聽session的撤銷
        count--;
        arg0.getSession().getServletContext().setAttribute("Count", count);
    }

}
```

2、在web.xml中配置監聽器。注意：監聽器>過濾器>servlet，配置的時候要注意先後順序

```xml
<listener>
  <listener-class>com.nnngu.listener.onLineCount</listener-class>
</listener>
```

如果使用 Servlet3.0 以上版本，監聽器的配置可以直接在程式碼中透過註解來完成，無需在 web.xml 中再配置。如下：

```java
@WebListener   //在此註明以下類是監聽器
public class onLineCount implements HttpSessionListener {

    public int count=0;
    public void sessionCreated(HttpSessionEvent arg0) {
        count++;
        arg0.getSession().getServletContext().setAttribute("Count", count);

    }

    @Override
    public void sessionDestroyed(HttpSessionEvent arg0) {
        count--;
        arg0.getSession().getServletContext().setAttribute("Count", count);
    }
```

3、在需要顯示線上人數的地方透過`session.getAttribute("Count")`即可獲取線上人數值。

## 附：常用監聽器

**除了上面監聽session建立與銷燬的listener外，還有以下幾個常用的監聽器。**

1、監聽session屬性的增加、移除以及屬性值改變的HttpSessionAttributeListener，如下圖：

![][1]

2、監聽web上下文的初始化（伺服器已準備好接收請求）與銷燬的ServletContextListener，如下圖：

![][2]

3、監聽web上下文屬性的增加、刪除、屬性值變化的ServletContextAttributeListener，如下圖：

![][3]

4、監聽request的建立與銷燬的ServletRequestListener，如下圖：

![][4]

5、監聽request的屬性的增加、刪除、屬性值變化的ServletRequestAttributeListener，如下圖：

![][5]

























---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-22-Java%E7%9B%91%E5%90%AC%E5%99%A8Listener%E7%9A%84%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.md](https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-22-Java%E7%9B%91%E5%90%AC%E5%99%A8Listener%E7%9A%84%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.md)


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/22/1521712268679.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/22/1521712339086.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/22/1521712392621.jpg
  [4]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/22/1521712435429.jpg
  [5]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/22/1521712488700.jpg