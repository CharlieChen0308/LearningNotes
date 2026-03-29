---
categories: Sword2offer
description: 用 Java 語言實現的《劍指 offer 》題解
---

本文永久更新地址：[https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-24-%E5%89%91%E6%8C%87%20offer%20%E9%A2%98%E8%A7%A3%EF%BC%88Java%E7%89%88%EF%BC%89.md](https://github.com/nnngu/LearningNotes/blob/master/_posts/2018-03-24-%E5%89%91%E6%8C%87%20offer%20%E9%A2%98%E8%A7%A3%EF%BC%88Java%E7%89%88%EF%BC%89.md)

---

## 1、本文的約定

### 變數命名約定

* nums 表示數字陣列，array 表示通用陣列，matrix 表示矩陣；
* n 表示陣列長度、字串長度、樹節點個數，以及其它具有一維性質的資料結構的元素個數；
* m, n 表示矩陣的總行數和總列數；
* first, last 表示閉區間，在需要作為函式引數時使用：\[first, last]；
* l, h 也表示閉區間，在只作為區域性變數時使用：\[l, h]；
* begin, end 表示左閉右開區間：\[begin, end)；
* ret 表示結果相關的變數；
* dp 表示動態規劃儲存子問題的陣列；

### 複雜度簡寫說明

O(nlog<sub>n</sub>) + O(n<sup>2</sup>)，+ 號前面的表示時間複雜度，+ 號後面的表示空間複雜度。

## 2、實現單例模式

> [單例模式的5種寫法](https://github.com/nnngu/LearningNotes/blob/master/_posts/2017-04-19-019%20%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F%E7%9A%845%E7%A7%8D%E5%86%99%E6%B3%95.md)

## 3、陣列中重複的數字

### 題目描述

在一個長度為 n 的陣列裡的所有數字都在 0 到 n-1 的範圍內。陣列中某些數字是重複的，但不知道有幾個數字是重複的。也不知道每個數字重複幾次。請找出陣列中任意一個重複的數字。例如，如果輸入長度為 7 的陣列 {2, 3, 1, 0, 2, 5, 3}，那麼對應的輸出是第一個重複的數字 2。

### 解題思路

這種陣列元素在 \[0, n-1] 範圍內的問題，可以將值為 i 的元素放到第 i 個位置上。

以 (2, 3, 1, 0, 2, 5) 為例，程式碼的執行過程為：

```
position-0 : (2,3,1,0,2,5) // 2 <-> 1
             (1,3,2,0,2,5) // 1 <-> 3
             (3,1,1,0,2,5) // 3 <-> 0
             (0,1,1,3,2,5) // already in position
position-1 : (0,1,1,3,2,5) // already in position
position-2 : (0,1,1,3,2,5) // nums[i] == nums[nums[i]], exit
```

遍歷到位置 2 時，該位置上的數為 1，但是第 1 個位置上已經有一個 1 的值了，因此可以知道 1 重複。

複雜度：O(n) + O(1)

```java
public boolean duplicate(int[] nums, int length, int[] duplication) {
    if (nums == null || length <= 0) return false;
    for (int i = 0; i < length; i++) {
        while (nums[i] != i && nums[i] != nums[nums[i]]) {
            swap(nums, i, nums[i]);
        }
        if (nums[i] != i && nums[i] == nums[nums[i]]) {
            duplication[0] = nums[i];
            return true;
        }
    }
    return false;
}

private void swap(int[] nums, int i, int j) {
    int t = nums[i]; nums[i] = nums[j]; nums[j] = t;
}
```

## 4、二維陣列中的查詢

### 題目描述

在一個二維陣列中，每一行都按照從左到右遞增的順序排序，每一列都按照從上到下遞增的順序排序。請完成一個函式，輸入這樣的一個二維陣列和一個整數，判斷陣列中是否含有該整數。

```
Consider the following matrix:
[
  [1,   4,  7, 11, 15],
  [2,   5,  8, 12, 19],
  [3,   6,  9, 16, 22],
  [10, 13, 14, 17, 24],
  [18, 21, 23, 26, 30]
]

Given target = 5, return true.
Given target = 20, return false.
```

### 解題思路

從右上角開始查詢。因為矩陣中的一個數，它左邊的數都比它小，下邊的數都比它大。因此，從右上角開始查詢，就可以根據 target 和當前元素的大小關係來縮小查詢區間。

複雜度：O(m + n) + O(1)

```java
public boolean Find(int target, int[][] matrix) {
    if (matrix == null || matrix.length == 0 || matrix[0].length == 0) return false;
    int m = matrix.length, n = matrix[0].length;
    int r = 0, c = n - 1; // 從右上角開始
    while (r <= m - 1 && c >= 0) {
        if (target == matrix[r][c]) return true;
        else if (target > matrix[r][c]) r++;
        else c--;
    }
    return false;
}
```

## 5、替換空格

### 題目描述

請實現一個函式，將一個字串中的空格替換成“%20”。例如，當字串為 We Are Happy. 則經過替換之後的字串為 We%20Are%20Happy。

### 解題思路

在字串尾部填充任意字元，使得字串的長度等於字串替換之後的長度。因為一個空格要替換成三個字元（%20），因此當遍歷到一個空格時，需要在尾部填充兩個任意字元。

令 P1 指向字串原來的末尾位置，P2 指向字串現在的末尾位置。P1 和 P2 從後向前遍歷，當 P1 遍歷到一個空格時，就需要令 P2 指向的位置依次填充 02%（注意是逆序的），否則就填充上 P1 指向字元的值。

從後向前遍是為了在改變 P2 所指向的內容時，不會影響到 P1 遍歷原來字串的內容。

複雜度：O(n) + O(1)

![][1]

```java
public String replaceSpace(StringBuffer str) {
    int oldLen = str.length();
    for (int i = 0; i < oldLen; i++) {
        if (str.charAt(i) == ' ') {
            str.append("  ");
        }
    }
    int idxOfOld = oldLen - 1;
    int idxOfNew = str.length() - 1;
    while (idxOfOld >= 0 && idxOfNew > idxOfOld) {
        char c = str.charAt(idxOfOld--);
        if (c == ' ') {
            str.setCharAt(idxOfNew--, '0');
            str.setCharAt(idxOfNew--, '2');
            str.setCharAt(idxOfNew--, '%');
        } else {
            str.setCharAt(idxOfNew--, c);
        }
    }
    return str.toString();
}
```

## 6、從尾到頭列印連結串列







  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/3/25/1521955019752.jpg