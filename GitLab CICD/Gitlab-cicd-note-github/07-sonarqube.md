# SonarQube 與 GitLab CI/CD 整合

## 目錄

- [SonarQube 與 GitLab CI/CD 整合](#sonarqube-與-gitlab-cicd-整合)
  - [目錄](#目錄)
  - [前言](#前言)
  - [運作架構](#運作架構)
  - [整合步驟](#整合步驟)
    - [Step 1：在 SonarQube 匯入專案](#step-1在-sonarqube-匯入專案)
    - [Step 2：設定 GitLab CI/CD Variables](#step-2設定-gitlab-cicd-variables)
    - [Step 3：建立 sonar-project.properties](#step-3建立-sonar-projectproperties)
    - [Step 4：在 .gitlab-ci.yml 加入掃描 job](#step-4在-gitlab-ciyml-加入掃描-job)
  - [技巧整理](#技巧整理)
    - [讓 Quality Gate 卡住 Pipeline](#讓-quality-gate-卡住-pipeline)
    - [排除特定檔案或目錄](#排除特定檔案或目錄)
    - [善用 Cache 加速掃描](#善用-cache-加速掃描)
    - [完整 clone 歷史紀錄](#完整-clone-歷史紀錄)

---

## 前言

本章專注在「**如何把 SonarQube 接到 GitLab CI/CD pipeline**」。

關於 SonarQube 本身的介紹、自架步驟與整合 GitLab 的完整設定流程，請直接參考 [`SonarQube/`](../../SonarQube/) 資料夾下的文章：

- [01-sonarqube.md](../../SonarQube/01-sonarqube.md)：SonarQube 簡介、核心概念、版本比較
- [02-gitlab-ci-integration.md](../../SonarQube/02-gitlab-ci-integration.md)：完整的 GitLab CI 整合步驟（匯入專案、設定變數、撰寫 `.gitlab-ci.yml`）
- [03-quality-gate-design.md](../../SonarQube/03-quality-gate-design.md)：Quality Gate 設計（PoC / Product）
- [04-quality-gate-and-results.md](../../SonarQube/04-quality-gate-and-results.md)：Quality Gate 與掃描結果判讀
- [05-quality-profile.md](../../SonarQube/05-quality-profile.md)：Quality Profile 規則調整

本章僅針對 GitLab CI 端常用的整合技巧做整理，使用前請先依上述文章完成基本整合。

整合的目標：每次 push 觸發 pipeline 時，自動跑一次掃描，並讓 Quality Gate 不過時 pipeline 直接失敗，阻擋問題程式碼合入主線。

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
      │  (回傳 Quality Gate 結果)
      ▼
Pipeline 通過 or 失敗
```

整個流程三個角色：

* **sonar-scanner**：CLI 工具，在 CI job 中被觸發，負責讀取原始碼並把分析結果送到 Server。

* **SonarQube Server**：接收分析結果後，由 Compute Engine 計算指標、更新資料庫，並透過 Web UI 呈現報告。

* **Quality Gate**：Server 端設定的門檻，掃描完成後由 Server 判定是否通過，結果回傳給 scanner，進而決定 Pipeline 成敗。

---

## 整合步驟

### Step 1：在 SonarQube 匯入專案

登入 SonarQube → **Create Project** → 選擇 **GitLab**，依畫面指示輸入：

* **Configuration name**：隨便取一個名字，例如 `GitLab CI`
* **GitLab API URL**：例如 `https://gitlab.com/api/v4`
* **Personal Access Token**：在 GitLab 產生一個 [Personal Access Token](https://docs.gitlab.com/user/profile/personal_access_tokens/)，權限選 `api` (可以使用 service account 產生，或是個人帳號產生都可以)

選擇要分析的專案後按 **Import**，**New Code definition** 選 **Follows the instance's default** 即可，最後按 **Create projects**。

匯入完成後，到 **My Account** → **Security** 產生一個 **User Token**（之後 CI 會用到），複製起來。

### Step 2：設定 GitLab CI/CD Variables

在 GitLab repo → **Settings** → **CI/CD** → **Variables**，新增以下兩個變數：

| 變數名稱 | 值 | 說明 |
|---------|---|------|
| `SONAR_HOST_URL` | `http://your-sonarqube-server:9000` | SonarQube Server 位址 |
| `SONAR_TOKEN` | （剛才產生的 token） | 分析用的認證 token，建議勾選 **Masked** |

### Step 3：建立 sonar-project.properties

在 repo 根目錄建立 `sonar-project.properties`：

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

### Step 4：在 .gitlab-ci.yml 加入掃描 job

```yaml
stages:
  - sonar_scan

sonarqube-check:
  stage: sonar_scan
  tags:
    - lndata-1-docker        # ← 換成你的 GitLab Runner tag
  image: maven:3.9-eclipse-temurin-21
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  cache:
    policy: pull-push
    key: "sonar-cache-$CI_COMMIT_REF_SLUG"
    paths:
      - "${SONAR_USER_HOME}/cache"
  script:
    - mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -DskipTests
  allow_failure: false
  rules:
    - if: $CI_COMMIT_BRANCH == "test-va" && $CI_PIPELINE_SOURCE == "push"
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
