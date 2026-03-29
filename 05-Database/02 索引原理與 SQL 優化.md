# 02 索引原理與 SQL 優化

> **版本**：PostgreSQL 15+ / MySQL 8.0+ (InnoDB)

---

## 1、索引結構

### B+ Tree 原理

B+ Tree 是關聯式資料庫最核心的索引結構，PostgreSQL 和 MySQL InnoDB 預設都使用 B+ Tree。

**B+ Tree 的特點**：
- **所有資料存在葉節點**：內部節點只存 key，不存資料，因此扇出（fan-out）更大
- **葉節點形成有序鏈結串列**：支援高效的範圍查詢（BETWEEN、>、< 等）
- **樹高通常 3~4 層**：每次查詢只需 3~4 次磁碟 I/O，百萬級資料也能毫秒級定位

**為何不用其他結構**：

| 結構 | 缺點 | 適用場景 |
|-----|------|---------|
| B-Tree | 資料分散在所有節點，範圍查詢效率低 | 較少使用 |
| Hash 索引 | 只支援等值查詢，不支援範圍、排序 | 精確匹配（Memory 引擎） |
| 紅黑樹 | 樹高過深，磁碟 I/O 次數多 | 記憶體內資料結構 |

### 聚簇索引 vs 非聚簇索引

| 特性 | 聚簇索引 (Clustered) | 非聚簇索引 (Non-Clustered) |
|-----|---------------------|--------------------------|
| 資料儲存 | 資料行依索引順序物理排列 | 索引與資料分開儲存 |
| 每張表數量 | 只能有一個 | 可以有多個 |
| 查詢效率 | 直接取得資料，無需回表 | 需要回表（除非覆蓋索引） |
| MySQL InnoDB | 主鍵即聚簇索引 | 二級索引，葉節點存主鍵值 |
| PostgreSQL | 無真正聚簇索引（CLUSTER 指令可排序一次但不維持） | 所有索引都指向 heap tuple |

**MySQL InnoDB 回表過程**：
```
二級索引查詢 → 取得主鍵值 → 回到聚簇索引 → 取得完整資料行
```

這就是為什麼 MySQL 中主鍵建議使用自增整數而非 UUID -- 聚簇索引按主鍵排列，UUID 的隨機性會造成頻繁的頁分裂。

---

## 2、索引類型

### PRIMARY INDEX（主鍵索引）

```sql
-- 建表時指定
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
```

### UNIQUE INDEX（唯一索引）

```sql
CREATE UNIQUE INDEX idx_users_email ON users (email);
```

### COMPOSITE INDEX（複合索引）

```sql
-- 最左前綴原則：(a, b, c) 可用於 a、a+b、a+b+c 的查詢
CREATE INDEX idx_orders_status_date ON orders (status, created_at);

-- 有效：
SELECT * FROM orders WHERE status = 'PENDING';                         -- 使用索引
SELECT * FROM orders WHERE status = 'PENDING' AND created_at > '2025-01-01'; -- 使用索引

-- 無效：
SELECT * FROM orders WHERE created_at > '2025-01-01';  -- 跳過 status，無法使用此索引
```

### PARTIAL INDEX / 條件索引（PostgreSQL 特有）

```sql
-- 只對未完成的訂單建索引，大幅減少索引大小
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status IN ('PENDING', 'PROCESSING');

-- 查詢時 WHERE 條件必須匹配索引條件才會使用
SELECT * FROM orders WHERE status = 'PENDING' AND created_at > '2025-01-01';
```

MySQL 不支援條件索引，但可以用類似效果的方式：
```sql
-- MySQL：用虛擬欄位模擬
ALTER TABLE orders
ADD COLUMN is_pending TINYINT GENERATED ALWAYS AS (IF(status IN ('PENDING','PROCESSING'), 1, NULL)) VIRTUAL;
CREATE INDEX idx_orders_pending ON orders (is_pending, created_at);
```

### COVERING INDEX（覆蓋索引）

```sql
-- 索引包含查詢所需的所有欄位，避免回表
CREATE INDEX idx_orders_covering ON orders (status, created_at, total);

-- 此查詢只需讀索引，不需回表
SELECT status, created_at, total FROM orders WHERE status = 'SHIPPED';
```

在 PostgreSQL 中可用 `INCLUDE` 語法：
```sql
CREATE INDEX idx_orders_covering ON orders (status, created_at) INCLUDE (total);
```

### GIN 索引（全文搜索 / JSONB）

```sql
-- PostgreSQL：JSONB 索引
CREATE INDEX idx_products_attrs ON products USING GIN (attrs);

-- PostgreSQL：全文搜索索引
CREATE INDEX idx_articles_fts ON articles USING GIN (to_tsvector('chinese', title || ' ' || content));
```

