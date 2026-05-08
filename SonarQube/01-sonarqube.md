# SonarQube

SonarQube 是一套**靜態程式碼分析**平台，能掃描程式碼並找出 Bug、安全漏洞與不良寫法。整合進 CI/CD 之後，每次 push 就會自動掃一次，讓問題在進主線之前就被攔下來。本文聚焦在 SonarQube 本身（介紹、部署、設定檔、Quality Gate）；如何把它接進 GitLab CI/CD pipeline 留在[第七章](../GitLab%20CICD/Gitlab-cicd-note-github/07-sonarqube.md)。

## 目錄

- [SonarQube](#sonarqube)
  - [目錄](#目錄)
  - [SonarQube 簡介](#sonarqube-簡介)
    - [核心概念](#核心概念)
    - [SonarQube vs SonarCloud](#sonarqube-vs-sonarcloud)
    - [Community Build vs Developer Edition](#community-build-vs-developer-edition)
    - [Community Build 在分支上的實務行為](#community-build-在分支上的實務行為)
  - [運作架構](#運作架構)
  - [部署 SonarQube：Docker Compose + PostgreSQL](#部署-sonarqubedocker-compose--postgresql)
    - [Linux Host 預先設定](#linux-host-預先設定)
    - [docker-compose.yml 範例](#docker-composeyml-範例)
    - [啟動與第一次登入](#啟動與第一次登入)
    - [升級流程](#升級流程)
  - [Scanner 介紹](#scanner-介紹)
  - [設定檔總覽](#設定檔總覽)
    - [Server 端：sonar.properties](#server-端sonarproperties)
    - [Scanner 端：sonar-project.properties](#scanner-端sonar-projectproperties)
    - [Scanner 端：Maven / Gradle 設定](#scanner-端maven--gradle-設定)
    - [Scanner 端：sonar-scanner.properties](#scanner-端sonar-scannerproperties)
  - [Quality Gate 入門](#quality-gate-入門)
  - [常見維運提醒](#常見維運提醒)
  - [小結](#小結)

---

## SonarQube 簡介

### 核心概念

| 名詞 | 說明 |
|------|------|
| **Bug** | 會導致程式執行錯誤或不預期行為的程式碼缺陷 |
| **Vulnerability** | 安全漏洞，例如 SQL injection、未驗證的輸入等，需要立即修復 |
| **Security Hotspot** | 安全敏感的程式碼段，需要人工判斷是否構成真正的風險 |
| **Code Smell** | 不影響功能、但影響可讀性與可維護性的壞味道，例如過度複雜的函式、重複的程式碼 |
| **Coverage** | 測試涵蓋率，代表有多少程式碼被自動化測試涵蓋（要靠專案另外產 coverage report 給 SonarQube 讀） |
| **Quality Gate** | 一組品質門檻條件，例如「新程式碼涵蓋率不得低於 80%」、「不得有未檢視的 Security Hotspot」，全部通過才算過關 |

其中 **Quality Gate** 是最關鍵的概念，它是 CI 的最後一道關卡——只要設定好門檻，掃描結果不達標就讓 Pipeline 失敗，阻止問題程式碼合入主線。

### SonarQube vs SonarCloud

* **SonarQube**：自架版本，自己準備 Server、管理資料庫與升級。資料留在內部，適合企業內網環境。

* **SonarCloud**：SaaS 版本，由 SonarSource 維護基礎設施。對開源專案免費，付費方案才能分析私有專案。

兩者底層分析引擎相同，差別主要在「自架 vs 雲端」。本文以自架的 SonarQube 為主。

### Community Build vs Developer Edition

| 功能 | Community Build (免費) | Developer Edition (付費) |
|------|----------------------|------------------------|
| 靜態分析 | ✅ | ✅ |
| Quality Gate | ✅ | ✅ |
| 支援語言 | 20+ | 30+ |
| **分支分析** | ❌（只能分析主線） | ✅ |
| **PR 裝飾**（在 MR 介面看分析結果） | ❌ | ✅ |

> Community Build 最大的限制是**只能分析主線**（main / master），無法針對 feature branch 或 Merge Request 做個別的分析。如果需要每個 MR 都看到獨立分析結果，必須升級到 Developer Edition 或改用 SonarCloud——這項是付費功能，沒有免費的迴避方法。

### Community Build 在分支上的實務行為

「不支援分支分析」的意思**不是 scanner 不能在 feature branch 上跑**——scanner 只是一個 CLI，CI 怎麼觸發它都行，分析結果也會照樣送到 Server。但 Server 在 Community Build 模式下，同一個 `projectKey` 只保留**一份**分析結果，所以從不同分支推上來的結果會**互相覆蓋**：

* feature branch 推一次掃描 → Dashboard 顯示 feature branch 的分析結果
* main 再推一次 → Dashboard 又被覆蓋成 main 的結果

這個行為帶出兩種實務做法：

1. **只在 main 跑**（官方建議）：在 `.gitlab-ci.yml` 用 `rules:` 限制只有 `$CI_COMMIT_BRANCH == "main"` 才觸發掃描，feature branch 不掃。Quality Gate 的把關就放在 main 合入後做。

2. **每個 branch 用不同 projectKey**：CI 裡動態組 `sonar.projectKey=$CI_PROJECT_NAME-$CI_COMMIT_REF_SLUG`，硬要每條分支都看到結果。代價是 Server 上會堆出一堆專案，是 hack 不是正規做法，**不推薦**。

> 在 `.gitlab-ci.yml` 裡實際怎麼寫 `rules:` 留到[第七章](../GitLab%20CICD/Gitlab-cicd-note-github/07-sonarqube.md)。本節只是先把「能跑 vs 結果是什麼」釐清。

---

## 運作架構

SonarQube Server 啟動後會跑起四個官方文件列出的子組件（[Server components overview](https://docs.sonarsource.com/sonarqube-community-build/server-installation/server-components-overview/)）：

| 組件 | 用途 |
|------|------|
| **Web Server** | 唯一面向外部的入口。使用者的瀏覽器、IDE 插件或 CI 系統透過 HTTP 連進來，它提供整個管理 UI 以及 REST API。它本身不做繁重計算，主要是讀取 DB / ES 的資料然後呈現出來。 |
| **Compute Engine (CE)** | 專門處理 Scanner 送來的分析報告。Scanner 跑完靜態分析後會把報告打包送到 CE，由 CE 來算 code smell、bug、覆蓋率等指標，最後把結果寫進資料庫。「任務佇列」也存在 DB 裡，所以 CE 掛掉再重啟不會遺失任務 |
| **Elasticsearch (ES)** | 扮演快取索引的角色。DB 是真正的 source of truth，但搜尋（例如搜尋 issue、跨專案查詢）如果直接打 SQL 會很慢，所以 SonarQube 會把 DB 的資料同步一份到 ES，讓搜尋和部分查詢走 ES 這條快速路徑。ES 的資料可以從 DB 重建，所以它是可拋棄的副本，不是主儲存。 |
| **SonarQube Database** |  是整個系統的核心儲存，保存所有 metric 數值、issue 清單、專案設定、使用者設定，以及 CE 的任務佇列。它是唯一不可丟失的資料層。官方建議 PostgreSQL，其次是 SQL Server 或 Oracle，不建議使用 H2（H2 只適合本機試玩）。 |

整體流程：

![alt text](image.png)

注意一個時序：**Quality Gate 是在 CE 處理完之後才判定的**——scanner 把報告送出去之後，CE 還要花幾秒到幾十秒處理，這也是為什麼後面講到「讓 CI 等 Quality Gate 結果」時，要加 `sonar.qualitygate.wait=true` 讓 scanner 持續 polling 直到結果出來。

---

## 部署 SonarQube：Docker Compose + PostgreSQL

SonarQube 內建一個 H2 資料庫供評估用，不可備份、效能不足，因此僅適合用來做測試、demo。Community Build 支援的資料庫有 PostgreSQL、Oracle、Microsoft SQL Server，其中以 PostgreSQL 最常見。

### Linux Host 預先設定

SonarQube 的 Elasticsearch 子組件對 host 的 kernel 參數有硬性要求，數值依[官方 Linux 預先設定文件](https://docs.sonarsource.com/sonarqube-community-build/server-installation/pre-installation/linux/)：

**`/etc/sysctl.d/99-sonarqube.conf`**

```conf
vm.max_map_count=524288
fs.file-max=131072
```

**`/etc/security/limits.d/99-sonarqube.conf`**

```conf
sonarqube   -   nofile   131072
sonarqube   -   nproc    8192
```

設好之後執行：

```bash
sudo sysctl --system
```

讓 sysctl 立即生效，limits 則需要該使用者重新登入才會套用。

每一項是給誰用的：

* `vm.max_map_count`、`nofile`：Elasticsearch 需要大量 memory map 與 file descriptor，不夠就直接拒絕啟動。
* `nproc`：CE 與 Web Server 在處理大型分析時的並行 thread 上限。
* **Kernel 必須啟用 seccomp**，用 `grep SECCOMP /boot/config-$(uname -r)` 確認回傳的內容包含 `CONFIG_SECCOMP=y`。主流 Linux 發行版預設都有開，這項通常不用動。

> Docker Desktop（Windows / macOS）的情況不一樣——容器是跑在 Docker 內部的 Linux VM 裡，要設的是那個 VM 的參數，不是 Windows / macOS 宿主。學習用途多半不會踩到，但部署到正式環境前請務必改用 Linux server 跑。

### docker-compose.yml 範例

下面這份是 SonarSource 官方 [docker-sonarqube repo](https://github.com/SonarSource/docker-sonarqube/tree/master/example-compose-files/sq-with-postgres) 提供的範本，僅做最小調整（移除 IPv6 dual-stack，預設帳密請務必改掉）：

```yaml
services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    depends_on:
      db:
        condition: service_healthy
    read_only: true
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: admin
      SONAR_JDBC_PASSWORD: admin
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_temp:/opt/sonarqube/temp
    tmpfs:
      - /tmp:size=256M,mode=1777
    ports:
      - "9000:9000"

  db:
    image: postgres:17
    container_name: sonarqube-db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin
      POSTGRES_DB: sonar
    volumes:
      - postgresql:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  sonarqube_temp:
  postgresql:
```

幾個關鍵點：

* **必須用 named volume，不能用 bind mount**——官方文件明確說 bind mount 會讓 plugin 安裝失敗。
* **四個 SonarQube volume 各自的用途**：
  * `data`：Elasticsearch 索引與內部狀態
  * `extensions`：plugin、JDBC driver（裝在這裡才會生效）
  * `logs`：Web / CE / ES 三個 process 的 log
  * `temp`：執行時暫存
* **`read_only: true` + `tmpfs: /tmp`**：把 container root filesystem 設成唯讀、`/tmp` 用 tmpfs，是 SonarSource 官方範本的安全建議。
* **`depends_on: condition: service_healthy`**：先等 PostgreSQL 通過 `pg_isready` 健康檢查，SonarQube 才啟動。沒這個的話 SonarQube 早於 DB 起來，會在連線重試後失敗退出。
* **`SONAR_JDBC_*`** 是 Server 認得的環境變數，下一節會講它跟 `sonar.properties` 設定的對應關係。


> **⚠️ 別把 volume 不小心刪掉**
>
> 除非你打算砍掉資料重來，否則：
>
> * 不要用 `docker compose down -v`（`-v` 會把 compose 帶起來的 volume 一起刪掉）
> * 不要在這台機器隨手 `docker system prune` 或 `docker volume prune`
>
> 不論你的 compose 有沒有寫 `external: true`，只要 volume 被刪到，**SonarQube 啟動到關閉之間累積的所有資料庫內容都會消失**——分析結果、issue 歷史、Quality Gate 設定全部歸零。

### 啟動與第一次登入

```bash
docker compose up -d
docker compose logs -f sonarqube
```

看到 log 出現 `SonarQube is operational` 就是起好了。打開 `http://<host>:9000`，預設帳密：

* 帳號：`admin`
* 密碼：`admin`

> 更改後的密碼：5tgb^YHN7ujm
> 
第一次登入會強制改密碼。改完之後到 **Administration → Security → Users** 為 CI 端產生 token，後面 Scanner 章節會用到。

### 升級流程

1. `docker compose down`（**不要** `down -v`，那會連 volume 一起刪掉）
2. 修改 `docker-compose.yml` 把 `sonarqube:community` 換成新版的 image tag
3. `docker compose up -d`，DB schema 由 SonarQube 自動 migrate
4. 看 logs 確認 migration 成功

**大版本升級前一定要先備份 PostgreSQL**：

```bash
docker compose exec db pg_dump -U sonar sonar > sonar_backup_$(date +%F).sql
```

---

## Scanner 介紹

Scanner 是在 CI 端跑的 CLI，負責讀原始碼、產出分析報告、推送給 SonarQube Server。**不同 build system 對應不同 scanner**，選錯會降低分析品質：

| 專案類型 | 對應 scanner | 設定檔放在 |
|----------|--------------|-----------|
| 通用（Python / JS / Go / PHP …） | **SonarScanner CLI** | `sonar-project.properties` |
| Maven 專案 | **SonarScanner for Maven** | `pom.xml` |
| Gradle 專案 | **SonarScanner for Gradle** | `build.gradle` |
| .NET 專案 | **SonarScanner for .NET** | 命令列參數 |

> [官方文件](https://docs.sonarsource.com/sonarqube-community-build/analyzing-source-code/scanners/sonarscanner/)明確警告：「Don't use the SonarScanner CLI for projects built with Maven, Gradle, or .NET. Doing so will degrade the quality of your analysis.」原因是 CLI 拿不到 build 過程的編譯產物（例如 `.class` 檔、bytecode），分析能做到的事就少了一截。Maven / Gradle / .NET 專案請改用對應的專屬 scanner。

各個 scanner 在實際專案裡怎麼跑、`.gitlab-ci.yml` 怎麼寫，留在[第七章](../GitLab%20CICD/Gitlab-cicd-note-github/07-sonarqube.md)整章專門處理。

---

## 設定檔總覽

SonarQube 的相關設定檔分**Server 端**與 **Scanner 端**兩組：

* Server 端：給 SonarQube Server 自己讀的（資料庫連線、port、LDAP 等）
* Scanner 端：告訴 scanner 「分析哪個專案、用什麼設定」，每個 build system 各有自己的設定檔

### Server 端：sonar.properties

`$SONARQUBE_HOME/conf/sonar.properties` 是 Server 主設定檔，常見欄位：

| 設定 | 用途 |
|------|------|
| `sonar.jdbc.url` | 資料庫連線字串，例 `jdbc:postgresql://db:5432/sonar` |
| `sonar.jdbc.username` / `sonar.jdbc.password` | 資料庫帳密 |
| `sonar.web.port` | Web Server port，預設 `9000` |
| `sonar.web.context` | 部署在子路徑時用（例如 `/sonarqube`） |
| `sonar.search.javaOpts` | Elasticsearch 的 JVM 參數，大流量環境調 heap |

**Docker 部署的人不用碰這個檔案**——SonarQube image 接受用 `SONAR_` 開頭的環境變數覆蓋，命名規則是「點轉底線、整個大寫」：

| sonar.properties 欄位 | 對應環境變數 |
|----------------------|-------------|
| `sonar.jdbc.url` | `SONAR_JDBC_URL` |
| `sonar.jdbc.username` | `SONAR_JDBC_USERNAME` |
| `sonar.web.port` | `SONAR_WEB_PORT` |
| `sonar.web.context` | `SONAR_WEB_CONTEXT` |

所以前面 `docker-compose.yml` 裡寫 `SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar`，效果等同於在 `sonar.properties` 寫 `sonar.jdbc.url=jdbc:postgresql://db:5432/sonar`。

> 還有一個 `$SONARQUBE_HOME/conf/wrapper.conf` 控制 Web / CE / ES 三個子 process 的 JVM wrapper 參數，大流量環境才需要調整，一般部署不用碰。

### Scanner 端：sonar-project.properties

這是 **SonarScanner CLI** 在掃描時讀的設定檔，放在專案根目錄。Maven / Gradle 用各自的 build script，**不需要**這個檔。

**唯一必填欄位**：

* `sonar.projectKey` — 必須對得上 SonarQube Server 上的 project key

常用欄位（每個都附預設值）：

| 設定 | 預設 | 用途 |
|------|------|------|
| `sonar.projectName` | = `projectKey` | Dashboard 上顯示的專案名稱 |
| `sonar.sources` | `.` | 原始碼根目錄。實務上一定要明確指定，避免把測試、文件、build artifact 全掃進去 |
| `sonar.tests` | （無） | 測試程式碼目錄；指定後 SonarQube 才知道哪些是測試檔，分析規則會不一樣 |
| `sonar.exclusions` | （無） | glob pattern，被匹配到的檔案**完全不會被掃描**（例：`**/node_modules/**,**/dist/**`） |
| `sonar.coverage.exclusions` | （無） | 被匹配到的檔案**仍然會掃描**，但**不算進涵蓋率分母**（適合排除 generated code） |
| `sonar.sourceEncoding` | 系統預設 | 建議寫 `UTF-8` |

語言特定的欄位另外列幾個常用的：

* `sonar.python.version=3.11`（讓分析器套用對應 Python 版本的規則）
* `sonar.java.binaries=target/classes`（Java 必填，指向 build 產出）
* `sonar.javascript.lcov.reportPaths=coverage/lcov.info`（吃 lcov 格式的涵蓋率報告）

**`sonar.token` 與 `sonar.host.url` 不應該寫在這個檔**——這個檔會 commit 進 git，token 進去就等於洩漏。改用環境變數 `SONAR_TOKEN` / `SONAR_HOST_URL`，或在執行時用 `-Dsonar.token=...` 命令列參數覆蓋。

完整範例（一個 Python 專案）：

```properties
sonar.projectKey=my-python-app
sonar.projectName=My Python App

sonar.sources=src
sonar.tests=tests
sonar.sourceEncoding=UTF-8

sonar.python.version=3.11

# 不要掃 venv、build 產物、自動生成檔
sonar.exclusions=**/.venv/**,**/build/**,**/*_pb2.py

# 仍掃描但不算進涵蓋率（autogenerated 之類）
sonar.coverage.exclusions=**/migrations/**,**/*_pb2.py

# 給 SonarQube 吃 coverage 報告
sonar.python.coverage.reportPaths=coverage.xml
```

### Scanner 端：Maven / Gradle 設定

**Maven** 在 `pom.xml` 加一段 properties，掃描時直接 `mvn sonar:sonar`：

```xml
<properties>
  <sonar.projectKey>my-java-app</sonar.projectKey>
  <sonar.host.url>http://your-sonarqube:9000</sonar.host.url>
</properties>
```

token 一樣用環境變數或 `-Dsonar.token=...` 帶入，**不要**寫進 `pom.xml`。

**Gradle** 在 `build.gradle` 套 plugin：

```groovy
plugins {
  id "org.sonarqube" version "5.1.0.4882"
}

sonar {
  properties {
    property "sonar.projectKey", "my-gradle-app"
    property "sonar.host.url", "http://your-sonarqube:9000"
  }
}
```

掃描指令是 `./gradlew sonar`（搭配環境變數 `SONAR_TOKEN`）。

### Scanner 端：sonar-scanner.properties

`<scanner_install>/conf/sonar-scanner.properties` 是 SonarScanner CLI 的**全域**預設值，可以放共用設定（例如固定的 `sonar.host.url`），讓所有專案不用各自重寫。多數情境用環境變數就夠了，這個檔很少需要動。

---

## Quality Gate 入門

Quality Gate 是 SonarQube 的品質門檻——一組條件，全部通過才算 Pass，沒過就 Fail。

**Clean as You Code 哲學**：SonarQube 的預設 Quality Gate **只看 New Code**（新增或修改的程式碼），不去糾結整個 codebase 既有的技術債。這個設計讓團隊可以從現狀往前推進，而不會被遺留問題卡住——只要新寫的 code 乾淨，整體品質自然會慢慢變好。

**預設的 "Sonar way"** 內建四個條件，全部只針對 New Code（[官方說明](https://docs.sonarsource.com/sonarqube-community-build/quality-standards-administration/managing-quality-gates/introduction-to-quality-gates/)）：

1. **No new issues are introduced**（沒有新的 issue）
2. **All new Security Hotspots are reviewed**（所有新的 Security Hotspot 都已人工審視）
3. **New code test coverage is greater than or equal to 80.0%**（新程式碼涵蓋率 ≥ 80%）
4. **Duplication in the new code is less than or equal to 3.0%**（新程式碼重複率 ≤ 3%）

> 條件數值會因 SonarQube 版本略有差異，請以你 Server **Quality Gates → Sonar way** 頁面顯示的為準。

在 Web UI 上看結果：專案 Dashboard 頂端會直接顯示 **Passed** / **Failed**，點進去能看到每一條條件的實際數值與差距。

**怎麼讓 CI 因 Quality Gate 失敗而停下來**：scanner 預設不等結果就退出，要加 `-Dsonar.qualitygate.wait=true` 讓它持續 polling，Quality Gate 一旦判定為 Fail，scanner 的 exit code 就會非零，pipeline 就會跟著失敗。完整的 GitLab CI 整合（含 timeout、cache、`rules:`、`allow_failure`）請看[第七章](../GitLab%20CICD/Gitlab-cicd-note-github/07-sonarqube.md)。

---

## 常見維運提醒

* **Token 種類**：SonarQube 有三種 token，影響範圍由大到小是 User Token > Global Analysis Token > Project Analysis Token。**官方建議用 Project Analysis Token**——影響範圍最小，外洩時只能用來掃描那一個專案。
* **備份**：要備份的有兩個東西——**PostgreSQL** 用 `pg_dump`，**`sonarqube_extensions` volume** 用 `tar` 包起來（裝過的 plugin、JDBC driver 都在這裡）。`data` volume 是 Elasticsearch 索引，可由 DB 重建，不用備；`logs`、`temp` 也不用。
* **Plugin / Marketplace**：SonarQube 在 Web UI 有 Marketplace 可以裝 plugin（例如額外語言支援），檔案會放進 `extensions` volume，**重啟 Server 才會生效**。

---

## 小結

* SonarQube 是自架的靜態分析平台，由 Web Server / CE / ES / Database 四個組件組成，正式環境一定要配外部 PostgreSQL。
* Scanner 端依 build system 選對工具（CLI / Maven / Gradle / .NET），並用 `sonar-project.properties` 或 build script 設定專案參數；token 與 host URL 永遠用環境變數帶。
* Quality Gate 的預設 "Sonar way" 只看 New Code 的四個條件——這是讓 CI 真正擋下品質問題的關鍵。

下一步：[第七章 — 把 SonarQube 整合進 GitLab CI/CD pipeline](../GitLab%20CICD/Gitlab-cicd-note-github/07-sonarqube.md)。
