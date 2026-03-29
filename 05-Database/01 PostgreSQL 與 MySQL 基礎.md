# 01 PostgreSQL 與 MySQL 基礎

> **版本**：PostgreSQL 15+ / MySQL 8.0+

---

## 1、PostgreSQL vs MySQL 比較表

| 比較項目 | PostgreSQL | MySQL (InnoDB) |
|---------|-----------|----------------|
| 授權 | PostgreSQL License (MIT-like) | GPL v2 (商業版需付費) |
| MVCC 實現 | Tuple versioning (xmin/xmax) | Undo log + Read View |
| JSON 支援 | JSONB（二進位、可索引） | JSON（文字儲存，8.0 起支援索引） |
| 全文搜索 | 內建 tsvector/tsquery + GIN 索引 | 內建 FULLTEXT 索引（InnoDB 支援） |
| 分區表 | 宣告式分區 (RANGE/LIST/HASH) | 原生分區 (RANGE/LIST/HASH/KEY) |
| CTE 支援 | 完整支援（含遞迴、可寫 CTE） | 8.0 起支援遞迴 CTE |
| Window Function | 完整支援 | 8.0 起支援 |
| 預設 Port | 5432 | 3306 |
| 預設隔離級別 | READ COMMITTED | REPEATABLE READ |
| 擴充能力 | Extension 機制 (PostGIS 等) | Plugin 架構 |
| 適用場景 | 複雜查詢、地理資訊、OLAP 混合 | 高併發讀取、Web 應用、OLTP |

**選擇建議**：需要複雜 JSON 操作、地理資訊或進階 SQL 功能選 PostgreSQL；追求高併發讀取效能、生態成熟度選 MySQL。兩者在現代版本中功能差距已大幅縮小。

---

## 2、資料型別比較

### 整數型別

| 型別 | PostgreSQL | MySQL | 範圍 |
|-----|-----------|-------|------|
| 小整數 | `SMALLINT` | `SMALLINT` | -32,768 ~ 32,767 |
| 整數 | `INTEGER` | `INT` | -2^31 ~ 2^31-1 |
| 大整數 | `BIGINT` | `BIGINT` | -2^63 ~ 2^63-1 |

### 文字型別

| 型別 | PostgreSQL | MySQL | 說明 |
|-----|-----------|-------|------|
| 定長 | `CHAR(n)` | `CHAR(n)` | 固定長度，空白填充 |
| 變長 | `VARCHAR(n)` | `VARCHAR(n)` | 最常用，MySQL 最大 65535 bytes |
| 無限長 | `TEXT` | `TEXT` | PostgreSQL 的 TEXT 與 VARCHAR 效能相同 |

### 時間型別

```sql
-- PostgreSQL：TIMESTAMPTZ 會自動轉換時區
CREATE TABLE events (
    created_at  TIMESTAMPTZ DEFAULT NOW(),   -- 推薦：帶時區
    local_time  TIMESTAMP                     -- 不帶時區
);

-- MySQL：TIMESTAMP 自動轉 UTC 儲存，DATETIME 原樣儲存
CREATE TABLE events (
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  -- 自動 UTC 轉換
    local_time  DATETIME                               -- 原樣儲存
);
```

### JSON 型別

| 特性 | PostgreSQL JSONB | MySQL JSON |
|-----|-----------------|------------|
| 儲存格式 | 二進位（解析後儲存） | 二進位（MySQL 8.0） |
| 可建索引 | GIN 索引 | 虛擬欄位 + B-Tree |
| 運算子 | `->`, `->>`, `@>`, `?` | `->`, `->>` (alias) |

### UUID

```sql
-- PostgreSQL：內建 UUID 型別
CREATE TABLE users (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY
);

-- MySQL：用 CHAR(36) 或 BINARY(16)
CREATE TABLE users (
    id CHAR(36) DEFAULT (UUID()) PRIMARY KEY
);
```

### Boolean

```sql
-- PostgreSQL：原生 BOOLEAN
ALTER TABLE users ADD COLUMN is_active BOOLEAN DEFAULT TRUE;

-- MySQL：用 TINYINT(1) 模擬，TRUE=1, FALSE=0
ALTER TABLE users ADD COLUMN is_active TINYINT(1) DEFAULT 1;
```

### ENUM

```sql
-- PostgreSQL：自訂型別
CREATE TYPE order_status AS ENUM ('PENDING', 'CONFIRMED', 'SHIPPED');
CREATE TABLE orders (status order_status NOT NULL);

-- MySQL：欄位層級定義
CREATE TABLE orders (
    status ENUM('PENDING', 'CONFIRMED', 'SHIPPED') NOT NULL
);
```

