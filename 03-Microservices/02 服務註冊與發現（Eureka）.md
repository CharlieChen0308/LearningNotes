# 02 服務註冊與發現（Eureka）

> **版本**：Spring Cloud 2023.x / Spring Boot 3.x / Java 17+

## 什麼是服務註冊與發現

在微服務架構中，服務例項的數量和網路位置可能動態變化。服務註冊與發現機制讓各服務能自動註冊自身資訊，並能查詢其他服務的位置，無需寫死 IP 和連接埠。

## Eureka 簡介

Netflix Eureka 是 Spring Cloud 中最經典的服務註冊與發現元件，包含兩個角色：

- **Eureka Server**：註冊中心，負責管理所有服務的註冊資訊
- **Eureka Client**：服務提供者和消費者，向 Eureka Server 註冊自身並獲取其他服務的資訊

> **注意**：Eureka 目前已進入維護模式（maintenance mode），不再新增功能，但仍廣泛運行於許多生產環境中。較新的替代方案包括 HashiCorp Consul 和 Alibaba Nacos，建議新專案評估後選用。

### 主流服務發現方案比較

| 比較維度 | Eureka | Consul | Nacos |
|---------|--------|--------|-------|
| 功能範圍 | 僅服務發現 | 服務發現 + KV 配置 + 服務網格 | 服務發現 + 配置管理 + 流量管理 |
| 部署複雜度 | 低（Spring Boot 應用直接啟動） | 中（需獨立部署 Agent） | 中（需獨立部署，提供 Docker 映像） |
| 社群活躍度 | 低（Netflix 已停止新功能開發） | 高（HashiCorp 持續維護） | 高（阿里巴巴活躍維護，國內社群龐大） |
| 配置管理 | 不支援（需搭配 Spring Cloud Config） | 內建 KV Store | 內建，支援多格式與命名空間 |
| CP/AP 模式 | AP（優先可用性，允許暫時不一致） | 預設 CP（強一致性），可切換 AP | 同時支援 CP 與 AP，依實例類型自動切換 |

## 搭建 Eureka Server

### 1. 新增依賴

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

### 2. 啟動類

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### 3. 配置檔案 application.yml

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    # 不向自己註冊
    register-with-eureka: false
    # 不從自己獲取註冊資訊
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

啟動後存取 `http://localhost:8761` 即可看到 Eureka 管理控制台。

## 搭建 Eureka Client（服務提供者）

### 1. 新增依賴

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### 2. 配置檔案

```yaml
spring:
  application:
    name: service-user

server:
  port: 8001

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

### 3. 服務提供者範例

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @GetMapping("/{id}")
    public Map<String, Object> getUser(@PathVariable Long id) {
        Map<String, Object> user = new HashMap<>();
        user.put("id", id);
        user.put("name", "張三");
        user.put("email", "zhangsan@example.com");
        return user;
    }
}
```

## 搭建服務消費者

### 使用 RestTemplate 呼叫

```java
@SpringBootApplication
public class OrderServiceApplication {

    @Bean
    @LoadBalanced  // 啟用負載均衡
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

```java
@RestController
@RequestMapping("/order")
public class OrderController {

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/{userId}")
    public Map<String, Object> getOrder(@PathVariable Long userId) {
        // 透過服務名稱呼叫，而非具體 IP
        Map<String, Object> user = restTemplate.getForObject(
            "http://service-user/user/" + userId, Map.class);

        Map<String, Object> order = new HashMap<>();
        order.put("orderId", 1001L);
        order.put("user", user);
        return order;
    }
}
```

## Eureka 高可用配置

生產環境中應部署多個 Eureka Server 互相註冊，形成叢集：

**Eureka Server 1（連接埠 8761）：**

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8762/eureka/
```

**Eureka Server 2（連接埠 8762）：**

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

## Eureka 的自我保護機制

當 Eureka Server 在短時間內丟失過多用戶端心跳時，會進入自我保護模式，不再移除因長時間沒有收到心跳的服務。這是為了避免網路分區故障時誤刪正常服務。

```yaml
eureka:
  server:
    # 開發環境可關閉自我保護
    enable-self-preservation: false
    # 清理間隔（毫秒）
    eviction-interval-timer-in-ms: 5000
```

## Nacos 替代方案

目前越來越多專案使用 Alibaba Nacos 替代 Eureka，Nacos 同時提供服務發現和配置管理功能：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
```

## 小結

服務註冊與發現是微服務架構的基礎設施。Eureka 提供了簡單可靠的解決方案，而 Nacos 則是目前更受歡迎的替代選擇，它整合了服務發現與配置管理功能。

## 延伸閱讀

- [01 Spring Cloud 概述與微服務架構](01%20Spring%20Cloud%20%E6%A6%82%E8%BF%B0%E8%88%87%E5%BE%AE%E6%9C%8D%E5%8B%99%E6%9E%B6%E6%A7%8B.md) — 微服務架構總覽
- [06 負載均衡（Spring Cloud LoadBalancer）](06%20%E8%B2%A0%E8%BC%89%E5%9D%87%E8%A1%A1%EF%BC%88Spring%20Cloud%20LoadBalancer%EF%BC%89.md) — 用戶端負載均衡
- [09 系統設計入門](../09-Software-Engineering/09%20系統設計入門.md) — 水平擴展與服務發現的架構背景
