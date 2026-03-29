# 04 Redis 快取實戰

> **版本**：Redis 7.x / Spring Boot 3.x / Spring Data Redis

---

## 1、Redis 資料型別

| 型別 | 說明 | 使用場景 | 常用指令 |
|-----|------|---------|---------|
| String | 字串/數值/二進位 | 快取、計數器、分散式鎖 | `SET`, `GET`, `INCR`, `SETNX` |
| List | 有序串列（雙向鏈結） | 訊息佇列、最新動態 | `LPUSH`, `RPOP`, `LRANGE` |
| Set | 無序集合（唯一值） | 標籤、共同好友、去重 | `SADD`, `SMEMBERS`, `SINTER` |
| Sorted Set | 有序集合（依 score 排序） | 排行榜、延遲佇列 | `ZADD`, `ZRANGE`, `ZRANK` |
| Hash | 雜湊表（field-value） | 物件快取（User、Order） | `HSET`, `HGET`, `HGETALL` |
| Stream | 訊息流（append-only log） | 事件溯源、訊息佇列 | `XADD`, `XREAD`, `XREADGROUP` |

### 型別選擇原則

```
快取單一值（token、設定）    → String
快取物件（使用者資料）       → Hash
需要排序/排行               → Sorted Set
任務佇列                   → List 或 Stream
去重/集合運算              → Set
可靠訊息佇列               → Stream（支援 consumer group、ACK）
```

---

## 2、Spring Boot 整合 Redis

### 依賴引入

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!-- 連線池 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

### application.yml 配置

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ""           # 正式環境必須設定密碼
      database: 0
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 16     # 最大連線數
          max-idle: 8        # 最大閒置連線
          min-idle: 2        # 最小閒置連線
          max-wait: 3000ms   # 等待連線的最大時間
```

### RedisTemplate 配置

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // Key 使用 String 序列化
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());

        // Value 使用 JSON 序列化（方便除錯，可讀性高）
        ObjectMapper om = new ObjectMapper();
        om.activateDefaultTyping(
                om.getPolymorphicTypeValidator(),
                ObjectMapper.DefaultTyping.NON_FINAL
        );

        // Spring Boot 3.x 建議使用建構子注入，setObjectMapper() 已標記為 @Deprecated
        Jackson2JsonRedisSerializer<Object> jsonSerializer =
                new Jackson2JsonRedisSerializer<>(om, Object.class);

        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);

        template.afterPropertiesSet();
        return template;
    }
}
```

### RedisTemplate vs StringRedisTemplate

| 特性 | RedisTemplate | StringRedisTemplate |
|-----|-------------|-------------------|
| Key 序列化 | 預設 JdkSerializationRedisSerializer | StringRedisSerializer |
| Value 序列化 | 預設 JdkSerializationRedisSerializer | StringRedisSerializer |
| 適用場景 | 需要存物件（搭配自訂序列化） | 只存字串（簡單 KV） |
| 可讀性 | 預設差（二進位），自訂 JSON 後好 | 好（純文字） |

**建議**：自訂 `RedisTemplate` 使用 JSON 序列化，同時保留 `StringRedisTemplate` 處理簡單字串場景。

---

## 3、Spring Cache 整合

### 啟用快取

```java
@SpringBootApplication
@EnableCaching
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### RedisCacheManager 配置

```java
@Configuration
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        // 預設快取配置
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(30))                         // 預設 TTL 30 分鐘
                .serializeKeysWith(
                        RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(
                        RedisSerializationContext.SerializationPair.fromSerializer(
                                new GenericJackson2JsonRedisSerializer()))
                .disableCachingNullValues();                              // 是否快取 null

        // 個別快取的自訂配置
        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
                "users",    defaultConfig.entryTtl(Duration.ofHours(1)),  // 使用者快取 1 小時
                "products", defaultConfig.entryTtl(Duration.ofMinutes(10)), // 商品快取 10 分鐘
                "configs",  defaultConfig.entryTtl(Duration.ofHours(24))  // 設定快取 24 小時
        );

        return RedisCacheManager.builder(factory)
                .cacheDefaults(defaultConfig)
                .withInitialCacheConfigurations(cacheConfigs)
                .transactionAware()      // 交易感知：commit 後才寫入快取
                .build();
    }
}
```

### 快取註解使用

```java
@Service
public class CustomerService {

    private final CustomerRepository customerRepo;

