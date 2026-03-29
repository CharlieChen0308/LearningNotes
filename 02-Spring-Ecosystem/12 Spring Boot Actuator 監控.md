# 12 Spring Boot Actuator 監控

> **版本**: Spring Boot 3.x
> **來源**: `_archive/Spring Boot/05 Spring Boot Actuator 監控與管理.md`

## 什麼是 Actuator

Spring Boot Actuator 提供了一系列**生產就緒**的監控和管理功能，透過 HTTP 端點或 JMX 暴露應用程式的健康狀況、指標、配置等資訊，是維運和除錯的重要工具。

## 快速開始

### 新增依賴

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 配置暴露端點

預設只暴露 `health` 端點，可依需求開放更多：

```yaml
management:
  endpoints:
    web:
      exposure:
        # 暴露指定端點
        include: health,info,metrics,beans,env,loggers,mappings
        # 或暴露全部（開發環境）
        # include: "*"
  endpoint:
    health:
      show-details: when-authorized  # always / never / when-authorized
```

## 常用端點

| 端點 | 路徑 | 功能 |
|------|------|------|
| health | `/actuator/health` | 應用程式健康狀態 |
| info | `/actuator/info` | 應用程式資訊 |
| metrics | `/actuator/metrics` | 各種指標（記憶體、CPU、HTTP 請求統計） |
| beans | `/actuator/beans` | 所有 Spring Bean 列表 |
| env | `/actuator/env` | 環境變數和配置屬性 |
| loggers | `/actuator/loggers` | 日誌等級查看與動態修改 |
| mappings | `/actuator/mappings` | 所有 `@RequestMapping` 路徑 |
| conditions | `/actuator/conditions` | 自動配置條件報告 |
| configprops | `/actuator/configprops` | 所有 `@ConfigurationProperties` |
| threaddump | `/actuator/threaddump` | 執行緒轉儲 |
| heapdump | `/actuator/heapdump` | 堆記憶體轉儲（下載檔案） |
| scheduledtasks | `/actuator/scheduledtasks` | 定時任務資訊 |
| shutdown | `/actuator/shutdown` | 優雅關閉應用程式（需特別啟用） |

## Health 端點

### 基本健康檢查

存取 `http://localhost:8080/actuator/health`：

```json
{
    "status": "UP",
    "components": {
        "db": {
            "status": "UP",
            "details": {
                "database": "PostgreSQL",
                "validationQuery": "isValid()"
            }
        },
        "diskSpace": {
            "status": "UP",
            "details": {
                "total": 499963174912,
                "free": 250000000000,
                "threshold": 10485760
            }
        },
        "redis": {
            "status": "UP"
        }
    }
}
```

Spring Boot 會自動偵測你引入的依賴，加入對應的健康指標（資料庫、Redis、Elasticsearch 等）。

### 自訂健康指標

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;

    public ExternalApiHealthIndicator(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                "https://api.example.com/health", String.class);
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("externalApi", "可用")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("externalApi", "不可用")
                .withException(e)
                .build();
        }
        return Health.down().build();
    }
}
```

## Metrics 端點

### 查看可用指標

`GET /actuator/metrics`：

```json
{
    "names": [
        "jvm.memory.used",
        "jvm.memory.max",
        "jvm.gc.pause",
        "http.server.requests",
        "process.cpu.usage",
        "system.cpu.usage",
        "tomcat.threads.current",
        "tomcat.threads.busy"
    ]
}
```

### 查看特定指標

`GET /actuator/metrics/jvm.memory.used`：

```json
{
    "name": "jvm.memory.used",
    "measurements": [
        {
            "statistic": "VALUE",
            "value": 268435456
        }
    ],
    "availableTags": [
        {
            "tag": "area",
            "values": ["heap", "nonheap"]
        }
    ]
}
```

### 自訂指標

```java
@RestController
public class OrderController {

    private final Counter orderCounter;
    private final Timer orderTimer;

    public OrderController(MeterRegistry meterRegistry) {
        this.orderCounter = meterRegistry.counter("orders.created");
        this.orderTimer = meterRegistry.timer("orders.processing.time");
    }

    @PostMapping("/api/orders")
    public OrderDto createOrder(@RequestBody CreateOrderRequest request) {
        return orderTimer.record(() -> {
            OrderDto order = orderService.create(request);
            orderCounter.increment();
            return order;
        });
    }
}
```

## Loggers 端點 — 動態修改日誌等級

### 查看日誌等級

`GET /actuator/loggers/com.example`：

```json
{
    "configuredLevel": null,
    "effectiveLevel": "INFO"
}
```

### 動態修改（不需重啟）

```bash
curl -X POST http://localhost:8080/actuator/loggers/com.example \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

在生產環境中出現問題時，可以即時開啟 DEBUG 日誌，排查完畢再改回來。

## 安全防護

Actuator 端點包含敏感資訊，**正式環境務必保護**：

### 使用 Spring Security

```java
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain actuatorFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN"))
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

### 使用獨立連接埠

```yaml
management:
  server:
    port: 9090           # Actuator 使用不同的連接埠
    address: 127.0.0.1   # 只允許本機存取
```

## 整合 Prometheus + Grafana

將 Actuator 指標匯出到 Prometheus，再用 Grafana 視覺化：

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus
```

> **Spring Boot 3.x 注意**：只要加入 `micrometer-registry-prometheus` 依賴並暴露 `prometheus` 端點，Prometheus 指標即自動啟用，無需額外設定 `management.prometheus.metrics.export.enabled`。

存取 `http://localhost:8080/actuator/prometheus` 即可取得 Prometheus 格式的指標資料。

## 小結

Spring Boot Actuator 是生產環境不可或缺的工具，提供了健康檢查、指標監控、日誌動態調整等功能。搭配 Prometheus + Grafana 可以建立完整的可觀測性方案，讓維運團隊即時掌握系統狀態。

> **延伸閱讀**：
> - [04 Spring Boot 自動配置與 Starters](04%20Spring%20Boot%20自動配置與%20Starters.md) — 理解 Actuator Starter 背後的自動配置機制
> - [05 Spring Boot 配置檔案與 Profiles](05%20Spring%20Boot%20配置檔案與%20Profiles.md) — 依環境切換 Actuator 暴露端點與安全設定

---
審查狀態：APPROVED — 2026-Q1
- [x] 技術正確性
- [x] 架構與方法論
- [x] 生產實戰
- [x] 內容結構
- [x] 術語與一致性
- [x] 讀者路徑
- [x] 時效性
