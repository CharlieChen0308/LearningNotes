---
categories: React
description: 這一篇就用這些圖片做一個音樂相簿吧！
---

[上一篇文章用爬蟲自動下載了一些圖片](https://github.com/nnngu/LearningNotes/blob/master/Spider/02%20Python%E7%88%AC%E8%99%AB%E5%AE%9E%E7%8E%B0%E7%99%BE%E5%BA%A6%E5%9B%BE%E7%89%87%E8%87%AA%E5%8A%A8%E4%B8%8B%E8%BD%BD.md)，這一篇就用這些圖片做一個音樂相簿吧！

## 效果預覽

![][1]

點選圖片，切換到背面：

![][2]

### 演示地址

演示地址：[https://nnngu.github.io/MusicPhoto/](https://nnngu.github.io/MusicPhoto/)

## 環境搭建

1、安裝 npm，安裝成功後，在終端輸入 `npm -v` 可以檢視它的版本。

![][3]

2、安裝`generator-react-webpack`，使用如下命令：

```
npm install -g generator-react-webpack
```

安裝完成之後，輸入`npm list --depth=0 -global` 可以檢視版本。

![][4]

3、建立專案，開啟你用來存放程式碼的目錄，然後輸入：`yo react-webpack MusicPhoto`

4、建立完成，專案的目錄如下圖：

![][5]

需要注意的幾個地方：

* ① cfg 目錄是配置檔案所在的目錄
  * 重點關注 cfg 目錄裡面的 defaults.js 檔案  
* ② src 專案的原始碼主要在這裡面
* ③ package.json 用來管理和配置依賴模組

## 新增 autoprefixer-loader 模組

autoprefixer-loader 是用來處理 css 的模組，安裝命令：

```
npm install autoprefixer-loader --save-dev
```

然後開啟 cfg 目錄中的 defaults.js 新增如下配置資訊：

![][6]

## 新增 json-loader 模組

json-loader 是用來處理 json 的模組，安裝命令：

```
npm install json-loader --save-dev
```

然後開啟 cfg 目錄中的 defaults.js 新增如下配置資訊：

![][7]

## 新增圖片

1、在 src 目錄下新增 images 目錄和一些圖片，如下圖：(圖片尺寸全部是 260 \* 260) 

![][8]

2、新增 imageDatas.json 如下圖：

![][9]

imageDatas.json 裡面的程式碼請參照專案的原始碼。

3、在`src/components/Main.js`中引入`imageDatas.json` 程式碼如下：

```javascript
// 獲取圖片的 json 資料
var imagesData = require('../data/imageDatas.json');
```

4、根據圖片的檔名，生成圖片URL。

*src/components/Main.js*

```javascript
/**
 * @imagesDataArray  {Array}
 * @return {Array}
 */
imagesData = (function getImageURL(imagesDataArray) {
  for (var i = 0, j = imagesDataArray.length; i < j; i++) {
    var singleImageData = imagesDataArray[i];

    singleImageData.imageURL = require('../images/' + singleImageData.fileName);

    imagesDataArray[i] = singleImageData;
  }
  return imagesDataArray;
})(imagesData);
```

## 配置字型

開啟 cfg 目錄中的 defaults.js 新增如下配置資訊：

![][10]

## 元件的繫結

src/index.html 中的關鍵程式碼：

![][11]

src/index.js 中的關鍵程式碼：

![][12]

## 程式碼邏輯

主要的程式碼邏輯在 `Main.js`中，主要的佈局樣式在 `App.scss`中。如下圖：

![][13]

具體程式碼請參照專案的原始碼 [https://github.com/nnngu/MusicPhoto](https://github.com/nnngu/MusicPhoto)

## 釋出到Github Pages

1、修改`cfg/defaults.js`中的`publicPath`，改為`publicPath: './assets/',` 如下圖：

![][14]

2、打包，使用`npm run dist`指令。打包完成可以看到如下目錄：

![][15]

3、把打包好的目錄 push 到 GitHub 的 gh-pages 分支，使用如下命令：

```
git subtree push --prefix=dist origin gh-pages
```

4、在GitHub 對應的倉庫裡面開啟 Github Pages 功能，並選擇 `gh-pages`分支即可。

**👇👇👇下一篇將會總結完成音樂播放器的過程。👇👇👇**

[05 (專案) 基於 React + Webpack 的音樂相簿（下）](https://github.com/nnngu/LearningNotes/blob/master/React/05%20(%E9%A1%B9%E7%9B%AE)%20%E5%9F%BA%E4%BA%8E%20React%20%2B%20Webpack%20%E7%9A%84%E9%9F%B3%E4%B9%90%E7%9B%B8%E5%86%8C%EF%BC%88%E4%B8%8B%EF%BC%89.md)



  [1]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/5/1517842690437.jpg
  [2]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/5/1517842775081.jpg
  [3]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517848578071.jpg
  [4]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517848855856.jpg
  [5]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517849337904.jpg
  [6]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517850101903.jpg
  [7]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517850270658.jpg
  [8]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517851778975.jpg
  [9]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517851939423.jpg
  [10]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517852817008.jpg
  [11]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517853041622.jpg
  [12]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517853081657.jpg
  [13]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517853295536.jpg
  [14]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517853662271.jpg
  [15]: https://www.github.com/nnngu/FigureBed/raw/master/2018/2/6/1517853876038.jpg