    // @Cacheable：查詢時先讀快取，沒有才查 DB 並寫入快取
    @Cacheable(value = "customers", key = "#id")
    public CustomerDTO getCustomer(Long id) {
        return customerRepo.findById(id)
                .map(this::toDTO)
                .orElseThrow(() -> new NotFoundException("Customer not found: " + id));
    }

    // @CachePut：更新時同步更新快取（先執行方法，再寫入快取）
    @CachePut(value = "customers", key = "#id")
    public CustomerDTO updateCustomer(Long id, CustomerDTO dto) {
        Customer entity = customerRepo.findById(id).orElseThrow();
        entity.setName(dto.getName());
        entity.setPhone(dto.getPhone());
        return toDTO(customerRepo.save(entity));
    }

    // @CacheEvict：刪除時清除快取
    @CacheEvict(value = "customers", key = "#id")
    public void deleteCustomer(Long id) {
        customerRepo.deleteById(id);
    }

    // @CacheEvict：清除整個快取空間
    @CacheEvict(value = "customers", allEntries = true)
    public void clearCustomerCache() {
        // 用於手動清除快取
    }

    // 條件式快取
    @Cacheable(value = "customers", key = "#id", condition = "#id > 0", unless = "#result == null")
    public CustomerDTO getCustomerConditional(Long id) {
        return customerRepo.findById(id).map(this::toDTO).orElse(null);
    }
}
```

---

## 4、快取問題與解決方案

### 4.1 快取穿透 (Cache Penetration)

**問題**：大量請求查詢不存在的資料（如 id=-1），每次都穿透到 DB。

**方案 A：快取空值**

```java
@Cacheable(value = "customers", key = "#id", unless = "#result == null")
public CustomerDTO getCustomer(Long id) {
    return customerRepo.findById(id)
            .map(this::toDTO)
            .orElse(null);  // null 也會被快取（需要短 TTL）
}
```

更精確的做法：

```java
public CustomerDTO getCustomer(Long id) {
    String key = "customer:" + id;
    Object cached = redisTemplate.opsForValue().get(key);

    if (cached != null) {
        if ("NULL".equals(cached)) return null;  // 空值標記
        return (CustomerDTO) cached;
    }

    CustomerDTO dto = customerRepo.findById(id).map(this::toDTO).orElse(null);
    if (dto != null) {
        redisTemplate.opsForValue().set(key, dto, Duration.ofHours(1));
    } else {
        redisTemplate.opsForValue().set(key, "NULL", Duration.ofMinutes(5));  // 空值短 TTL
    }
    return dto;
}
```

**方案 B：布隆過濾器（Bloom Filter）**

```java
// 使用 Redisson 的布隆過濾器
@Component
public class CustomerBloomFilter {

    private final RBloomFilter<Long> bloomFilter;

    public CustomerBloomFilter(RedissonClient redisson) {
        this.bloomFilter = redisson.getBloomFilter("customer:bloom");
        // 預期元素數量 100 萬，誤判率 1%
        this.bloomFilter.tryInit(1_000_000L, 0.01);
    }

    public void add(Long customerId) {
        bloomFilter.add(customerId);
    }

    public boolean mightExist(Long customerId) {
        return bloomFilter.contains(customerId);  // false = 一定不存在
    }
}

// Service 中使用
public CustomerDTO getCustomer(Long id) {
    if (!bloomFilter.mightExist(id)) {
        return null;  // 一定不存在，直接返回
    }
    // ... 查快取 → 查 DB
}
```

### 4.2 快取擊穿 (Cache Breakdown)

**問題**：熱點 key 過期的瞬間，大量併發請求同時打到 DB。

**解決方案：分散式鎖**

> 此為教學簡化範例，生產環境需額外考慮：
> - 重試上限與退避策略（exponential backoff）
> - 鎖的原子性釋放（Lua 腳本或 Redisson）
> - 降級策略（重試耗盡後返回預設值或拋出明確例外）

```java
private static final int MAX_RETRY = 3;

public CustomerDTO getCustomerWithLock(Long id) {
    return getCustomerWithLock(id, 0);
}

