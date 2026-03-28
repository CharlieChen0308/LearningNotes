# Java中的String類能否被繼承？為什麼？

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

---

不能被繼承，因為String類有final修飾符，而final修飾的類是不能被繼承的。

## **Java對String類的定義：**

```
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    // 省略...　
}
```

## final修飾符的用法：

1.修飾類

　　當用final修飾一個類時，表明這個類不能被繼承。final類中的成員變數可以根據需要設為final，但是要注意final類中的所有成員方法都會被隱式地指定為final方法。

![][1]

2.修飾方法

　　使用final修飾方法的原因有兩個。第一個原因是把方法鎖定，以防任何繼承類修改它的含義；第二個原因是效率。在早期的Java實現版本中，會將final方法轉為內嵌呼叫。但是如果方法過於龐大，可能看不到內嵌呼叫帶來的任何效能提升。在最近的Java版本中，不需要使用final方法進行這些最佳化了。

　　因此，只有在想明確禁止該方法在子類中被覆蓋的情況下才將方法設定為final。

　　注：一個類中的private方法會隱式地被指定為final方法。

3.修飾變數

　　對於被final修飾的變數，如果是基本資料型別的變數，則其數值一旦在初始化之後便不能更改；如果是引用型別的變數，則在對其初始化之後便不能再讓其指向另一個物件。雖然不能再指向其他物件，但是它指向的物件的內容是可變的。

## final和static的區別：

很多時候會容易把static和final關鍵字混淆，static作用於成員變數用來表示只儲存一份副本，而final的作用是用來保證變數不可變。看下面這個例子：

```
 1 public class Demo1 {
 2     public static void main(String[] args)  {
 3         MyClass myClass1 = new MyClass();
 4         MyClass myClass2 = new MyClass();
 5         System.out.println(myClass1.i);
 6         System.out.println(myClass2.i);
 7         System.out.println(myClass1.j);
 8         System.out.println(myClass2.j);
 9 
10     }
11 }
12 
13 class MyClass {
14     public final double i = Math.random();
15     public static double j = Math.random();
16 }
```

 執行結果：

```
0.3222977275463088
0.2565532218939688
0.36856868882926397
0.36856868882926397
```

每次列印的兩個j值都是一樣的，而i的值卻是不同的。從這裡就可以知道final和static變數的區別了。


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/21/1516470278278.jpg