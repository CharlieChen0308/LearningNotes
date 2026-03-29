# 05 Playwright 前端自動化測試

> **版本**：Playwright 1.40+ / TypeScript (or Java)

前端自動化測試能夠模擬真實使用者操作瀏覽器的行為（點擊、輸入、導航），確保應用程式在不同瀏覽器上正確運作。Playwright 是 Microsoft 開發的新一代瀏覽器自動化框架，原生支援多瀏覽器、自動等待機制，以及現代 Web 應用所需的各種測試能力。

---

## 1、為什麼用 Playwright

### 與其他框架比較

| 比較項目 | Playwright | Selenium | Cypress |
|---------|-----------|----------|---------|
| 開發商 | Microsoft | 社群 | Cypress.io |
| 支援瀏覽器 | Chromium, Firefox, WebKit | 所有主流瀏覽器 | Chromium 系列為主 |
| 語言支援 | TypeScript, JavaScript, Python, Java, C# | Java, Python, C#, JS 等 | JavaScript / TypeScript |
| 自動等待 | 內建（auto-wait） | 需手動處理 | 內建 |
| 多分頁/多視窗 | 原生支援 | 支援但較繁瑣 | 不支援 |
| iframe 操作 | 原生支援 | 支援 | 有限支援 |
| 網路攔截 | 內建 `route()` API | 需額外工具 | `cy.intercept()` |
| 平行測試 | 內建 worker | 需 Selenium Grid | 付費功能 |
| 執行速度 | 快（直接與瀏覽器通訊） | 中（透過 WebDriver 協議） | 快（在瀏覽器內執行） |
| 測試隔離 | 每個測試獨立 Browser Context | 需手動管理 | 每個測試獨立 |

Playwright 的核心優勢在於：**真正的多瀏覽器支援**（包括 Safari 的 WebKit 引擎）、**自動等待元素可操作**（不需要手動寫 `sleep` 或 `waitFor`），以及**優秀的開發者體驗**（Codegen、Trace Viewer、UI Mode）。

---

## 2、安裝與設定

### 初始化專案

```bash
# 建立新的 Playwright 專案
npm init playwright@latest

# 互動式選項：
# - TypeScript / JavaScript
# - tests 目錄名稱
# - 是否加入 GitHub Actions workflow
# - 是否安裝瀏覽器
```

### playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
    testDir: './tests',
    timeout: 30_000,                        // 每個測試的超時時間
    expect: {
        timeout: 5_000,                     // assertion 的超時時間
    },
    fullyParallel: true,                    // 測試平行執行
    retries: process.env.CI ? 2 : 0,        // CI 環境失敗重試 2 次
    workers: process.env.CI ? 1 : undefined, // CI 環境用 1 個 worker

    reporter: [
        ['html', { open: 'never' }],        // 產生 HTML 報告
        ['list'],                            // 終端機輸出
    ],

    use: {
        baseURL: 'http://localhost:3000',    // 所有 page.goto('/') 的基準 URL
        trace: 'on-first-retry',            // 失敗重試時錄製 trace
        screenshot: 'only-on-failure',       // 失敗時自動截圖
        video: 'retain-on-failure',          // 失敗時保留錄影
    },

    projects: [
        {
            name: 'chromium',
            use: { ...devices['Desktop Chrome'] },
        },
        {
            name: 'firefox',
            use: { ...devices['Desktop Firefox'] },
        },
        {
            name: 'webkit',
            use: { ...devices['Desktop Safari'] },
        },
        // 行動裝置測試
        {
            name: 'mobile-chrome',
            use: { ...devices['Pixel 5'] },
        },
    ],
});
```

---

## 3、基本測試

### test / expect 與基本操作

```typescript
import { test, expect } from '@playwright/test';

test('首頁標題正確', async ({ page }) => {
    await page.goto('/');
    await expect(page).toHaveTitle(/我的應用/);
});

test('登入流程', async ({ page }) => {
    // 導航
    await page.goto('/login');

    // 填入表單
    await page.fill('input[name="username"]', 'admin');
    await page.fill('input[name="password"]', 'password123');

    // 點擊按鈕
    await page.click('button[type="submit"]');

    // 斷言：等待導航到 dashboard
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('h1')).toHaveText('歡迎回來');
});