private CustomerDTO getCustomerWithLock(Long id, int retryCount) {
    String cacheKey = "customer:" + id;
    String lockKey = "lock:customer:" + id;

    // 1. 先查快取
    CustomerDTO dto = (CustomerDTO) redisTemplate.opsForValue().get(cacheKey);
    if (dto != null) return dto;

    // 2. 取得分散式鎖
    Boolean locked = redisTemplate.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(10));
    if (Boolean.TRUE.equals(locked)) {
        try {
            // Double check
            dto = (CustomerDTO) redisTemplate.opsForValue().get(cacheKey);
            if (dto != null) return dto;

            // 查 DB 並寫入快取
            dto = customerRepo.findById(id).map(this::toDTO).orElse(null);
            if (dto != null) {
                redisTemplate.opsForValue().set(cacheKey, dto, Duration.ofHours(1));
            }
            return dto;
        } finally {
            redisTemplate.delete(lockKey);
        }
    } else {
        // 未取得鎖，等待後重試（設定重試上限避免無限遞迴）
        if (retryCount >= MAX_RETRY) {
            throw new RuntimeException("快取重建等待逾時，key=" + cacheKey);
        }
        try { Thread.sleep(50); } catch (InterruptedException ignored) {
            Thread.currentThread().interrupt();
        }
        return getCustomerWithLock(id, retryCount + 1);
    }
}
```

### 4.3 快取雪崩 (Cache Avalanche)

**問題**：大量 key 同時過期（或 Redis 當機），請求全部打到 DB。

**方案 1：隨機 TTL**

```java
public void cacheCustomer(Long id, CustomerDTO dto) {
    // 基礎 TTL + 隨機偏移，避免同時過期
    long baseTtl = 3600; // 1 小時
    long randomOffset = ThreadLocalRandom.current().nextLong(0, 600); // 0~10 分鐘
    redisTemplate.opsForValue().set("customer:" + id, dto, Duration.ofSeconds(baseTtl + randomOffset));
}
```

**方案 2：多級快取**

```
請求 → 本地快取 (Caffeine, 1 分鐘) → Redis (30 分鐘) → DB
```

```java
@Configuration
public class MultiLevelCacheConfig {

    @Bean
    public CaffeineCacheManager localCacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(1)));
        return manager;
    }
}
```

---

## 5、分散式鎖

### 5.1 SETNX + EXPIRE（簡易版）

```java
public boolean tryLock(String lockKey, String value, long expireSeconds) {
    Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, value, Duration.ofSeconds(expireSeconds));
    return Boolean.TRUE.equals(result);
}

public void unlock(String lockKey, String value) {
    // 用 Lua 腳本保證原子性：只刪除自己的鎖
    String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
    redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            List.of(lockKey),
            value
    );
}
```

**缺點**：鎖無法自動續期，業務執行時間超過 expire 時間會導致鎖提前釋放。

### 5.2 Redisson RLock（推薦）

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.27.0</version> <!-- 請依實際 Spring Boot 版本查閱 Redisson 相容性矩陣 -->
</dependency>
```

