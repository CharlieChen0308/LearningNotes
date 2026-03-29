# 06 前端建置工具（Vite）

> **版本**：Vite 5.x / Vue 3 / React 19

Vite 是由 Vue.js 作者尤雨溪開發的新一代前端建置工具，利用瀏覽器原生的 ES Module 支援實現極速開發體驗。本篇將從 Vite 的核心優勢出發，介紹專案建立、常用配置、環境變數、代理設定、打包最佳化，以及與 Spring Boot 後端的整合部署。

---

## 1、為什麼用 Vite

### 傳統 Webpack 的痛點

Webpack 在開發模式下需要將整個專案打包成 bundle 後才能啟動開發伺服器。隨著專案規模增長，冷啟動時間可能達到數十秒甚至數分鐘，每次修改也需要等待數秒才能看到效果。

### Vite 的核心理念

Vite 採用截然不同的策略：

- **開發模式**：利用瀏覽器原生的 ES Module（`import` / `export`），不需要打包。Vite 啟動開發伺服器時只需要處理當前頁面實際引用的模組，未使用的模組根本不會被載入。
- **生產模式**：使用 Rollup 進行打包，產出高度最佳化的靜態資源。

### Vite vs Webpack 比較

| 比較項目 | Vite | Webpack |
|---------|------|---------|
| 開發伺服器啟動 | 毫秒級（ESM 按需載入） | 秒~分鐘級（全量打包） |
| HMR 熱更新 | 幾乎即時，不受專案大小影響 | 隨專案增大而變慢 |
| 設定複雜度 | 開箱即用，設定簡潔 | 需要大量 loader / plugin 設定 |
| 生態成熟度 | 快速成長中，主流框架皆已支援 | 極為成熟，Plugin 生態龐大 |
| 生產打包工具 | Rollup（Tree-shaking 優秀） | 自身的打包引擎 |
| 學習曲線 | 低，設定直觀 | 高，概念多且設定繁瑣 |
| TypeScript 支援 | 原生支援，無需額外設定 | 需要 ts-loader 或 babel 設定 |

### 為什麼 HMR 如此快？

Webpack 的 HMR 需要重新構建受影響的模組鏈並傳送完整的更新 bundle。Vite 的 HMR 則精確地只讓瀏覽器重新請求被修改的那一個模組，再透過 ESM 的 `import` 自動解析相依關係。因此無論專案有 100 個還是 10,000 個模組，HMR 的速度幾乎不變。

---

## 2、快速開始

### 建立專案

```bash
# 互動式選擇框架與模板
npm create vite@latest my-project

# 直接指定模板
npm create vite@latest my-vue-app -- --template vue-ts
npm create vite@latest my-react-app -- --template react-ts
```

### 支援的框架模板

| 模板名稱 | 說明 |
|---------|------|
| `vanilla` / `vanilla-ts` | 純 JavaScript / TypeScript |
| `vue` / `vue-ts` | Vue 3 |
| `react` / `react-ts` | React 19 |
| `preact` / `preact-ts` | Preact |
| `svelte` / `svelte-ts` | Svelte |
| `lit` / `lit-ts` | Lit |
| `solid` / `solid-ts` | SolidJS |
| `qwik` / `qwik-ts` | Qwik |

### 專案結構（以 Vue 為例）

```
my-vue-app/
  index.html          # 入口 HTML（Vite 以此為進入點）
  vite.config.ts      # Vite 設定檔
  tsconfig.json       # TypeScript 設定
  package.json
  public/             # 靜態資源（不經過打包處理）
  src/
    main.ts           # 應用程式進入點
    App.vue
    components/
    assets/           # 靜態資源（會經過打包處理）
```

與 Webpack 不同，Vite 的入口是 `index.html` 而非 JavaScript 檔案。`index.html` 中直接使用 `<script type="module" src="/src/main.ts">` 引入模組，瀏覽器原生解析 ESM。

### 啟動開發伺服器