test('搜尋商品', async ({ page }) => {
    await page.goto('/products');

    // 輸入搜尋關鍵字
    await page.fill('[placeholder="搜尋商品..."]', '輪胎');
    await page.press('[placeholder="搜尋商品..."]', 'Enter');

    // 斷言：搜尋結果至少有一筆
    const results = page.locator('.product-card');
    await expect(results).not.toHaveCount(0);

    // 斷言：每個結果都包含關鍵字
    const firstResult = results.first();
    await expect(firstResult).toContainText('輪胎');
});
```

### 常用斷言

```typescript
// 頁面層級
await expect(page).toHaveTitle('首頁');
await expect(page).toHaveURL('/dashboard');

// 元素層級
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toBeEnabled();
await expect(locator).toBeDisabled();
await expect(locator).toHaveText('確切文字');
await expect(locator).toContainText('部分文字');
await expect(locator).toHaveValue('input 的值');
await expect(locator).toHaveCount(3);
await expect(locator).toHaveAttribute('href', '/about');
await expect(locator).toHaveClass(/active/);
```

---

## 4、選擇器策略

Playwright 推薦使用語意化選擇器（Semantic Locators），優先使用使用者可見的屬性，而非 CSS class 或 XPath。

### 優先級建議

| 優先順序 | 選擇器 | 範例 | 說明 |
|---------|--------|------|------|
| 1 | `getByRole` | `page.getByRole('button', { name: '送出' })` | 基於 ARIA 角色，最貼近使用者感知 |
| 2 | `getByText` | `page.getByText('歡迎回來')` | 基於可見文字 |
| 3 | `getByLabel` | `page.getByLabel('電子郵件')` | 基於 `<label>` 關聯 |
| 4 | `getByPlaceholder` | `page.getByPlaceholder('請輸入帳號')` | 基於 placeholder 文字 |
| 5 | `getByTestId` | `page.getByTestId('submit-btn')` | 基於 `data-testid` 屬性 |
| 6 | CSS / XPath | `page.locator('.btn-primary')` | 最後手段，容易因樣式變更而失效 |

```typescript
// 推薦：語意化選擇器
await page.getByRole('button', { name: '新增商品' }).click();
await page.getByLabel('商品名稱').fill('米其林 PS5');
await page.getByRole('combobox', { name: '品牌' }).selectOption('Michelin');

// 次選：data-testid（當 UI 文字可能變動時）
await page.getByTestId('product-price').fill('3500');

// 避免：脆弱的 CSS 選擇器
await page.locator('.form-group:nth-child(3) > input').fill('3500');  // 不推薦
```

---

## 5、Page Object Model

當測試數量增加後，相同的頁面操作會散落在多個測試檔案中。Page Object Model（POM）將頁面操作封裝成 class，提升可維護性與可讀性。

### LoginPage 範例

```typescript
// pages/LoginPage.ts
import { type Page, type Locator } from '@playwright/test';

export class LoginPage {
    readonly page: Page;
    readonly usernameInput: Locator;
    readonly passwordInput: Locator;
    readonly submitButton: Locator;
    readonly errorMessage: Locator;

    constructor(page: Page) {
        this.page = page;
        this.usernameInput = page.getByLabel('帳號');
        this.passwordInput = page.getByLabel('密碼');
        this.submitButton = page.getByRole('button', { name: '登入' });
        this.errorMessage = page.getByRole('alert');
    }

    async goto() {
        await this.page.goto('/login');
    }

