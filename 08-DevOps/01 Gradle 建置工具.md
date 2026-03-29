# 01 Gradle 建置工具

> **版本**：Gradle 8.x / Spring Boot 3.x / Java 17+

Gradle 是當今 JVM 生態系最主流的建置工具之一，結合了 Maven 的依賴管理能力與 Ant 的彈性，並以 Kotlin DSL 提供類型安全的建置腳本。本篇將從 Gradle 與 Maven 的比較出發，逐步介紹 `build.gradle.kts` 的撰寫、多模組專案配置、Spring Boot Plugin、依賴管理，以及常用 Plugin 的整合。

---

## 1、Gradle vs Maven 比較

| 比較項目 | Gradle | Maven |
|---------|--------|-------|
| 配置語言 | Kotlin DSL（`.kts`）或 Groovy DSL | XML（`pom.xml`） |
| 效能 | 增量建置 + Build Cache，大型專案快 2~10 倍 | 每次完整建置，無增量機制 |
| 彈性 | 可程式化（完整的程式語言） | 宣告式，擴展需撰寫 Plugin |
| 學習曲線 | 中高（需理解 DSL 與 Task 概念） | 中低（XML 結構固定，上手快） |
| IDE 支援 | IntelliJ IDEA 原生支援 | IntelliJ IDEA 原生支援 |
| 多模組專案 | 原生支援，設定靈活 | 原生支援，但設定較為冗長 |
| 依賴管理 | 支援 Maven Central、自定義 Repository | 以 Maven Central 為主 |
| Spring Boot 支援 | 官方 Plugin（`spring-boot`） | 官方 Plugin（`spring-boot-maven-plugin`） |
| 社群採用 | Android 官方指定、Spring 生態系主流 | Java 企業專案傳統首選 |

### 為什麼選擇 Kotlin DSL？

Gradle 支援 Groovy DSL（`build.gradle`）和 Kotlin DSL（`build.gradle.kts`）兩種寫法。推薦使用 Kotlin DSL 的原因：

- **類型安全**：編譯期檢查錯誤，而非執行期才發現
- **IDE 自動補全**：IntelliJ IDEA 對 `.kts` 提供完整的程式碼補全、跳轉定義、重構支援
- **與專案語言一致**：Spring Boot + Kotlin 專案無需額外學習 Groovy
- **官方推薦**：Gradle 官方文件已將 Kotlin DSL 作為首選範例

---

## 2、build.gradle.kts 基礎

### 完整範例

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.4.3"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "com.tiremaster"
version = "1.0.0"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

repositories {
    mavenCentral()
    // 如需私有 Repository
    // maven { url = uri("https://nexus.example.com/repository/maven-releases/") }
}

