# Java中String、StringBuffer、StringBuilder的區別

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

---

## 1.從是否可變的角度

　　String類中使用字元陣列儲存字串，因為有“final”修飾符，所以String物件是不可變的。

```
/** The value is used for character storage. */
    private final char value[];
``` 

　　**StringBuffer和StringBuilder**都繼承自AbstractStringBuilder類，在AbstractStringBuilder中也是使用字元陣列儲存字串，但沒有“final”修飾符，所以兩種物件都是可變的。

```
/**
     * The value is used for character storage.
     */
    char[] value;
```

## 2.是否多執行緒安全

　　String中的物件是不可變的，也就可以理解為常量，所以**是執行緒安全的**。

　　AbstractStringBuilder是StringBuffer和StringBuilder的公共父類，定義了一些字串的基本操作，如append、insert、indexOf等公共方法。

　　StringBuffer對方法加了同步鎖(synchronized) ，所以是**執行緒安全的**。看如下原始碼：

```
1     public synchronized StringBuffer append(String str) {
2         toStringCache = null;
3         super.append(str);
4         return this;
5     }
```

　　StringBuilder並沒有對方法進行加同步鎖，所以是**非執行緒安全的**。如下原始碼：

```
1     public StringBuilder append(String str) {
2         super.append(str);
3         return this;
4     }
```

## 3.StringBuffer和StringBuilder的共同點

　　StringBuffer和StringBuilder有公共父類AbstractStringBuilder(**抽象類**)。

　　StringBuffer、StringBuilder的方法都會呼叫AbstractStringBuilder中的公共方法，如上面的兩段原始碼中都呼叫了super.append(str);  只是StringBuffer會在方法上加synchronized關鍵字，進行同步。

　　**最後，如果程式不是多執行緒的，那麼使用StringBuilder效率高於StringBuffer。**

