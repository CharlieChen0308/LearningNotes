---
categories: JavaBasis
description: 代理模式是常用設計模式的一種，我們在軟體設計時常用的代理一般是指靜態代理，也就是在程式碼中顯式指定的代理。
---

## 一、靜態代理

代理模式是常用設計模式的一種，我們在軟體設計時常用的代理一般是指靜態代理，也就是在程式碼中顯式指定的代理。

靜態代理由業務實現類、業務代理類兩部分組成。業務實現類負責實現主要的業務方法，業務代理類負責對呼叫的業務方法作攔截、過濾、預處理。在需要呼叫業務時，不是直接透過業務實現類來呼叫的，而是透過業務代理類的同名方法來呼叫被處理過的業務方法。

### 靜態代理的實現：

1、首先定義一個介面，說明業務邏輯。 

```java
package com.nnngu.dao;      
    /** 
     * 定義一個賬戶介面 
     */  
    public interface Count {  
        // 查詢賬戶
        public void queryCount();  
      
        // 修改賬戶  
        public void updateCount();  
      
    }
```

2、然後定義業務實現類，實現業務邏輯介面。

```java
import com.nnngu.dao.Count;    
/** 
 * 委託類(包含業務邏輯) 
 */  
public class CountImpl implements Count {  
  
    @Override  
    public void queryCount() {  
        System.out.println("檢視賬戶...");   
    }  
  
    @Override  
    public void updateCount() {  
        System.out.println("修改賬戶...");    
    }  
  
}
```

3、定義業務代理類：在代理類中建立一個業務實現類的物件來呼叫具體的業務方法；

在代理類中實現業務邏輯介面中的方法時：①進行預處理操作、②透過業務實現類的物件呼叫真正的業務方法、③進行呼叫後的操作。

```java
public class CountProxy implements Count {  
    private CountImpl countImpl;  // 組合一個業務實現類的物件來進行真正的業務方法的呼叫
  
    /** 
     * 覆蓋預設構造器 
     *  
     * @param countImpl 
     */  
    public CountProxy(CountImpl countImpl) {  
        this.countImpl = countImpl;  
    }  
  
    @Override  
    public void queryCount() {  
        System.out.println("查詢賬戶的預處理——————");  
        // 呼叫真正的查詢賬戶方法
        countImpl.queryCount();  
        System.out.println("查詢賬戶之後————————");  
    }  
  
    @Override  
    public void updateCount() {  
        System.out.println("修改賬戶之前的預處理——————");  
        // 呼叫真正的修改賬戶操作
        countImpl.updateCount();  
        System.out.println("修改賬戶之後——————————");  
  
    }  
  
}
```

4、在使用時，首先建立業務實現類物件，然後把業務實現類物件當作構造引數建立一個代理類物件，最後透過代理類物件進行業務方法的呼叫。

```java
public static void main(String[] args) {  

        CountImpl countImpl = new CountImpl();  
        CountProxy countProxy = new CountProxy(countImpl);  
        countProxy.updateCount();  
        countProxy.queryCount();  
  
    }
```

靜態代理的缺點很明顯：一個代理類只能對一個業務介面的實現類進行包裝，如果有多個業務介面的話就要定義很多實現類和代理類才行。而且，如果代理類對業務方法的預處理、呼叫後操作都是一樣的（比如：呼叫前輸出提示、呼叫後自動關閉連線），則多個代理類就會有很多重複程式碼。這時我們可以定義這樣一個代理類，它能代理所有實現類的方法呼叫：根據傳進來的業務實現類和方法名進行具體呼叫。——這就是動態代理。

## 二、JDK動態代理

JDK動態代理所用到的代理類在程式呼叫到代理類物件時才由JVM真正建立，JVM根據傳進來的 業務實現類物件 以及 方法名 ，動態地建立了一個代理類的class檔案並被位元組碼引擎執行，然後透過該代理類物件進行方法呼叫。我們需要做的，只需指定代理類的預處理、呼叫後操作即可。

1、首先，定義業務邏輯介面

```java
public interface BookFacade {  
    public void addBook();  
}
```

2、然後，實現業務邏輯介面建立業務實現類

```java
public class BookFacadeImpl implements BookFacade {   
    @Override  
    public void addBook() {  
        System.out.println("增加圖書方法。。。");  
    }  
  
}
```

3、最後，實現 呼叫管理介面`InvocationHandler`  建立動態代理類