### 自增主鍵

```sql
-- PostgreSQL：推薦 IDENTITY（SQL 標準）
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
-- 也支援 SERIAL（舊寫法，底層是 SEQUENCE）
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY
);

-- MySQL：AUTO_INCREMENT
CREATE TABLE products (
    id BIGINT AUTO_INCREMENT PRIMARY KEY
);
```

---

## 3、DDL 基礎

### CREATE TABLE 完整範例

```sql
-- PostgreSQL
CREATE TABLE customers (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    name        VARCHAR(100) NOT NULL,
    phone       VARCHAR(20),
    tier        VARCHAR(20) DEFAULT 'BASIC' CHECK (tier IN ('BASIC', 'VIP', 'PREMIUM')),
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
    total       NUMERIC(12, 2) NOT NULL CHECK (total >= 0),
    status      VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

```sql
-- MySQL
CREATE TABLE customers (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    name        VARCHAR(100) NOT NULL,
    phone       VARCHAR(20),
    tier        VARCHAR(20) DEFAULT 'BASIC' CHECK (tier IN ('BASIC', 'VIP', 'PREMIUM')),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE orders (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    total       DECIMAL(12, 2) NOT NULL CHECK (total >= 0),
    status      VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT
) ENGINE=InnoDB;
```

### 約束條件摘要

| 約束 | 說明 | 注意事項 |
|-----|------|---------|
| `PRIMARY KEY` | 主鍵，唯一且非空 | 每張表只能有一個 |
| `FOREIGN KEY` | 外鍵，參照完整性 | ON DELETE: CASCADE / RESTRICT / SET NULL |
| `UNIQUE` | 唯一值約束 | 允許多個 NULL（PostgreSQL）；MySQL 中多個 NULL 也允許 |
| `CHECK` | 條件約束 | MySQL 8.0.16 起才真正執行 CHECK 約束 |
| `DEFAULT` | 預設值 | 可用函數，如 NOW()、gen_random_uuid() |
| `NOT NULL` | 非空約束 | 最基礎的資料完整性保證 |

---

## 4、DML 操作

### INSERT

```sql
-- 單筆插入
INSERT INTO customers (email, name, phone)
VALUES ('alice@example.com', 'Alice', '0912345678');

-- 批次插入
INSERT INTO customers (email, name) VALUES
    ('bob@example.com', 'Bob'),
    ('carol@example.com', 'Carol');
```

### SELECT

```sql
-- 基本查詢 + 排序 + 分頁
SELECT id, email, name, created_at
FROM customers
WHERE tier = 'VIP'
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

### UPDATE

```sql
UPDATE customers
SET tier = 'VIP', updated_at = NOW()
WHERE id = 1;
```

### DELETE

```sql
DELETE FROM orders WHERE status = 'CANCELLED' AND created_at < NOW() - INTERVAL '90 days';
```

### UPSERT（衝突處理）

```sql
-- PostgreSQL：ON CONFLICT
INSERT INTO customers (email, name, phone)
VALUES ('alice@example.com', 'Alice Chen', '0987654321')
ON CONFLICT (email) DO UPDATE SET
    name = EXCLUDED.name,
    phone = EXCLUDED.phone,
    updated_at = NOW();

-- MySQL：ON DUPLICATE KEY UPDATE
INSERT INTO customers (email, name, phone)
VALUES ('alice@example.com', 'Alice Chen', '0987654321')
ON DUPLICATE KEY UPDATE
    name = VALUES(name),
    phone = VALUES(phone),
    updated_at = CURRENT_TIMESTAMP;
```

> **MySQL 8.0.20+ 注意**：`VALUES(col)` 語法已被標記為棄用，建議改用別名語法：`INSERT INTO ... AS new_row ON DUPLICATE KEY UPDATE name = new_row.name`。

---

## 5、JSON 操作

### PostgreSQL JSONB

```sql
-- 建表
CREATE TABLE products (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name  VARCHAR(200) NOT NULL,
    attrs JSONB DEFAULT '{}'::jsonb
);

-- 插入
INSERT INTO products (name, attrs) VALUES
('輪胎 A', '{"brand": "Michelin", "size": "205/55R16", "specs": {"load": 91, "speed": "V"}}');

-- 取值：-> 回傳 JSON，->> 回傳 TEXT
SELECT name,
       attrs -> 'brand' AS brand_json,          -- "Michelin"（JSON 格式）
       attrs ->> 'brand' AS brand_text,          -- Michelin（純文字）
       attrs -> 'specs' ->> 'load' AS load_index -- 91（純文字）
FROM products;

-- 包含查詢：@> 運算子（可用 GIN 索引加速）
SELECT * FROM products WHERE attrs @> '{"brand": "Michelin"}';

-- JSON Path 查詢（PostgreSQL 12+）
SELECT * FROM products
WHERE jsonb_path_exists(attrs, '$.specs ? (@.load > 90)');

-- 更新 JSONB 欄位
UPDATE products
SET attrs = jsonb_set(attrs, '{specs, speed}', '"W"')
WHERE id = 1;

-- 建立 GIN 索引
CREATE INDEX idx_products_attrs ON products USING GIN (attrs);
```

### MySQL JSON

```sql
-- 建表
CREATE TABLE products (
    id    BIGINT AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(200) NOT NULL,
    attrs JSON DEFAULT (JSON_OBJECT())
);

-- 取值：JSON_EXTRACT 或 -> 語法糖
SELECT name,
       JSON_EXTRACT(attrs, '$.brand') AS brand,
       attrs -> '$.specs.load' AS load_index,
       JSON_UNQUOTE(attrs -> '$.brand') AS brand_text  -- 去掉引號
FROM products;

-- 更新
UPDATE products
SET attrs = JSON_SET(attrs, '$.specs.speed', 'W')
WHERE id = 1;

-- 虛擬欄位 + 索引（MySQL 建立 JSON 索引的方式）
ALTER TABLE products
ADD COLUMN brand VARCHAR(100) GENERATED ALWAYS AS (JSON_UNQUOTE(attrs -> '$.brand')) STORED,
ADD INDEX idx_brand (brand);
```

---

## 6、時區處理

### 核心原則：永遠存 UTC

```
應用程式 (Local TZ) → 轉 UTC → 存入 DB → 讀取 UTC → 轉 Local TZ → 顯示
```

### PostgreSQL 時區處理

```sql
-- 查看目前時區設定
SHOW timezone;  -- 通常為 UTC

-- TIMESTAMPTZ：存入時自動轉 UTC，讀取時依 session timezone 轉換
SET timezone = 'Asia/Taipei';
SELECT NOW();  -- 顯示 +08:00

-- 明確轉換
SELECT created_at AT TIME ZONE 'Asia/Taipei' AS local_time
FROM orders;

-- 建議：application.yml 設定連線時區
-- spring.datasource.url=jdbc:postgresql://host/db?currentSchema=public&TimeZone=UTC
```

### MySQL 時區處理

```sql
-- 查看時區
SELECT @@global.time_zone, @@session.time_zone;

-- TIMESTAMP 欄位：自動存 UTC，讀取時依 session time_zone 轉換
-- DATETIME 欄位：原樣存取，不做轉換
SET time_zone = '+08:00';

-- 建議：JDBC 連線字串指定時區
-- jdbc:mysql://host/db?serverTimezone=UTC&useSSL=false
```

### Java / Spring Boot 配合

```java
// Entity 中使用 Instant（UTC）或 OffsetDateTime
@Column(name = "created_at", nullable = false, updatable = false)
private Instant createdAt;

// 前端顯示時才轉為當地時區
// Jackson 設定
// spring.jackson.time-zone=Asia/Taipei
```

**關鍵提醒**：
- 永遠用 `TIMESTAMPTZ`（PostgreSQL）或 `TIMESTAMP`（MySQL）儲存時間
- Java 層使用 `Instant` 或 `OffsetDateTime`，避免使用 `LocalDateTime` 儲存帶時區的時間
- 前端透過 Jackson 或手動轉換顯示當地時間

---

## 7、小結

| 要點 | 說明 |
|-----|------|
| 型別選擇 | 優先使用標準型別（BIGINT, VARCHAR, TIMESTAMPTZ） |
| JSON | PostgreSQL JSONB 功能最完整；MySQL 用虛擬欄位補索引 |
| UPSERT | PostgreSQL 用 ON CONFLICT；MySQL 用 ON DUPLICATE KEY |
| 時區 | 永遠存 UTC，讀取時轉換 |
| 主鍵策略 | PostgreSQL 推薦 IDENTITY；MySQL 使用 AUTO_INCREMENT |

### 延伸閱讀

- [02 索引原理與 SQL 優化](./02%20索引原理與%20SQL%20優化.md) -- 索引結構、EXPLAIN 分析、查詢最佳化
- [03 交易與鎖機制](./03%20交易與鎖機制.md) -- ACID、隔離級別、MVCC、死鎖排查
- [04 Redis 快取實戰](./04%20Redis%20快取實戰.md) -- Redis 整合 Spring Boot、快取策略、分散式鎖
