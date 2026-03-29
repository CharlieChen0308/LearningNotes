# 015 反射中的 Class.forName() 與 ClassLoader.loadClass() 的區別

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

---

Class.forName() 與 ClassLoader.loadClass() 大家都知道是反射用來構造類的方法，但是他們的用法還是有一定區別的。

 在講區別之前，我覺得很有必要把類的載入過程在此整理一下。

 在Java中，類載入器把一個類載入進Java虛擬機器中，要經過三個步驟來完成：載入、連結和初始化，其中連結又可以分成驗證、準備和解析三步，除了解析外，其它步驟是嚴格按照順序完成的，各個步驟的主要工作如下：
 
* 載入：查詢和匯入類或介面的二進位制資料； 

* 連結：執行下面的校驗、準備和解析步驟，其中解析步驟是可以選擇的； 

  * 驗證：檢查匯入類或介面的二進位制資料的正確性； 

  * 準備：給類的靜態變數分配並初始化儲存空間； 

  * 解析：將符號引用轉成直接引用； 

* 初始化：啟用類的靜態變數的初始化Java程式碼和靜態Java程式碼塊。

![][1]

 於是乎我們可以開始看2者的區別了。
 
Class.forName(className) 方法，其實呼叫的方法是`Class.forName(className,true,classloader);` 注意看第2個boolean引數，它表示的意思，在載入之後必須初始化。在執行過此方法後，目標物件的靜態塊程式碼已經被執行，靜態引數也已經被初始化。

再看ClassLoader.loadClass(className) 方法，其實他呼叫的方法是`ClassLoader.loadClass(className,false);` 注意看第2個 boolean 引數，該參數列示目標物件被載入後不進行連結，這就意味著不會去執行該類靜態塊中的內容。因此兩者的區別就顯而易見了。




  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/26/1516908236033.jpg