# 03 Kubernetes 入門

> **版本**：Kubernetes 1.28+ / Spring Boot 3.x

Kubernetes（簡稱 K8s）是 Google 開源的容器編排平台，用於自動化部署、擴縮容、負載均衡與自我修復。當應用從「一台機器跑幾個容器」成長為「多台機器跑數十甚至數百個容器」時，Docker Compose 已不足以應付，K8s 就是這個階段的解決方案。

---

## 1、K8s 核心概念

### 架構總覽

K8s 叢集由兩種角色組成：

- **Control Plane（控制平面）**：負責整個叢集的決策（排程、偵測故障、回應 API 請求）。核心元件包括 API Server、etcd（叢集狀態資料庫）、Scheduler、Controller Manager。
- **Worker Node（工作節點）**：實際運行應用程式容器的機器。每個 Node 上運行 kubelet（接收指令、管理 Pod）和 kube-proxy（處理網路規則）。

### 核心術語

| 概念 | 說明 |
|------|------|
| **Cluster** | 一組 Node 的集合，由 Control Plane 統一管理 |
| **Node** | 叢集中的一台機器（實體機或虛擬機） |
| **Pod** | K8s 最小的部署單位，包含一個或多個容器 |
| **Service** | 為一組 Pod 提供穩定的網路存取端點 |
| **Deployment** | 宣告式管理 Pod 的副本數量、更新策略 |
| **Namespace** | 邏輯隔離，用於區分不同環境或團隊 |

### K8s vs Docker Compose 比較

| 比較項目 | Docker Compose | Kubernetes |
|---------|---------------|------------|
| 定位 | 單機多容器編排 | 多機容器編排平台 |
| 擴縮容 | 手動 `scale` 指令 | 自動水平擴縮（HPA） |
| 自我修復 | 僅 `restart: always` | Pod 故障自動重建、重新排程 |
| 負載均衡 | 需額外設定（nginx） | Service 內建負載均衡 |
| 滾動更新 | 不支援 | 原生支援，零停機部署 |
| 設定管理 | `.env` 檔案 | ConfigMap / Secret |
| 適用場景 | 本地開發、小型部署 | 正式環境、大規模部署 |

Docker Compose 適合本地開發和小型專案；當需要高可用、自動擴縮、滾動更新時，就該考慮 K8s。

---

## 2、核心資源

K8s 的一切都是「資源」，透過 YAML 檔案宣告期望狀態，K8s 負責將實際狀態調整到與期望一致。

### 2.1 Pod

Pod 是 K8s 最小的部署單位。一個 Pod 包含一個或多個共享網路和儲存的容器。

```yaml
# pod-example.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-spring-app
  labels:
    app: spring-app
spec:
  containers:
    - name: app
      image: my-spring-app:1.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"
          cpu: "500m"
```

**多容器 Pod 的使用時機**：當兩個容器必須緊密協作時才放在同一個 Pod，例如：

- **Sidecar 模式**：主容器 + 日誌收集容器（Fluentd）
- **Ambassador 模式**：主容器 + 代理容器（處理外部連線）
- **Adapter 模式**：主容器 + 資料格式轉換容器

一般情況下，一個 Pod 只放一個應用容器。

### 2.2 Deployment

實務上幾乎不會直接建立 Pod，而是透過 Deployment 來管理。Deployment 負責維持指定數量的 Pod 副本，並支援滾動更新與回滾。

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
  labels:
    app: spring-app
spec:
  replicas: 3                    # 維持 3 個 Pod 副本
  selector:
    matchLabels:
      app: spring-app
  strategy:
    type: RollingUpdate          # 滾動更新策略
    rollingUpdate:
      maxSurge: 1                # 更新時最多多出 1 個 Pod
      maxUnavailable: 0          # 更新時不允許 Pod 不可用
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: app
          image: my-spring-app:1.0
          ports:
            - containerPort: 8080
```

**滾動更新流程**：修改映像檔版本後執行 `kubectl apply`，K8s 會逐步建立新 Pod、等待就緒、再終止舊 Pod，確保服務不中斷。

**回滾**：

```bash
# 查看部署歷史
kubectl rollout history deployment/spring-app

# 回滾到上一個版本
kubectl rollout undo deployment/spring-app