```bash
cd my-vue-app
npm install
npm run dev
```

---

## 3、vite.config.ts 常用配置

```ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'

export default defineConfig({
  plugins: [vue()],

  // ===== 路徑別名 =====
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '@components': resolve(__dirname, 'src/components'),
      '@utils': resolve(__dirname, 'src/utils'),
    },
  },

  // ===== 開發伺服器 =====
  server: {
    port: 3000,          // 開發伺服器埠號
    open: true,          // 啟動後自動開啟瀏覽器
    host: true,          // 允許外部裝置存取（0.0.0.0）

    // 後端 API 代理（詳見第 5 節）
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },

  // ===== 打包設定 =====
  build: {
    outDir: 'dist',           // 輸出目錄
    sourcemap: false,         // 生產環境不產生 sourcemap
    chunkSizeWarningLimit: 500, // chunk 大小警告門檻（KB）

    rollupOptions: {
      output: {
        // 手動分割 chunk（詳見第 6 節）
        manualChunks: {
          vendor: ['vue', 'vue-router', 'pinia'],
          ui: ['element-plus'],
        },
      },
    },
  },
})
```

### 路徑別名搭配 TypeScript

設定 `@` 別名後，還需要在 `tsconfig.json` 中同步設定，讓 TypeScript 編譯器能正確解析路徑：

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

設定完成後即可使用別名引入模組：

```ts
import UserList from '@/components/UserList.vue'
import { formatDate } from '@utils/date'
```

---

## 4、環境變數

### 環境檔案

Vite 支援多層環境變數檔案，依優先級由高到低：

| 檔案 | 載入時機 |
|------|---------|
| `.env.production.local` | `build` 時載入，不進版控 |
| `.env.production` | `build` 時載入 |
| `.env.development.local` | `dev` 時載入，不進版控 |
| `.env.development` | `dev` 時載入 |
| `.env.local` | 所有模式載入，不進版控 |
| `.env` | 所有模式載入 |

### VITE_ 前綴規則

基於安全考量，只有以 `VITE_` 開頭的變數才會暴露給前端程式碼。這是為了防止伺服器端的敏感資訊（如資料庫密碼）意外洩漏到瀏覽器端。

```bash
# .env.development
VITE_API_BASE_URL=http://localhost:8080/api
VITE_APP_TITLE=TireMaster Dev

# 這個變數不會暴露給前端（沒有 VITE_ 前綴）
DB_PASSWORD=secret123
```

```bash
# .env.production
VITE_API_BASE_URL=https://api.tiremaster.com/api
VITE_APP_TITLE=TireMaster
```

### 在程式碼中使用

透過 `import.meta.env` 存取環境變數：

```ts
// 讀取自訂變數
const apiUrl = import.meta.env.VITE_API_BASE_URL

// Vite 內建變數
console.log(import.meta.env.MODE)       // 'development' 或 'production'
console.log(import.meta.env.DEV)        // true（開發模式）
console.log(import.meta.env.PROD)       // true（生產模式）
console.log(import.meta.env.BASE_URL)   // 由 base 設定決定
```

### TypeScript 型別提示

為了讓 `import.meta.env` 有正確的型別提示，建立 `src/env.d.ts`：

```ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string
  readonly VITE_APP_TITLE: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

---

## 5、代理設定（前端開發與 Spring Boot 後端整合）

前後端分離開發時，前端跑在 `localhost:3000`，後端跑在 `localhost:8080`，直接呼叫 API 會遇到跨域（CORS）問題。Vite 的 `server.proxy` 可以在開發環境中透過反向代理解決這個問題。

### 基本代理

```ts
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
})
```

設定後，前端發送 `GET /api/users` 的請求會被 Vite 開發伺服器代理到 `http://localhost:8080/api/users`。

### 路徑重寫

如果後端 API 路徑不帶 `/api` 前綴，可以使用 `rewrite` 去除：