dependencies {
    // ===== 核心依賴 =====
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-security")

    // ===== 資料庫 =====
    runtimeOnly("org.postgresql:postgresql")

    // ===== 編譯期工具 =====
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
    annotationProcessor("org.mapstruct:mapstruct-processor:1.6.3")
    implementation("org.mapstruct:mapstruct:1.6.3")

    // ===== 開發工具 =====
    developmentOnly("org.springframework.boot:spring-boot-devtools")

    // ===== 測試 =====
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### 依賴範圍（Configuration）說明

| Configuration | 說明 | 等同 Maven Scope |
|---------------|------|-----------------|
| `implementation` | 編譯與執行皆需要，不傳遞給消費者 | `compile`（但不傳遞） |
| `api` | 編譯與執行皆需要，傳遞給消費者（需 `java-library` plugin） | `compile` |
| `compileOnly` | 只在編譯期需要，不打包進產出 | `provided` |
| `runtimeOnly` | 只在執行期需要，編譯期不可見 | `runtime` |
| `annotationProcessor` | 註解處理器，編譯期使用 | Maven `annotationProcessorPaths` |
| `testImplementation` | 測試編譯與執行 | `test` |
| `testRuntimeOnly` | 只在測試執行期需要 | `test`（runtime） |
| `developmentOnly` | Spring Boot 開發時專用，不打包進 JAR | 無對應 |

### plugins block 說明

```kotlin
plugins {
    java                    // Java 編譯支援
    id("org.springframework.boot") version "3.4.3"  // Spring Boot Plugin
    id("io.spring.dependency-management") version "1.1.7"  // BOM 依賴管理
}
```

- `java`：Gradle 內建 Plugin，提供 `compileJava`、`test`、`jar` 等基礎 Task。
- `org.springframework.boot`：提供 `bootJar`、`bootRun` 等 Spring Boot 專屬 Task。
- `io.spring.dependency-management`：自動引入 Spring Boot BOM，讓 Spring 相關依賴不需要指定版本號。

---

## 3、常用指令

### 基本操作

```bash
# 完整建置（編譯 + 測試 + 打包）
./gradlew build

# 清除建置產出
./gradlew clean

# 清除後重新建置
./gradlew clean build

# 執行測試
./gradlew test

# 啟動 Spring Boot 應用（開發模式）
./gradlew bootRun

# 打包為可執行 JAR
./gradlew bootJar
```

### 效能加速參數

```bash
# 平行建置（多模組專案效果顯著）
./gradlew build --parallel

# 啟用建置快取
./gradlew build --build-cache

# 組合使用
./gradlew build --parallel --build-cache

# 跳過測試（加速建置，不建議在 CI 使用）
./gradlew build -x test

# 顯示詳細建置日誌
./gradlew build --info

# 持續建置（監聽檔案變更自動重新建置）
./gradlew build --continuous
```

### 在 `gradle.properties` 中設定預設參數

```properties
# gradle.properties
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.daemon=true
org.gradle.jvmargs=-Xmx2g -XX:+UseParallelGC
```

設定後無需每次手動加參數，所有建置自動套用。

---

## 4、多模組專案

大型專案（如 TireMaster）通常會拆分為多個模組，共用核心程式碼並各自獨立建置。

### 專案結構

```
tiremaster/
  settings.gradle.kts     # 定義子模組
  build.gradle.kts         # 根專案設定（共用配置）
  common/                  # 共用模組（Entity、DTO、工具類）
    build.gradle.kts
    src/
  store/                   # 門市後端
    build.gradle.kts
    src/
  employee/                # 員工後端
    build.gradle.kts
    src/
  account/                 # 帳務後端
    build.gradle.kts
    src/
```

### settings.gradle.kts

```kotlin
rootProject.name = "tiremaster"

include("common")
include("store")
include("employee")
include("account")
```

### 根專案 build.gradle.kts（共用配置）

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.4.3" apply false
    id("io.spring.dependency-management") version "1.1.7" apply false
}

// 所有子專案共用設定
subprojects {
    apply(plugin = "java")
    apply(plugin = "io.spring.dependency-management")

    group = "com.tiremaster"
    version = "1.0.0"

    java {
        toolchain {
            languageVersion = JavaLanguageVersion.of(17)
        }
    }

    repositories {
        mavenCentral()
    }

    dependencies {
        implementation("org.projectlombok:lombok")
        annotationProcessor("org.projectlombok:lombok")
        testImplementation("org.springframework.boot:spring-boot-starter-test")
        testRuntimeOnly("org.junit.platform:junit-platform-launcher")
    }

    tasks.withType<Test> {
        useJUnitPlatform()
    }
}
```

注意根專案的 `org.springframework.boot` Plugin 加上 `apply false`，表示只宣告版本但不套用到根專案，讓各子模組自行決定是否套用。

### 子模組 build.gradle.kts（以 store 為例）

```kotlin
plugins {
    id("org.springframework.boot")
}