    async login(username: string, password: string) {
        await this.usernameInput.fill(username);
        await this.passwordInput.fill(password);
        await this.submitButton.click();
    }
}
```

### 在測試中使用

```typescript
// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test.describe('登入功能', () => {
    let loginPage: LoginPage;

    test.beforeEach(async ({ page }) => {
        loginPage = new LoginPage(page);
        await loginPage.goto();
    });

    test('正確帳密可以登入', async ({ page }) => {
        await loginPage.login('admin', 'password123');
        await expect(page).toHaveURL('/dashboard');
    });

    test('錯誤密碼顯示提示', async () => {
        await loginPage.login('admin', 'wrong');
        await expect(loginPage.errorMessage).toHaveText('帳號或密碼錯誤');
    });

    test('空白帳號無法送出', async () => {
        await loginPage.login('', 'password123');
        await expect(loginPage.submitButton).toBeVisible();  // 停留在登入頁
    });
});
```

---

## 6、截圖與錄影

### 失敗自動截圖

在 `playwright.config.ts` 中設定（前面已展示）：

```typescript
use: {
    screenshot: 'only-on-failure',  // 'off' | 'on' | 'only-on-failure'
}
```

也可以在測試中手動截圖：

```typescript
test('手動截圖範例', async ({ page }) => {
    await page.goto('/dashboard');

    // 整頁截圖
    await page.screenshot({ path: 'screenshots/dashboard.png', fullPage: true });

    // 特定元素截圖
    await page.locator('.chart-container').screenshot({ path: 'screenshots/chart.png' });
});
```

### 錄影設定

```typescript
// playwright.config.ts
use: {
    video: 'retain-on-failure',  // 'off' | 'on' | 'retain-on-failure' | 'on-first-retry'
}
```

影片檔案儲存在 `test-results/` 目錄下，方便除錯重現失敗場景。

### Trace Viewer

Trace 是 Playwright 最強大的除錯工具，記錄了測試執行的每一步操作、DOM 快照、網路請求與 console 輸出：

```typescript
use: {
    trace: 'on-first-retry',  // 失敗重試時錄製 trace
}
```

查看 trace：

```bash
npx playwright show-trace test-results/login-test/trace.zip
```

Trace Viewer 提供時間軸檢視，可以逐步回放每個操作當下的頁面狀態，是排查 flaky test 的利器。

---

## 7、CI 整合

### GitHub Actions Workflow

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Run Playwright tests
        run: npx playwright test
        env:
          BASE_URL: http://localhost:3000

      - name: Upload test report
        uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30

      - name: Upload test results (traces, screenshots)
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: test-results
          path: test-results/
          retention-days: 7
```

**重點說明**：

- `npx playwright install --with-deps`：安裝瀏覽器及其系統相依套件（Linux 上需要額外的共享函式庫）
- `if: ${{ !cancelled() }}`：即使測試失敗也上傳報告（但手動取消時不上傳）
- `if: failure()`：只在失敗時上傳 trace 和截圖，節省儲存空間
- CI 環境預設以 headless 模式運行，不需要額外設定

---

## 8、Java 版 Playwright

Playwright 也提供 Java API，適合後端團隊或已有 Java 測試基礎建設的專案。

### Maven 依賴

```xml
<dependency>
    <groupId>com.microsoft.playwright</groupId>
    <artifactId>playwright</artifactId>
    <version>1.41.0</version>
    <scope>test</scope>
</dependency>
```

### 基本範例

```java
import com.microsoft.playwright.*;
import org.junit.jupiter.api.*;
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

class LoginTest {
    static Playwright playwright;
    static Browser browser;
    BrowserContext context;
    Page page;

    @BeforeAll
    static void setup() {
        playwright = Playwright.create();
        browser = playwright.chromium().launch(
            new BrowserType.LaunchOptions().setHeadless(true)
        );
    }

    @BeforeEach
    void createContext() {
        context = browser.newContext();
        page = context.newPage();
    }

    @AfterEach
    void closeContext() {
        context.close();
    }

    @AfterAll
    static void teardown() {
        browser.close();
        playwright.close();
    }

    @Test
    void shouldLoginSuccessfully() {
        page.navigate("http://localhost:3000/login");
        page.getByLabel("帳號").fill("admin");
        page.getByLabel("密碼").fill("password123");
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("登入")).click();
        assertThat(page).hasURL(Pattern.compile(".*/dashboard"));
    }
}
```

Java 版的 API 設計與 TypeScript 版幾乎一致，選擇器、斷言、等待機制都相同，主要差異在語法風格。如果團隊已使用 JUnit 5 + Spring Boot 測試體系，Java 版可以無縫整合。

---

## 9、小結

本篇涵蓋了 Playwright 前端自動化測試的核心知識：

- **基本測試**：test/expect、頁面操作、常用斷言
- **選擇器策略**：優先使用 getByRole、getByLabel 等語意化選擇器
- **Page Object Model**：封裝頁面操作，提升測試可維護性
- **截圖與錄影**：失敗自動截圖、Trace Viewer 除錯
- **CI 整合**：GitHub Actions 自動化執行、報告上傳
- **Java 版**：與 JUnit 5 整合，API 設計與 TypeScript 版一致

延伸閱讀：

- **JUnit 5 測試實戰**：`08-DevOps/04 JUnit 5 測試實戰.md`
- **CI/CD 流程（GitHub Actions）**：`08-DevOps/06 CI／CD 流程（GitHub Actions）.md`

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
