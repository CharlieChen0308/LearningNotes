# 01 React 函式元件與 Hooks

> 本系列前五篇使用 React 15.x 的 `React.createClass` 和 CDN 引入方式。本篇對照 React 官方文件（react.dev），補充現代 React 19 的開發方式——函式元件（Function Components）與 Hooks。

## 開發方式演進對照

| 項目 | 舊版（第 01~05 篇） | 現代版（React 19） |
|------|---------------------|-------------------|
| React 版本 | 15.x | 19.x |
| 引入方式 | CDN `<script>` 標籤 | npm + 建構工具 |
| 元件定義 | `React.createClass({})` | 函式元件 `function App() {}` |
| 狀態管理 | `getInitialState` + `this.setState` | `useState` Hook |
| 生命週期 | `componentDidMount` 等 | `useEffect` Hook |
| Props | `this.props.name` | 函式參數 `props.name` |
| PropTypes | `React.PropTypes` | 獨立套件 `prop-types` |

## 建立 React 專案

現代 React 專案使用 Vite 作為建構工具：

```bash
npm create vite@latest my-react-app -- --template react
cd my-react-app
npm install
npm run dev
```

## 函式元件

### 第 01 篇的 HelloMessage 元件（對比）

**舊版（React.createClass）：**

```jsx
var HelloMessage = React.createClass({
    render: function() {
        return <h1>Hello {this.props.name}</h1>;
    }
});
```

**現代版（函式元件）：**

```jsx
function HelloMessage({ name }) {
    return <h1>Hello {name}</h1>;
}

// 或使用箭頭函式
const HelloMessage = ({ name }) => <h1>Hello {name}</h1>;
```

## useState — 狀態管理

### 第 02 篇的 LikeButton 元件（對比）

**舊版：**

```jsx
var LikeButton = React.createClass({
    getInitialState: function() {
        return { liked: false };
    },
    handleClick: function() {
        this.setState({ liked: !this.state.liked });
    },
    render: function() {
        var text = this.state.liked ? '喜歡' : '不喜歡';
        return <p onClick={this.handleClick}>你<b>{text}</b>我</p>;
    }
});
```

**現代版：**

```jsx
import { useState } from 'react';

function LikeButton() {
    const [liked, setLiked] = useState(false);

    return (
        <p onClick={() => setLiked(!liked)}>
            你<b>{liked ? '喜歡' : '不喜歡'}</b>我
        </p>
    );
}
```

### 多個狀態

```jsx
function UserForm() {
    const [name, setName] = useState('');
    const [age, setAge] = useState(0);
    const [email, setEmail] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        console.log({ name, age, email });
    };

    return (
        <form onSubmit={handleSubmit}>
            <input value={name} onChange={e => setName(e.target.value)} placeholder="姓名" />
            <input type="number" value={age} onChange={e => setAge(Number(e.target.value))} placeholder="年齡" />
            <input value={email} onChange={e => setEmail(e.target.value)} placeholder="Email" />
            <button type="submit">送出</button>
        </form>
    );
}
```

## Props — 父子元件通訊

### 第 03 篇的 Props 範例（對比）

**舊版：**

```jsx
var WebSite = React.createClass({
    getInitialState: function() {
        return { name: "LearningNotes", site: "https://github.com/nnngu" };
    },
    render: function() {
        return (
            <div>
                <Name name={this.state.name} />
                <Link site={this.state.site} />
            </div>
        );
    }
});
```

**現代版：**

```jsx
import { useState } from 'react';

function WebSite() {
    const [name] = useState('LearningNotes');
    const [site] = useState('https://github.com/nnngu');

    return (
        <div>
            <Name name={name} />
            <Link site={site} />
        </div>
    );
}

function Name({ name }) {
    return <h1>{name}</h1>;
}

function Link({ site }) {
    return <a href={site}>{site}</a>;
}
```

### 預設 Props

**舊版：** `getDefaultProps()`

**現代版：** 直接使用 JavaScript 預設參數

```jsx
function HelloMessage({ name = 'World' }) {
    return <h1>Hello {name}</h1>;
}
```

## useEffect — 副作用處理

取代舊版的 `componentDidMount`、`componentDidUpdate`、`componentWillUnmount`：

```jsx
import { useState, useEffect } from 'react';

function Clock() {
    const [time, setTime] = useState(new Date());

    useEffect(() => {
        // 等同 componentDidMount + componentDidUpdate
        const timer = setInterval(() => {
            setTime(new Date());
        }, 1000);

        // 返回清理函式，等同 componentWillUnmount
        return () => clearInterval(timer);
    }, []);  // 空陣列 = 只在 mount 時執行一次

    return <h2>現在時間：{time.toLocaleTimeString()}</h2>;
}
```

### useEffect 的依賴陣列

```jsx
// 每次渲染都執行
useEffect(() => { ... });

// 只在 mount 時執行一次
useEffect(() => { ... }, []);

// 當 userId 變化時執行
useEffect(() => {
    fetchUser(userId);
}, [userId]);
```

## 常用 Hooks 總覽

| Hook | 用途 |
|------|------|
| `useState` | 管理元件狀態 |
| `useEffect` | 副作用處理（API 呼叫、訂閱、計時器） |
| `useContext` | 跨元件共享資料（取代 Context.Consumer） |
| `useRef` | 取得 DOM 元素參考 / 保存不觸發渲染的值 |
| `useMemo` | 快取計算結果，避免不必要的重算 |
| `useCallback` | 快取函式參考，避免不必要的子元件重新渲染 |
| `useReducer` | 複雜狀態邏輯（類似 Redux 的 reducer） |

## 完整範例：待辦事項

```jsx
import { useState } from 'react';

function TodoApp() {
    const [todos, setTodos] = useState([]);
    const [input, setInput] = useState('');

    const addTodo = () => {
        if (input.trim() === '') return;
        setTodos([...todos, { id: Date.now(), text: input, done: false }]);
        setInput('');
    };

    const toggleTodo = (id) => {
        setTodos(todos.map(todo =>
            todo.id === id ? { ...todo, done: !todo.done } : todo
        ));
    };

    const deleteTodo = (id) => {
        setTodos(todos.filter(todo => todo.id !== id));
    };

    return (
        <div>
            <h1>待辦事項</h1>
            <input
                value={input}
                onChange={e => setInput(e.target.value)}
                onKeyDown={e => e.key === 'Enter' && addTodo()}
                placeholder="新增待辦事項..."
            />
            <button onClick={addTodo}>新增</button>

            <ul>
                {todos.map(todo => (
                    <li key={todo.id}>
                        <span
                            style={{ textDecoration: todo.done ? 'line-through' : 'none' }}
                            onClick={() => toggleTodo(todo.id)}>
                            {todo.text}
                        </span>
                        <button onClick={() => deleteTodo(todo.id)}>刪除</button>
                    </li>
                ))}
            </ul>

            <p>共 {todos.length} 項，已完成 {todos.filter(t => t.done).length} 項</p>
        </div>
    );
}

export default TodoApp;
```

## 小結

React 19 的函式元件 + Hooks 是現代 React 開發的標準方式，完全取代了 `React.createClass` 和 class 元件。`useState` 管理狀態、`useEffect` 處理副作用、函式參數接收 Props——整體更加簡潔直覺。

> **延伸閱讀**：
> - [05 React 進階與狀態管理](05%20React%20進階與狀態管理.md) — useContext、useReducer、React Router、Zustand 與效能最佳化
> - [02 TypeScript 基礎](02%20TypeScript%20基礎.md) — 為 React 元件加上型別安全
