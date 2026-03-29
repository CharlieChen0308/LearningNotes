# 02 Docker 容器化部署

> **版本**：Docker 24+ / Spring Boot 3.x / Java 17+

Docker 是當今最主流的容器化技術，讓開發者能夠將應用程式與其相依環境打包成一個可攜帶的單元，實現「一次建置，到處運行」。本篇將從核心概念出發，逐步介紹 Dockerfile 撰寫、多階段建置、Docker Compose 編排，以及 Spring Boot 容器化的最佳實踐。

---

## 1、Docker 核心概念

Docker 的三大核心元件：

- **Image（映像檔）**：唯讀的範本，包含應用程式執行所需的一切（程式碼、Runtime、系統工具、函式庫）。映像檔採用分層架構，每一層都是前一層的增量變更。
- **Container（容器）**：映像檔的執行實例。容器是輕量、隔離的執行環境，可以啟動、停止、刪除，且彼此互不影響。
- **Registry（註冊中心）**：儲存與分發映像檔的倉庫。Docker Hub 是最常見的公開 Registry，企業也可以架設私有 Registry（如 Harbor、AWS ECR）。

### Docker 容器 vs 虛擬機比較

| 比較項目 | Docker 容器 | 虛擬機（VM） |
|---------|------------|-------------|
| 啟動速度 | 秒級 | 分鐘級 |
| 資源佔用 | 數十 MB | 數 GB |
| 隔離層級 | 程序級（共享 Host Kernel） | 系統級（獨立 Kernel） |
| 效能損耗 | 接近原生 | 約 5-15% |
| 可攜性 | 高，映像檔即可部署 | 中，需要完整 VM 映像 |
| 適用場景 | 微服務、CI/CD、快速擴縮 | 需完整 OS 隔離、異構 OS |

容器並非取代虛擬機，而是在不同場景各有優勢。實務上，許多企業在虛擬機中運行容器，兩者互補。

---

## 2、Dockerfile 撰寫

Dockerfile 是一個文字檔，定義了映像檔的建置步驟。Docker 依序執行每一條指令，每條指令產生一個新的分層。

### 2.1 基本範例

```dockerfile
# 指定基礎映像檔
FROM eclipse-temurin:17-jre

# 設定工作目錄
WORKDIR /app

# 複製 JAR 檔到容器中
COPY target/my-app.jar app.jar

# 安裝必要工具（範例）
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# 宣告容器對外的埠號
EXPOSE 8080

# 定義容器啟動時執行的命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**常用指令說明**：

| 指令 | 說明 |
|------|------|
| `FROM` | 指定基礎映像檔，所有 Dockerfile 必須以此開頭 |
| `WORKDIR` | 設定工作目錄，後續指令以此為基準路徑 |
| `COPY` | 從建置上下文複製檔案到映像檔中 |
| `RUN` | 在建置階段執行命令（安裝套件、編譯等） |
| `EXPOSE` | 宣告容器監聽的埠號（文件用途，不會自動開啟） |
| `ENTRYPOINT` | 容器啟動時執行的命令，不易被覆寫 |
| `CMD` | 容器啟動時的預設參數，可被 `docker run` 覆寫 |
| `ENV` | 設定環境變數 |
| `ARG` | 定義建置時的參數 |

### 2.2 Spring Boot 最佳化 Dockerfile

針對 Spring Boot 專案，推薦使用多階段建置（下一節詳述），將編譯與執行環境分離：

- **建置階段**：使用 `eclipse-temurin:17-jdk`，包含完整 JDK 與建置工具
- **執行階段**：使用 `eclipse-temurin:17-jre`，僅包含 JRE，映像檔更小更安全

### 2.3 .dockerignore

類似 `.gitignore`，用來排除不需要送入建置上下文的檔案，加快建置速度並避免洩漏敏感資訊：

```
.git
.gitignore
.idea
*.md
build/
target/
node_modules/
.env
*.log
```

---

## 3、多階段建置（Multi-stage Build）

多階段建置是 Docker 最重要的最佳化技巧之一。它允許在同一個 Dockerfile 中使用多個 `FROM` 指令，每個 `FROM` 開啟一個新的建置階段。最終映像檔只保留最後一個階段的內容，大幅縮小映像檔體積。

### Spring Boot + Gradle 完整範例

```dockerfile
# ===== 第一階段：建置 =====
FROM eclipse-temurin:17-jdk AS build
WORKDIR /app