```java
public class BookFacadeProxy implements InvocationHandler {  
    private Object target; // 這是業務實現類物件，用來呼叫具體的業務方法 
	
    /** 
     * 繫結業務物件並返回一個代理類  
     */  
    public Object bind(Object target) {  
        this.target = target;  // 接收業務實現類物件引數

       // 透過反射機制，建立一個代理類物件例項並返回。
       // 建立代理物件時，需要傳遞該業務類的類載入器、介面、handler實現類
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),  
                target.getClass().getInterfaces(), this); 
	}  
	
    /** 
     * 包裝呼叫方法：進行預處理、呼叫後處理 
     */  
    public Object invoke(Object proxy, Method method, Object[] args)  
            throws Throwable {  
        Object result=null;  

        System.out.println("預處理操作——————");  
		
        // 呼叫真正的業務方法  
        result=method.invoke(target, args);  

        System.out.println("呼叫後處理——————");  
        return result;  
    }  
  
}
```

4、在使用時，首先建立一個業務實現類物件和一個代理類物件，然後定義介面引用，並用代理物件.bind(業務實現類物件)的返回值進行賦值。最後透過介面引用呼叫業務方法即可。

```java
public static void main(String[] args) {  
        BookFacadeImpl bookFacadeImpl=new BookFacadeImpl();
        BookFacadeProxy proxy = new BookFacadeProxy();  
        BookFacade bookfacade = (BookFacade) proxy.bind(bookFacadeImpl);  
        bookfacade.addBook();  
    }
```

JDK動態代理的代理物件在建立時，需要使用業務實現類所實現的介面作為引數。如果業務實現類是沒有實現介面而是直接定義業務方法的話，就無法使用JDK動態代理了。並且，如果業務實現類中新增了介面中沒有的方法，這些方法是無法被代理的（因為無法被呼叫）。

## 三、cglib動態代理

**cglib是針對類來實現代理的，原理是對指定的業務類生成一個子類，並覆蓋其中的業務方法來實現代理。因為採用的是繼承，所以不能對final修飾的類進行代理。**

1、首先定義業務類，無需實現介面

```java
public class BookFacadeImpl2 {  
    public void addBook() {  
        System.out.println("新增圖書...");  
    }  
}
```

2、實現 `MethodInterceptor` 方法代理介面，建立代理類

```java
public class BookFacadeCglib implements MethodInterceptor {  
    private Object target; // 業務類物件，供代理方法中進行真正的業務方法呼叫
  
    // 相當於JDK動態代理中的繫結
    public Object getInstance(Object target) {  
        this.target = target;  // 給業務物件賦值
        Enhancer enhancer = new Enhancer(); // 建立加強器，用來建立動態代理類
        enhancer.setSuperclass(this.target.getClass());  // 為加強器指定要代理的業務類
        // 設定回撥：對於代理類上所有方法的呼叫，都會呼叫CallBack，而Callback則需要實現intercept() 方法進行攔截
        enhancer.setCallback(this); 
       // 建立動態代理類物件並返回  
       return enhancer.create(); 
    }
	
    // 實現回撥方法 
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable { 
        System.out.println("預處理——————");
        proxy.invokeSuper(obj, args); // 呼叫業務類（父類中）的方法
        System.out.println("呼叫後操作——————");
        return null; 
    }
```

3、建立業務類物件和代理類物件，然後透過  代理類物件.getInstance(業務類物件)  返回一個動態代理類物件（它是業務類的子類，可以用業務類引用指向它）。最後透過動態代理類物件進行方法呼叫。

```java
public static void main(String[] args) {      
        BookFacadeImpl2 bookFacade=new BookFacadeImpl2()；
        BookFacadeCglib  cglib=new BookFacadeCglib();  
        BookFacadeImpl2 bookCglib=(BookFacadeImpl2)cglib.getInstance(bookFacade);  
        bookCglib.addBook();  
    }
```

## 四、總結

靜態代理是透過在程式碼中顯式定義一個業務實現類一個代理，在代理類中對同名的業務方法進行包裝，使用者透過代理類呼叫被包裝過的業務方法；

JDK動態代理是透過介面中的方法名，在動態生成的代理類中呼叫業務實現類的同名方法；

CGlib動態代理是透過繼承業務類，生成的動態代理類是業務類的子類，透過重寫業務方法進行代理；




















---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/Java%20Basis/017%20Java%E4%B8%AD%E7%9A%84%E9%9D%99%E6%80%81%E4%BB%A3%E7%90%86%E3%80%81JDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E3%80%81cglib%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86.md](https://github.com/nnngu/LearningNotes/blob/master/Java%20Basis/017%20Java%E4%B8%AD%E7%9A%84%E9%9D%99%E6%80%81%E4%BB%A3%E7%90%86%E3%80%81JDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E3%80%81cglib%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86.md)