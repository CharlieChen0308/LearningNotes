# 02 TypeScript 基礎

> **版本**：TypeScript 5.7.x

TypeScript 是 JavaScript 的超集（superset），由 Microsoft 開發維護。它在 JavaScript 之上加入了靜態型別系統，讓開發者能在編譯期就捕捉到型別錯誤，而非等到執行期才爆炸。對於後端出身的 Java 開發者來說，TypeScript 的型別概念會感到非常親切。

---

## 1、為什麼使用 TypeScript

### JavaScript 的痛點

JavaScript 是動態型別語言，變數可以隨時被賦予任意型別的值：

```javascript
let price = 100;
price = "免費";     // 合法，但邏輯上可能是 bug
price.toFixed(2);   // 執行期才會爆 TypeError
```

這類問題在小型專案中尚可接受，但當專案規模成長、多人協作時，缺乏型別檢查會導致大量隱性 bug。

### TypeScript 的三大優勢

**編譯期型別檢查**：在程式碼執行前就能發現型別錯誤。

```typescript
let price: number = 100;
price = "免費"; // 編譯錯誤：Type 'string' is not assignable to type 'number'
```

**IDE 智慧支援**：有了型別資訊，IDE 能提供精確的自動補全、參數提示、重構工具。不再需要靠記憶或翻文件確認 API。

**程式碼即文件**：型別定義本身就是最新、最準確的文件。函式簽名清楚標示了「需要什麼、回傳什麼」。

```typescript
// 一看就知道：接收使用者 ID（數字），回傳 Promise 包裹的 User 物件
function getUser(id: number): Promise<User> { ... }
```

### 安裝與編譯

```bash
npm install -D typescript
npx tsc --init          # 產生 tsconfig.json
npx tsc                 # 編譯所有 .ts 檔案為 .js
```

現代前端框架（Vue、React）搭配 Vite 時，TypeScript 已內建支援，不需額外設定編譯流程。

---

## 2、基礎型別

TypeScript 的型別系統涵蓋了 JavaScript 的所有原始型別，並額外提供數個實用型別。

### 原始型別

```typescript
// 字串、數字、布林
let name: string = "TireMaster";
let price: number = 3500;
let inStock: boolean = true;

// null 與 undefined
let empty: null = null;
let notDefined: undefined = undefined;
```

### 陣列與元組

```typescript
// 陣列：兩種寫法
let sizes: number[] = [14, 15, 16, 17];
let brands: Array<string> = ["Michelin", "Bridgestone"];

// 元組（Tuple）：固定長度、各位置型別不同
let tire: [string, number] = ["Michelin PS5", 3500];
// tire[0] 是 string，tire[1] 是 number
```

### 列舉 Enum

```typescript
enum TireType {
  Summer = "SUMMER",
  Winter = "WINTER",
  AllSeason = "ALL_SEASON"
}

let selected: TireType = TireType.Summer;
```

### 特殊型別

```typescript
// any：關閉型別檢查（盡量避免使用）
let legacy: any = "anything";
legacy = 42; // 不報錯，但失去型別安全

// unknown：比 any 安全，使用前必須先檢查型別
let input: unknown = "hello";
if (typeof input === "string") {
  console.log(input.toUpperCase()); // 通過型別檢查
}

// void：函式沒有回傳值
function logMessage(msg: string): void {
  console.log(msg);
}

// never：永遠不會有回傳值（拋出例外或無窮迴圈）
function throwError(msg: string): never {
  throw new Error(msg);
}
```

> **經驗法則**：優先用 `unknown` 取代 `any`。`unknown` 強制你在使用前做型別檢查，保留了型別安全性。

---

## 3、介面與型別別名

### interface：定義物件結構

```typescript
interface Tire {
  brand: string;
  model: string;
  size: number;
  price: number;
  dot?: string;            // 可選屬性（Optional）
  readonly id: number;     // 唯讀屬性
}

const tire: Tire = {
  id: 1,
  brand: "Michelin",
  model: "Pilot Sport 5",
  size: 17,
  price: 4200
  // dot 可省略
};

tire.id = 2; // 編譯錯誤：Cannot assign to 'id' because it is a read-only property
```

### type：型別別名

```typescript
type TireSize = 14 | 15 | 16 | 17 | 18 | 19 | 20;
type PaymentMethod = "cash" | "credit" | "transfer";

type TireInfo = {
  brand: string;
  model: string;
  size: TireSize;
};
```

