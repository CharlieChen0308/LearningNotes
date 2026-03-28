# 演算法08 五大查詢之：二叉排序樹(BSTree)

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

---

上一篇總結了[索引查詢](http://www.cnblogs.com/nnngu/p/8290367.html)，這一篇要總結的是二叉排序樹(Binary Sort Tree)，又稱為二叉查詢樹(Binary Search Tree) ，即BSTree。

構造一棵二叉排序樹的目的，其實並不是為了排序，而是為了提高查詢和插入刪除的效率。

什麼是二叉排序樹呢？二叉排序樹具有以下幾個特點。

（1）若根節點有左子樹，則左子樹的所有節點都比根節點小。

（2）若根節點有右子樹，則右子樹的所有節點都比根節點大。

（3）根節點的左，右子樹也分別是二叉排序樹。

## 1、二叉排序樹的圖示

下面是二叉排序樹的圖示，透過它可以加深對二叉排序樹的理解。

![][1]

## 2、二叉排序樹常見的操作及思路

下面是二叉排序樹常見的操作及思路。

### 2-1、插入節點

思路：比如我們要插入數字20到這棵二叉排序樹中。那麼步驟如下：

（1）首先將20與根節點進行比較，發現比根節點小，所以繼續與根節點的左子樹30比較。

（2）發現20比30也要小，所以繼續與30的左子樹10進行比較。

（3）發現20比10要大，所以就將20插入到10的右子樹中。

**此時的二叉排序樹如下圖：**

![][2]

### 2-2、查詢節點

比如我們要查詢節點10，那麼思路如下：

（1）還是一樣，首先將10與根節點50進行比較，發現比根節點要小，所以繼續與根節點的左子樹30進行比較。

（2）發現10比左子樹30要小，所以繼續與30的左子樹10進行比較。

（3）發現兩值相等，即查詢成功，返回10的位置。

### 2-3、刪除節點

刪除節點的情況相對複雜，主要分為以下三種情形：

（1）刪除的是葉節點(即沒有孩子節點的)。比如20，刪除它不會破壞原來樹的結構，最簡單。如圖所示。

![][3]

（2）刪除的是單孩子節點。比如90，刪除它後需要將它的孩子節點與自己的父節點相連。情形比第一種複雜一些。

![][4]

（3）刪除的是有左右孩子的節點。比如根節點50

這裡有一個問題就是刪除它後，誰將作為根節點？**利用二叉樹的中序遍歷，就是右節點的左子樹的最左孩子**。

![][5]

## 3、程式碼

有了思路之後，下面就開始寫程式碼來實現這些功能。

BSTreeNode.java

<pre>public class BSTreeNode {
    public int data;
    public BSTreeNode left;
    public BSTreeNode right;

    public BSTreeNode(int data) {
        this.data = data;
    }
}</pre>

BSTreeOperate.java

<pre>/**
 * 二叉排序樹的常見操作
 */
public class BSTreeOperate {

    // 樹的根節點
    public BSTreeNode root;
    // 記錄樹的節點個數
    public int size;

    /**
     * 建立二叉排序樹
     *
     * @param list
     * @return
     */
    public BSTreeNode create(int[] list) {

        for (int i = 0; i < list.length; i++) {
            insert(list[i]);
        }
        return root;
    }

    /**
     * 插入一個值為data的節點
     *
     * @param data
     */
    public void insert(int data) {
        insert(new BSTreeNode(data));
    }

    /**
     * 插入一個節點
     *
     * @param bsTreeNode
     */
    public void insert(BSTreeNode bsTreeNode) {
        if (root == null) {
            root = bsTreeNode;
            size++;
            return;
        }
        BSTreeNode current = root;
        while (true) {
            if (bsTreeNode.data <= current.data) {
                // 如果插入節點的值小於當前節點的值，說明應該插入到當前節點左子樹，而此時如果左子樹為空，就直接設定當前節點的左子樹為插入節點。
                if (current.left == null) {
                    current.left = bsTreeNode;
                    size++;
                    return;
                }
                current = current.left;
            } else {
                // 如果插入節點的值大於當前節點的值，說明應該插入到當前節點右子樹，而此時如果右子樹為空，就直接設定當前節點的右子樹為插入節點。
                if (current.right == null) {
                    current.right = bsTreeNode;
                    size++;
                    return;
                }
                current = current.right;
            }
        }
    }

    /**
     * 中序遍歷
     *
     * @param bsTreeNode
     */
    public void LDR(BSTreeNode bsTreeNode) {
        if (bsTreeNode != null) {
            // 遍歷左子樹
            LDR(bsTreeNode.left);
            // 輸出節點資料
            System.out.print(bsTreeNode.data + " ");
            // 遍歷右子樹
            LDR(bsTreeNode.right);
        }
    }

    /**
     * 查詢節點
     */
    public boolean search(BSTreeNode bsTreeNode, int key) {
        // 遍歷完沒有找到，查詢失敗
        if (bsTreeNode == null) {
            return false;
        }
        // 要查詢的元素為當前節點，查詢成功
        if (key == bsTreeNode.data) {
            return true;
        }
        // 繼續去當前節點的左子樹中查詢，否則去當前節點的右子樹中查詢
        if (key < bsTreeNode.data) {
            return search(bsTreeNode.left, key);
        } else {
            return search(bsTreeNode.right, key);
        }
    }
}</pre>

BSTreeOperateTest.java

<pre>public class BSTreeOperateTest {
    public static void main(String[] args) {
        BSTreeOperate bsTreeOperate = new BSTreeOperate();
        int[] list = new int[]{50, 30, 70, 10, 40, 90, 80};
        System.out.println("*********建立二叉排序樹*********");
        BSTreeNode bsTreeNode = bsTreeOperate.create(list);
        System.out.println("中序遍歷原始的資料：");
        bsTreeOperate.LDR(bsTreeNode);
        System.out.println("");
        System.out.println("");

        System.out.println("********查詢節點*******");
        System.out.println("元素20是否在樹中：" + bsTreeOperate.search(bsTreeNode, 20));
        System.out.println("");

        System.out.println("********插入節點*******");
        System.out.println("將元素20插入到樹中");
        bsTreeOperate.insert(20);
        System.out.println("中序遍歷：");
        bsTreeOperate.LDR(bsTreeNode);
        System.out.println("");
        System.out.println("");

        System.out.println("********查詢節點*******");
        System.out.println("元素20是否在樹中：" + bsTreeOperate.search(bsTreeNode, 20));
        System.out.println("");
    }
}</pre>

執行結果：

![][6]


  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/21/1516485386411.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/21/1516485447505.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/21/1516485491037.jpg
  [4]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/21/1516485533633.jpg
  [5]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/21/1516485588434.jpg
  [6]: https://www.github.com/nnngu/FigureBed/raw/master/2018/1/21/1516467321982.jpg