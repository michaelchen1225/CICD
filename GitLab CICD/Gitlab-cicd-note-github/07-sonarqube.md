# SonarQube

## 目錄

- [SonarQube](#sonarqube)
  - [目錄](#目錄)
  - [SonarQube 簡介](#sonarqube-簡介)
    - [核心概念](#核心概念)
    - [SonarQube vs SonarCloud](#sonarqube-vs-sonarcloud)
    - [Community Build vs Developer Edition](#community-build-vs-developer-edition)
  - [運作架構](#運作架構)
  - [實作](#實作)
    - [用 Docker 架設 SonarQube Server](#用-docker-架設-sonarqube-server)
    - [設定 GitLab CI/CD](#設定-gitlab-cicd)
    - [設定 sonar-project.properties](#設定-sonar-projectproperties)
    - [整合 GitLab CI/CD](#整合-gitlab-cicd)
  - [技巧整理](#技巧整理)
    - [讓 Quality Gate 卡住 Pipeline](#讓-quality-gate-卡住-pipeline)
    - [排除特定檔案或目錄](#排除特定檔案或目錄)
    - [善用 Cache 加速掃描](#善用-cache-加速掃描)
    - [完整 clone 歷史紀錄](#完整-clone-歷史紀錄)

---

## SonarQube 簡介

SonarQube 是一套**靜態程式碼分析**平台，能夠自動掃描程式碼並找出潛在的問題，例如 Bug、安全漏洞、或不良的寫法。將 SonarQube 整合進 CI/CD pipeline 後，每次 push 就會自動跑一次掃描，讓程式碼品質問題在進入主線之前就被抓出來。

### 核心概念

| 名詞 | 說明 |
|------|------|
| **Bug** | 會導致程式執行錯誤或不預期行為的程式碼缺陷 |
| **Vulnerability** | 安全漏洞，例如 SQL injection、未驗證的輸入等，需要立即修復 |
| **Security Hotspot** | 安全敏感的程式碼段，需要人工判斷是否構成真正的風險 |
| **Code Smell** | 不影響功能、但影響可讀性與可維護性的壞味道，例如過度複雜的函式、重複的程式碼 |
| **Coverage** | 測試涵蓋率，代表有多少程式碼被自動化測試涵蓋 |
| **Quality Gate** | 一組品質門檻條件，例如「新增的 Bug 必須為零」、「Coverage 不得低於 80%」，全部通過才算過關 |

其中 **Quality Gate** 是最重要的概念，它就像是 CI 的最後一道關卡——只要設定好門檻，掃描結果不達標就讓 Pipeline 失敗，阻止問題程式碼合入主線。

### SonarQube vs SonarCloud

* **SonarQube**：自架版本，需要自己準備 Server、管理資料庫與升級。適合需要資料留在內部的環境。

* **SonarCloud**：SaaS 版本，由 SonarSource 負責維護基礎設施。對開源專案免費，且 Free Plan 就包含完整的分支分析功能。

兩者底層的分析引擎相同，差別主要在於「自架 vs 雲端」。本文以自架的 SonarQube 為主。

### Community Build vs Developer Edition

| 功能 | Community Build (免費) | Developer Edition (付費) |
|------|----------------------|------------------------|
| 靜態分析 | ✅ | ✅ |
| Quality Gate | ✅ | ✅ |
| 支援語言 | 20+ | 30+ |
| **分支分析** | ❌（只能分析主線） | ✅ |
| **PR 裝飾** | ❌ | ✅ |

> Community Build 最大的限制是只能分析主線（main/master），無法針對 feature branch 或 Merge Request 做個別的分析。如果要在每個 MR 上看到掃描結果，需要升級到 Developer Edition 或改用 SonarCloud。

---

## 運作架構

```
Developer push
      │
      ▼
GitLab CI/CD Pipeline
      │  (執行 sonar-scanner)
      ▼
SonarQube Server  ◄──── sonar-project.properties (專案設定)
      │  (Compute Engine 處理結果)
      ▼
SonarQube Dashboard (Web UI)
      │  (顯示 Quality Gate 結果)
      ▼
Pipeline 通過 or 失敗
```

整個流程分三個角色：

* **sonar-scanner**：CLI 工具，在 CI 中被觸發，負責讀取原始碼並把分析結果送到 Server。

* **SonarQube Server**：接收分析結果後，由 Compute Engine 計算指標、更新資料庫，並透過 Web UI 呈現報告。

* **Quality Gate**：Server 端設定的門檻，掃描完成後由 Server 判定是否通過，結果回傳給 scanner，進而決定 Pipeline 成敗。

---

## 實作

> [Official Documentation](https://docs.sonarsource.com/sonarqube-community-build/server-installation/from-docker-image/prepare-installation)

### 用 Docker 架設 SonarQube Server

最快的方式是直接用 Docker 跑 Community Build：

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  -v sonarqube_logs:/opt/sonarqube/logs \
  sonarqube:community
```

啟動後，打開瀏覽器前往 `http://localhost:9000`，預設帳密為：

* 帳號：`admin`
* 密碼：`admin`

> 第一次登入後系統會要求更改密碼，這裡改成 `5tgb^YHN7ujm` 就好。

登入後選擇 Gitlab，他會要求輸入一些資訊：

* Configuration name：隨便取一個名字，例如 `GitLab CI`

* Gitlab api url：`https://gitlab.com/api/v4`

* Personal access token：在 GitLab 產生一個 [Personal Access Token](https://docs.gitlab.com/user/profile/personal_access_tokens/)，權限選 `api`，複製 token 貼上去。


然後選擇你想要分析的專案，右邊按下 Import，最後會要你選擇 **New Code definition**——這個設定決定 Quality Gate 把哪些程式碼當成「新程式碼」來檢查。直接選 **Follows the instance's default**（預設是 Previous version）即可，之後可以再針對個別專案調整。按下 **Create projects** 完成匯入。

### 設定 GitLab CI/CD 

接著點選你想要做分析的專案，然後選擇 Gitlab CI，他會一步步帶你設定：



### 設定 sonar-project.properties

在 GitLab repo 的根目錄建立 `sonar-project.properties`：

```properties
sonar.projectKey=my-project-key
sonar.projectName=My Project

# 原始碼位置
sonar.sources=src

# 測試程式碼位置（選填）
sonar.tests=tests

# 排除不需要掃描的路徑
sonar.exclusions=**/node_modules/**,**/dist/**,**/coverage/**
```

> `sonar.projectKey` 必須與 SonarQube Server 上建立的 Project Key 完全一致。

### 整合 GitLab CI/CD

**Step 1**：在 GitLab 的 repo 中，進入 **Settings** → **CI/CD** → **Variables**，新增以下兩個變數：

| 變數名稱 | 值 | 說明 |
|---------|---|------|
| `SONAR_HOST_URL` | `http://your-sonarqube-server:9000` | SonarQube Server 位址 |
| `SONAR_TOKEN` | （剛才產生的 token） | 分析用的認證 token，建議勾選 **Masked** |

**Step 2**：在 `.gitlab-ci.yml` 加入掃描 job：

```yaml
stages:
  - test

sonarqube-check:
  stage: test
  image: sonarsource/sonar-scanner-cli:latest
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: 0
  cache:
    key: sonar-cache-${CI_COMMIT_REF_SLUG}
    paths:
      - .sonar/cache
  script:
    - sonar-scanner
      -Dsonar.host.url=${SONAR_HOST_URL}
      -Dsonar.token=${SONAR_TOKEN}
      -Dsonar.qualitygate.wait=true
  allow_failure: false
```

push 之後，Pipeline 跑完就能在 SonarQube Dashboard 上看到分析結果。

---

## 技巧整理

### 讓 Quality Gate 卡住 Pipeline

預設情況下，`sonar-scanner` 把分析結果送出去之後就結束了，不管 Quality Gate 有沒有過，Pipeline 都會成功。

若要讓 Quality Gate 失敗時 Pipeline 也跟著失敗，加上 `-Dsonar.qualitygate.wait=true`：

```yaml
script:
  - sonar-scanner -Dsonar.qualitygate.wait=true
```

加上這個參數後，scanner 會持續 polling Server，直到 Quality Gate 有結果為止，結果失敗則 exit code 非零，Pipeline 就會跟著停掉。

> 預設 timeout 是 300 秒，可以用 `-Dsonar.qualitygate.timeout=600` 調整。

搭配 `allow_failure: false`（GitLab CI 預設值），就能確保 Pipeline 被阻斷：

```yaml
sonarqube-check:
  script:
    - sonar-scanner -Dsonar.qualitygate.wait=true
  allow_failure: false
```

### 排除特定檔案或目錄

在 `sonar-project.properties` 中用 `sonar.exclusions` 指定要忽略的路徑，支援 glob 語法：

```properties
# 排除測試檔、打包輸出、node_modules
sonar.exclusions=**/*.spec.ts,**/dist/**,**/build/**,**/node_modules/**
```

常見 glob 規則：

* `*`：匹配單層路徑中的任意字元（不含 `/`）
* `**`：匹配任意層的目錄
* `?`：匹配單一字元

如果只想排除 Coverage 報告不計入覆蓋率，可以用：

```properties
sonar.coverage.exclusions=**/tests/**,**/*.spec.ts
```

### 善用 Cache 加速掃描

sonar-scanner 每次執行時會從 Server 下載 plugin 與分析器，加上 cache 可以省下 30~60 秒：

```yaml
sonarqube-check:
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  cache:
    key: sonar-cache-${CI_COMMIT_REF_SLUG}
    paths:
      - .sonar/cache
    policy: pull-push
```

> `SONAR_USER_HOME` 要指到 `${CI_PROJECT_DIR}` 底下，cache 路徑才能被 GitLab CI 的 cache 機制管到。

### 完整 clone 歷史紀錄

GitLab CI 預設只做 shallow clone（`GIT_DEPTH: 20`），SonarQube 在分析 blame 資訊時會因為歷史不完整而產生警告，建議設成 `0` 做完整 clone：

```yaml
sonarqube-check:
  variables:
    GIT_DEPTH: 0
```
