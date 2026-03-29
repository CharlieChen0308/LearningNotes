# 03 Vue 3 Composition API

> **版本**：Vue 3.5.x / TypeScript / Vite

Vue 3 的 Composition API 是一種全新的元件邏輯組織方式，讓開發者能以「功能」為單位組織程式碼，而非被迫分散到 `data`、`methods`、`computed` 等選項中。搭配 TypeScript，能獲得完整的型別推斷與 IDE 支援。

---

## 1、Options API vs Composition API

### 比較表

| 項目 | Options API | Composition API |
|------|-------------|-----------------|
| 組織方式 | 依選項分類（data, methods, computed） | 依功能邏輯分組 |
| 邏輯重用 | Mixins（有命名衝突風險） | Composables（明確匯入匯出） |
| TypeScript 支援 | 需要額外宣告，推斷有限 | 原生友好，完整型別推斷 |
| 學習曲線 | 較低（結構固定） | 略高（自由度大） |
| 程式碼組織 | 相關邏輯被選項拆散 | 相關邏輯集中在一起 |

### 為什麼 Composition API 更好

Options API 的問題在於：當元件變複雜時，同一個功能的程式碼被分散到 `data`、`computed`、`methods`、`watch` 等不同區塊。假設一個元件同時處理「搜尋」和「分頁」，你必須在各個選項區塊間跳來跳去。

Composition API 讓你把「搜尋」的所有邏輯放在一起、「分頁」的所有邏輯放在一起，甚至可以抽取為獨立的 composable 函式。

```vue
<script setup lang="ts">
// 搜尋相關邏輯集中
const keyword = ref("")
const searchResults = computed(() => { ... })
function handleSearch() { ... }

// 分頁相關邏輯集中
const currentPage = ref(1)
const pageSize = ref(10)
function changePage(page: number) { ... }
</script>
```

---

## 2、響應式核心

### ref() vs reactive()

`ref()` 和 `reactive()` 是 Vue 3 的兩個響應式 API，任何被包裹的資料在變更時都會自動觸發畫面更新。

**ref()：包裹單一值**

```typescript
import { ref } from "vue"

const count = ref<number>(0)       // Ref<number>
const name = ref<string>("Vue")    // Ref<string>
const isReady = ref(false)         // 自動推斷為 Ref<boolean>

// 在 script 中需要 .value 存取
count.value++
console.log(count.value) // 1

// 在 template 中自動解包，不需要 .value
// <template>{{ count }}</template>
```

**reactive()：包裹物件**

```typescript
import { reactive } from "vue"

interface SearchForm {
  keyword: string
  category: string
  page: number
}

const form = reactive<SearchForm>({
  keyword: "",
  category: "all",
  page: 1
})

// 直接存取屬性，不需要 .value
form.keyword = "Michelin"
form.page = 2
```

**何時用哪個？**

| 場景 | 推薦 | 原因 |
|------|------|------|
| 原始值（string, number, boolean） | `ref()` | `reactive()` 不支援原始值 |
| 物件/陣列 | `ref()` 或 `reactive()` 皆可 | `ref()` 更通用，可整個替換 |
| 表單資料 | `reactive()` | 多個欄位，存取不需 `.value` |
| 函式回傳值 | `ref()` | `reactive()` 解構會失去響應性 |

> **經驗法則**：不確定時用 `ref()`，它適用於所有情境。`reactive()` 的主要陷阱是解構會失去響應性。

### computed()

```typescript
import { ref, computed } from "vue"

const price = ref(3500)
const quantity = ref(4)

// 唯讀 computed（getter only）
const total = computed(() => price.value * quantity.value)
console.log(total.value) // 14000

// 可寫 computed（getter + setter）
const displayPrice = computed({
  get: () => `NT$ ${price.value.toLocaleString()}`,
  set: (val: string) => {
    price.value = parseInt(val.replace(/[^\d]/g, ""), 10)
  }
})
```

### watch() vs watchEffect()

```typescript
import { ref, watch, watchEffect } from "vue"

const keyword = ref("")
const category = ref("all")

// watch()：明確指定監聽對象，可取得新舊值
watch(keyword, (newVal, oldVal) => {
  console.log(`搜尋從 "${oldVal}" 變為 "${newVal}"`)
  fetchResults(newVal)
})

// 監聽多個來源
watch([keyword, category], ([newKeyword, newCategory]) => {
  fetchResults(newKeyword, newCategory)
})

// watch 選項
watch(keyword, (val) => { fetchResults(val) }, {
  immediate: true,    // 立即執行一次
  deep: true          // 深層監聽（物件內部變化）
})

// watchEffect()：自動追蹤內部使用的響應式依賴
watchEffect(() => {
  // 自動追蹤 keyword.value 和 category.value
  console.log(`搜尋 ${keyword.value}，分類 ${category.value}`)
})
```

