# 05 React 進階與狀態管理

> **版本**：React 19 / TypeScript / React Router v7

掌握 `useState` 和 `useEffect` 之後，下一步是學會跨元件狀態共享、複雜狀態邏輯、路由導航，以及全域狀態管理。本篇涵蓋 `useContext`、`useReducer`、自定義 Hook、React Router v7、Zustand，以及效能最佳化策略。

---

## 1、useContext

當多個元件需要共享同一份資料時（例如使用者登入狀態、語言設定、主題切換），逐層傳遞 Props（prop drilling）會讓程式碼變得冗長且難以維護。`useContext` 讓任何深度的子元件都能直接存取共享資料。

### 建立 Context 與 Provider

```tsx
// context/AuthContext.tsx
import { createContext, useContext, useState, type ReactNode } from 'react';

// 1. 定義型別
interface AuthContextType {
    user: string | null;
    login: (username: string) => void;
    logout: () => void;
}

// 2. 建立 Context（給定預設值）
const AuthContext = createContext<AuthContextType | null>(null);

// 3. 建立 Provider 元件
export function AuthProvider({ children }: { children: ReactNode }) {
    const [user, setUser] = useState<string | null>(null);

    const login = (username: string) => setUser(username);
    const logout = () => setUser(null);

    return (
        <AuthContext.Provider value={{ user, login, logout }}>
            {children}
        </AuthContext.Provider>
    );
}

// 4. 建立自定義 Hook，方便消費
export function useAuth() {
    const context = useContext(AuthContext);
    if (!context) {
        throw new Error('useAuth 必須在 AuthProvider 內使用');
    }
    return context;
}
```

### 在子元件中消費

```tsx
// components/Navbar.tsx
import { useAuth } from '../context/AuthContext';

function Navbar() {
    const { user, logout } = useAuth();

    return (
        <nav>
            {user ? (
                <>
                    <span>歡迎，{user}</span>
                    <button onClick={logout}>登出</button>
                </>
            ) : (
                <span>請先登入</span>
            )}
        </nav>
    );
}
```

在 App 最外層包上 Provider：

```tsx
function App() {
    return (
        <AuthProvider>
            <Navbar />
            <MainContent />
        </AuthProvider>
    );
}
```

**注意**：Context 值變化時，所有消費該 Context 的元件都會重新渲染。如果 Context 包含太多不相關的資料，會導致不必要的渲染。解法是將不同關注點拆成多個 Context。

---

## 2、useReducer

當元件的狀態邏輯變得複雜（多個欄位互相關聯、狀態轉換有明確規則），`useReducer` 比多個 `useState` 更容易維護和測試。

### dispatch + action 模式

```tsx
import { useReducer } from 'react';

// 定義狀態型別
interface FormState {
    name: string;
    email: string;
    loading: boolean;
    error: string | null;
}

// 定義 Action 型別（Discriminated Union）
type FormAction =
    | { type: 'SET_FIELD'; field: keyof FormState; value: string }
    | { type: 'SUBMIT_START' }
    | { type: 'SUBMIT_SUCCESS' }
    | { type: 'SUBMIT_ERROR'; error: string };

const initialState: FormState = {
    name: '',
    email: '',
    loading: false,
    error: null,
};

// Reducer：純函式，易於測試
function formReducer(state: FormState, action: FormAction): FormState {
    switch (action.type) {
        case 'SET_FIELD':
            return { ...state, [action.field]: action.value };
        case 'SUBMIT_START':
            return { ...state, loading: true, error: null };
        case 'SUBMIT_SUCCESS':
            return { ...initialState };
        case 'SUBMIT_ERROR':
            return { ...state, loading: false, error: action.error };
        default:
            return state;
    }
}

function ContactForm() {
    const [state, dispatch] = useReducer(formReducer, initialState);

    const handleSubmit = async () => {
        dispatch({ type: 'SUBMIT_START' });
        try {
            await fetch('/api/contact', {
                method: 'POST',
                body: JSON.stringify({ name: state.name, email: state.email }),
            });
            dispatch({ type: 'SUBMIT_SUCCESS' });
        } catch (err) {
            dispatch({ type: 'SUBMIT_ERROR', error: '送出失敗，請稍後再試' });
        }
    };

    return (
        <form onSubmit={e => { e.preventDefault(); handleSubmit(); }}>
            <input
                value={state.name}
                onChange={e => dispatch({ type: 'SET_FIELD', field: 'name', value: e.target.value })}
            />
            <input
                value={state.email}
                onChange={e => dispatch({ type: 'SET_FIELD', field: 'email', value: e.target.value })}
            />
            <button disabled={state.loading}>
                {state.loading ? '送出中...' : '送出'}
            </button>
            {state.error && <p style={{ color: 'red' }}>{state.error}</p>}
        </form>
    );
}
```