```sql
-- MySQL：FULLTEXT 索引
CREATE FULLTEXT INDEX idx_articles_fts ON articles (title, content);
SELECT * FROM articles WHERE MATCH(title, content) AGAINST('輪胎' IN NATURAL LANGUAGE MODE);
```

---

## 3、索引失效場景

以下情況會導致索引無法使用，退化為全表掃描：

### 3.1 函數操作

```sql
-- 失效：對索引欄位使用函數
SELECT * FROM users WHERE UPPER(email) = 'ALICE@EXAMPLE.COM';

-- 修復方案 1：建立函數索引（PostgreSQL）
CREATE INDEX idx_users_email_upper ON users (UPPER(email));

-- 修復方案 2：在應用層處理
SELECT * FROM users WHERE email = LOWER('ALICE@EXAMPLE.COM');
```

### 3.2 隱式型別轉換

```sql
-- 失效：phone 是 VARCHAR，但傳入 INTEGER
SELECT * FROM users WHERE phone = 912345678;  -- 隱式轉型，索引失效

-- 修復：確保型別一致
SELECT * FROM users WHERE phone = '912345678';
```

### 3.3 LIKE 前綴萬用字元

```sql
-- 失效：前綴模糊查詢
SELECT * FROM users WHERE name LIKE '%Alice%';

-- 有效：後綴模糊查詢可以用索引
SELECT * FROM users WHERE name LIKE 'Alice%';

-- 替代方案：全文搜索索引
```

### 3.4 OR 條件

```sql
-- 可能失效：OR 連接不同欄位
SELECT * FROM users WHERE email = 'x' OR phone = 'y';

-- 修復：改用 UNION
SELECT * FROM users WHERE email = 'x'
UNION
SELECT * FROM users WHERE phone = 'y';
```

### 3.5 最左前綴原則違反

```sql
-- 複合索引 (a, b, c)
-- 失效的查詢：
SELECT * FROM t WHERE b = 1;          -- 跳過 a
SELECT * FROM t WHERE b = 1 AND c = 2; -- 跳過 a
SELECT * FROM t WHERE c = 2;          -- 只用 c
```

### 3.6 NOT NULL / IS NULL

```sql
-- 在部分情況下，IS NULL 可以使用索引（MySQL 8.0+ 和 PostgreSQL 都支援）
-- 但 NOT IN、<>、!= 通常會導致索引失效
SELECT * FROM users WHERE status <> 'DELETED';  -- 可能全表掃描
```

---

## 4、EXPLAIN 分析

### PostgreSQL EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'PENDING' AND created_at > '2025-01-01'
ORDER BY created_at DESC LIMIT 20;
```

輸出解讀：
```
Limit  (cost=0.43..52.70 rows=20 width=120) (actual time=0.035..0.089 rows=20 loops=1)
  ->  Index Scan Backward using idx_orders_status_date on orders
        (cost=0.43..1250.33 rows=478 width=120)
        (actual time=0.033..0.084 rows=20 loops=1)
        Index Cond: ((status = 'PENDING') AND (created_at > '2025-01-01'))
Planning Time: 0.152 ms
Execution Time: 0.112 ms
```

| 關鍵欄位 | 說明 |
|---------|------|
| `cost` | 啟動成本..總成本（單位：磁碟頁讀取次數） |
| `rows` | 預估/實際回傳行數 |
| `actual time` | 實際執行時間（毫秒） |
| `Index Scan` | 使用索引掃描（好） |
| `Seq Scan` | 全表循序掃描（通常不好，小表例外） |
| `Bitmap Index Scan` | 先從索引取得位圖，再批次讀資料頁 |
| `loops` | 執行次數（在巢狀迴圈 JOIN 中特別重要） |

### MySQL EXPLAIN FORMAT=TREE

```sql
EXPLAIN FORMAT=TREE
SELECT * FROM orders WHERE status = 'PENDING' AND created_at > '2025-01-01'
ORDER BY created_at DESC LIMIT 20;
```

```sql
-- 傳統 EXPLAIN 關鍵欄位
EXPLAIN SELECT * FROM orders WHERE status = 'PENDING';
```

| 欄位 | 含義 | 好的值 |
|-----|------|-------|
| `type` | 存取類型 | system > const > eq_ref > ref > range > index > ALL |
| `key` | 實際使用的索引 | 應有值，NULL 表示全表掃描 |
| `rows` | 預估掃描行數 | 越小越好 |
| `filtered` | 過濾百分比 | 越高越好（100% 最佳） |
| `Extra` | 額外資訊 | `Using index` (覆蓋索引)、`Using filesort` (需要排序，較慢) |

**type 等級說明**：
- `const`：主鍵或唯一索引等值查詢，最多一行
- `eq_ref`：JOIN 時使用主鍵/唯一索引，每次只匹配一行
- `ref`：使用非唯一索引，可能多行
- `range`：索引範圍掃描（BETWEEN、>、< 等）
- `index`：全索引掃描（比 ALL 好，但仍掃描所有索引項）
- `ALL`：全表掃描，最差

---

## 5、SQL 優化實戰

### 5.1 避免 SELECT *

```sql
-- 差：讀取所有欄位，無法利用覆蓋索引
SELECT * FROM orders WHERE status = 'PENDING';