> **版本相容性注意**：Redisson 的 major 版本與 Spring Boot 版本有對應關係。
> 請參考 [Redisson GitHub Wiki](https://github.com/redisson/redisson#spring-boot-integration) 確認
> 當前 Spring Boot 版本對應的 Redisson 版本，避免執行時期相容性問題。

```java
@Service
public class OrderService {

    private final RedissonClient redisson;

    public void processOrder(Long orderId) {
        RLock lock = redisson.getLock("lock:order:" + orderId);
        try {
            // 等待 5 秒取得鎖，鎖自動過期 30 秒（看門狗會自動續期）
            boolean locked = lock.tryLock(5, 30, TimeUnit.SECONDS);
            if (!locked) {
                throw new RuntimeException("無法取得鎖，訂單處理中：" + orderId);
            }
            // 業務邏輯
            doProcessOrder(orderId);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("鎖等待被中斷", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### 5.3 看門狗自動續期（Watchdog）

Redisson 的看門狗機制：

- 如果不指定 `leaseTime`（或設為 -1），Redisson 會啟動看門狗
- 看門狗每隔 `lockWatchdogTimeout / 3`（預設 10 秒）自動續期
- 預設鎖過期時間 30 秒（`lockWatchdogTimeout`）
- 當持有鎖的 JVM 程序正常結束或當機，看門狗停止，鎖自動過期釋放

```java
// 觸發看門狗：不指定 leaseTime
RLock lock = redisson.getLock("my-lock");
lock.lock();  // 看門狗自動續期，直到 unlock() 或 JVM 結束

// 不觸發看門狗：指定了 leaseTime
lock.lock(30, TimeUnit.SECONDS);  // 30 秒後自動釋放，不續期
```

---

## 6、常用操作範例

### RedisTemplate 操作

```java
@Service
public class RedisExampleService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final StringRedisTemplate stringRedisTemplate;

    // String 操作
    public void stringOps() {
        // 設定值 + TTL
        redisTemplate.opsForValue().set("key1", "value1", Duration.ofMinutes(30));

        // 取得值
        Object value = redisTemplate.opsForValue().get("key1");

        // 計數器
        stringRedisTemplate.opsForValue().increment("counter:orders");

        // SETNX（不存在才設定）
        Boolean success = redisTemplate.opsForValue()
                .setIfAbsent("lock:key", "value", Duration.ofSeconds(10));
    }

    // Hash 操作（適合存物件）
    public void hashOps() {
        String key = "user:1001";
        redisTemplate.opsForHash().put(key, "name", "Alice");
        redisTemplate.opsForHash().put(key, "email", "alice@example.com");

        // 批次設定
        Map<String, String> fields = Map.of("name", "Alice", "email", "alice@example.com");
        redisTemplate.opsForHash().putAll(key, fields);

        // 取得單一欄位
        Object name = redisTemplate.opsForHash().get(key, "name");

        // 取得所有欄位
        Map<Object, Object> allFields = redisTemplate.opsForHash().entries(key);

        redisTemplate.expire(key, Duration.ofHours(1));
    }

    // Sorted Set 操作（排行榜）
    public void sortedSetOps() {
        String key = "leaderboard:sales";

        // 新增/更新分數
        redisTemplate.opsForZSet().add(key, "employee:1", 15000);
        redisTemplate.opsForZSet().add(key, "employee:2", 23000);
        redisTemplate.opsForZSet().add(key, "employee:3", 18000);

        // 增加分數
        redisTemplate.opsForZSet().incrementScore(key, "employee:1", 5000);

        // 取得排行榜 Top 10（分數由高到低）
        Set<ZSetOperations.TypedTuple<Object>> top10 =
                redisTemplate.opsForZSet().reverseRangeWithScores(key, 0, 9);

        // 取得某成員排名（0-based，分數最高排第 0）
        Long rank = redisTemplate.opsForZSet().reverseRank(key, "employee:1");
    }

    // List 操作（任務佇列）
    public void listOps() {
        String key = "queue:notifications";

        // 推入佇列（左進右出 = FIFO）
        redisTemplate.opsForList().leftPush(key, "notification:1");
        redisTemplate.opsForList().leftPush(key, "notification:2");

        // 取出（阻塞式，最多等 5 秒）
        Object item = redisTemplate.opsForList().rightPop(key, Duration.ofSeconds(5));
    }
}
```

### Lua 腳本實現原子操作

```java
@Service
public class RateLimiterService {

    private final StringRedisTemplate redisTemplate;

    /**
     * 滑動視窗限流：每分鐘最多 maxRequests 次
     */
    public boolean isAllowed(String clientId, int maxRequests) {
        String script = """
                local key = KEYS[1]
                local max = tonumber(ARGV[1])
                local window = tonumber(ARGV[2])
                local now = tonumber(ARGV[3])

                -- 移除視窗外的記錄
                redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

                -- 計算當前視窗內的請求數
                local count = redis.call('ZCARD', key)

                if count < max then
                    -- 允許請求，記錄時間戳
                    redis.call('ZADD', key, now, now .. ':' .. math.random())
                    redis.call('EXPIRE', key, window / 1000)
                    return 1
                else
                    return 0
                end
                """;

        Long result = redisTemplate.execute(
                new DefaultRedisScript<>(script, Long.class),
                List.of("rate:" + clientId),
                String.valueOf(maxRequests),
                String.valueOf(60_000),               // 60 秒視窗
                String.valueOf(System.currentTimeMillis())
        );

        return result != null && result == 1L;
    }

