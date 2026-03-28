# 如何用java語言實現C#中的ref關鍵字(按引用傳遞引數)的效果

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

---

在上一篇文章中（[Java的引數傳遞是值傳遞還是引用傳遞](http://www.cnblogs.com/nnngu/p/8299724.html)），主要分析了java語言的引數傳遞只有按值傳遞而沒有按引用傳遞。

先看一下微軟的C#文件對按引用傳遞的定義（如下截圖）：<https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/ref#passing-an-argument-by-reference>

![][1]

**那麼java語言如何實現C#中ref關鍵字(按引用傳遞引數)的效果呢？**

## 思路

我們可以把需要傳遞的引數再封裝一層，即定義一個新的類，使得需要傳遞的引數成為新類的成員變數，傳遞引數時就傳遞這個新類的例項。以此達到ref關鍵字的效果。

## 程式碼演示

RefDemo.java

<pre>public class RefDemo {
    public static void main(String[] args) {
        Person person1 = new Person();
        PersonPack personPack = new PersonPack();
        personPack.person = person1;

        // 列印person
        System.out.println(personPack.person);

        change(personPack);

        // 再列印person
        System.out.println(personPack.person);
    }

    public static void change(PersonPack personPack) {
        Person person2 = new Person();
        personPack.person = person2;
    }
}

/**
 * Person類
 */
class Person {

}

/**
 * 包裝類：PersonPack
 */
class PersonPack {
    public Person person;
}</pre>

執行結果：

![][2]

可以看出兩次列印person的地址值不一樣，即呼叫完change() 方法之後，person引用(指向) 了另一個物件！


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/21/1516472077285.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/21/1516472129252.jpg