### interface vs type

| 特性 | interface | type |
|------|-----------|------|
| 物件結構定義 | 適合 | 適合 |
| 聯合型別 `A \| B` | 不支援 | 支援 |
| 交叉型別 `A & B` | 用 extends | 用 `&` |
| 擴展 | `extends` | `&`（intersection） |
| 宣告合併 | 支援（同名自動合併） | 不支援 |
| 類別實作 | `implements` | `implements` |

**extends（介面繼承）：**

```typescript
interface Vehicle {
  brand: string;
  year: number;
}

interface Car extends Vehicle {
  doors: number;
}
```

**intersection（交叉型別）：**

```typescript
type Vehicle = {
  brand: string;
  year: number;
};

type Electric = {
  batteryCapacity: number;
};

type ElectricCar = Vehicle & Electric;
// 同時擁有 brand, year, batteryCapacity
```

> **經驗法則**：定義物件結構用 `interface`，定義聯合型別、工具型別用 `type`。

---

## 4、泛型

泛型（Generics）讓你撰寫可重用的型別安全程式碼，概念與 Java 的泛型相同。

### 泛型函式

```typescript
// 不用泛型：要嘛用 any 失去型別安全，要嘛每種型別寫一個函式
function identity<T>(value: T): T {
  return value;
}

const num = identity<number>(42);      // T = number
const str = identity("hello");          // TypeScript 自動推斷 T = string
```

### 泛型介面

```typescript
interface ApiResponse<T> {
  code: number;
  message: string;
  data: T;
}

// 使用時指定 T
type TireResponse = ApiResponse<Tire>;
type TireListResponse = ApiResponse<Tire[]>;
```

### 泛型約束（constraints with extends）

```typescript
// T 必須包含 length 屬性
function logLength<T extends { length: number }>(item: T): void {
  console.log(item.length);
}

logLength("hello");     // OK，string 有 length
logLength([1, 2, 3]);   // OK，array 有 length
logLength(123);          // 編譯錯誤：number 沒有 length
```

### 預設型別

```typescript
interface PaginatedResult<T = any> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
}

// 不指定 T 時，預設為 any
const result: PaginatedResult = { items: [], total: 0, page: 1, pageSize: 10 };

// 指定 T
const tireResult: PaginatedResult<Tire> = { ... };
```

---

## 5、實用工具型別

TypeScript 內建多個實用工具型別（Utility Types），能從已有型別快速衍生出新型別。

### Partial\<T\>：所有屬性變可選

```typescript
interface Tire {
  brand: string;
  model: string;
  price: number;
}

// 更新時不需要傳所有欄位
function updateTire(id: number, updates: Partial<Tire>): void {
  // updates 可以是 { price: 3800 }，不需帶 brand 和 model
}

updateTire(1, { price: 3800 }); // OK
```

### Required\<T\>：所有屬性變必填

```typescript
interface Config {
  host?: string;
  port?: number;
  debug?: boolean;
}

// 載入設定後，確保所有值都存在
const finalConfig: Required<Config> = {
  host: "localhost",
  port: 8080,
  debug: false
  // 少任何一個都會報錯
};
```

### Pick\<T, K\>：從 T 中選取部分屬性

```typescript
type TireSummary = Pick<Tire, "brand" | "model">;
// 等同 { brand: string; model: string; }
```

### Omit\<T, K\>：從 T 中排除部分屬性

```typescript
type CreateTireDto = Omit<Tire, "id" | "createdAt">;
// 建立時不需要 id 和 createdAt
```

### Record\<K, V\>：鍵值對映射

```typescript
type TireStock = Record<string, number>;

const stock: TireStock = {
  "Michelin-PS5-17": 12,
  "Bridgestone-RE71RS-16": 8
};
```

### Readonly\<T\>：所有屬性變唯讀

```typescript
const defaultConfig: Readonly<Config> = {
  host: "localhost",
  port: 3000,
  debug: false
};

defaultConfig.port = 8080; // 編譯錯誤：Cannot assign to 'port'
```

> 這些工具型別可以組合使用：`Partial<Pick<Tire, "price" | "stock">>` 表示「只有 price 和 stock，而且都是可選的」。

---

## 6、型別守衛

型別守衛（Type Guards）讓你在條件分支中縮窄（narrow）變數的型別。

### typeof

