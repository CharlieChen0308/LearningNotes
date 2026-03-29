# 02 Python爬蟲實現百度圖片自動下載

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)    

---

## 製作爬蟲的步驟

製作一個爬蟲一般分以下幾個步驟：

* 分析需求
* 分析網頁原始碼，配合開發者工具
* 編寫正規表示式或者XPath表示式
* 正式編寫 python 爬蟲程式碼

## 效果預覽

執行效果如下：

![][1]

存放圖片的資料夾：

![][2]

## 需求分析

我們的爬蟲至少要實現兩個功能：一是搜尋圖片，二是自動下載。

搜尋圖片：最容易想到的是爬百度圖片的結果，我們就上百度圖片看看：

![][3]

隨便搜尋幾個關鍵字，可以看到已經搜尋出來很多張圖片：

![][4]

## 分析網頁

我們點選右鍵，檢視原始碼：

![][5]

開啟原始碼之後，發現一堆原始碼比較難找出我們想要的資源。

這個時候，就要用開發者工具！我們回到上一頁面，調出開發者工具，我們需要用的是左上角那個東西：(滑鼠跟隨)。

![][6]

然後選擇你想看原始碼的地方，就可以發現，下面的程式碼區自動定位到了相應的位置。如下圖：

![][7]

![][8]

我們複製這個地址，然後到剛才的一堆原始碼裡搜尋一下，發現了它的位置，但是這裡我們又疑惑了，這個圖片有這麼多地址，到底用哪個呢？我們可以看到有thumbURL，middleURL，hoverURL，objURL

![][9]

透過分析可以知道，前面兩個是縮小的版本，hoverURL 是滑鼠移動過後顯示的版本，objURL 應該是我們需要的，可以分別開啟這幾個網址看看，發現 objURL 的那個最大最清晰。

找到了圖片地址，接下來我們分析原始碼。看看是不是所有的 objURL 都是圖片。

![][10]

發現都是以.jpg格式結尾的圖片。

## 編寫正規表示式

```python
pic_url = re.findall('"objURL":"(.*?)",',html,re.S)
```

## 編寫爬蟲程式碼

這裡我們用了2個包，一個是正則，一個是 requests 包

```python
#-*- coding:utf-8 -*-
import re
import requests
```

複製百度圖片搜尋的連結，傳入 requests ，然後把正規表示式寫好

![][11]

```python
url = 'https://image.baidu.com/search/index?tn=baiduimage&ie=utf-8&word=%E6%A0%97%E5%B1%B1%E6%9C%AA%E6%9D%A5%E5%A4%B4%E5%83%8F&ct=201326592&ic=0&lm=-1&width=&height=&v=index'

html = requests.get(url).text
pic_url = re.findall('"objURL":"(.*?)",',html,re.S)

```

因為有很多張圖片，所以要迴圈，我們列印出結果來看看，然後用 requests 獲取網址，由於有些圖片可能存在網址打不開的情況，所以加了10秒超時控制。

```python
pic_url = re.findall('"objURL":"(.*?)",',html,re.S)
i = 1
for each in pic_url:
    print each
    try:
        pic= requests.get(each, timeout=10)
    except requests.exceptions.ConnectionError:
        print('【錯誤】當前圖片無法下載')
        continue

```

接著就是把圖片儲存下來，我們事先建立好一個 images 目錄，把圖片都放進去，命名的時候，以數字命名。

```python
        dir = '../images/' + keyword + '_' + str(i) + '.jpg'
        fp = open(dir, 'wb')
        fp.write(pic.content)
        fp.close()
        i += 1
```

## 完整的程式碼

```python
# -*- coding:utf-8 -*-
import re
import requests


def dowmloadPic(html, keyword):
    pic_url = re.findall('"objURL":"(.*?)",', html, re.S)
    i = 1
    print('找到關鍵詞:' + keyword + '的圖片，現在開始下載圖片...')
    for each in pic_url:
        print('正在下載第' + str(i) + '張圖片，圖片地址:' + str(each))
        try:
            pic = requests.get(each, timeout=10)
        except requests.exceptions.ConnectionError:
            print('【錯誤】當前圖片無法下載')
            continue

        dir = '../images/' + keyword + '_' + str(i) + '.jpg'
        fp = open(dir, 'wb')
        fp.write(pic.content)
        fp.close()
        i += 1


if __name__ == '__main__':
    word = input("Input key word: ")
    url = 'http://image.baidu.com/search/flip?tn=baiduimage&ie=utf-8&word=' + word + '&ct=201326592&v=flip'
    result = requests.get(url)
    dowmloadPic(result.text, word)

```

![][12]

![][13]

我們看到有的圖片沒顯示出來，開啟網址看，發現確實沒了。

![][14]

因為百度有些圖片它快取到百度的伺服器上，所以我們在百度上還能看見它，但它的實際連結已經失效了。

## 總結

enjoy 我們的第一個圖片下載爬蟲吧！當然它不僅能下載百度的圖片，依葫蘆畫瓢，你現在應該能做很多事情了，比如爬取頭像，爬淘寶展示圖等等。

完整程式碼已經放到Github上 [https://github.com/nnngu/BaiduImageDownload](https://github.com/nnngu/BaiduImageDownload)

下一篇文章，[我們將用這些圖片做一個音樂相簿](https://github.com/nnngu/LearningNotes/blob/master/React/04%20(專案)%20基於%20React%20%2B%20%20Webpack%20的音樂相簿（上）.md)。




  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517624440357.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517624588214.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517624851741.jpg
  [4]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517625097976.jpg
  [5]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517625636570.jpg
  [6]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517626066422.jpg
  [7]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517626276983.jpg
  [8]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517626329451.jpg
  [9]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517626739154.jpg
  [10]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517627100214.jpg
  [11]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517627638515.jpg
  [12]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517629256979.jpg
  [13]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517629346426.jpg
  [14]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/3/1517629377850.jpg