# 01 React 元件

> **版本提示**：本篇基於 React 15.x，使用已廢棄的 `React.createClass` 和 CDN 引入方式。`React.createClass` 自 React 16.3 起已移除。現代 React 19 使用函式元件與 Hooks，請參考 [06 現代 React 開發：函式元件與 Hooks](06%20%E7%8F%BE%E4%BB%A3%20React%20%E9%96%8B%E7%99%BC%EF%BC%9A%E5%87%BD%E5%BC%8F%E5%85%83%E4%BB%B6%E8%88%87%20Hooks.md)。

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)

---

本篇文章我們來總結 React 元件，接下來我們封裝一個輸出 "Hello World！" 的元件，元件名為 HelloMessage

## HelloMessage元件

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8" />
    <title>React 學習</title>
    <script src="https://cdn.bootcss.com/react/15.4.2/react.min.js"></script>
    <script src="https://cdn.bootcss.com/react/15.4.2/react-dom.min.js"></script>
    <script src="https://cdn.bootcss.com/babel-standalone/6.22.1/babel.min.js"></script>
</head>
<body>
<div id="example"></div>
<script type="text/babel">
    var HelloMessage = React.createClass({
        render: function() {
            return <h1>Hello World！</h1>;
        }
    });

    ReactDOM.render(
        <HelloMessage />,
        document.getElementById('example')
    );
</script>
</body>
</html>
```

### 解析

React.createClass 方法用於生成一個元件類 HelloMessage

<HelloMessage /> 例項化元件類並輸出資訊。

**注意，原生 HTML 元素名以小寫字母開頭，而自定義的 React 類名以大寫字母開頭，比如 HelloMessage 不能寫成 helloMessage。除此之外還需要注意元件類只能包含一個頂層標籤，否則也會報錯。**

## 向元件傳遞引數

如果我們需要向元件傳遞引數，可以使用 this.props 物件，例項如下：

```html
<body>
<div id="example"></div>
<script type="text/babel">
    var HelloMessage = React.createClass({
        render: function() {
            return <h1>Hello {this.props.name}</h1>;
        }
    });

    ReactDOM.render(
        <HelloMessage name="React！！" />,
        document.getElementById('example')
    );
</script>
</body>
```

## 複合元件

我們可以透過建立多個元件來合成一個元件，即把元件的不同功能進行分離。

以下例子實現了輸出網站名字和網址的元件

```html
<body>
<div id="example"></div>
<script type="text/babel">
    var WebSite = React.createClass({
        render: function() {
            return (
                <div>
                    <Name name={this.props.name} />
                    <Link site={this.props.site} />
                </div>
            );
        }
    });

    var Name = React.createClass({
        render: function() {
            return (
                <h1>{this.props.name}</h1>
            );
        }
    });

    var Link = React.createClass({
        render: function() {
            return (
                <a href={this.props.site}>
                    {this.props.site}
                </a>
            );
        }
    });

    ReactDOM.render(
        <WebSite name="LearningNotes" site="https://github.com/nnngu/LearningNotes" />,
        document.getElementById('example')
    );
</script>
</body>
```

上面的例子中，WebSite 元件使用了 Name 和 Link 元件來輸出對應的資訊，也就是說 WebSite 擁有 Name 和 Link 的例項。