```ts
proxy: {
  '/api': {
    target: 'http://localhost:8080',
    changeOrigin: true,
    rewrite: (path) => path.replace(/^\/api/, ''),
  },
}
```

這樣前端 `GET /api/users` 會被代理到 `http://localhost:8080/users`。

### 多後端代理

TireMaster 這類微服務架構，不同 API 路徑對應不同的後端服務：

```ts
proxy: {
  '/api/store': {
    target: 'http://localhost:8091',
    changeOrigin: true,
  },
  '/api/employee': {
    target: 'http://localhost:8092',
    changeOrigin: true,
  },
  '/api/account': {
    target: 'http://localhost:8093',
    changeOrigin: true,
  },
  '/api/auth': {
    target: 'http://localhost:8099',
    changeOrigin: true,
  },
}
```

### 注意事項

- `changeOrigin: true` 會將請求的 `Host` 標頭改為目標伺服器的位址，解決部分後端框架的 Host 檢查問題。
- 代理只在開發環境生效。生產環境需要透過 nginx 或其他方式處理 API 路由。
- 如果後端使用 WebSocket，需要額外設定 `ws: true`。

---

## 6、打包最佳化

### Code Splitting（程式碼分割）

利用動態 `import()` 實現路由級別的程式碼分割，讓使用者只下載當前頁面所需的程式碼：

```ts
// Vue Router 搭配動態載入
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: () => import('@/views/Home.vue'),
    },
    {
      path: '/orders',
      component: () => import('@/views/Orders.vue'),
    },
    {
      path: '/customers',
      component: () => import('@/views/Customers.vue'),
    },
  ],
})
```

每個 `import()` 會產生一個獨立的 chunk 檔案，只在使用者導航到該路由時才載入。

### 手動分割 Chunk（manualChunks）

將第三方套件從主 bundle 中分離，利用瀏覽器快取機制。業務程式碼頻繁更新，但第三方套件很少變動，分離後使用者只需重新下載業務程式碼：

```ts
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        // 核心框架（幾乎不會更新）
        'vue-vendor': ['vue', 'vue-router', 'pinia'],
        // UI 框架
        'ui-vendor': ['element-plus'],
        // 工具類函式庫
        'utils-vendor': ['axios', 'dayjs', 'lodash-es'],
      },
    },
  },
}
```

### CSS Code Splitting

Vite 預設會自動進行 CSS 分割。非同步載入的元件（如動態 `import()` 的路由元件），其 CSS 會被提取為獨立檔案，與 JavaScript chunk 一同按需載入。

如果需要將所有 CSS 合併為單一檔案：

```ts
build: {
  cssCodeSplit: false, // 關閉 CSS 分割
}
```

### 靜態資源處理

Vite 對靜態資源有兩種處理方式：

- **`src/assets/`**：會經過打包處理，小於 4KB 的檔案自動轉為 Base64 內聯，超過的則加上 hash 指紋方便快取管理。
- **`public/`**：直接複製到輸出目錄，不經過任何處理。適合放置 `favicon.ico`、`robots.txt` 等不需要打包的檔案。

```ts
// 調整內聯門檻值（預設 4KB）
build: {
  assetsInlineLimit: 8192, // 8KB 以下的資源轉為 Base64
}
```

### Gzip / Brotli 壓縮

使用 `vite-plugin-compression` 在打包時預先產生壓縮檔案，搭配 nginx 直接回傳壓縮版本，減少傳輸量：

```bash
npm install -D vite-plugin-compression
```

```ts
import viteCompression from 'vite-plugin-compression'

export default defineConfig({
  plugins: [
    vue(),
    // Gzip 壓縮
    viteCompression({
      algorithm: 'gzip',
      threshold: 10240,    // 大於 10KB 的檔案才壓縮
      ext: '.gz',
    }),
    // Brotli 壓縮（壓縮率更高）
    viteCompression({
      algorithm: 'brotliCompress',
      threshold: 10240,
      ext: '.br',
    }),
  ],
})
```

### 打包分析