dependencies {
    // 依賴共用模組
    implementation(project(":common"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")
}
```

### common 模組

共用模組不需要 Spring Boot Plugin（不產生可執行 JAR），但需要停用 `bootJar` 並啟用普通 `jar`：

```kotlin
plugins {
    id("org.springframework.boot") apply false
}

// common 模組只需產生普通 JAR
tasks.named<Jar>("jar") {
    enabled = true
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

### buildSrc 共享配置（進階）

當多個子模組有相同的 Plugin 組合時，可以使用 `buildSrc` 建立自訂 Convention Plugin：

```
tiremaster/
  buildSrc/
    build.gradle.kts
    src/main/kotlin/
      spring-boot-conventions.gradle.kts
```

```kotlin
// buildSrc/build.gradle.kts
plugins {
    `kotlin-dsl`
}

repositories {
    gradlePluginPortal()
}
```

```kotlin
// buildSrc/src/main/kotlin/spring-boot-conventions.gradle.kts
plugins {
    java
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

// 共用的 Spring Boot 子模組設定
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

子模組只需一行即可套用：

```kotlin
// store/build.gradle.kts
plugins {
    id("spring-boot-conventions")
}
```

---

## 5、Spring Boot Plugin

### 核心 Task

| Task | 說明 |
|------|------|
| `bootRun` | 啟動 Spring Boot 應用程式（開發模式） |
| `bootJar` | 打包為可執行的 Fat JAR（包含所有依賴） |
| `bootBuildImage` | 使用 Cloud Native Buildpacks 建置 Docker 映像檔 |

### bootJar 設定

```kotlin
tasks.named<org.springframework.boot.gradle.tasks.bundling.BootJar>("bootJar") {
    archiveBaseName.set("tiremaster-store")
    archiveVersion.set("1.0.0")
    // 產出：tiremaster-store-1.0.0.jar
}
```

### bootRun 設定

```kotlin
tasks.named<org.springframework.boot.gradle.tasks.run.BootRun>("bootRun") {
    // 傳遞 JVM 參數
    jvmArgs = listOf(
        "-Xmx512m",
        "-Dspring.profiles.active=dev"
    )

    // 傳遞應用程式參數
    args = listOf("--server.port=8091")
}
```

### 配置 mainClass

如果專案中有多個 `main` 方法，需要明確指定：

```kotlin
springBoot {
    mainClass = "com.tiremaster.store.StoreApplication"
}
```

### Layer Tools for Docker

Spring Boot 的分層 JAR 可以搭配 Docker 多階段建置，最大化快取命中率：

```kotlin
tasks.named<org.springframework.boot.gradle.tasks.bundling.BootJar>("bootJar") {
    layered {
        enabled = true
    }
}
```

在 Dockerfile 中提取分層：

```dockerfile
FROM eclipse-temurin:17-jdk AS extract
WORKDIR /app
COPY build/libs/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=extract /app/dependencies/ ./
COPY --from=extract /app/spring-boot-loader/ ./
COPY --from=extract /app/snapshot-dependencies/ ./
COPY --from=extract /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## 6、依賴管理

### Spring Boot BOM

`io.spring.dependency-management` Plugin 會自動引入 Spring Boot 的 BOM（Bill of Materials），統一管理所有 Spring 相關依賴的版本。因此 Spring 相關依賴不需要手動指定版本：

```kotlin
dependencies {
    // 不需要寫版本號，由 BOM 統一管理
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
}
```

### Version Catalog（libs.versions.toml）

Gradle 7.0+ 引入的 Version Catalog 是管理非 Spring 依賴版本的最佳方式。將所有版本號集中在 `gradle/libs.versions.toml` 檔案中：

```toml
# gradle/libs.versions.toml

[versions]
mapstruct = "1.6.3"
querydsl = "5.1.0"
jwt = "0.12.6"
poi = "5.3.0"

[libraries]
mapstruct = { module = "org.mapstruct:mapstruct", version.ref = "mapstruct" }
mapstruct-processor = { module = "org.mapstruct:mapstruct-processor", version.ref = "mapstruct" }
querydsl-jpa = { module = "com.querydsl:querydsl-jpa", version.ref = "querydsl" }
querydsl-apt = { module = "com.querydsl:querydsl-apt", version.ref = "querydsl" }
jwt-api = { module = "io.jsonwebtoken:jjwt-api", version.ref = "jwt" }
jwt-impl = { module = "io.jsonwebtoken:jjwt-impl", version.ref = "jwt" }
jwt-jackson = { module = "io.jsonwebtoken:jjwt-jackson", version.ref = "jwt" }
poi-ooxml = { module = "org.apache.poi:poi-ooxml", version.ref = "poi" }

[bundles]
jwt = ["jwt-api", "jwt-impl", "jwt-jackson"]

[plugins]
spring-boot = { id = "org.springframework.boot", version = "3.4.3" }
dependency-management = { id = "io.spring.dependency-management", version = "1.1.7" }
```

在 `build.gradle.kts` 中使用：

```kotlin
plugins {
    java
    alias(libs.plugins.spring.boot)
    alias(libs.plugins.dependency.management)
}

dependencies {
    implementation(libs.mapstruct)
    annotationProcessor(libs.mapstruct.processor)

    // 使用 bundle 一次引入多個相關依賴
    implementation(libs.bundles.jwt)

    implementation(libs.poi.ooxml)
}
```

Version Catalog 的優勢：
- **單一來源**：所有版本號集中管理，避免多模組版本不一致
- **IDE 支援**：`libs.` 前綴提供完整的自動補全
- **可共享**：多專案可以共用同一份 catalog

### 依賴衝突排查

```bash
# 列出所有依賴樹
./gradlew dependencies

# 只看 runtimeClasspath 的依賴
./gradlew dependencies --configuration runtimeClasspath

# 搜尋特定依賴的來源
./gradlew dependencyInsight --dependency jackson-databind --configuration runtimeClasspath
```

輸出範例：

```
com.fasterxml.jackson.core:jackson-databind:2.17.3 (selected by rule)
   variant "runtime" [
      org.gradle.status = release (not requested)
   ]

com.fasterxml.jackson.core:jackson-databind:2.17.3
\--- org.springframework.boot:spring-boot-starter-json:3.4.3
     \--- org.springframework.boot:spring-boot-starter-web:3.4.3
```

### 強制指定版本（解決衝突）

```kotlin
configurations.all {
    resolutionStrategy {
        // 強制使用指定版本
        force("com.fasterxml.jackson.core:jackson-databind:2.17.3")

        // 遇到版本衝突時直接報錯（而非靜默選擇）
        failOnVersionConflict()
    }
}
```

---

## 7、常用 Plugin

### Lombok

Lombok 透過編譯期註解處理器自動產生 getter/setter、constructor、builder 等模板程式碼：

```kotlin
dependencies {
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    // 測試中也需要 Lombok
    testCompileOnly("org.projectlombok:lombok")
    testAnnotationProcessor("org.projectlombok:lombok")
}
```

### MapStruct

MapStruct 在編譯期產生類型安全的物件映射程式碼（Entity to DTO），效能遠優於 Runtime 反射方案：

```kotlin
dependencies {
    implementation("org.mapstruct:mapstruct:1.6.3")
    annotationProcessor("org.mapstruct:mapstruct-processor:1.6.3")

    // Lombok + MapStruct 同時使用時需要額外的 binding
    annotationProcessor("org.projectlombok:lombok-mapstruct-binding:0.2.0")
}
```

注意：`annotationProcessor` 的宣告順序很重要。Lombok 必須在 MapStruct 之前處理，否則 MapStruct 無法識別 Lombok 產生的 getter/setter。

### JaCoCo（程式碼覆蓋率）

```kotlin
plugins {
    java
    jacoco
}

jacoco {
    toolVersion = "0.8.12"
}

tasks.test {
    finalizedBy(tasks.jacocoTestReport)  // 測試完成後自動產生報告
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required = true    // CI 使用（SonarQube 等）
        html.required = true   // 人工閱讀
        csv.required = false
    }
}

// 設定覆蓋率門檻
tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.70".toBigDecimal()  // 最低 70% 覆蓋率
            }
        }
    }
}
```

執行覆蓋率檢查：

```bash
./gradlew test jacocoTestReport jacocoTestCoverageVerification
```

報告產出於 `build/reports/jacoco/test/html/index.html`。

### Spotless（程式碼格式化）

Spotless 是跨語言的程式碼格式化工具，確保團隊成員的程式碼風格一致：

```kotlin
plugins {
    id("com.diffplug.spotless") version "7.0.2"
}

spotless {
    java {
        target("src/**/*.java")

        // 移除未使用的 import
        removeUnusedImports()

        // 使用 Google Java Style
        googleJavaFormat("1.25.2")

        // 或使用 Eclipse formatter
        // eclipse().configFile("eclipse-formatter.xml")

        // 自訂規則
        trimTrailingWhitespace()
        endWithNewline()
    }

    kotlinGradle {
        target("*.gradle.kts")
        ktlint()
    }
}
```

```bash
# 檢查格式（CI 使用）
./gradlew spotlessCheck

# 自動修正格式
./gradlew spotlessApply
```

可以搭配 Git pre-commit hook，在每次提交前自動檢查格式。

---

## 8、小結

本篇涵蓋了 Gradle 建置工具的核心知識：

- **Gradle vs Maven**：Gradle 以增量建置、Build Cache 帶來顯著的效能優勢，Kotlin DSL 提供類型安全的建置腳本
- **build.gradle.kts**：plugins、repositories、dependencies 三大區塊構成建置腳本的基礎
- **多模組專案**：透過 `settings.gradle.kts` 組織子模組，使用 `subprojects` 或 `buildSrc` 共享設定
- **Spring Boot Plugin**：`bootJar` 打包可執行 JAR、`bootRun` 開發模式啟動、Layer Tools 搭配 Docker
- **依賴管理**：Spring Boot BOM 管理 Spring 生態版本、Version Catalog 統一管理第三方版本
- **品質工具**：JaCoCo 追蹤覆蓋率、Spotless 統一程式碼風格

延伸學習資源：

- [Gradle 官方文件](https://docs.gradle.org/)
- [Spring Boot Gradle Plugin 文件](https://docs.spring.io/spring-boot/gradle-plugin/)
- [Gradle Kotlin DSL Primer](https://docs.gradle.org/current/userguide/kotlin_dsl.html)
- [Version Catalog 官方指南](https://docs.gradle.org/current/userguide/platforms.html#sub:version-catalog)

> **延伸閱讀**：
> - [02 Docker 容器化部署](02%20Docker%20容器化部署.md) — 將 Gradle 建置產出打包為 Docker 映像檔
> - [06 CI／CD 流程（GitHub Actions）](06%20CI／CD%20流程（GitHub%20Actions）.md) — 在 CI 管線中整合 Gradle 建置與測試