    /**
     * 庫存扣減（原子操作）
     */
    public boolean deductStock(String productId, int quantity) {
        String script = """
                local stock = tonumber(redis.call('GET', KEYS[1]))
                if stock == nil then
                    return -1
                end
                if stock >= tonumber(ARGV[1]) then
                    redis.call('DECRBY', KEYS[1], ARGV[1])
                    return 1
                else
                    return 0
                end
                """;

        Long result = redisTemplate.execute(
                new DefaultRedisScript<>(script, Long.class),
                List.of("stock:" + productId),
                String.valueOf(quantity)
        );

        return result != null && result == 1L;
    }
}
```

---

## 7、Redis 運維與生產注意事項

### 7.1 持久化策略

| 方式 | 原理 | 優點 | 缺點 |
|-----|------|------|------|
| RDB（快照） | 定時將記憶體快照寫入磁碟 | 備份/還原速度快，檔案緊湊 | 兩次快照之間的資料可能遺失 |
| AOF（日誌追加） | 記錄每筆寫入指令 | 資料持久性高（可設 `everysec`/`always`） | 檔案較大，重播還原較慢 |
| RDB + AOF 混合 | 結合兩者優勢 | 兼顧備份速度與資料安全 | 需同時管理兩種機制 |

**建議**：生產環境同時啟用 RDB（用於定期備份與災難復原）與 AOF（`appendfsync everysec` 確保最多遺失 1 秒資料），達到最佳安全性。

### 7.2 高可用架構

| 方案 | 用途 | 適用場景 |
|-----|------|---------|
| Redis Sentinel | 自動故障轉移（failover）+ 監控 | 主從架構，資料量單機可承受 |
| Redis Cluster | 資料分片（sharding）+ 高可用 | 資料量超過單機記憶體，需水平擴展 |

**選擇原則**：

- 資料量 < 單機記憶體上限 → Sentinel（架構簡單，維運成本低）
- 資料量需水平擴展或寫入吞吐量極高 → Cluster（自動分片，但應用端需注意跨 slot 操作限制）

### 7.3 記憶體淘汰策略

透過 `maxmemory-policy` 控制記憶體滿時的行為：

| 策略 | 說明 | 適用場景 |
|-----|------|---------|
| `allkeys-lru` | 淘汰最近最少使用的 key | **快取場景**（推薦） |
| `volatile-lru` | 只淘汰有設定 TTL 的 key（LRU） | 快取與持久資料混用 |
| `allkeys-lfu` | 淘汰最不常使用的 key（Redis 4.0+） | 存取頻率差異大的場景 |
| `noeviction` | 記憶體滿時拒絕寫入，回傳錯誤 | **資料儲存場景**（不可遺失） |

**建議**：純快取用途使用 `allkeys-lru`；若 Redis 同時作為資料儲存，使用 `noeviction` 並搭配監控告警。

### 7.4 連線安全

| 項目 | 說明 |
|-----|------|
| 密碼認證 | 生產環境必須設定 `requirepass`，禁止無密碼暴露 |
| TLS 加密 | Redis 6.0+ 原生支援 TLS，跨網路傳輸時務必啟用 |
| Redis ACL | Redis 6.0+ 支援細粒度存取控制，可限制使用者只能操作特定 key 或指令 |
| 網路隔離 | `bind` 限定監聽介面，禁止直接暴露至公網，搭配防火牆規則 |

```yaml
# application.yml 生產環境範例
spring:
  data:
    redis:
      host: redis.internal.example.com
      port: 6379
      password: ${REDIS_PASSWORD}    # 從環境變數或 secret manager 注入
      ssl:
        enabled: true                # 啟用 TLS
      username: app_user             # Redis ACL 使用者（Redis 6.0+）
```

---

## 8、小結

| 要點 | 說明 |
|-----|------|
| 資料型別 | 根據場景選擇：物件用 Hash，排行用 Sorted Set，佇列用 List/Stream |
| 序列化 | 推薦 JSON 序列化（可讀性好），Key 用 StringRedisSerializer |
| Spring Cache | @Cacheable / @CachePut / @CacheEvict，搭配 RedisCacheManager 設定 TTL |
| 快取穿透 | 空值快取（短 TTL）+ 布隆過濾器 |
| 快取擊穿 | 分散式鎖保護熱點 key 重建（注意重試上限） |
| 快取雪崩 | 隨機 TTL + 多級快取（Caffeine + Redis） |
| 分散式鎖 | 推薦 Redisson RLock，支援看門狗自動續期 |
| 原子操作 | Lua 腳本保證多步驟原子性（限流、庫存扣減） |
| 持久化 | RDB + AOF 混合模式，兼顧備份速度與資料安全 |
| 高可用 | 單機容量足夠用 Sentinel，需水平擴展用 Cluster |
| 記憶體淘汰 | 快取用 `allkeys-lru`，資料儲存用 `noeviction` |
| 連線安全 | 密碼 + TLS + ACL + 網路隔離，缺一不可 |

### 延伸閱讀


- [01 PostgreSQL 與 MySQL 基礎](./01%20PostgreSQL%20與%20MySQL%20基礎.md) -- 資料型別、DDL/DML、JSON 操作
- [02 索引原理與 SQL 優化](./02%20索引原理與%20SQL%20優化.md) -- 索引結構、EXPLAIN 分析、查詢最佳化
- [03 交易與鎖機制](./03%20交易與鎖機制.md) -- ACID、隔離級別、MVCC、死鎖排查