# 先複製 Gradle 相關檔案，利用 Docker 快取機制
COPY gradle/ gradle/
COPY gradlew build.gradle settings.gradle ./
RUN ./gradlew dependencies --no-daemon

# 複製原始碼並編譯
COPY src/ src/
RUN ./gradlew bootJar --no-daemon

# ===== 第二階段：執行 =====
FROM eclipse-temurin:17-jre
WORKDIR /app

# 從建置階段複製產出的 JAR
COPY --from=build /app/build/libs/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**為什麼要多階段建置？**

| 項目 | 單階段建置 | 多階段建置 |
|------|----------|----------|
| 映像檔大小 | ~500MB（含 JDK + 原始碼） | ~200MB（僅 JRE + JAR） |
| 安全性 | 包含建置工具、原始碼 | 僅包含執行所需檔案 |
| 建置快取 | 較差 | 利用分層快取加速 |

上面範例中，先複製 `gradlew` 和 `build.gradle` 並下載相依套件，再複製原始碼。這樣當只修改程式碼時，相依套件的快取層不會失效，大幅加速重複建置。

### Maven 版本對照

```dockerfile
FROM eclipse-temurin:17-jdk AS build
WORKDIR /app
COPY pom.xml mvnw ./
COPY .mvn/ .mvn/
RUN ./mvnw dependency:resolve --no-transfer-progress
COPY src/ src/
RUN ./mvnw package -DskipTests --no-transfer-progress

FROM eclipse-temurin:17-jre
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 4、Docker Compose

Docker Compose 用於定義和管理多容器應用程式。透過一個 `docker-compose.yml` 檔案描述所有服務及其關聯，一條命令即可啟動整個環境。

### Spring Boot + PostgreSQL + Redis 完整範例

```yaml
version: "3.9"

services:
  # ===== Spring Boot 應用程式 =====
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: my-spring-app
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydb
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD:-secret}
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PORT: 6379
      JAVA_OPTS: "-XX:MaxRAMPercentage=75.0"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

  # ===== PostgreSQL =====
  db:
    image: postgres:15-alpine
    container_name: my-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

  # ===== Redis =====
  redis:
    image: redis:7-alpine
    container_name: my-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  backend:
    driver: bridge
```

### 關鍵設定說明

**environment（環境變數）**：透過環境變數注入設定，避免將敏感資訊寫死在映像檔中。`${DB_PASSWORD:-secret}` 語法會優先讀取主機的環境變數，找不到時使用預設值。

**volumes（資料卷）**：將資料持久化到主機，即使容器刪除資料也不會遺失。`postgres_data` 是具名資料卷，由 Docker 管理；`./init-scripts` 是綁定掛載，將主機目錄映射到容器。

**networks（網路）**：同一網路內的容器可透過服務名稱互相存取（如 `db:5432`），無需使用 IP 位址。

**depends_on + healthcheck**：`depends_on` 單獨使用只保證啟動順序，搭配 `condition: service_healthy` 才能確保相依服務真正就緒後再啟動。

---

## 5、常用指令速查

### 映像檔與容器操作

```bash
# 建置映像檔（-t 指定標籤，. 表示當前目錄為建置上下文）
docker build -t my-app:1.0 .

# 執行容器（-d 背景執行，-p 埠號映射，--name 指定名稱）
docker run -d -p 8080:8080 --name my-app my-app:1.0

# 列出執行中的容器（-a 包含已停止的）
docker ps
docker ps -a

# 查看容器日誌（-f 即時追蹤）
docker logs -f my-app

# 進入容器內部執行命令
docker exec -it my-app /bin/bash

# 停止容器
docker stop my-app

# 刪除容器
docker rm my-app

# 刪除映像檔
docker rmi my-app:1.0

# 清理無用資源（已停止容器、懸空映像檔、未使用網路）
docker system prune
```

### Docker Compose 操作

```bash
# 啟動所有服務（-d 背景執行，--build 強制重新建置）
docker compose up -d
docker compose up -d --build

