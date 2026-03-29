---
categories: JavaBasis
description: Map主要用於儲存健值對，根據鍵得到值，因此不允許鍵重複，但允許值重複。
---

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

---

Map主要用於儲存健值對，根據鍵得到值，因此不允許鍵重複，但允許值重複。

## HashMap

　　說到Map，首先能想起的是HashMap，它是一個最常用的Map，它根據鍵的HashCode 來儲存資料，根據鍵可以直接獲取它的值，具有很快的訪問速度。**遍歷時，取得資料的順序是完全隨機的。**
  
　　HashMap最多隻允許一條記錄的鍵為Null；允許多條記錄的值為 Null。（不允許鍵重複，但允許值重複）
  
　　HashMap不支援執行緒的同步（任一時刻可以有多個執行緒同時寫HashMap，即執行緒非安全），可能會導致資料的不一致。如果需要同步，可以用 Collections的synchronizedMap() 方法使HashMap具有同步的能力，或者使用ConcurrentHashMap。
  
　　Hashtable與 HashMap類似。不同的是：它不允許記錄的鍵或者值為空；它支援執行緒的同步（任一時刻只有一個執行緒能寫Hashtable，即執行緒安全），因此也導致了 Hashtable 在寫入時會比較慢。

## TreeMap

　　TreeMap實現SortMap介面，能夠把它儲存的記錄根據鍵排序。
  
　　**預設是按鍵的升序排序，也可以指定排序的比較器**，當用Iterator 遍歷TreeMap時，得到的記錄是排過序的。

## LinkedHashMap

**LinkedHashMap儲存了記錄的插入順序，在用Iterator遍歷LinkedHashMap時，先得到的記錄肯定是先插入的**。

在遍歷的時候會比HashMap慢，不過有種情況例外：當HashMap容量很大，實際資料較少時，遍歷起來可能會比LinkedHashMap慢。因為LinkedHashMap的遍歷速度只和實際資料有關，和容量無關，而HashMap的遍歷速度和它的容量有關。

## 三種型別的Map分別在什麼時候使用

　　1、一般情況下，我們用的最多的是HashMap。HashMap裡面存入的值在取出的時候是隨機的，它根據鍵的HashCode來儲存資料，根據鍵可以直接獲取它的值，具有很快的訪問速度。在Map 中插入、刪除和定位元素，HashMap 是最好的選擇。
  
　　2、TreeMap取出來的是排序後的鍵值對。但如果您要按自然順序或自定義順序遍歷鍵，那麼TreeMap會更好。
  
　　3、LinkedHashMap 是HashMap的一個子類，如果需要輸出的順序和輸入的順序相同，那麼用LinkedHashMap可以實現。

