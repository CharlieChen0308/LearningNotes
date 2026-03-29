# 02 React State(狀態)

> **版本提示**：本篇基於 React 15.x，使用已廢棄的 `getInitialState` 和 `this.setState`。現代 React 使用 `useState` Hook 管理狀態，請參考 [06 現代 React 開發：函式元件與 Hooks](06%20%E7%8F%BE%E4%BB%A3%20React%20%E9%96%8B%E7%99%BC%EF%BC%9A%E5%87%BD%E5%BC%8F%E5%85%83%E4%BB%B6%E8%88%87%20Hooks.md)。

筆記倉庫：[https://github.com/nnngu/LearningNotes](https://github.com/nnngu/LearningNotes)

---

React 把元件看成是一個狀態機（State Machines）。透過與使用者的互動，實現不同狀態，然後渲染 UI，讓使用者介面和資料保持一致。

React 裡，只需更新元件的 state，然後根據新的 state 重新渲染使用者介面（不用操作 DOM）。

下面的例子中建立了 LikeButton 元件，getInitialState 方法用於定義初始狀態，也就是一個物件，這個物件可以透過 this.state 屬性讀取。當使用者點選元件，導致狀態變化，this.setState 方法就修改狀態值，每次修改以後，自動呼叫 this.render 方法，再次渲染元件。

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
    var LikeButton = React.createClass({
        getInitialState: function() {
            return {liked: false};
        },
        handleClick: function(event) {
            this.setState({liked: !this.state.liked});
        },
        render: function() {
            var text = this.state.liked ? '喜歡' : '不喜歡';
            return (
                <p onClick={this.handleClick}>
                    你<b>{text}</b>我。點我切換狀態。
                </p>
            );
        }
    });

    ReactDOM.render(
        <LikeButton />,
        document.getElementById('example')
    );
</script>
</body>

</html>
```

注意：onClick 等事件，on 後面第一個字母是大寫的！