# 停止並移除所有容器、網路
docker compose down

# 停止並移除，包含資料卷（注意：會刪除持久化資料）
docker compose down -v

# 查看所有服務日誌
docker compose logs -f

# 查看特定服務日誌
docker compose logs -f app

# 重啟特定服務
docker compose restart app

# 查看服務狀態
docker compose ps
```

---

## 6、Spring Boot 容器化最佳實踐

### 6.1 使用非 root 使用者

容器預設以 root 執行，存在安全風險。建議建立專用使用者：

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app

# 建立非 root 使用者
RUN groupadd -r appuser && useradd -r -g appuser appuser

COPY --from=build /app/build/libs/*.jar app.jar

# 調整檔案權限後切換使用者
RUN chown appuser:appuser app.jar
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 6.2 JVM 記憶體設定

容器環境下，JVM 不應使用固定的 `-Xmx` 值，而應使用百分比參數讓 JVM 自動感知容器的記憶體限制：

```dockerfile
ENTRYPOINT ["java", \
    "-XX:MaxRAMPercentage=75.0", \
    "-XX:InitialRAMPercentage=50.0", \
    "-XX:+UseG1GC", \
    "-jar", "app.jar"]
```

在 `docker-compose.yml` 中也可以限制容器記憶體：

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          memory: 512M
```

`-XX:MaxRAMPercentage=75.0` 表示 JVM 最多使用容器記憶體的 75%，剩餘留給 JVM 的非堆積記憶體與作業系統。

### 6.3 健康檢查端點

Spring Boot Actuator 提供 `/actuator/health` 端點，可搭配 Docker 的 `HEALTHCHECK` 指令：

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1
```

確保 `application.yml` 已啟用 Actuator：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info
  endpoint:
    health:
      show-details: always
```

### 6.4 分層提取（Layer Extraction）

Spring Boot 2.3+ 支援將 JAR 拆分為多個層，讓 Docker 更好地利用快取。當只修改業務邏輯時，相依套件的層不需要重新建置：

```dockerfile
# ===== 第一階段：建置 =====
FROM eclipse-temurin:17-jdk AS build
WORKDIR /app
COPY . .
RUN ./gradlew bootJar --no-daemon

# ===== 第二階段：提取分層 =====
FROM eclipse-temurin:17-jdk AS extract
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# ===== 第三階段：執行 =====
FROM eclipse-temurin:17-jre
WORKDIR /app

RUN groupadd -r appuser && useradd -r -g appuser appuser

# 按變更頻率由低到高複製各層（最大化快取命中率）
COPY --from=extract /app/dependencies/ ./
COPY --from=extract /app/spring-boot-loader/ ./
COPY --from=extract /app/snapshot-dependencies/ ./
COPY --from=extract /app/application/ ./

RUN chown -R appuser:appuser /app
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

分層順序的設計邏輯：`dependencies`（第三方套件，幾乎不變）> `spring-boot-loader`（Spring Boot 啟動器）> `snapshot-dependencies`（快照版套件）> `application`（業務程式碼，最常變更）。

---

## 7、小結

本篇涵蓋了 Docker 容器化部署的核心知識：

- **核心概念**：Image、Container、Registry 的關係與運作機制
- **Dockerfile**：從基本指令到多階段建置的漸進式學習
- **Docker Compose**：多容器應用的編排與管理
- **Spring Boot 最佳實踐**：非 root 使用者、JVM 記憶體調校、健康檢查、分層建置

掌握 Docker 後，下一步可以進一步學習：

- **Kubernetes（K8s）**：容器編排平台，管理大規模容器叢集的部署、擴縮與自我修復
- **CI/CD 整合**：將 Docker 建置整合到 GitHub Actions、GitLab CI 等持續整合流程中，實現自動化建置與部署

> **延伸閱讀**：
> - [03 Kubernetes 入門](03%20Kubernetes%20入門.md) — 容器編排與叢集管理的下一步
> - [01 Gradle 建置工具](01%20Gradle%20建置工具.md) — 搭配 Gradle 建置 Spring Boot JAR 供 Docker 打包
