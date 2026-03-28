---
categories: WebSocket
description: 最近有同學問我有沒有做過線上諮詢功能。同時，公司也剛好讓我接手一個 IM 專案。所以今天抽時間記錄一下最近學習的內容。本文主要剖析了 WebSocket 的原理，以及附上一個完整的聊天室實戰 Demo （包含前端和後端，程式碼下載連結在文末）。
---

## 1、前言

最近有同學問我有沒有做過線上諮詢功能。同時，公司也剛好讓我接手一個 IM 專案。所以今天抽時間記錄一下最近學習的內容。本文主要剖析了 WebSocket 的原理，以及附上一個完整的聊天室實戰 Demo （包含前端和後端，程式碼下載連結在文末）。

## 2、WebSocket 與 HTTP

WebSocket 協議在2008年誕生，2011年成為國際標準。現在所有瀏覽器都已經支援了。WebSocket 的最大特點就是，伺服器可以主動向客戶端推送資訊，客戶端也可以主動向伺服器傳送資訊，是真正的雙向平等對話。

HTTP 有 1.1 和 1.0 之說，也就是所謂的 keep-alive ，把多個 HTTP 請求合併為一個，但是 Websocket 其實是一個新協議，跟 HTTP 協議基本沒有關係，只是為了相容現有瀏覽器，所以在握手階段使用了 HTTP 。

下面一張圖說明了 HTTP 與 WebSocket 的主要區別：

![][1]

WebSocket 的其他特點：

- 建立在 TCP 協議之上，伺服器端的實現比較容易。
- 與 HTTP 協議有著良好的相容性。預設埠也是80和443，並且握手階段採用 HTTP 協議，因此握手時不容易遮蔽，能透過各種 HTTP 代理伺服器。
- 資料格式比較輕量，效能開銷小，通訊高效。
- 可以傳送文字，也可以傳送二進位制資料。
- 沒有同源限制，客戶端可以與任意伺服器通訊。
- 協議識別符號是ws（如果加密，則為wss），伺服器網址就是 URL。

## 3、WebSocket 是什麼樣的協議，具體有什麼優點

首先，WebSocket 是一個持久化的協議，相對於 HTTP 這種非持久的協議來說。簡單的舉個例子吧，用目前應用比較廣泛的 PHP 生命週期來解釋。

HTTP 的生命週期透過 Request 來界定，也就是一個 Request 一個 Response ，那麼在 HTTP1.0 中，這次 HTTP 請求就結束了。

在 HTTP1.1 中進行了改進，使得有一個 keep-alive，也就是說，在一個 HTTP 連線中，可以傳送多個 Request，接收多個 Response。但是請記住 Request = Response， 在 HTTP 中永遠是這樣，也就是說一個 Request 只能有一個 Response。而且這個 Response 也是被動的，不能主動發起。

你 BB 了這麼多，跟 WebSocket 有什麼關係呢？ 好吧，我正準備說 WebSocket 呢。

首先 WebSocket 是基於 HTTP 協議的，或者說借用了 HTTP 協議來完成一部分握手。

首先我們來看個典型的 WebSocket 握手

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://example.com
```

熟悉 HTTP 的童鞋可能發現了，這段類似 HTTP 協議的握手請求中，多了這麼幾個東西。

```
Upgrade: websocket
Connection: Upgrade
```

這個就是 WebSocket 的核心了，告訴 Apache 、 Nginx 等伺服器：注意啦，我發起的請求要用 WebSocket 協議，快點幫我找到對應的助理處理~而不是那個老土的 HTTP。

```
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

首先， Sec-WebSocket-Key 是一個 Base64 encode 的值，這個是瀏覽器隨機生成的，告訴伺服器：泥煤，不要忽悠我，我要驗證你是不是真的是 WebSocket 助理。

然後， Sec_WebSocket-Protocol 是一個使用者定義的字串，用來區分同 URL 下，不同的服務所需要的協議。簡單理解：今晚我要服務A，別搞錯啦~

最後， Sec-WebSocket-Version 是告訴伺服器所使用的 WebSocket Draft （協議版本），在最初的時候，WebSocket 協議還在 Draft 階段，各種奇奇怪怪的協議都有，而且還有很多期奇奇怪怪不同的東西，什麼 Firefox 和 Chrome 用的不是一個版本之類的，當初 WebSocket 協議太多可是一個大難題。。不過現在還好，已經定下來啦~大家都使用同一個版本： 服務員，我要的是13歲的噢→_→

然後伺服器會返回下列東西，表示已經接受到請求， 成功建立 WebSocket 啦！

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```

這裡開始就是 HTTP 最後負責的區域了，告訴客戶，我已經成功切換協議啦~

```
Upgrade: websocket
Connection: Upgrade
```

依然是固定的，告訴客戶端即將升級的是 WebSocket 協議，而不是 mozillasocket，lurnarsocket 或者 shitsocket。

然後， Sec-WebSocket-Accept 這個則是經過伺服器確認，並且加密過後的 Sec-WebSocket-Key 。 伺服器：好啦好啦，知道啦，給你看我的 ID CARD 來證明行了吧。

後面的， Sec-WebSocket-Protocol 則是表示最終使用的協議。

至此，HTTP 已經完成它所有工作了，接下來就是完全按照 WebSocket 協議進行了。

## 4、WebSocket 的作用

在講 WebSocket之前，我就順帶著講下 ajax輪詢 和 long poll 的原理。

### 4-1、ajax輪詢

ajax輪詢的原理非常簡單，讓瀏覽器隔個幾秒就傳送一次請求，詢問伺服器是否有新資訊。

**場景再現：**

```
客戶端：啦啦啦，有沒有新資訊(Request)

