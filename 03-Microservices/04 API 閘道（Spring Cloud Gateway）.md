# 04 API 閘道（Spring Cloud Gateway）

> **版本**：Spring Cloud 2023.x / Spring Boot 3.x / Java 17+

## 什麼是 API 閘道

API 閘道是微服務架構中的統一入口，所有外部請求都先經過閘道，再由閘道路由到對應的微服務。閘道可以處理：

- **路由轉發**：根據規則將請求轉發到不同服務
- **負載均衡**：將請求分配到多個服務例項
- **認證授權**：統一的身份驗證
- **限流熔斷**：保護後端服務
- **日誌監控**：統一的請求日誌

## Spring Cloud Gateway 簡介

Spring Cloud Gateway 是基於 Spring 6、Spring Boot 3 和 Project Reactor 構建的 API 閘道，提供了高效能的非同步非阻塞路由機制。

核心概念：

- **Route（路由）**：閘道的基本構建塊，由 ID、目標 URI、Predicate 集合和 Filter 集合組成
- **Predicate（斷言）**：匹配 HTTP 請求的條件（路徑、標頭、引數等）
- **Filter（過濾器）**：對請求和回應進行處理

## 搭建 Gateway

### 1. 新增依賴

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### 2. 配置路由（YAML 方式）

```yaml
server:
  port: 9000

spring:
  application:
    name: gateway-server
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://service-user
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1

        - id: order-service
          uri: lb://service-order
          predicates:
            - Path=/api/order/**
          filters:
            - StripPrefix=1
```

上述配置表示：
- `http://gateway:9000/api/user/1` → `http://service-user/user/1`
- `http://gateway:9000/api/order/1` → `http://service-order/order/1`

### 3. 配置路由（Java 程式碼方式）

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/api/user/**")
                .filters(f -> f.stripPrefix(1))
                .uri("lb://service-user"))
            .route("order-service", r -> r
                .path("/api/order/**")
                .filters(f -> f.stripPrefix(1))
                .uri("lb://service-order"))
            .build();
    }
}
```

## 常用 Predicate

```yaml
predicates:
  # 路徑匹配
  - Path=/api/**

  # 請求方法
  - Method=GET,POST

  # 時間之後
  - After=2024-01-01T00:00:00+08:00

  # 時間之前
  - Before=2025-12-31T23:59:59+08:00

  # 請求標頭
  - Header=X-Request-Id, \d+

  # 查詢引數
  - Query=name, zhangsan

  # 來源 IP
  - RemoteAddr=192.168.1.0/24
```

## 常用 Filter

### 內建過濾器

```yaml
filters:
  # 移除路徑前綴
  - StripPrefix=1

  # 新增請求標頭
  - AddRequestHeader=X-Request-Source, gateway

  # 新增回應標頭
  - AddResponseHeader=X-Response-Time, 100

  # 重寫路徑
  - RewritePath=/api/(?<segment>.*), /$\{segment}

  # 重試
  - name: Retry
    args:
      retries: 3
      statuses: BAD_GATEWAY

  # 熔斷器
  - name: CircuitBreaker
    args:
      name: myCircuitBreaker
      fallbackUri: forward:/fallback
```

### 自訂全域過濾器

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");

        if (token == null || token.isEmpty()) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // 驗證 token 邏輯...
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -1; // 數字越小，優先度越高
    }
}
```

## Gateway 整合熔斷器

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("user-service", r -> r.path("/api/user/**")
            .filters(f -> f
                .circuitBreaker(c -> c
                    .setName("userCircuitBreaker")
                    .setFallbackUri("forward:/fallback"))
                .stripPrefix(1))
            .uri("lb://service-user"))
        .build();
}
```

## 限流配置

使用 Redis 實現限流：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 10   # 每秒允許 10 個請求
      redis-rate-limiter.burstCapacity: 20   # 最大突發量
      key-resolver: "#{@ipKeyResolver}"
```

```java
@Bean
public KeyResolver ipKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
    );
}
```

## 小結

Spring Cloud Gateway 是微服務架構中的重要元件，作為統一入口提供了路由、過濾、限流、熔斷等功能。它基於 WebFlux 的非同步非阻塞模型，具有出色的效能表現。

## 延伸閱讀

- [05 熔斷與限流（Resilience4j）](05%20%E7%86%94%E6%96%B7%E8%88%87%E9%99%90%E6%B5%81%EF%BC%88Resilience4j%EF%BC%89.md) — 容錯與限流機制
- [01 Spring Cloud 概述與微服務架構](01%20Spring%20Cloud%20%E6%A6%82%E8%BF%B0%E8%88%87%E5%BE%AE%E6%9C%8D%E5%8B%99%E6%9E%B6%E6%A7%8B.md) — 微服務架構總覽
- [04 API 設計最佳實踐](../09-Software-Engineering/04%20API%20設計最佳實踐.md) — RESTful 設計原則、版本策略、冪等性
- [06 安全開發實踐](../09-Software-Engineering/06%20安全開發實踐.md) — OWASP Top 10 與閘道層安全
