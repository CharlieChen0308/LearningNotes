---
categories: JavaWeb
description: 本文主要介紹Java過濾器以及它的使用。
---

## 過濾器

過濾器是處於客戶端與伺服器資原始檔之間的一道過濾網，在訪問資原始檔之前，透過一系列的過濾器對請求進行修改、判斷等，把不符合規則的請求在中途攔截或修改。也可以對響應進行過濾，攔截或修改響應。

如下圖，瀏覽器發出的請求先遞交給第一個filter進行過濾，符合規則則放行，遞交給filter鏈中的下一個過濾器進行過濾。過濾器在鏈中的順序與它在web.xml中配置的順序有關，配置在前的則位於鏈的前端。當請求透過了鏈中所有過濾器後就可以訪問資原始檔了，如果不能透過，則可能在中間某個過濾器中被處理掉。

![][1]

過濾器一般用於登入許可權驗證、資源訪問許可權控制、敏感詞彙過濾、字元編碼轉換等等操作，便於程式碼的重用，不必每個servlet中還要進行相應的操作。

## 過濾器的簡單應用：

1、新建一個class，實現介面Filter（注意：是javax.servlet中的Filter）。

2、重寫過濾器的doFilter(request，response，chain)方法。另外兩個init()、destroy()方法一般不需要重寫。在doFilter方法中進行過濾操作。

3、在web.xml中配置過濾器。這裡要謹記一條原則：在web.xml中，監聽器\>過濾器\>servlet。也就是說web.xml中監聽器配置在過濾器之前，過濾器配置在servlet之前，否則會出錯。

```xml
<filter>  
    <filter-name>loginFilter</filter-name>//過濾器名稱  
    <filter-class>com.nnngu.filter.loginFilter</filter-class>//過濾器類的包路徑
	<init—param> //可選 
		<param—name>引數名</param-name>//過濾器初始化引數
		<param-value>引數值</param-value>  
	</init—pamm>  
</filter> 
 
<filter-mapping>//過濾器對映  
    <filter-name>loginFilter</filter-name>  
    <url—pattern>指定過濾器作用的範圍</url-pattern>
</filter-mapping>
```

`<url-pattren>`處定義過濾器作用的範圍。一般有以下規則：

1、作用與所有web資源：`<url—pattern>/*</url-pattern>`。則客戶端請求訪問任意資原始檔時都要經過過濾器的過濾，透過則可以訪問，否則不能訪問。

2、作用於某一資料夾下所有檔案：`<url—pattern>/dir/*</url-pattern>`

3、作用於某一種型別的檔案：`<url—pattern>*.副檔名</url-pattern>`。比如`<url—pattern>*.jsp</url-pattern>`過濾所有對jsp檔案的訪問請求。

4、作用於某一資料夾下某一型別檔案：`<url—pattern>/dir/*.副檔名</url-pattern>`

如果一個過濾器需要過濾多種檔案，則可以配置多個`<filter-mapping>`，一個mapping定義一個url-pattern來定義過濾規則。如下：

```xml
<filter>
  <filter-name>loginFilter</filter-name>
  <filter-class>com.nnngu.filter.loginFilter</filter-class>
</filter>
<filter-mapping>
  <filter-name>loginFilter</filter-name>
  <url-pattern>*.jsp</url-pattern>
</filter-mapping>
<filter-mapping>
  <filter-name>loginFilter</filter-name>
  <url-pattern>*.do</url-pattern>
</filter-mapping>
```

### 例1：用過濾器實現登入驗證，沒登入則駁回訪問請求並重定向到登入頁面。

```java
public void doFilter(ServletRequest arg0, ServletResponse arg1,
            FilterChain arg2) throws IOException, ServletException {
        
        HttpServletRequest request=(HttpServletRequest) arg0;
        HttpServletResponse response=(HttpServletResponse) arg1;
        HttpSession session=request.getSession();
        
        String path=request.getRequestURI();
        
        Integer uid=(Integer)session.getAttribute("userid");
        
        if(path.indexOf("/login.jsp")>-1){//登入頁面不過濾
            arg2.doFilter(arg0, arg1);//遞交給下一個過濾器
            return;
        }
        if(path.indexOf("/register.jsp")>-1){//註冊頁面不過濾
            arg2.doFilter(request, response);
            return;
        }
        
        if(uid!=null){//已經登入
            arg2.doFilter(request, response);//放行，遞交給下一個過濾器
            
        }else{
            response.sendRedirect("login.jsp");
        }

    }
```

### 例2：設定字元編碼

```java
public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {
          HttpServletRequest request2=(HttpServletRequest) request;
          HttpServletResponse response2=(HttpServletResponse) response;
          
          request2.setCharacterEncoding("UTF-8");  
          response2.setCharacterEncoding("UTF-8"); 
          
          chain.doFilter(request, response); 

    }
```





















---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-22-Java%E8%BF%87%E6%BB%A4%E5%99%A8Filter%E7%9A%84%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.md](https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-22-Java%E8%BF%87%E6%BB%A4%E5%99%A8Filter%E7%9A%84%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.md)


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/22/1521719259966.jpg