服務端：沒有（Response）

客戶端：啦啦啦，有沒有新資訊(Request)

服務端：沒有。。（Response）

客戶端：啦啦啦，有沒有新資訊(Request)

服務端：你好煩啊，沒有啊。。（Response）

客戶端：啦啦啦，有沒有新訊息（Request）

服務端：好啦好啦，有啦給你。（Response）

客戶端：啦啦啦，有沒有新訊息（Request）

服務端：。。。。。沒。。。。沒。。。沒有（Response） —- loop
```

### 4-2、long poll

long poll 其實原理跟 ajax輪詢 差不多，都是採用輪詢的方式，不過採取的是阻塞模型（一直打電話，沒收到就不掛電話），也就是說，客戶端發起請求後，如果沒訊息，就一直不返回 Response 給客戶端。直到有訊息才返回，返回完之後，客戶端再次建立連線，週而復始。

**場景再現：**

```
客戶端：啦啦啦，有沒有新資訊，沒有的話就等有了才返回給我吧（Request）

服務端：額。。 等待到有訊息的時候。。來 給你（Response）

客戶端：啦啦啦，有沒有新資訊，沒有的話就等有了才返回給我吧（Request） -loop
```

**從上面可以看出其實這兩種方式，都是在不斷地建立HTTP連線，然後等待服務端處理，可以體現HTTP協議的另外一個特點，被動性。**

何為被動性呢，其實就是，服務端不能主動聯絡客戶端，只能有客戶端發起。

從上面很容易看出來，不管怎麼樣，上面這兩種都是非常消耗資源的。

ajax輪詢 需要伺服器有很快的處理速度和資源。long poll 需要有很高的併發，也就是說同時接待客戶的能力。

所以 ajax輪詢 和 long poll 都有可能發生這種情況。

```

客戶端：啦啦啦啦，有新資訊麼？

服務端：正忙，請稍後再試（503 Server Unavailable）

客戶端：。。。。好吧，啦啦啦，有新資訊麼？

服務端：正忙，請稍後再試（503 Server Unavailable）

```

### 4-3、WebSocket

透過上面這兩個例子，我們可以看出，這兩種方式都不是最好的方式，需要很多資源。

一種需要更快的速度，一種需要更多的’電話’。這兩種都會導致’電話’的需求越來越高。

哦對了，忘記說了 HTTP 還是一個無狀態協議。通俗的說就是，伺服器因為每天要接待太多客戶了，是個健忘鬼，你一掛電話，他就把你的東西全忘光了，把你的東西全丟掉了。你第二次還得再告訴伺服器一遍。

所以在這種情況下出現了 WebSocket 。他解決了 HTTP 的這幾個難題。首先，被動性，當伺服器完成協議升級後（HTTP->Websocket），服務端就可以主動推送資訊給客戶端啦。所以上面的情景可以做如下修改。

```

客戶端：啦啦啦，我要建立Websocket協議，需要的服務：chat，Websocket協議版本：17（HTTP Request）

服務端：ok，確認，已升級為Websocket協議（HTTP Protocols Switched）

客戶端：麻煩你有資訊的時候推送給我噢。。

服務端：ok，有的時候會告訴你的。

服務端：balabalabalabala

服務端：balabalabalabala

服務端：哈哈哈哈哈啊哈哈哈哈

服務端：笑死我了哈哈哈哈哈哈哈

```

這樣，只需要經過一次 HTTP 請求，就可以做到源源不斷的資訊傳送了。

## 5、實戰程式碼

[本文的更新源 託管於GitHub](https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-07-21-%E7%9C%8B%E5%AE%8C%E8%AE%A9%E4%BD%A0%E5%BD%BB%E5%BA%95%E7%90%86%E8%A7%A3%20WebSocket%20%E5%8E%9F%E7%90%86%EF%BC%8C%E9%99%84%E5%AE%8C%E6%95%B4%E7%9A%84%E5%AE%9E%E6%88%98%E4%BB%A3%E7%A0%81%EF%BC%88%E5%8C%85%E5%90%AB%E5%89%8D%E7%AB%AF%E5%92%8C%E5%90%8E%E7%AB%AF%EF%BC%89.md)

參考文件：   
[php socket 文件](http://php.net/manual/zh/ref.sockets.php)      
[js 的 WebSocket 文件](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)   

前端程式碼：[https://github.com/nnngu/WebSocketDemo-js](https://github.com/nnngu/WebSocketDemo-js)      
後端程式碼：[https://github.com/nnngu/WebSocketDemo-php](https://github.com/nnngu/WebSocketDemo-php)  

執行步驟：

1. 在終端開啟 `WebSocketDemo-php` 目錄，執行 `php -q server.php`
2. 用瀏覽器訪問 `WebSocketDemo-js` 目錄裡面的 `index.html`

執行截圖：

![][2]






  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/7/21/1532166645819.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/7/21/1532169032942.jpg