**差異**：`watch()` 需要明確指定監聽目標，能取得新舊值；`watchEffect()` 自動收集依賴，但無法取得舊值。

---

## 3、生命週期鉤子

Composition API 中，生命週期鉤子以 `on` 開頭的函式形式使用：

```typescript
import { onMounted, onUpdated, onUnmounted, onBeforeMount, onBeforeUpdate } from "vue"

// 元件掛載後（DOM 已就緒）
onMounted(() => {
  console.log("元件已掛載")
  fetchInitialData()
})

// DOM 更新後
onUpdated(() => {
  console.log("DOM 已更新")
})

// 元件卸載前（清理資源）
onUnmounted(() => {
  console.log("元件即將卸載")
  clearInterval(timer)
  unsubscribe()
})
```

| Options API | Composition API | 用途 |
|-------------|-----------------|------|
| `beforeCreate` / `created` | `setup()` 本身 | 初始化（`<script setup>` 即為 setup） |
| `beforeMount` | `onBeforeMount()` | DOM 掛載前 |
| `mounted` | `onMounted()` | DOM 掛載後，常用於 API 呼叫 |
| `beforeUpdate` | `onBeforeUpdate()` | DOM 更新前 |
| `updated` | `onUpdated()` | DOM 更新後 |
| `beforeUnmount` | `onBeforeUnmount()` | 卸載前，清理資源 |
| `unmounted` | `onUnmounted()` | 卸載後 |

---

## 4、組件通訊

### defineProps()：父傳子

```vue
<!-- ChildComponent.vue -->
<script setup lang="ts">
interface Props {
  title: string
  count?: number       // 可選，預設 0
  items: string[]
}

const props = withDefaults(defineProps<Props>(), {
  count: 0
})

// 直接使用 props.title, props.count
</script>

<template>
  <h2>{{ title }}</h2>
  <span>數量：{{ count }}</span>
</template>
```

```vue
<!-- ParentComponent.vue -->
<template>
  <ChildComponent title="輪胎清單" :count="5" :items="['A', 'B']" />
</template>
```

### defineEmits()：子傳父

```vue
<!-- ChildComponent.vue -->
<script setup lang="ts">
const emit = defineEmits<{
  (e: "update", id: number): void
  (e: "delete", id: number): void
  (e: "search", keyword: string, page: number): void
}>()

function handleClick(id: number) {
  emit("update", id)
}
</script>
```

```vue
<!-- ParentComponent.vue -->
<template>
  <ChildComponent
    @update="handleUpdate"
    @delete="handleDelete"
    @search="handleSearch"
  />
</template>

<script setup lang="ts">
function handleUpdate(id: number) {
  console.log("更新", id)
}

function handleDelete(id: number) {
  console.log("刪除", id)
}

function handleSearch(keyword: string, page: number) {
  console.log("搜尋", keyword, "頁碼", page)
}
</script>
```

### provide / inject：跨層傳遞

當資料需要跨越多層元件傳遞（避免 prop drilling）時使用：

```typescript
// 祖先元件
import { provide, ref } from "vue"

const theme = ref("light")
provide("theme", theme)         // 提供響應式資料
provide("appName", "TireMaster") // 提供靜態值

// 後代元件（任意深度）
import { inject } from "vue"

const theme = inject<Ref<string>>("theme", ref("light"))  // 第二參數是預設值
const appName = inject<string>("appName", "Default")
```

---

## 5、組合式函式（Composables）

Composables 是 Composition API 最強大的特性之一：將可重用的邏輯抽取為獨立函式。

### 命名規範

- 檔案名稱與函式名稱以 `use` 開頭
- 放在 `composables/` 目錄下

### 範例：useFetch

```typescript
// composables/useFetch.ts
import { ref, type Ref } from "vue"

interface UseFetchReturn<T> {
  data: Ref<T | null>
  error: Ref<string | null>
  loading: Ref<boolean>
  execute: () => Promise<void>
}

export function useFetch<T>(url: string): UseFetchReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<string | null>(null)
  const loading = ref(false)

  async function execute() {
    loading.value = true
    error.value = null
    try {
      const response = await fetch(url)
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      data.value = await response.json()
    } catch (e) {
      error.value = (e as Error).message
    } finally {
      loading.value = false
    }
  }

  return { data, error, loading, execute }
}
```

```vue
<!-- 在元件中使用 -->
<script setup lang="ts">
import { onMounted } from "vue"
import { useFetch } from "@/composables/useFetch"

interface Tire {
  id: number
  brand: string
  model: string
}

const { data: tires, loading, error, execute } = useFetch<Tire[]>("/api/tires")

onMounted(() => execute())
</script>

<template>
  <div v-if="loading">載入中...</div>
  <div v-else-if="error">錯誤：{{ error }}</div>
  <ul v-else>
    <li v-for="tire in tires" :key="tire.id">{{ tire.brand }} {{ tire.model }}</li>
  </ul>
</template>
```

