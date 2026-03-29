# 06 負載均衡（Spring Cloud LoadBalancer）

## 什麼是負載均衡

負載均衡是將網路流量分散到多個服務例項上，以提高系統的可用性和吞吐量。在微服務架構中，同一個服務通常會部署多個例項。

負載均衡分為兩種：

- **伺服器端負載均衡**：由獨立的負載均衡器（如 Nginx、HAProxy）進行流量分配
- **用戶端負載均衡**：由服務消費者自己從註冊中心獲取服務清單，自行選擇呼叫哪個例項

Spring Cloud LoadBalancer 屬於**用戶端負載均衡**。

## Spring Cloud LoadBalancer 簡介

Spring Cloud LoadBalancer 是 Spring Cloud 官方提供的用戶端負載均衡元件，取代了早期的 Netflix Ribbon。它與 `@LoadBalanced` 註解配合使用，自動將服務名稱解析為具體的服務例項地址。

### 新增依賴

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

> 注意：`spring-cloud-starter-netflix-eureka-client` 已經包含了 LoadBalancer 依賴，通常不需要額外新增。

## 基本使用

### 搭配 RestTemplate

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```java
@Service
public class OrderService {

    @Autowired
    private RestTemplate restTemplate;

    public Map<String, Object> getOrder(Long userId) {
        // "service-user" 是服務名稱，LoadBalancer 會自動解析為具體 IP:Port
        Map user = restTemplate.getForObject(
            "http://service-user/user/" + userId, Map.class);
        // ...
        return orderInfo;
    }
}
```

### 搭配 WebClient（WebFlux 環境）

```java
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}
```

```java
@Service
public class OrderService {

    @Autowired
    private WebClient.Builder webClientBuilder;

    public Mono<Map> getUser(Long userId) {
        return webClientBuilder.build()
            .get()
            .uri("http://service-user/user/" + userId)
            .retrieve()
            .bodyToMono(Map.class);
    }
}
```

## 負載均衡策略

### 內建策略

Spring Cloud LoadBalancer 預設提供兩種策略：

| 策略 | 說明 |
|------|------|
| RoundRobinLoadBalancer | 輪詢（預設），依序呼叫每個例項 |
| RandomLoadBalancer | 隨機選擇一個例項 |

### 切換為隨機策略

```java
public class RandomLoadBalancerConfig {

    @Bean
    ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {

        String name = environment.getProperty(
            LoadBalancerClientFactory.PROPERTY_NAME);

        return new RandomLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name);
    }
}
```

指定特定服務使用隨機策略：

```java
@Configuration
@LoadBalancerClient(name = "service-user", configuration = RandomLoadBalancerConfig.class)
public class LoadBalancerConfig {
}
```

### 自訂負載均衡策略

```java
public class WeightedLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final String serviceId;
    private final ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;

    public WeightedLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> provider,
            String serviceId) {
        this.serviceInstanceListSupplierProvider = provider;
        this.serviceId = serviceId;
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        return serviceInstanceListSupplierProvider.getIfAvailable()
            .get(request)
            .next()
            .map(this::getInstanceResponse);
    }

    private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return new EmptyResponse();
        }
        // 自訂選擇邏輯，例如根據權重選擇
        ServiceInstance instance = selectByWeight(instances);
        return new DefaultResponse(instance);
    }

    private ServiceInstance selectByWeight(List<ServiceInstance> instances) {
        // 根據 metadata 中的 weight 欄位進行加權選擇
        // 實作細節略...
        return instances.get(0);
    }
}
```

## 服務例項快取

LoadBalancer 預設會快取服務例項列表，避免頻繁查詢註冊中心：

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 35s           # 快取存活時間
        capacity: 256       # 快取容量
```

## 健康檢查

可以配置 LoadBalancer 定期檢查服務例項的健康狀態：

```yaml
spring:
  cloud:
    loadbalancer:
      health-check:
        interval: 25s
        path:
          default: /actuator/health
```

## 搭配 OpenFeign 使用

OpenFeign 內建了 LoadBalancer 支援，直接使用服務名稱即可：

```java
@FeignClient(name = "service-user")
public interface UserClient {

    @GetMapping("/user/{id}")
    Map<String, Object> getUser(@PathVariable("id") Long id);
}
```

## 小結

Spring Cloud LoadBalancer 提供了簡單而強大的用戶端負載均衡能力。透過 `@LoadBalanced` 註解和內建策略，開發者可以輕鬆實現微服務間的負載均衡，同時也支援自訂策略以滿足特殊需求。
