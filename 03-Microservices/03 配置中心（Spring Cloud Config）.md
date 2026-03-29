# 03 配置中心（Spring Cloud Config）

> **版本**：Spring Cloud 2023.x / Spring Boot 3.x / Java 17+

## 為什麼需要配置中心

在微服務架構中，每個服務都有自己的配置檔案。當服務數量增多時，分散的配置管理會帶來以下問題：

- 配置分散在各處，難以統一管理
- 修改配置需要重新打包部署
- 不同環境（開發、測試、正式）的配置切換麻煩
- 缺乏配置的版本管理和稽核追蹤

## Spring Cloud Config 簡介

Spring Cloud Config 提供了集中化的外部配置管理，分為：

- **Config Server**：配置伺服器，集中管理各環境配置
- **Config Client**：各微服務，從 Config Server 獲取配置

配置可以存放在 Git 倉庫、本地檔案系統或資料庫中。

## 搭建 Config Server

### 1. 新增依賴

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

### 2. 啟動類

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### 3. 配置檔案（使用 Git 倉庫）

```yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: main
          search-paths: '{application}'
```

### 4. Git 倉庫結構

```
config-repo/
├── service-user/
│   ├── service-user-dev.yml
│   ├── service-user-test.yml
│   └── service-user-prod.yml
├── service-order/
│   ├── service-order-dev.yml
│   └── service-order-prod.yml
└── application.yml          # 共用配置
```

### 5. 存取規則

Config Server 提供以下 REST 端點：

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{application}-{profile}.properties
```

例如：`http://localhost:8888/service-user/dev` 會返回 `service-user-dev.yml` 的內容。

## 搭建 Config Client

### 1. 新增依賴

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

### 2. 配置檔案 application.yml

```yaml
spring:
  application:
    name: service-user
  profiles:
    active: dev
  config:
    import: "configserver:http://localhost:8888"
```

### 3. 使用配置

```java
@RestController
public class ConfigTestController {

    @Value("${custom.message:預設訊息}")
    private String message;

    @GetMapping("/config")
    public String getConfig() {
        return message;
    }
}
```

## 配置動態重新整理

### 使用 @RefreshScope

```java
@RestController
@RefreshScope
public class ConfigTestController {

    @Value("${custom.message}")
    private String message;

    @GetMapping("/config")
    public String getConfig() {
        return message;
    }
}
```

需要新增 Actuator 依賴：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

暴露 refresh 端點：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh
```

手動觸發重新整理：

```bash
curl -X POST http://localhost:8001/actuator/refresh
```

### 使用 Spring Cloud Bus 自動重新整理

結合訊息中介軟體（如 RabbitMQ），可以實現配置修改後自動通知所有服務：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

## 配置加密

Spring Cloud Config 支援對敏感配置進行加密：

```yaml
# 對稱加密
encrypt:
  key: my-secret-key
```

加密配置值以 `{cipher}` 前綴標記：

```yaml
spring:
  datasource:
    password: '{cipher}AQBHn3...'
```

## Nacos Config 替代方案

與服務發現類似，Nacos 同時提供配置管理功能：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: localhost:8848
        file-extension: yml
```

Nacos 的優勢在於提供了視覺化的管理介面，支援配置的即時推送，無需手動觸發重新整理。

## 小結

配置中心是微服務架構中不可或缺的基礎設施。Spring Cloud Config 基於 Git 提供了版本化的配置管理，而 Nacos Config 則提供了更便捷的操作介面和即時推送能力。

## 延伸閱讀

- [01 Spring Cloud 概述與微服務架構](01%20Spring%20Cloud%20%E6%A6%82%E8%BF%B0%E8%88%87%E5%BE%AE%E6%9C%8D%E5%8B%99%E6%9E%B6%E6%A7%8B.md) — 微服務架構總覽
- [05 Spring Boot 配置檔案與 Profiles](../02-Spring-Ecosystem/05%20Spring%20Boot%20配置檔案與%20Profiles.md) — Spring Boot 配置檔案基礎