### 範例：useLocalStorage

```typescript
// composables/useLocalStorage.ts
import { ref, watch, type Ref } from "vue"

export function useLocalStorage<T>(key: string, defaultValue: T): Ref<T> {
  const stored = localStorage.getItem(key)
  const data = ref<T>(stored ? JSON.parse(stored) : defaultValue) as Ref<T>

  watch(data, (newValue) => {
    localStorage.setItem(key, JSON.stringify(newValue))
  }, { deep: true })

  return data
}
```

```vue
<script setup lang="ts">
import { useLocalStorage } from "@/composables/useLocalStorage"

// 自動同步到 localStorage
const settings = useLocalStorage("user-settings", {
  theme: "light",
  pageSize: 10
})

settings.value.theme = "dark" // 自動寫入 localStorage
</script>
```

---

## 6、Pinia 狀態管理

### 為什麼用 Pinia 取代 Vuex

Pinia 是 Vue 官方推薦的狀態管理方案，取代了 Vuex：

| 項目 | Vuex | Pinia |
|------|------|-------|
| API 風格 | Options（mutations, actions） | Composition API 友好 |
| TypeScript | 需要額外型別宣告 | 原生完整支援 |
| Mutations | 必須透過 mutations 修改 state | 可直接修改 state |
| 模組 | 巢狀模組，需要 namespaced | 扁平 store，各自獨立 |
| 體積 | 較大 | 約 1KB |

### 安裝

```bash
npm install pinia
```

```typescript
// main.ts
import { createApp } from "vue"
import { createPinia } from "pinia"
import App from "./App.vue"

const app = createApp(App)
app.use(createPinia())
app.mount("#app")
```

### defineStore：定義 Store

```typescript
// stores/tireStore.ts
import { defineStore } from "pinia"
import { ref, computed } from "vue"
import type { Tire } from "@/types"

export const useTireStore = defineStore("tire", () => {
  // state
  const tires = ref<Tire[]>([])
  const loading = ref(false)
  const selectedCategory = ref<string>("all")

  // getters（用 computed）
  const filteredTires = computed(() => {
    if (selectedCategory.value === "all") return tires.value
    return tires.value.filter(t => t.category === selectedCategory.value)
  })

  const totalCount = computed(() => tires.value.length)

  // actions（普通函式）
  async function fetchTires() {
    loading.value = true
    try {
      const response = await fetch("/api/tires")
      tires.value = await response.json()
    } finally {
      loading.value = false
    }
  }

  function setCategory(category: string) {
    selectedCategory.value = category
  }

  return {
    tires,
    loading,
    selectedCategory,
    filteredTires,
    totalCount,
    fetchTires,
    setCategory
  }
})
```

### 在元件中使用

```vue
<script setup lang="ts">
import { onMounted } from "vue"
import { storeToRefs } from "pinia"
import { useTireStore } from "@/stores/tireStore"

const tireStore = useTireStore()

// storeToRefs 保持響應性（只解構 state 和 getters）
const { filteredTires, loading, totalCount } = storeToRefs(tireStore)

// actions 直接解構即可
const { fetchTires, setCategory } = tireStore

onMounted(() => fetchTires())
</script>

<template>
  <div>
    <p>共 {{ totalCount }} 筆</p>
    <button @click="setCategory('summer')">夏季胎</button>
    <button @click="setCategory('all')">全部</button>
    <div v-if="loading">載入中...</div>
    <ul v-else>
      <li v-for="tire in filteredTires" :key="tire.id">
        {{ tire.brand }} {{ tire.model }}
      </li>
    </ul>
  </div>
</template>
```

> **注意**：解構 store 的 state 和 getters 時必須使用 `storeToRefs()`，否則會失去響應性。actions（函式）則可以直接解構。

---

## 7、小結

Vue 3 Composition API 的核心概念整理：

- **響應式**：`ref()` 適用所有場景，`reactive()` 適合表單物件；`computed()` 做衍生計算；`watch()` / `watchEffect()` 監聽變化
- **生命週期**：`onMounted()` 最常用，`onUnmounted()` 負責清理
- **組件通訊**：`defineProps()` 父傳子、`defineEmits()` 子傳父、`provide/inject` 跨層傳遞
- **Composables**：以 `use` 前綴函式封裝可重用邏輯，取代 Mixins
- **Pinia**：扁平化 store，原生 TypeScript 支援，用 `defineStore()` + Composition API 風格定義

延伸閱讀：

- 上一篇 [02 TypeScript 基礎](./02%20TypeScript%20基礎.md)：型別系統基礎知識
- 下一篇 [04 Vue 3 元件開發實戰](./04%20Vue%203%20元件開發實戰.md)：路由、Axios 封裝、表單驗證、表格分頁
- 參考 [01 React 函式元件與 Hooks](./01%20React%20函式元件與%20Hooks.md)：React 生態系的對照

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