# 回滾到指定版本
kubectl rollout undo deployment/spring-app --to-revision=2
```

### 2.3 Service

Pod 的 IP 是短暫的（Pod 重建後 IP 會改變），Service 提供穩定的網路端點，透過 label selector 找到對應的 Pod，並自動負載均衡。

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-app-svc
spec:
  selector:
    app: spring-app              # 找到所有 label 為 app=spring-app 的 Pod
  ports:
    - port: 80                   # Service 對外的 port
      targetPort: 8080           # 轉發到 Pod 的 port
  type: ClusterIP                # 預設類型
```

**Service 類型**：

| 類型 | 說明 | 適用場景 |
|------|------|---------|
| **ClusterIP** | 僅叢集內部可存取（預設） | 內部微服務溝通 |
| **NodePort** | 在每個 Node 開放指定埠號（30000-32767） | 開發測試、簡單對外存取 |
| **LoadBalancer** | 向雲端供應商申請外部負載均衡器 | 正式環境對外服務 |

叢集內部的服務發現：Pod 可透過 Service 名稱直接存取，例如 `http://spring-app-svc:80`。K8s 的 DNS 會自動解析 Service 名稱為 ClusterIP。

### 2.4 ConfigMap / Secret

將設定與機密資訊從映像檔中分離，方便管理不同環境的設定。

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-app-config
data:
  SPRING_PROFILES_ACTIVE: "production"
  SERVER_PORT: "8080"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres-svc:5432/mydb"
```

```yaml
# secret.yaml（值需 base64 編碼）
apiVersion: v1
kind: Secret
metadata:
  name: spring-app-secret
type: Opaque
data:
  SPRING_DATASOURCE_USERNAME: cG9zdGdyZXM=     # echo -n 'postgres' | base64
  SPRING_DATASOURCE_PASSWORD: c2VjcmV0MTIz     # echo -n 'secret123' | base64
```

**ConfigMap 與 Secret 的差異**：Secret 的值以 base64 編碼儲存（注意：base64 不是加密，正式環境應搭配 RBAC 與外部密鑰管理工具如 Vault）。

### 2.5 Ingress

Ingress 定義了從叢集外部到內部 Service 的 HTTP/HTTPS 路由規則，取代為每個服務建立 LoadBalancer 的方式。

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /store
            pathType: Prefix
            backend:
              service:
                name: store-svc
                port:
                  number: 80
          - path: /auth
            pathType: Prefix
            backend:
              service:
                name: auth-svc
                port:
                  number: 80
  tls:
    - hosts:
        - api.example.com
      secretName: tls-secret
```

Ingress 本身只是規則定義，需要搭配 **Ingress Controller**（如 nginx-ingress-controller）才能生效。Ingress Controller 是實際處理流量的元件，負責讀取 Ingress 規則並設定反向代理。

---

## 3、Spring Boot 部署到 K8s

### 完整 Deployment + Service YAML

```yaml
# spring-app-k8s.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: app
          image: ghcr.io/myorg/spring-app:1.0.0
          ports:
            - containerPort: 8080

          # ===== 環境變數（來自 ConfigMap 和 Secret）=====
          envFrom:
            - configMapRef:
                name: spring-app-config
            - secretRef:
                name: spring-app-secret

          # ===== 健康檢查 =====
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30     # 容器啟動後等待 30 秒才開始探測
            periodSeconds: 10           # 每 10 秒探測一次
            failureThreshold: 3         # 連續失敗 3 次視為不健康，重啟容器
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
            failureThreshold: 3         # 連續失敗 3 次，從 Service 移除此 Pod

          # ===== 資源限制 =====
          resources:
            requests:                   # 排程用：K8s 保證至少分配這些資源
              memory: "256Mi"
              cpu: "250m"
            limits:                     # 上限：超過 memory limit 會被 OOMKilled
              memory: "512Mi"
              cpu: "500m"

---
apiVersion: v1
kind: Service
metadata:
  name: spring-app-svc
  namespace: production
spec:
  selector:
    app: spring-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### 健康檢查說明

Spring Boot Actuator 在 2.3+ 提供了獨立的 liveness 和 readiness 端點：

| 探測類型 | 端點 | 用途 |
|---------|------|------|
| **livenessProbe** | `/actuator/health/liveness` | 容器是否存活。失敗時 K8s 重啟容器 |
| **readinessProbe** | `/actuator/health/readiness` | 容器是否準備好接收流量。失敗時從 Service 移除 |

在 `application.yml` 中啟用：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info
  endpoint:
    health:
      probes:
        enabled: true        # 啟用 liveness/readiness 端點
      show-details: always
```

