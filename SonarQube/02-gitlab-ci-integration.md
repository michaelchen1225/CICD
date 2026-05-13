# SonarQube --- 整合 GitLab CI/CD (Java)

本文處理「Server 起好之後，怎麼把 SonarQube 接進 GitLab CI/CD pipeline」。主軸是 **Java + Maven**（公司多數專案的組合），Gradle 給一個小範例帶過。

## 目錄

- [整合流程一覽](#整合流程一覽)
- [前置作業：在 SonarQube 建立專案 + 產 Token](#前置作業在-sonarqube-建立專案--產-token)
  - [Path A — Create a local project](#path-a--create-a-local-project)
  - [Path B — Import from GitLab](#path-b--import-from-gitlab)
  - [三種 Token 一次釐清](#三種-token-一次釐清)
- [在 GitLab 設 CI/CD Variables](#在-gitlab-設-cicd-variables)
- [設定 pom.xml](#設定-pomxml)
- [補充：`sonar-project.properties` 何時用](#補充sonar-projectproperties-何時用)
- [.gitlab-ci.yml 範例（Maven，跳過測試）](#gitlab-ciyml-範例maven跳過測試)

---

## 整合流程一覽

```
Developer push (main / 指定 branch)
        │
        ▼
GitLab Runner 拉 maven image
        │
        ▼
  mvn verify sonar:sonar
        │  (sonar-maven-plugin 把分析報告打包送出)
        ▼
   SonarQube Server
        │  (Compute Engine 處理報告，更新 DB / ES)
        ▼
   Quality Gate 判定
        │  (scanner 持續 polling /api/qualitygates/project_status)
        ▼
  Pipeline pass / fail
```

關鍵在於：**scanner 送出報告後，CE 還要幾秒到幾十秒才會處理完**。scanner 預設不等就退出，pipeline 直接 pass、看不到 Quality Gate 結果。要讓 pipeline 真的等需加 `sonar.qualitygate.wait=true`（後續設定 pom.xml 一節會處理）。

---

## 前置作業：在 SonarQube 建立專案 + 產 Token

`Projects → Create Project` 提供兩種建立路徑：

| 項目 | Create a local project | Import from GitLab |
|------|------------------------|--------------------|
| 前置作業 | 無 | Admin 要先設 DevOps Platform Integration |
| 匯入時要的 token | 無 | 一次性的 Service Account PAT |
| Project key | 自己取 | 由 GitLab repo path 自動帶入 |
| 主分支名稱 | 自己填 | 自動從 GitLab 抓 |
| 適合誰 | 單一專案、最快跑通 | 公司多 repo 想批量接 |

Community Build 下兩條路徑的分析能力完全一樣（MR decoration、branch analysis 都還是付費功能）。「重新接一個 Java 專案」的情境用 Path A 最快。

### Path A — Create a local project

依[官方 GitLab integration 文件](https://docs.sonarsource.com/sonarqube-community-build/devops-platform-integration/gitlab-integration/adding-analysis-to-gitlab-ci-cd/)：

1. `Projects → Create Project` → 頁面下方 `Are you just testing or have an advanced use-case?` 區塊按 `Create a local project`
   - **Project display name**：Dashboard 顯示名稱
   - **Project key**：CI 端 `mvn sonar:sonar` 對應的 key，**必須一致**。建議用 `<group>-<repo>`（例 `lndata-gen-bi`）
   - **Main branch name**：填 GitLab repo 實際主分支名（公司多為 `test-va`）
2. **New Code definition**：選 `Use the global setting`，之後可個別專案再調
3. **產 Project Analysis Token**：建好後 SonarQube 帶入 Analysis Method wizard，過程引導產生 token。建議用 **Project Analysis Token**（外洩時只能掃這個專案）、Expires 選 90 天
4. Wizard 會給一段 `mvn` 範例指令，[.gitlab-ci.yml 範例](#gitlab-ciyml-範例maven跳過測試) 會用到

### Path B — Import from GitLab

前置較多但可批量匯入，公司多 repo 划算。分三段：**(1) GitLab 建 Service Account → (2) SonarQube 設 DevOps integration → (3) 匯入專案**。

#### B.1 在 GitLab 建立 Service Account 並產 PAT

選 Service Account 而非個人帳號或一般 bot user 的理由（[官方文件](https://docs.gitlab.com/user/profile/service_accounts/)）：

- **不佔 license 席位**：普通 bot user 會吃一個付費席位，Service Account 完全不算 billable user
- **不能 UI 登入**：token 外洩時攻擊者沒辦法登進 GitLab 介面操作
- **不依賴特定員工**：沒人離職時跟著掛掉
- **不需要 instance admin**：group owner 就能建

> Service Account 是 GitLab 16.x+ 的功能、**CE 不支援**。CE 環境見本節最後的 fallback。

步驟：

1. **建 Service Account**：目標 group（例 `LnData/fusion`）→ `Settings → Service accounts → Add service account`，取名（例 `SonarQube Service`、username `fusion_sonarqube_service`），按 `Create`
2. **為 Service Account 產 PAT**：Service accounts 列表 → 該筆 → 右側三點 → `Manage access tokens` → `Add new token`，scope 勾 **`api`**、Expires 設 365 天，按 `Create token`，**字串只顯示一次**立刻複製到密碼管理器
3. **把 Service Account 加進 project / group**：`Manage → Members → Invite members`，搜尋帳號、Role 選 **Reporter**，按 `Invite`。也可加進整個 group 讓所有 project 自動繼承

#### B.2 SonarQube 設 DevOps Platform Integration（一次性、需 admin）

依[官方文件](https://docs.sonarsource.com/sonarqube-community-build/devops-platform-integration/gitlab-integration/global-setup/)，`Administration → Configuration → General Settings → DevOps Platform Integrations → GitLab → Create configuration`：

- **Configuration name**：例 `Company GitLab`
- **GitLab URL**：`https://gitlab.com/api/v4` 或 self-hosted 的 `https://<your-gitlab>/api/v4`
- **Personal Access Token**：貼 B.1 拿到的 Service Account PAT
- `Save configuration`

#### B.3 匯入專案

`Projects → Create Project → Import from GitLab`，分兩步：

**1 of 2 — 選擇 repo**：SonarQube 直接用 Service Account PAT 列出 repo 清單，勾選後 `Next`。

**2 of 2 — Set up new code for N projects**：決定哪些 commit 視為 **New Code**。Quality Gate 只檢查 New Code，這個基準直接決定 pipeline 何時紅綠。三個選項（[官方文件](https://docs.sonarsource.com/sonarqube-community-build/project-administration/adjusting-analysis/configuring-new-code-calculation/)）：

| 選項 | 「New Code」是什麼 | 基準怎麼決定 |
|------|------------------|------------|
| **Previous version** | 自上一個版本以來變更的 code | Maven 從 `pom.xml` 的 `<version>` 讀，基準 = 「目前版本第一次被掃描」 |
| **Number of days** | 過去 X 天內變更的 code | 滾動視窗，預設 30 天、最多 90 |
| **Reference branch** | 「目前分支有、reference 沒有」的 code（類似 `git diff main..HEAD`） | 指定的 reference branch 名稱 |

實務適用：

- **Previous version**：適合有明確版本發佈節奏（Maven release plugin、semver 嚴謹）的專案。陷阱：若 `pom.xml` 的 `<version>` 長期停留在 `0.0.1-SNAPSHOT`，基準永遠不動，所有 commit 都算 New Code、QG 涵蓋率條件等同作用於整個專案歷史 → pipeline 持續紅
- **Number of days**（預設 30）：適合無明確版本、持續交付的專案。基準隨時間滑動、不依賴 `<version>` bump，是 CI/CD 場景最常見的選擇
- **Reference branch**：適合長期分支模型（GitFlow）。**Community Build 限制**：reference branch 必須也曾被 SonarQube 掃描過才有效基準

**情境建議**：在「Java + Maven、未隨 release 維護 `pom.xml` 版本、僅在單一開發分支執行 SonarQube」這種情境下，選 `Number of days → 30` 最合理；其餘兩種在此情境皆無法有效運作。

按 `Create projects` 完成匯入。

**匯入完成後**：N 個專案出現在 Projects 列表，但 **Project Analysis Token 還沒產**——需分別進每個專案的 Analysis Method wizard：

- 點進專案 → 「How do you want to analyze your repository?」選 **`With GitLab CI/CD`**
- **第 1 段** Add environment variables：點 `Generate a token` 產 Project Analysis Token（**Expires 選 90 天**），依指示在 GitLab 設 `SONAR_TOKEN` 與 `SONAR_HOST_URL`（細節見[下一節](#在-gitlab-設-cicd-variables)）
- **第 2 段** Create or update the configuration file：選 build tool，wizard 產出對應 yaml 片段

**Project Analysis Token Expires 怎麼選**

| 選項 | 取捨 |
|------|------|
| 30 / 60 天 | 安全性最好但輪替負擔重，容易忘 |
| **90 天**（推薦）| 一季輪替一次，與安全月、季度檢查節奏對得上 |
| 1 年 | 維運輕鬆，但外洩到失效最大 365 天，太久 |
| No expiration | **絕對不要選** |

統一用 90 天，到期日登記在密碼管理工具，到期前重產一把、更新 GitLab Variable 即可（`.gitlab-ci.yml` 不用改）。

#### CE 用戶的 fallback：開 bot user

GitLab CE 沒 Service Account，改用 instance admin 開普通 user 當 bot：

1. `Admin Area → Users → New user`，username 例如 `sonarqube-bot`，email 用團隊共用信箱
2. 邀請 bot 到 project / group 為 Reporter
3. 用 bot 帳號登入 → `User Settings → Access Tokens` → 產 token，scope `api`
4. token 貼到 B.2 的 SonarQube DevOps integration

代價：佔 license 席位、bot 帳號能 UI 登入（密碼要保管好）、需 instance admin。能用 Service Account 就用 Service Account。

### 三種 Token 一次釐清

Path B 流程裡其實有三個 token，常被搞混：

| Token | 在哪產 | 給誰用 | 做什麼 |
|-------|--------|--------|--------|
| **Service Account PAT**（`api`）| GitLab → group 的 Service accounts | SonarQube **Server** 永久使用 | 列 repo、處理 webhook 等 admin 級 GitLab API |
| **個人 GitLab PAT**（`read_api`，選用）| GitLab → 自己 Access Tokens | SonarQube import wizard 暫時用 | 預設不需要；只有想用自己的權限視角覆寫 Service Account 時才用 |
| **Project Analysis Token** | SonarQube → 專案 Analysis Method wizard | GitLab CI **Runner** | 上傳分析結果回 SonarQube |

方向：**Service Account PAT 是 SonarQube → GitLab；Project Analysis Token 是 GitLab Runner → SonarQube**。

---

## 在 GitLab 設 CI/CD Variables

`Settings → CI/CD → Variables` 新增兩個變數：

| 變數 | 值 | Masked | Protected |
|------|---|--------|-----------|
| `SONAR_TOKEN` | wizard 產的 Project Analysis Token | 勾 | 不勾 |
| `SONAR_HOST_URL` | wizard 自動帶入（例 `http://192.168.10.201:9000`） | 不勾 | 不勾 |

旗標意義：

- **Masked**：log 自動遮成 `[MASKED]`，token 必勾、URL 不必
- **Protected**：勾起來時只有 protected branch / tag 觸發的 pipeline 才讀得到。dev branch（`test-va`、`feature/*`）通常不是 protected branch，勾了會直接 `Not authorized` fail
  - **想再收緊**：把實際跑掃描的 branch 設成 protected branch，再把 `SONAR_TOKEN` 的 Protected 勾起來
  - **不收緊也安全**：靠 `.gitlab-ci.yml` 的 `rules:` 限制掃描 job 只在指定 branch 觸發

---

## 設定 pom.xml

Analysis Method wizard 給三段 `<properties>`：

```xml
<properties>
  <sonar.projectKey>lndata_data-transform_gen-bi_6c5840f1-xxx-4xxx-XXXXXXXX</sonar.projectKey>
  <sonar.projectName>gen-bi</sonar.projectName>
  <sonar.qualitygate.wait>true</sonar.qualitygate.wait>
</properties>
```

每條的意義：

- **`sonar.projectKey`**：必填，CI 端 `mvn` 用這個 key 對應 Server 專案。Path B 自動產生形如 `<group>_<repo>_<UUID>` 的長 key，原樣保留即可（想換乾淨命名要先在 SonarQube `Project Settings → Update Key` 改、再同步改 pom.xml）
- **`sonar.projectName`**：Dashboard 顯示用
- **`sonar.qualitygate.wait`**：讓 scanner 等 QG 結果才退出。寫在 pom.xml 的好處是本機跑 `mvn sonar:sonar` 也生效、CI 命令列不必每次帶 `-D`

**另外建議：在 `<pluginManagement>` 鎖定 `sonar-maven-plugin` 版本**（wizard 不會主動幫加）：

```xml
<build>
  <pluginManagement>
    <plugins>
      <plugin>
        <groupId>org.sonarsource.scanner.maven</groupId>
        <artifactId>sonar-maven-plugin</artifactId>
        <version>5.6.0.6792</version>
      </plugin>
    </plugins>
  </pluginManagement>
</build>
```

避免 CI 抓到不可預期的最新版（寫作時為 `5.6.0.6792`，請去 [Maven Central](https://central.sonatype.com/artifact/org.sonarsource.scanner.maven/sonar-maven-plugin) 確認）。

> **不要寫進 `pom.xml`**：`sonar.token`、`sonar.host.url`——這個檔會 commit 進 git。改走 GitLab CI/CD Variables。

---

## 補充：`sonar-project.properties` 何時用

官方文件提到的 `sonar-project.properties` 是放在專案 root 的 `key=value` 設定檔，給 **SonarScanner CLI**（standalone）讀。最小範例：

```properties
sonar.projectKey=my-project
sonar.projectName=My Project
sonar.sources=.
sonar.sourceEncoding=UTF-8
```

[官方文件](https://docs.sonarsource.com/sonarqube-community-build/analyzing-source-code/scanners/sonarscanner/)直接點名：

> Don't use the SonarScanner CLI for projects built with Maven, Gradle, or .NET. Doing so will degrade the quality of your analysis.

對 Java + Maven 情境，正確做法就是上一節寫進 `pom.xml` 的 `<properties>`、由 `sonar-maven-plugin` 讀取；Gradle 對應 `build.gradle` 的 `sonar { properties { ... } }`。混用 `sonar-project.properties` 不會多保險，反而可能讓 plugin 取不到正確的編譯產物路徑、降低分析品質。

`sonar-project.properties` 只在「**沒有對應 build tool plugin**」的專案才會出現——純前端 JS/TS、Python、Go、C/C++ 等用 CLI 掃描的情境。

**設定參數優先序**（[官方文件](https://docs.sonarsource.com/sonarqube-community-build/analyzing-source-code/analysis-parameters/configuration-overview/)，由低到高）：

1. SonarQube UI 全域設定
2. SonarQube UI 該專案設定
3. Scanner 設定檔（`sonar-project.properties` / `pom.xml` `<properties>` / `build.gradle` 的 sonar 區塊）
4. 命令列 `-D` 旗標

`sonar.token` / `sonar.host.url` 永遠走環境變數，不寫進 commit 進 git 的設定檔。

---

## .gitlab-ci.yml 範例（Maven，跳過測試）

### Wizard 給的版本

```yaml
image: maven:3-eclipse-temurin-17

variables:
  SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  GIT_DEPTH: "0"

stages:
  - build-sonar

build-sonar:
  stage: build-sonar
  cache:
    policy: pull-push
    key: "sonar-cache-$CI_COMMIT_REF_SLUG"
    paths:
      - "${SONAR_USER_HOME}/cache"
      - sonar-scanner/
  script:
    - mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar
  allow_failure: true
  rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
    - if: $CI_COMMIT_BRANCH == 'master'
    - if: $CI_COMMIT_BRANCH == 'main'
    - if: $CI_COMMIT_BRANCH == 'develop'
```

可直接跑，但有幾處對 Community Build / 跳過測試專案不適合。

### 修改後的正式環境版本

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

### 對 wizard 版本的修改

| 修改 | 理由 |
|------|------|
| cache.paths 移除 `sonar-scanner/` | 那是 standalone CLI scanner 才會產的目錄，Maven plugin 用不到 |
| script 加 `-DskipTests` | 公司專案目前沒可用測試（Spring Boot 自動產的 context-load 測試需要 MongoDB / AWS 認證，會 fail） |
| rules 縮到一條 | Community Build 不支援 MR / branch analysis，多 branch 推會互相覆蓋 Dashboard。導入 [06 章 plugin](06-mr-pr-integration.md) 才能加回 MR rule |
| 加 `tags` | self-hosted Runner 需要 tag；wizard 不知道公司 runner 名 |
| `allow_failure: false` | **接好後再切**——第一階段先保留 `true`，調整 QG 後再切 `false` 正式把關 |

`sonar.qualitygate.wait` 沒出現在 yaml——已寫在 `pom.xml` `<properties>`，CI 不必再帶 `-D`。

### 關鍵欄位說明

- **`image: maven:3.9-eclipse-temurin-21`**：`sonar-maven-plugin 5.x` 要求 **JRE 21+**。Image 用 JDK 21 可省下 plugin auto-provisioning JRE 21 的時間。Wizard 預設 `maven:3-eclipse-temurin-17` 也能跑，但每次 job 都要拉 JRE
- **`SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"`**：scanner 預設把 plugin 放 `~/.sonar`，但 GitLab CI cache 只能涵蓋 `${CI_PROJECT_DIR}` 底下的路徑，要把 user home 改指過來 cache 才管得到
- **`GIT_DEPTH: "0"`**：拉完整 git 歷史。SonarQube 用 `git blame` 時間戳判定 New Code；GitLab 預設只 clone 最近 20 個 commit，blame 資訊不全會讓 New Code 判定失準
- **`cache.key: "sonar-cache-$CI_COMMIT_REF_SLUG"`**：cache 按 branch 分，可重用又不互相污染
- **`mvn verify ... sonar:sonar -DskipTests`**：
  - 用 `verify` 而非 `package`：跑到 verify 階段確保 `sonar-maven-plugin` 拿得到完整編譯結果。`-DskipTests` 跳測試但 build 仍跑到 verify
  - 用完整 plugin coordinate `org.sonarsource.scanner.maven:sonar-maven-plugin:sonar` 而非短名 `sonar:sonar`：版本可控（前提是 `<pluginManagement>` 已鎖好版本）
- **`rules: $CI_COMMIT_BRANCH == "test-va" && $CI_PIPELINE_SOURCE == "push"`**：只在 `test-va` push 跑分析（公司把 `test-va` 當主分析線）

下一步：[04-quality-gate-and-results.md — Quality Gate 與掃描結果判讀](04-quality-gate-and-results.md)。
