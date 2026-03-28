# 03 Spring的父子容器

## 1、概念理解和知識鋪墊

在Spring整體框架的核心概念中，容器是核心思想，就是用來管理Bean的整個生命週期的，而在一個專案中，容器不一定只有一個，Spring中可以包括多個容器，而且容器有上下層關係，目前最常見的一種場景就是在一個專案中引入Spring和SpringMVC這兩個框架，那麼它其實就是兩個容器，Spring是父容器，SpringMVC是其子容器，並且在父容器中註冊的Bean對於子容器是可見的，而在子容器中註冊的Bean對於父容器是不可見的，也就是子容器可以看見父容器中的註冊的Bean，反之就不行。

我們可以使用統一的如下註解配置來對Bean進行批次註冊，而不需要再給每個Bean單獨使用xml的方式進行配置。

```xml
<context:component-scan base-package="com.nnngu" />
```

從Spring提供的參考手冊中我們得知該配置的功能是掃描配置的base-package包下的所有使用了\@Component註解的類，並且將它們自動註冊到容器中，同時也掃描\@Controller，\@Service，\@Respository這三個註解，因為他們是繼承自\@Component。

在專案中我們經常見到還有如下這個配置，其實有了上面的配置，這個是可以省略掉的，因為上面的配置會預設開啟以下配置。以下配置會預設宣告瞭\@Required、\@Autowired、 \@PostConstruct、\@PersistenceContext、\@Resource、\@PreDestroy等註解。

```xml
<context:annotation-config/>
```

另外，還有一個和SpringMVC相關如下配置，經過驗證，這個是SpringMVC必須要配置的，因為它宣告瞭\@RequestMapping、\@RequestBody、\@ResponseBody等。並且，該配置預設載入很多的引數繫結方法，比如json轉換解析器等。

```xml
<mvc:annotation-driven />
```

## 2、使用場景分析

我們共有Spring和SpringMVC兩個容器，它們的配置檔案分別為applicationContext.xml和applicationContext-MVC.xml。

1. 在applicationContext.xml中配置了`<context:component-scan base-package=“com.nnngu" />`，負責所有需要註冊的Bean的掃描和註冊工作。

2. 在applicationContext-MVC.xml中配置`<mvc:annotation-driven />`，負責SpringMVC相關注解的使用。

3. 啟動專案我們發現SpringMVC無法進行跳轉，將log的日誌列印級別設定為DEBUG進行除錯，發現SpringMVC容器中的請求好像沒有對映到具體controller中。

4. 在applicationContext-MVC.xml中配置`<context:component-scan base-package=“com.nnngu" />`，重啟後，驗證成功，springMVC跳轉有效。

下面我們來檢視具體原因，翻看原始碼，從SpringMVC的DispatcherServlet開始往下找，我們發現SpringMVC初始化時，會尋找SpringMVC容器中的所有使用了\@Controller註解的Bean，來確定其是否是一個handler。1,2兩步的配置使得當前springMVC容器中並沒有註冊帶有\@Controller註解的Bean，而是把所有帶有\@Controller註解的Bean都註冊在Spring這個父容器中了，所以springMVC找不到處理器，不能進行跳轉。

而在第4步配置中，SpringMVC容器中也註冊了所有帶有\@Controller註解的Bean，故SpringMVC能找到處理器進行處理，從而正常跳轉。

## 3、官方推薦配置

在實際工程中會包括很多配置，我們按照官方推薦根據不同的業務模組來劃分不同容器中註冊不同型別的Bean：Spring父容器負責所有其他非\@Controller註解的Bean的註冊，而SpringMVC只負責\@Controller註解的Bean的註冊，使得他們各負其責、明確邊界。配置方式如下：

1. 在applicationContext.xml中配置:

```xml
<!-- Spring容器中註冊非@Controller註解的Bean -->
<context:component-scan base-package="com.nnngu">
   <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

2. applicationContext-MVC.xml中配置

```xml
<!-- SpringMVC容器中只註冊帶有@Controller註解的Bean -->
<context:component-scan base-package="com.nnngu" use-default-filters="false">
   <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller" />
</context:component-scan>
```

## 4、總結

我們在清楚了Spring和SpringMVC的父子容器關係、以及掃描註冊的原理以後，根據官方建議，就可以很好把不同型別的Bean分配到不同的容器中進行管理。出現Bean找不到或者SpringMVC不能跳轉以及事務的配置失效的問題時，我們就可以很快的定位以及解決問題了。













---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/Spring/03%20Spring%E7%9A%84%E7%88%B6%E5%AD%90%E5%AE%B9%E5%99%A8.md](https://github.com/nnngu/LearningNotes/blob/master/Spring/03%20Spring%E7%9A%84%E7%88%B6%E5%AD%90%E5%AE%B9%E5%99%A8.md)