### useState vs useReducer 選擇時機

| 情境 | 推薦 |
|------|------|
| 單一值、簡單切換（boolean、string） | `useState` |
| 多個欄位、互相關聯的狀態 | `useReducer` |
| 狀態轉換有明確規則（如表單流程、狀態機） | `useReducer` |
| 需要將狀態邏輯抽離元件測試 | `useReducer` |

---

## 3、自定義 Hook

自定義 Hook 是 React 最強大的程式碼複用機制。它的規則很簡單：**函式名稱以 `use` 開頭**，內部可以呼叫其他 Hook。

### useFetch — 通用 API 請求

```tsx
import { useState, useEffect } from 'react';

interface UseFetchResult<T> {
    data: T | null;
    loading: boolean;
    error: string | null;
}

function useFetch<T>(url: string): UseFetchResult<T> {
    const [data, setData] = useState<T | null>(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState<string | null>(null);

    useEffect(() => {
        const controller = new AbortController();

        setLoading(true);
        fetch(url, { signal: controller.signal })
            .then(res => {
                if (!res.ok) throw new Error(`HTTP ${res.status}`);
                return res.json();
            })
            .then(setData)
            .catch(err => {
                if (err.name !== 'AbortError') setError(err.message);
            })
            .finally(() => setLoading(false));

        return () => controller.abort(); // 元件卸載時取消請求
    }, [url]);

    return { data, loading, error };
}

// 使用方式
function UserList() {
    const { data: users, loading, error } = useFetch<User[]>('/api/users');
    if (loading) return <p>載入中...</p>;
    if (error) return <p>錯誤：{error}</p>;
    return <ul>{users?.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### useDebounce — 延遲搜尋

```tsx
import { useState, useEffect } from 'react';

function useDebounce<T>(value: T, delay: number = 300): T {
    const [debouncedValue, setDebouncedValue] = useState(value);

    useEffect(() => {
        const timer = setTimeout(() => setDebouncedValue(value), delay);
        return () => clearTimeout(timer);
    }, [value, delay]);

    return debouncedValue;
}

