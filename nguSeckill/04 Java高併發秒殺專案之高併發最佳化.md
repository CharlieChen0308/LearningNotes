# 04 Java高併發秒殺專案之高併發最佳化

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

專案原始碼：[https://github.com/nnngu/nguSeckill](https://github.com/nnngu/nguSeckill)  

---

## 關於併發

併發性上不去是因為當多個執行緒同時訪問一行資料時，產生了事務，因此產生寫鎖，當一個獲取了事務的執行緒把鎖釋放，另一個排隊執行緒才能拿到寫鎖，QPS(Query Per Second每秒查詢率)和事務執行的時間有密切關係，事務執行時間越短，併發性越高，這也是要將費時的 IO 操作移出事務的原因。

### 專案中的高併發發生在哪？

下圖中，紅色的部分就表示會發生高併發的地方，綠色部分表示對於高併發沒有影響。

![][1]

### 為什麼要單獨獲取系統時間？

這是為了我們的秒殺系統的最佳化做鋪墊。比如在秒殺還未開始的時候，使用者大量重新整理秒殺商品詳情頁面是很正常的情況，這時候秒殺還未開始，大量的請求傳送到伺服器會造成不必要的負擔。

![][2]

我們將這個詳情頁放置到CDN中，這樣使用者在訪問該頁面時就不需要訪問我們的伺服器了，起到了降低伺服器壓力的作用。而CDN中儲存的是靜態化的詳情頁和一些靜態資源（css，js等），這樣我們就拿不到系統的時間來進行秒殺時段的控制，所以我們需要單獨設計一個請求來獲取我們伺服器的系統時間。

### CDN（Content Delivery Network）的理解

![][3]

### 獲取系統時間不需要最佳化

因為Java訪問一次記憶體（Cacheline）大約10ns，1s=10億ns，也就是如果不考慮GC，這個操作1s可以做1億次。

### 秒殺地址介面分析

* 無法使用CDN快取，因為CDN適合請求對應的資源不變化的，比如靜態資源、JavaScript；秒殺地址返回的資料是變化的，不適合放在CDN快取；

* 適合服務端快取：Redis等，1秒鐘可以承受10萬QPS。多個Redis組成叢集，可以到100萬個QPS。所以後端快取可以用業務系統控制。

### 秒殺地址介面最佳化

![][4]

### 秒殺操作最佳化分析

* 無法使用cdn快取

* 後端快取困難： 庫存問題

* 一行資料競爭：熱門商品

大部分寫的操作和核心操作無法使用CDN，也不可能在快取中減庫存。你在Redis中減庫存，那麼使用者也可能透過快取來減庫存，這樣庫存會不一致，所以要透過mysql的事務來保證一致性。

比如一個熱門商品所有人都在搶，那麼會在同一時間對資料表中的一行資料進行大量的`update set`操作。

行級鎖在commit之後才釋放，所以最佳化方向是減少行級鎖的持有時間。

### 延遲問題很關鍵

* 同城機房網路（0.5ms~2ms），最高併發性是1000qps。

* update後JVM -GC(垃圾回收機制)大約50ms，最高併發性是20qps。併發性越高，GC就越可能發生，雖然不一定每次都會發生，但一定會發生。

* 異地機房，比如北京到上海之間的網路延遲，經過計算大概13~20ms。

![][5]

### 如何判斷update更新庫存成功？

有兩個條件：

* update自身沒報錯；

* 客戶端確認update影響記錄數

最佳化思路：把客戶端邏輯放到MySQL服務端，避免網路延遲和GC影響


### 如何把客戶端邏輯放到MySQL服務端

有兩種方案：

* 定製SQL方案，在每次update後都會自動提交，但需要修改MySQL原始碼，成本很高，不是大公司（BAT等）一般不會使用這種方法。

* 使用儲存過程：整個事務在MySQL端完成，用儲存過程寫業務邏輯，服務端負責呼叫。

**接下來先分析第一種方案**

![][6]

![][7]

**根據上圖的成本分析，我們的秒殺系統採用第二種方案，即使用儲存過程。**

### 最佳化總結

* 前端控制。暴露介面，按鈕防重複（點選一次按鈕後就變成灰色，禁止重複點選按鈕）

* 動靜態資料分離。CDN快取，後端快取

* 事務競爭最佳化。減少事務行級鎖的持有時間

## 下載安裝Redis

Redis是一個開源的、支援網路、可基於記憶體亦可持久化的日誌型、Key-Value資料庫

下載安裝Redis的步驟，搜尋引擎能找到相關的資料，本文不做展開。

下載安裝完Redis之後就可以繼續進行操作。

## 使用Java操作Redis

Java操作Redis使用的是jedis包。

在`pom.xml`新增jedis的依賴，如下圖：

![][8]

新增protostuff-core 以及protostuff-runtime 序列化jar包，如下圖：

![][9]

序列化是處理物件流的機制，就是將物件的內容進行流化，可以對流化後的物件進行讀寫操作，也可以將流化後的物件在網路間傳輸。反序列化就是將流化後的物件重新轉化成原來的物件。

在Java中內建了序列化機制，透過implements Serializable來標識一個物件實現了序列化介面，不過其效能並不高。

### 建立操作Redis的dao類

原本查詢秒殺商品時是透過主鍵直接去資料庫查詢的，選擇將資料快取在Redis，在查詢秒殺商品時先去Redis快取中查詢，以此降低資料庫的壓力。如果在快取中查詢不到資料再去資料庫中查詢，再將查詢到的資料放入Redis快取中，這樣下次就可以直接去快取中直接查詢到。

新增`RedisDao.java`檔案，位於下圖所示的位置：

![][10]

`RedisDao.java`檔案裡面的程式碼請參照專案的原始碼。

### 在applicationContext-dao.xml中注入redisDao

在`applicationContext-dao.xml`中新增下圖所示的內容：

![][11]

### 改造exportSeckillUrl方法：

修改`SeckillServiceImpl.java`檔案中的`exportSeckillUrl`方法：

```java
/**
     * 在秒殺開啟時輸出秒殺介面的地址，否則輸出系統時間跟秒殺地址
     *
     * @param seckillId 秒殺商品Id
     * @return 根據對應的狀態返回對應的狀態實體
     */
    @Override
    public Exposer exportSeckillUrl(long seckillId) {
        
        Seckill seckill = redisDao.getSeckill(seckillId);
        if (seckill == null) {
            // 訪問資料庫讀取資料
            seckill = seckillMapper.queryById(seckillId);
            if (seckill == null) {
                return new Exposer(false, seckillId);
            } else {
                // 放入redis
                redisDao.putSeckill(seckill);
            }
        }

        // 判斷是否還沒到秒殺時間或者是過了秒殺時間
        Date startTime = seckill.getStartTime();
        Date endTime = seckill.getEndTime();
        Date nowTime = new Date();
        // 開始時間大於現在的時候說明沒有開始秒殺活動；秒殺活動結束時間小於現在的時間說明秒殺已經結束了
        if (nowTime.getTime() > startTime.getTime() && nowTime.getTime() < endTime.getTime()) {
            //秒殺開啟,返回秒殺商品的id,用給介面加密的md5
            String md5 = getMd5(seckillId);
            return new Exposer(true, md5, seckillId);
        }
        return new Exposer(false, seckillId, nowTime.getTime(), startTime.getTime(), endTime.getTime());
        
    }
```

## 簡單的最佳化

以前的實現流程：

![][12]

使用者的秒殺操作分為兩步：減庫存、插入購買明細，我們在這裡進行簡單的最佳化，就是將原本先update（減庫存）再進行insert（插入購買明細）的步驟改成：先insert再update。

![][13]

### 為什麼要先insert再update

* 首先是在更新操作的時候給行加鎖，插入並不會加鎖，如果更新操作在前，那麼就需要執行完更新和插入以後事務提交或回滾才釋放鎖。而如果插入在前，更新在後，那麼只有在更新時才會加行鎖，之後在更新完以後事務提交或回滾釋放鎖。

* 在這裡，插入是可以並行的，而更新由於會加行級鎖是序列的。

* 也就是說是如果更新在前，加鎖和釋放鎖之間兩次的網路延遲和GC，如果更新在後，則加鎖和釋放鎖之間只有一次的網路延遲和GC，也就是減少的持有鎖的時間。

* 這裡先insert並不是忽略了庫存不足的情況，而是因為insert和update是在同一個事務裡，光是insert並不一定會提交，只有在update成功才會提交，所以並不會造成過量插入秒殺成功記錄。

**去程式碼裡改造執行秒殺的`executeSeckill()`方法，最佳化效能。**

## 深度最佳化（使用儲存過程）

前邊透過調整insert和update的執行順序來實現簡單的最佳化，但依然存在著Java客戶端和伺服器通訊時的網路延遲和GC影響，我們可以將執行秒殺操作時的insert和update放到MySQL服務端的儲存過程裡，而Java客戶端直接呼叫這個儲存過程，這樣就可以避免網路延遲和可能發生的GC影響。另外，由於我們使用了儲存過程，也就用不到Spring的事務管理了，因為在儲存過程裡我們會直接啟用一個事務。

### 去MySQL的控制檯執行儲存過程`procedure.sql`裡面的程式碼

**去MySQL的控制檯執行儲存過程`procedure.sql`裡面的程式碼。**

`procedure.sql`檔案位於專案的sql目錄下。

注意點：在儲存過程中，row_count() 函式用來返回上一條 sql（delete, insert, update）影響的行數。

根據row_count() 的返回值，可以進行接下來的流程判斷：

`0`：未修改資料；

`>0`：表示修改的行數；

`<0`：表示SQL錯誤或未執行修改SQL

## 修改原始碼以呼叫儲存過程

在`SeckillMapper.java`介面中宣告`killByProcedure()`方法

```java
    /**
     * 使用儲存過程執行秒殺
     *
     * @param paramMap
     */
    void killByProcedure(Map<String, Object> paramMap);
```

然後在`SeckillMapper.xml`中寫`sql`語句，具體程式碼請參照專案的原始碼。

接著在`SeckillService.java`介面中宣告 `executeSeckillProcedure()`方法

在pom.xml中新增`commons-collections`的依賴，如下圖：

![][14]

然後在`SeckillServiceImpl.java`中實現`executeSeckillProcedure()`方法。

在`SeckillServiceImplTest.java`中編寫測試方法`executeSeckillProcedureTest()`。

測試結果：

![][15]

修改`SeckillController.java`中的`execute()`方法，把一開始呼叫普通方法的改成呼叫儲存過程的方法。

## 儲存過程最佳化總結

* 儲存過程最佳化：事務行級鎖持有的時間

* 不要過度依賴儲存過程

* 簡單的邏輯依賴儲存過程

* QPS:一個秒殺單6000/qps

經過簡單最佳化和深度最佳化之後，本專案大概能達到一個秒殺單6000qps，這個資料對於一個秒殺商品來說其實已經挺ok了，注意這裡是指同一個秒殺商品6000qps，如果是不同商品不存在行級鎖競爭的問題。

### 系統部署架構

![][16]

CDN：放置一些靜態化資源，或者可以將動態資料分離。一些js依賴直接用公網的CDN，自己開發的一些頁面也做靜態化處理推送到CDN。使用者在CDN獲取到的資料不需要再訪問我們的伺服器，動靜態分離可以降低伺服器請求量。比如秒殺詳情頁，做成HTML放在CDN上，動態資料可以透過ajax請求後臺獲取。

Nginx：作為http伺服器，響應客戶請求，為後端的servlet容器做反向代理，以達到負載均衡的效果。

Redis：用來做伺服器端的快取，透過Jedis提供的API來達到熱點資料的一個快速存取的過程，減少資料庫的請求量。

MySQL：保證秒殺過程的資料一致性與完整性。

智慧DNS解析+智慧CDN加速+Nginx併發+Redis快取+MySQL分庫分表，如下圖：

![][17]

大型系統部署架構，邏輯叢集就是開發的部分。

* Nginx做負載均衡

* 分庫分表：在秒殺系統中，一般透過關鍵的秒殺商品id取模進行分庫分表，以512為一張表，1024為一張表。分庫分表一般採用開源架構，如阿里巴巴的tddl分庫分表框架。

* 統計分析：一般使用hadoop等架構進行分析

在這樣一個架構中，可能參與的角色如下：

![][18]

到此，該專案已經全部完成，感謝閱讀本文。


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517334205021.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517334438454.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517334600302.jpg
  [4]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517334903423.jpg
  [5]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517335304628.jpg
  [6]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517335653112.jpg
  [7]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517335681904.jpg
  [8]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/30/1517325699734.jpg
  [9]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/30/1517325829466.jpg
  [10]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/30/1517325955175.jpg
  [11]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/30/1517326483818.jpg
  [12]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517337117090.jpg
  [13]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517337228818.jpg
  [14]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517328581037.jpg
  [15]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517330110383.jpg
  [16]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517339924527.jpg
  [17]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517340069294.jpg
  [18]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/31/1517340432690.jpg