-- 好：只取需要的欄位，可能命中覆蓋索引
SELECT id, status, total, created_at FROM orders WHERE status = 'PENDING';
```

### 5.2 分頁最佳化：Keyset Pagination

```sql
-- 差：OFFSET 越大越慢，DB 需要掃描並丟棄前面所有行
SELECT * FROM orders ORDER BY id DESC LIMIT 20 OFFSET 100000;

-- 好：Keyset Pagination（又稱 Cursor-based Pagination）
-- 利用上一頁最後一筆的 id 作為游標
SELECT * FROM orders
WHERE id < 99000    -- 上一頁最後一筆的 id
ORDER BY id DESC
LIMIT 20;
```

Keyset Pagination 優點：
- 效能恆定，不受頁數影響
- 搭配索引，只需讀取 20 筆資料

限制：
- 不支援「跳到第 N 頁」
- 排序欄位必須有索引且值唯一（或搭配 id 作為 tiebreaker）

### 5.3 JOIN 最佳化

```sql
-- 確保 JOIN 欄位有索引
SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id  -- customer_id 需要索引
WHERE o.status = 'PENDING';

CREATE INDEX idx_orders_customer ON orders (customer_id);
```

**JOIN 順序**：優化器通常會自動選擇最佳順序，但在複雜查詢中可以用 hint 引導：
```sql
-- PostgreSQL：可透過 SET 控制
SET join_collapse_limit = 1;  -- 按照 SQL 中的 JOIN 順序

-- MySQL：STRAIGHT_JOIN 強制順序
SELECT STRAIGHT_JOIN o.*, c.name
FROM orders o JOIN customers c ON c.id = o.customer_id;
```

### 5.4 子查詢 vs JOIN

```sql
-- 差：相關子查詢，每行執行一次
SELECT *, (SELECT name FROM customers c WHERE c.id = o.customer_id) AS customer_name
FROM orders o;

-- 好：改用 JOIN
SELECT o.*, c.name AS customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id;

-- 例外：EXISTS 子查詢在某些場景下比 JOIN 更快
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'PENDING');
```

### 5.5 批次操作

```sql
-- 差：逐筆 INSERT
INSERT INTO logs (message) VALUES ('msg1');
INSERT INTO logs (message) VALUES ('msg2');

-- 好：批次 INSERT
INSERT INTO logs (message) VALUES ('msg1'), ('msg2'), ('msg3');

-- 大量更新：分批處理，避免長時間鎖定
-- PostgreSQL 範例
UPDATE orders SET status = 'ARCHIVED'
WHERE id IN (
    SELECT id FROM orders WHERE status = 'COMPLETED' AND created_at < '2024-01-01'
    LIMIT 1000
);
```

### 5.6 統計查詢最佳化

```sql
-- 差：COUNT(*)  大表上很慢
SELECT COUNT(*) FROM orders;

-- PostgreSQL 近似值（毫秒級）
SELECT reltuples::bigint AS estimate FROM pg_class WHERE relname = 'orders';

-- 精確計數但有條件：善用索引
SELECT COUNT(*) FROM orders WHERE status = 'PENDING';  -- 有索引就快
```

---

## 6、小結

| 要點 | 說明 |
|-----|------|
| 索引結構 | B+ Tree 是核心，葉節點有序鏈結支援範圍查詢 |
| 索引設計 | 遵循最左前綴原則，考慮覆蓋索引減少回表 |
| 索引失效 | 函數、隱式轉型、前綴萬用字元、OR 條件 |
| EXPLAIN | 必學的診斷工具，關注 type、rows、Extra |
| 分頁 | 大量資料用 Keyset Pagination 取代 OFFSET |
| JOIN | 確保 JOIN 欄位有索引，優先用 JOIN 取代相關子查詢 |

### 延伸閱讀

- [01 PostgreSQL 與 MySQL 基礎](./01%20PostgreSQL%20與%20MySQL%20基礎.md) -- 資料型別、DDL/DML、JSON 操作
- [03 交易與鎖機制](./03%20交易與鎖機制.md) -- ACID、隔離級別、MVCC、死鎖排查
- [04 Redis 快取實戰](./04%20Redis%20快取實戰.md) -- Redis 整合 Spring Boot、快取策略、分散式鎖
