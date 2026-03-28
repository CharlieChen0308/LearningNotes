# 018 final 關鍵字的用途

## final關鍵字的含義

`final`在`Java`中是一個保留的關鍵字，可以宣告成員變數、方法、類以及本地變數。一旦你將引用宣告作`final`，你將不能改變這個引用了，編譯器會檢查程式碼，如果你試圖將變數再次初始化的話，編譯器會報編譯錯誤。

## final變數

凡是對成員變數或者本地變數(在方法中的或者程式碼塊中的變數稱為本地變數)宣告為`final`的都叫作`final`變數。`final`變數經常和`static`關鍵字一起使用，作為常量。`final`變數是隻讀的。

## final方法

`final`也可以宣告方法。方法前面加上`final`關鍵字，代表這個方法不可以被子類的方法重寫。如果你認為一個方法的功能已經足夠完整了，在子類中不需要改變的話，你可以宣告此方法為`final`。`final`方法比`非final`方法要快，因為在編譯的時候已經靜態繫結了，不需要在執行時再動態繫結。

## final類

使用`final`來修飾的類叫做`final類`。`final類`通常功能是完整的，它們不能被繼承。`Java`中有許多類是`final`的，比如`String`、`Interger`以及其他包裝類。

## 使用 final關鍵字的好處

1. `final`關鍵字提高了效能。`JVM`和`Java應用`都會快取`final變數`。
2. `final變數`可以安全的在多執行緒環境下進行共享，而不需要額外的同步開銷。
3. 使用`final關鍵字`，`JVM`會對變數、方法以及類進行最佳化。

















---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/Java%20Basis/018%20final%20%E5%85%B3%E9%94%AE%E5%AD%97%E7%9A%84%E7%94%A8%E9%80%94.md](https://github.com/nnngu/LearningNotes/blob/master/Java%20Basis/018%20final%20%E5%85%B3%E9%94%AE%E5%AD%97%E7%9A%84%E7%94%A8%E9%80%94.md)