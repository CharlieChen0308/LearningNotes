# Java中ArrayList與LinkedList的區別

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

---

**一般大家都知道ArrayList和LinkedList的區別：**

      1、ArrayList的實現是基於陣列，LinkedList的實現是基於雙向連結串列。
	  
      2、對於隨機訪問，ArrayList優於LinkedList

      3、對於插入和刪除操作，LinkedList優於ArrayList

      4、LinkedList比ArrayList更佔記憶體，因為LinkedList的節點除了儲存資料，還儲存了兩個引用，一個指向前一個元素，一個指向後一個元素。

##  **一．在時間複雜度上的區別**

假設我們有兩個很大的列表，它們裡面的元素已經排好序了，這兩個列表分別是ArrayList型別和LinkedList型別的，現在我們對這兩個列表來進行二分查詢(binary search)，比較它們的查詢速度。

程式碼如下：

<pre> 1 package com.demo;
 2 
 3 import java.util.ArrayList;
 4 import java.util.Collections;
 5 import java.util.LinkedList;
 6 import java.util.List;
 7 
 8 public class Demo1 {
 9     static List<Integer> array = new ArrayList<Integer>();
10     static List<Integer> linked = new LinkedList<Integer>();
11 
12     public static void main(String[] args) {
13 
14         for (int i = 0; i < 10000; i++) {
15             array.add(i);
16             linked.add(i);
17         }
18         System.out.println("ArrayList訪問消耗的時間：" + getTime(array));
19         System.out.println("LinkedList訪問消耗的時間：" + getTime(linked));
20     }
21 
22     public static long getTime(List list) {
23         long time = System.currentTimeMillis();
24         for (int i = 0; i < 10000; i++) {
25             int index = Collections.binarySearch(list, list.get(i));
26             if (index != i) {
27                 System.out.println("ERROR!");
28             }
29         }
30         return System.currentTimeMillis() - time;
31     }
32 
33 }</pre>

執行結果：

<pre>ArrayList訪問消耗的時間：10
LinkedList訪問消耗的時間：383</pre>

可以看出，對於隨機訪問，ArrayList的訪問速度更快。 

ArrayList和LinkedList的插入資料耗時：

<pre> 1 package com.demo;
 2 
 3 import java.util.ArrayList; 
 4 import java.util.LinkedList;
 5 import java.util.List;
 6 
 7 public class Demo2 {
 8     static List<Integer> array = new ArrayList<Integer>();
 9     static List<Integer> linked = new LinkedList<Integer>();
10 
11     public static void main(String[] args) {
12 
13         for (int i = 0; i < 10000; i++) {
14             array.add(i);
15             linked.add(i);
16         }
17         System.out.println("ArrayList插入消耗的時間：" + insertTime(array));
18         System.out.println("LinkedList插入消耗的時間：" + insertTime(linked));
19     }
20 
21     public static long insertTime(List list) {
22         long time = System.currentTimeMillis();
23         for (int i = 100; i < 10000; i++) {
24             list.add(10, i); // 在索引為10的位置插入i
25         }
26         return System.currentTimeMillis() - time;
27     }
28 }</pre>

執行結果：

<pre>ArrayList插入消耗的時間：31
LinkedList插入消耗的時間：4</pre>

可以看出，對於插入操作，LinkedList 的速度更快。

##  二．在空間複雜度上的區別

在LinkedList中有一個私有的內部類，定義如下：

<pre>private static class Entry {   
         Object element;   
         Entry next;   
         Entry previous;   
     }   </pre>

LinkedList中的每一個元素中還儲存了它的前一個元素的索引和後一個元素的索引。

ArrayList使用一個內建的陣列來儲存元素，這個陣列的起始容量是10，當陣列需要增長時，新的容量按如下公式獲得：新容量 = 舊容量&times;1.5 + 1，也就是說每一次容量大概會增長50% 

## 總結：

**ArrayList和LinkedList的區別如下：**

      1\. ArrayList的實現是基於陣列，LinkedList的實現是基於雙向連結串列。 
	  
      2\. 對於隨機訪問，ArrayList優於LinkedList，ArrayList可以根據下標以O(1)時間複雜度對元素進行隨機訪問。而LinkedList的每一個元素都依靠地址指標和它後一個元素連線在一起，在這種情況下，查詢某個元素的時間複雜度是O(n) 

      3\. 對於插入和刪除操作，LinkedList優於ArrayList，因為當元素被新增到LinkedList任意位置的時候，不需要像ArrayList那樣重新計算大小或者是更新索引。 

　　4\. LinkedList比ArrayList更佔記憶體，因為LinkedList的節點除了儲存資料，還儲存了兩個引用，一個指向前一個元素，一個指向後一個元素。