使用 `rollup-plugin-visualizer` 視覺化分析各 chunk 的大小，找出體積過大的模組：

```bash
npm install -D rollup-plugin-visualizer
```

```ts
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  plugins: [
    vue(),
    visualizer({
      open: true,           // 打包後自動開啟分析頁面
      gzipSize: true,       // 顯示 gzip 後的大小
      filename: 'stats.html',
    }),
  ],
})
```

---

## 7、與 Spring Boot 整合部署

### 方案一：打包後放入 Spring Boot static/

將 Vite 打包產出直接放到 Spring Boot 的靜態資源目錄，由 Spring Boot 同時提供前後端：

```bash
# 前端打包
cd my-vue-app
npm run build

# 複製到 Spring Boot 靜態資源目錄
cp -r dist/* ../my-spring-app/src/main/resources/static/
```

Spring Boot 設定（`application.yml`）：

```yaml
spring:
  web:
    resources:
      static-locations: classpath:/static/
  mvc:
    # SPA 路由支援：所有未匹配的路徑都導向 index.html
    throw-exception-if-no-handler-found: true
```

需要額外處理 SPA 路由，讓前端路由（如 `/orders`、`/customers`）不被 Spring Boot 攔截為 404：

```java
@Controller
public class SpaController {

    @GetMapping(value = {"/", "/{path:[^\\.]*}", "/{path1}/{path2:[^\\.]*}"})
    public String forward() {
        return "forward:/index.html";
    }
}
```

**適用場景**：小型專案、單體架構、不想額外維護 nginx。

### 方案二：nginx 反向代理（推薦）

前後端各自獨立部署，由 nginx 統一接收請求並分發：

```nginx
server {
    listen 80;
    server_name tiremaster.example.com;

    # 前端靜態資源
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;  # SPA 路由支援
    }

    # 靜態資源快取
    location /assets/ {
        root /usr/share/nginx/html;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # 後端 API 代理
    location /api/ {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 開啟 gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1024;

    # 預壓縮檔案優先（搭配 vite-plugin-compression）
    gzip_static on;
}
```

**適用場景**：前後端獨立部署、微服務架構、需要負載平衡、需要 HTTPS 終端。

### 兩種方案比較

| 項目 | 放入 static/ | nginx 反向代理 |
|------|-------------|---------------|
| 部署複雜度 | 低（單一服務） | 中（需維護 nginx） |
| 前後端耦合 | 高（打包在一起） | 低（各自獨立） |
| 靜態資源快取 | 由 Spring Boot 處理 | 由 nginx 處理（效能更好） |
| 水平擴展 | 前後端一起擴展 | 前後端獨立擴展 |
| 適合規模 | 小型、單體專案 | 中大型、微服務專案 |

---

## 8、小結

本篇涵蓋了 Vite 前端建置工具的核心知識：

- **核心優勢**：ESM-based 開發伺服器帶來毫秒級啟動與即時 HMR，開發體驗遠超 Webpack
- **設定簡潔**：路徑別名、代理、打包最佳化都能在 `vite.config.ts` 中清晰配置
- **環境變數**：`VITE_` 前綴機制確保敏感資訊不會洩漏到前端
- **打包最佳化**：Code Splitting、manualChunks、gzip 壓縮，有效減少生產環境的載入時間
- **後端整合**：根據專案規模選擇 Spring Boot static 或 nginx 反向代理方案

延伸學習資源：

- [Vite 官方文件](https://vite.dev/)
- [Rollup 官方文件](https://rollupjs.org/)
- [Awesome Vite](https://github.com/vitejs/awesome-vite) -- 社群 Plugin 與模板合集

> **延伸閱讀**：
> - [03 Vue 3 Composition API](03%20Vue%203%20Composition%20API.md) — 搭配 Vite 開發 Vue 3 專案的核心語法
> - [01 React 函式元件與 Hooks](01%20React%20函式元件與%20Hooks.md) — 搭配 Vite 開發 React 專案的基礎知識