### 資源限制建議

- **requests**：設定為應用正常運行所需的最低資源。K8s 排程器根據 requests 決定將 Pod 放到哪個 Node。
- **limits**：設定為應用能容忍的最大資源。CPU 超過 limit 會被限流（throttle），記憶體超過 limit 會被強制終止（OOMKilled）。
- **JVM 記憶體**：搭配 `-XX:MaxRAMPercentage=75.0`，讓 JVM 自動感知容器的記憶體限制。

---

## 4、常用指令速查

```bash
# ===== 查詢資源 =====
kubectl get pods                              # 列出所有 Pod
kubectl get pods -n production                # 指定 namespace
kubectl get pods -o wide                      # 顯示更多資訊（IP、Node）
kubectl get deployments                       # 列出所有 Deployment
kubectl get services                          # 列出所有 Service
kubectl get all                               # 列出所有資源

# ===== 詳細資訊 =====
kubectl describe pod <pod-name>               # Pod 詳細資訊（事件、狀態）
kubectl describe deployment <deployment-name> # Deployment 詳細資訊

# ===== 日誌 =====
kubectl logs <pod-name>                       # 查看 Pod 日誌
kubectl logs <pod-name> -f                    # 即時追蹤日誌
kubectl logs <pod-name> --previous            # 查看上一個容器的日誌（容器重啟後）
kubectl logs <pod-name> -c <container-name>   # 多容器 Pod 指定容器

# ===== 部署與刪除 =====
kubectl apply -f deployment.yaml              # 建立或更新資源（宣告式，推薦）
kubectl delete -f deployment.yaml             # 刪除 YAML 中定義的資源
kubectl delete pod <pod-name>                 # 刪除特定 Pod（Deployment 會自動重建）

# ===== 偵錯 =====
kubectl exec -it <pod-name> -- /bin/bash      # 進入 Pod 內部
kubectl port-forward <pod-name> 8080:8080     # 將本地 8080 轉發到 Pod 的 8080
kubectl port-forward svc/spring-app-svc 8080:80  # 轉發到 Service

# ===== 擴縮容 =====
kubectl scale deployment spring-app --replicas=5  # 手動調整副本數
```

---

## 5、Spring Cloud Kubernetes

傳統的 Spring Cloud 微服務使用 Spring Cloud Config Server 作為集中式設定管理、Eureka 作為服務發現。當部署到 K8s 後，這些功能可以由 K8s 原生機制取代：

| Spring Cloud 元件 | K8s 原生替代 |
|-------------------|-------------|
| Spring Cloud Config | ConfigMap / Secret |
| Eureka（服務發現） | K8s Service + DNS |
| Ribbon（負載均衡） | K8s Service 內建負載均衡 |

**Spring Cloud Kubernetes** 是一個橋接專案，讓 Spring Boot 直接讀取 K8s 的 ConfigMap 和 Secret 作為 Spring 設定來源：

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  cloud:
    kubernetes:
      config:
        name: spring-app-config    # 對應 ConfigMap 名稱
        enabled: true
      secrets:
        name: spring-app-secret
        enabled: true
```

修改 ConfigMap 後，Spring Cloud Kubernetes 可以自動偵測變更並重新載入設定（搭配 `@RefreshScope`），無需重啟 Pod。

對於新專案，建議直接使用 K8s 原生的 `envFrom` 注入環境變數，簡單且不增加額外依賴。Spring Cloud Kubernetes 適合已有 Spring Cloud 架構、需要動態設定更新的場景。

---

## 6、小結

本篇涵蓋了 Kubernetes 入門所需的核心知識：

- **核心概念**：Cluster、Node、Pod、Service、Deployment 的關係
- **核心資源**：Pod、Deployment（滾動更新）、Service（服務發現）、ConfigMap/Secret、Ingress
- **Spring Boot 部署**：完整 YAML、健康檢查（liveness/readiness）、資源限制
- **常用指令**：kubectl 的日常操作速查
- **Spring Cloud Kubernetes**：用 K8s 原生機制取代 Spring Cloud 元件

延伸閱讀：

- **Docker 容器化部署**：`08-DevOps/02 Docker 容器化部署.md`