// 使用方式：搜尋輸入框
function SearchBar() {
    const [query, setQuery] = useState('');
    const debouncedQuery = useDebounce(query, 500);

    useEffect(() => {
        if (debouncedQuery) {
            fetch(`/api/search?q=${debouncedQuery}`);
        }
    }, [debouncedQuery]);

    return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

### useLocalStorage — 持久化狀態

```tsx
import { useState } from 'react';

function useLocalStorage<T>(key: string, initialValue: T) {
    const [storedValue, setStoredValue] = useState<T>(() => {
        try {
            const item = window.localStorage.getItem(key);
            return item ? JSON.parse(item) : initialValue;
        } catch {
            return initialValue;
        }
    });

    const setValue = (value: T | ((prev: T) => T)) => {
        const valueToStore = value instanceof Function ? value(storedValue) : value;
        setStoredValue(valueToStore);
        window.localStorage.setItem(key, JSON.stringify(valueToStore));
    };

    return [storedValue, setValue] as const;
}

// 使用方式
const [theme, setTheme] = useLocalStorage('theme', 'light');
```

---

## 4、React Router v7

React Router v7 是 React 生態系的標準路由方案，採用 Data Router 架構，將資料載入（loader）與路由定義結合。

### createBrowserRouter 與路由設定

```tsx
// router.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import RootLayout from './layouts/RootLayout';
import Home from './pages/Home';
import Products from './pages/Products';
import ProductDetail from './pages/ProductDetail';
import Login from './pages/Login';

const router = createBrowserRouter([
    {
        path: '/',
        element: <RootLayout />,   // 巢狀路由的共用佈局
        children: [
            { index: true, element: <Home /> },
            { path: 'products', element: <Products />, loader: productsLoader },
            { path: 'products/:id', element: <ProductDetail />, loader: productLoader },
            { path: 'login', element: <Login /> },
        ],
    },
]);

function App() {
    return <RouterProvider router={router} />;
}
```

### 巢狀路由與 Outlet

```tsx
// layouts/RootLayout.tsx
import { Outlet, Link } from 'react-router-dom';

function RootLayout() {
    return (
        <div>
            <nav>
                <Link to="/">首頁</Link>
                <Link to="/products">商品</Link>
            </nav>
            <main>
                <Outlet />  {/* 子路由渲染在這裡 */}
            </main>
        </div>
    );
}
```

### useParams 與 loader

```tsx
// pages/ProductDetail.tsx
import { useLoaderData, useParams, type LoaderFunctionArgs } from 'react-router-dom';

// loader：在路由匹配時自動載入資料
export async function productLoader({ params }: LoaderFunctionArgs) {
    const res = await fetch(`/api/products/${params.id}`);
    if (!res.ok) throw new Response('找不到商品', { status: 404 });
    return res.json();
}

function ProductDetail() {
    const product = useLoaderData() as Product;
    const { id } = useParams();

    return (
        <div>
            <h1>{product.name}</h1>
            <p>商品編號：{id}</p>
        </div>
    );
}
```

### useNavigate 與保護路由

```tsx
import { Navigate, useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

// 程式化導航
function LoginForm() {
    const navigate = useNavigate();

    const handleLogin = async () => {
        await loginAPI();
        navigate('/dashboard');      // 登入成功後跳轉
        // navigate(-1);             // 回上一頁
    };

    return <button onClick={handleLogin}>登入</button>;
}

// 保護路由元件
function ProtectedRoute({ children }: { children: React.ReactNode }) {
    const { user } = useAuth();
    if (!user) return <Navigate to="/login" replace />;
    return <>{children}</>;
}

// 在路由設定中使用
{
    path: 'dashboard',
    element: <ProtectedRoute><Dashboard /></ProtectedRoute>,
}
```

---

## 5、Zustand 狀態管理

### 為什麼不用 Redux

Redux 是 React 生態系的老牌狀態管理方案，功能強大但樣板程式碼多（action types、action creators、reducers、middleware、connect/useSelector）。Zustand 提供同等的全域狀態管理能力，但 API 極度精簡。

| 比較項目 | Redux Toolkit | Zustand |
|---------|--------------|---------|
| 樣板程式碼 | 中等（slice + store 設定） | 極少（一個 `create` 呼叫） |
| Bundle 大小 | ~12 KB | ~1 KB |
| Provider 需求 | 需要 `<Provider>` 包裹 | 不需要 |
| DevTools | 內建支援 | 需額外設定（但也支援） |
| 學習曲線 | 中等 | 低 |
| 適用場景 | 大型團隊、複雜業務 | 中小型專案、快速開發 |

### create store

```tsx
// stores/useCartStore.ts
import { create } from 'zustand';

interface CartItem {
    id: string;
    name: string;
    price: number;
    quantity: number;
}

interface CartStore {
    items: CartItem[];
    addItem: (item: Omit<CartItem, 'quantity'>) => void;
    removeItem: (id: string) => void;
    updateQuantity: (id: string, quantity: number) => void;
    totalPrice: () => number;
    clearCart: () => void;
}

export const useCartStore = create<CartStore>((set, get) => ({
    items: [],

    addItem: (item) =>
        set((state) => {
            const existing = state.items.find(i => i.id === item.id);
            if (existing) {
                return {
                    items: state.items.map(i =>
                        i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
                    ),
                };
            }
            return { items: [...state.items, { ...item, quantity: 1 }] };
        }),

    removeItem: (id) =>
        set((state) => ({ items: state.items.filter(i => i.id !== id) })),

    updateQuantity: (id, quantity) =>
        set((state) => ({
            items: state.items.map(i => (i.id === id ? { ...i, quantity } : i)),
        })),

    totalPrice: () =>
        get().items.reduce((sum, item) => sum + item.price * item.quantity, 0),

    clearCart: () => set({ items: [] }),
}));
```

### 使用 selector 避免不必要的渲染

```tsx
function CartIcon() {
    // 只訂閱 items.length，其他狀態變化不會觸發重新渲染
    const itemCount = useCartStore((state) => state.items.length);
    return <span>購物車 ({itemCount})</span>;
}

function CartTotal() {
    const totalPrice = useCartStore((state) => state.totalPrice());
    return <p>總計：${totalPrice}</p>;
}

function AddToCartButton({ product }: { product: Product }) {
    const addItem = useCartStore((state) => state.addItem);
    return <button onClick={() => addItem(product)}>加入購物車</button>;
}
```

### persist middleware — 持久化儲存

```tsx
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useCartStore = create<CartStore>()(
    persist(
        (set, get) => ({
            items: [],
            addItem: (item) => set((state) => ({ /* ... */ })),
            // ...其他 action
        }),
        {
            name: 'cart-storage',       // localStorage 的 key
            // storage: createJSONStorage(() => sessionStorage),  // 可切換為 sessionStorage
        }
    )
);
```

重新整理頁面後，購物車資料仍然保留——Zustand 會自動從 localStorage 還原狀態。

---

## 6、效能最佳化

React 的渲染機制是：**當元件的 state 或 props 變化時，該元件及其所有子元件都會重新渲染**。大多數情況下這不是問題，但當渲染成本較高時，可以使用以下工具進行最佳化。

### React.memo — 避免子元件不必要的重新渲染

```tsx
interface ProductCardProps {
    name: string;
    price: number;
    onAddToCart: () => void;
}

// 用 React.memo 包裹後，只有 props 實際變化時才重新渲染
const ProductCard = React.memo(function ProductCard({ name, price, onAddToCart }: ProductCardProps) {
    console.log(`渲染 ProductCard: ${name}`);
    return (
        <div>
            <h3>{name}</h3>
            <p>${price}</p>
            <button onClick={onAddToCart}>加入購物車</button>
        </div>
    );
});
```

### useMemo — 快取計算結果

```tsx
function OrderSummary({ items }: { items: CartItem[] }) {
    // 只有 items 變化時才重新計算，避免每次渲染都遍歷陣列
    const stats = useMemo(() => ({
        totalItems: items.reduce((sum, i) => sum + i.quantity, 0),
        totalPrice: items.reduce((sum, i) => sum + i.price * i.quantity, 0),
        uniqueItems: items.length,
    }), [items]);

    return (
        <div>
            <p>{stats.uniqueItems} 種商品，共 {stats.totalItems} 件</p>
            <p>總金額：${stats.totalPrice}</p>
        </div>
    );
}
```

### useCallback — 快取函式參考

```tsx
function ProductList({ products }: { products: Product[] }) {
    const addItem = useCartStore((state) => state.addItem);

    // 如果不用 useCallback，每次 ProductList 渲染都會產生新的函式參考
    // 導致所有 ProductCard（即使用了 React.memo）都重新渲染
    const handleAddToCart = useCallback((product: Product) => {
        addItem(product);
    }, [addItem]);

    return (
        <div>
            {products.map(p => (
                <ProductCard
                    key={p.id}
                    name={p.name}
                    price={p.price}
                    onAddToCart={() => handleAddToCart(p)}
                />
            ))}
        </div>
    );
}
```

### 何時使用、何時不使用

| 工具 | 適合使用 | 不需要使用 |
|------|---------|----------|
| `React.memo` | 渲染成本高的元件（複雜 UI、長列表項目） | 輕量元件、props 本來就經常變化 |
| `useMemo` | 複雜計算（排序、篩選、統計大量資料） | 簡單計算（字串串接、簡單算術） |
| `useCallback` | 傳給 `React.memo` 子元件的回呼函式 | 不傳給子元件的函式、子元件未用 memo |

**原則**：先寫出正確、可讀的程式碼，**確認有效能問題後**再加入最佳化。過早最佳化反而增加程式碼複雜度。React 團隊也在持續改進編譯器（React Compiler），未來有可能自動處理這些最佳化。

---

## 7、小結

本篇涵蓋了 React 進階開發的核心知識：

- **useContext**：跨元件資料共享，解決 prop drilling
- **useReducer**：複雜狀態邏輯的 dispatch + action 模式
- **自定義 Hook**：封裝可複用的邏輯（useFetch、useDebounce、useLocalStorage）
- **React Router v7**：Data Router 架構、loader、巢狀路由、保護路由
- **Zustand**：輕量全域狀態管理，搭配 selector 與 persist middleware
- **效能最佳化**：React.memo、useMemo、useCallback 的正確使用時機

延伸閱讀：

- **React 函式元件與 Hooks**：`06-Frontend/01 React 函式元件與 Hooks.md`
- **TypeScript 基礎**：`06-Frontend/02 TypeScript 基礎.md`