```typescript
function formatValue(value: string | number): string {
  if (typeof value === "string") {
    return value.toUpperCase();   // 這裡 TypeScript 知道 value 是 string
  }
  return value.toFixed(2);        // 這裡知道 value 是 number
}
```

### instanceof

```typescript
class ApiError extends Error {
  code: number;
  constructor(message: string, code: number) {
    super(message);
    this.code = code;
  }
}

function handleError(error: Error | ApiError): void {
  if (error instanceof ApiError) {
    console.log(`API Error ${error.code}: ${error.message}`);
  } else {
    console.log(`Error: ${error.message}`);
  }
}
```

### in 運算子

```typescript
interface Car { drive(): void; doors: number; }
interface Bike { ride(): void; gears: number; }

function move(vehicle: Car | Bike): void {
  if ("drive" in vehicle) {
    vehicle.drive();    // TypeScript 知道是 Car
  } else {
    vehicle.ride();     // TypeScript 知道是 Bike
  }
}
```

### 自定義型別守衛（is）

```typescript
interface Tire {
  brand: string;
  size: number;
}

interface Wheel {
  material: string;
  diameter: number;
}

// 回傳型別 `item is Tire` 告訴 TypeScript：若回傳 true，item 就是 Tire
function isTire(item: Tire | Wheel): item is Tire {
  return "brand" in item && "size" in item;
}

function describe(item: Tire | Wheel): string {
  if (isTire(item)) {
    return `${item.brand} ${item.size}吋`;    // 安全存取 Tire 屬性
  }
  return `${item.material} ${item.diameter}吋`;
}
```

---

## 7、與 Java 概念對照表

如果你是 Java 開發者，以下對照表能幫助你快速理解 TypeScript 的對應概念：

| 概念 | Java | TypeScript |
|------|------|-----------|
| 介面 | `interface`（方法簽名） | `interface`（物件結構，含屬性） |
| 泛型 | `<T extends Comparable<T>>` | `<T extends { length: number }>` |
| 泛型擦除 | 有（Runtime 無泛型資訊） | 有（編譯為 JS 後型別全部消失） |
| 列舉 | `enum`（類別，可有方法和屬性） | `enum`（較簡單，通常是常數映射） |
| 型別推斷 | `var`（Java 10+） | `let` / `const`（推斷更強大） |
| 空安全 | `Optional<T>` | `T \| null`、可選鏈 `?.`、`??` |
| 介面多繼承 | `extends A, B`（介面層） | `extends A` 或 `A & B`（交叉型別） |
| 型別聯合 | 不支援（靠多型） | `string \| number` 原生支援 |
| 裝飾器 | 註解 `@Override` | 裝飾器 `@decorator`（Stage 3） |
| 存取修飾 | `public/private/protected` | 相同關鍵字（僅編譯期檢查） |
| 不可變 | `final` | `readonly`（屬性）/ `as const`（值） |

**關鍵差異**：

- TypeScript 的型別系統是「結構型別」（Structural Typing），只要結構吻合就相容。Java 是「名義型別」（Nominal Typing），必須明確宣告實作。
- TypeScript 的型別只存在於編譯期，產出的 JavaScript 完全沒有型別資訊。Java 的型別在 Runtime 仍然存在（反射）。
- TypeScript 的 `interface` 可以描述物件的形狀（包含屬性），Java 的 `interface` 主要用於定義方法契約。

---

## 8、小結

TypeScript 在不改變 JavaScript 執行環境的前提下，提供了強大的靜態型別系統。核心觀念整理：

- **基礎型別**：涵蓋 `string`、`number`、`boolean` 等原始型別，以及 `any`、`unknown`、`never` 等特殊型別
- **interface vs type**：物件結構優先用 `interface`，聯合型別和組合用 `type`
- **泛型**：撰寫可重用且型別安全的程式碼，搭配 `extends` 做約束
- **工具型別**：`Partial`、`Pick`、`Omit` 等大幅減少重複的型別定義
- **型別守衛**：在條件分支中安全地縮窄型別

掌握這些基礎後，就能順利進入前端框架的 TypeScript 開發：

- 下一篇 [03 Vue 3 Composition API](./03%20Vue%203%20Composition%20API.md)：在 Vue 3 中運用 TypeScript
- 參考 [01 React 函式元件與 Hooks](./01%20React%20函式元件與%20Hooks.md)：React 生態系的現代開發方式

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
