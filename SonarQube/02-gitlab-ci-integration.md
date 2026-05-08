# SonarQube --- 整合 GitLab CI/CD (Java)

[前一篇](sonarqube.md)講完了 SonarQube 的部署、設定檔與 Quality Gate 概念，這篇接著處理「Server 起好之後，怎麼把它接進 GitLab CI/CD pipeline」。本文以 **Java + Maven** 為主軸（公司多數專案用這個組合），Gradle 給一個小範例帶過。

## 目錄

- [SonarQube --- 整合 GitLab CI/CD (Java)](#sonarqube-----整合-gitlab-cicd-java)
  - [目錄](#目錄)
  - [整合流程一覽](#整合流程一覽)
  - [前置作業：在 SonarQube 建立專案 + 產 Token](#前置作業在-sonarqube-建立專案--產-token)
    - [兩條建立路徑比較](#兩條建立路徑比較)
    - [Path A — Create a local project（手動建立）](#path-a--create-a-local-project手動建立)
    - [Path B — Import from GitLab（DevOps 平台匯入）](#path-b--import-from-gitlabdevops-平台匯入)
      - [B.1 在 GitLab 建立 Service Account 並產 PAT](#b1-在-gitlab-建立-service-account-並產-pat)
      - [B.2 在 SonarQube 設 DevOps Platform Integration（一次性，需 SonarQube admin）](#b2-在-sonarqube-設-devops-platform-integration一次性需-sonarqube-admin)
      - [B.3 匯入專案](#b3-匯入專案)
        - [1 of 2 — 選擇 repo](#1-of-2--選擇-repo)
        - [2 of 2 — Set up new code for N projects](#2-of-2--set-up-new-code-for-n-projects)
        - [匯入完成後](#匯入完成後)
        - [Project Analysis Token 的 Expires 怎麼選](#project-analysis-token-的-expires-怎麼選)
      - [三種 Token 一次釐清](#三種-token-一次釐清)
      - [CE 用戶的 fallback：開 bot user](#ce-用戶的-fallback開-bot-user)
  - [在 GitLab 設 CI/CD Variables](#在-gitlab-設-cicd-variables)
  - [設定 pom.xml](#設定-pomxml)
  - [.gitlab-ci.yml 範例（Maven，跳過測試）](#gitlab-ciyml-範例maven跳過測試)
    - [逐行解釋](#逐行解釋)
  - [Quality Gate 在 CI 的把關](#quality-gate-在-ci-的把關)
    - [`sonar.qualitygate.wait=true` 機制](#sonarqualitygatewaittrue-機制)
    - [跳過測試的專案怎麼處理 Quality Gate](#跳過測試的專案怎麼處理-quality-gate)
    - [漸進採用：先 allow\_failure 後切 false](#漸進採用先-allow_failure-後切-false)
  - [Branch / MR pipeline 行為](#branch--mr-pipeline-行為)
  - [Cache 與 GIT\_DEPTH 細節](#cache-與-git_depth-細節)
  - [Gradle 簡短範例](#gradle-簡短範例)
  - [常見錯誤排查](#常見錯誤排查)
  - [小結](#小結)

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

整段流程的關鍵在於：**scanner 把報告送出後，Compute Engine 還要幾秒到幾十秒才會處理完**。預設情況 scanner 不等就退出，pipeline 直接 pass，看不到 Quality Gate 結果。後面 [Quality Gate 在 CI 的把關](#quality-gate-在-ci-的把關) 會說明怎麼用 `sonar.qualitygate.wait=true` 讓 pipeline 真的等。

---

## 前置作業：在 SonarQube 建立專案 + 產 Token

進入 `Projects → Create Project` 之後，SonarQube 會顯示兩組選項：

* **主要區塊**：`Create your project from your favorite DevOps platform.`
  列出 Azure DevOps / Bitbucket Cloud / Bitbucket Server / GitHub / **GitLab**，各自有 `Setup` 按鈕。
* **下方區塊**：`Are you just testing or have an advanced use-case?` 底下一個 `Create a local project` 按鈕。

從 UX 設計就看得出 SonarQube 在**引導使用者走 DevOps 平台匯入**，把手動建立放在「測試或進階用途」位置。但對 Community Build 來說，這兩條路徑做出來的專案**分析能力完全一樣**——MR decoration、branch analysis 都還是付費功能，匯入並不會解鎖。差別只在便利性、前置成本，以及主分支名稱要不要自己填。

### 兩條建立路徑比較

| 項目 | Create a local project | Import from GitLab |
|------|------------------------|--------------------|
| 前置作業 | 無 | Admin 要先按 GitLab 區塊的 `Setup` 設好 DevOps Platform Integration（見 Path B） |
| 匯入時要的 token | 無 | 一次性設定 Service Account PAT（admin 做） |
| Project key | 自己取 | 由 GitLab repo path 自動帶入 |
| 主分支名稱 | 自己填 | 自動從 GitLab 抓 |
| 適合誰 | 單一專案、最快跑通 | 公司多個 repo 想批量接 |

對「重新接一個 Java 專案」的情境，**`Create a local project` 最直接**——少一步 admin 前置工作，幾分鐘就能把專案開好。下面兩條路徑都列，看你需求挑。

### Path A — Create a local project（手動建立）

依[官方 GitLab integration 文件](https://docs.sonarsource.com/sonarqube-community-build/devops-platform-integration/gitlab-integration/adding-analysis-to-gitlab-ci-cd/)：

* **Step 1**：登入 SonarQube → `Projects` → `Create Project` → 在頁面下方 `Are you just testing or have an advanced use-case?` 區塊按 `Create a local project`
  * **Project display name**：在 Dashboard 上顯示的名字，可中可英
  * **Project key**：之後 CI 端 `mvn sonar:sonar` 會用這個對應到 Server，**必須一致**。建議用 `<group>-<repo>` 之類的命名（例如 `lndata-gen-bi`）
  * **Main branch name**：填你 GitLab repo 上實際的主分支名（公司若用 `test-va` 就填 `test-va`、用 `main` 就填 `main`）

* **Step 2 — New Code definition**：選 `Use the global setting`（預設是 Previous version），之後可以再針對個別專案調整

* **Step 3 — 產 Project Analysis Token**：建立完之後 SonarQube 會帶你進 Analysis Method wizard，選擇 GitLab CI / Maven / Gradle 等情境，過程中會引導你產生 token
  * 建議用 **Project Analysis Token**（理由見 [sonarqube.md 常見維運提醒](sonarqube.md#常見維運提醒)：影響範圍最小，外洩時只能掃這個專案）
  * Expires：建議 90 天，到期前換新；不要選 `No expiration`

* **Step 4**：Wizard 會給一段 `mvn` 範例指令，先複製起來，[.gitlab-ci.yml 範例](#gitlab-ciyml-範例maven跳過測試) 會用到

### Path B — Import from GitLab（DevOps 平台匯入）

這條路徑前置作業比較多，但建好之後可以批量匯入 repo，公司有多個專案要接時划算。整個前置分三段：**(1) 在 GitLab 建 Service Account → (2) 在 SonarQube 設 DevOps integration → (3) 匯入專案**。

#### B.1 在 GitLab 建立 Service Account 並產 PAT

**為什麼用 Service Account 而不是個人帳號或一般 bot user**（依[官方文件](https://docs.gitlab.com/user/profile/service_accounts/)）：

* **不佔 license 席位** — 自架 GitLab 開普通 bot user 會吃一個付費席位；Service Account 完全不算 billable user
* **不能 UI 登入** — 即使 token 外洩，攻擊者也沒辦法登進 GitLab 介面操作，攻擊面更小
* **不依賴特定員工** — 沒人離職時跟著掛掉的問題
* **不需要 instance admin** — 你只要是 group owner 就能建

> Service Account 是 GitLab 16.x 之後的功能，[官方文件](https://docs.gitlab.com/user/profile/service_accounts/)寫明 **CE (Community Edition) 不支援**，需要 Premium / Ultimate / EE 才有。如果你的 GitLab 是 CE，看本節最後的 fallback 做法。

* **Step 1 — 建立 Service Account**：到要被分析的 group（例如 `LnData/fusion`） → `Settings → Service accounts → Add service account`
  * Name：取個有意義的名字，例如 `SonarQube Service`
  * Username：`fusion_sonarqube_service`
  * 按 `Create`

* **Step 2 — 為 Service Account 產 Personal Access Token**：回到 Service accounts 列表 → 找到剛建的那筆 → 點右邊 **垂直三點 (⋮)** → `Manage access tokens` → `Add new token`
  * Token name：`SonarQube DevOps Integration`
  * Expires at：建議 365 天（GitLab 預設值）
  * Scopes：勾 **`api`**
  * 按 `Create token` → **token 字串只會顯示一次**，立刻複製到安全的地方（密碼管理器）
  
  > 不需要「登入 Service Account 再產 token」——Service Account 本來就不能 UI 登入，所以 GitLab 直接在 group owner 介面提供「幫它產 token」的入口。

* **Step 3 — 把 Service Account 加進要分析的 project / group**：到要被 SonarQube 分析的 project（或整個 group）→ `Manage → Members → Invite members`
  * 搜尋 Step 1 取的名字
  * Role 選 **Reporter**（最低能讀 repo 內容的權限，剛好夠 SonarQube 用）
  * 按 `Invite`

  > 想懶人一點：直接把 Service Account 加進整個 group 為 Reporter，group 內所有 project 自動繼承權限。

#### B.2 在 SonarQube 設 DevOps Platform Integration（一次性，需 SonarQube admin）

依[官方 Setting up GitLab integration 文件](https://docs.sonarsource.com/sonarqube-community-build/devops-platform-integration/gitlab-integration/global-setup/)：

`Administration → Configuration → General Settings → DevOps Platform Integrations → GitLab → Create configuration`：

* **Configuration name**：自己取一個識別用的名字，例如 `Company GitLab`
* **GitLab URL**：填 `https://gitlab.com/api/v4` 或公司 self-hosted 的 `https://<your-gitlab>/api/v4`
* **Personal Access Token**：貼上 B.1 Step 2 拿到的 Service Account PAT
* `Save configuration`

#### B.3 匯入專案

Import 流程分兩步：**1 of 2** 選 repo、**2 of 2** 一次幫所有匯入的 repo 設 New Code definition。匯入完之後，再各自進每個專案產 Project Analysis Token。

##### 1 of 2 — 選擇 repo

* `Projects → Create Project → Import from GitLab`
* 出現 `GitLab project onboarding` 頁面，SonarQube 直接用 B.2 設好的 Service Account PAT 去 GitLab 列出 repo 清單（頁面上方的 `Reset your GitLab personal access token` 連結是之後換 token 才會用到，目前不用點）
* 勾選要匯入的 repo（可多選；也可勾左上角 `Select currently visible repositories` 全選）→ 按 `Next`

##### 2 of 2 — Set up new code for N projects

這頁是要決定 SonarQube 把哪些 commit 視為「**New Code**」。Quality Gate（[Sonar way](sonarqube.md#quality-gate-入門) 那 4 條）只檢查 New Code，所以這個基準怎麼設**直接決定 pipeline 哪天紅、哪天綠**，不是隨便選的擺設。

依[官方 Configuring new code calculation 文件](https://docs.sonarsource.com/sonarqube-community-build/project-administration/adjusting-analysis/configuring-new-code-calculation/)，三個選項實際的行為是：

| 選項 | 「New Code」是什麼 | 基準怎麼決定 |
|------|------------------|------------|
| **Previous version** | 自上一個版本以來變更的所有 code | Maven 從 `pom.xml` 的 `<version>` 讀；Gradle 從 `build.gradle`；其他語言用 `sonar.projectVersion` 參數。基準時間點 = 「目前版本第一次被掃描」的那次分析 |
| **Number of days** | 過去 X 天內變更的所有 code | 滾動視窗。預設 30 天，可設 7 / 14 / 最多 90 |
| **Reference branch** | 「目前分支有、但 reference branch 沒有」的 code（類似 `git diff main..HEAD`） | 你指定的 reference branch 名稱（例如 `main`） |

各選項適用情境分析：

* **Previous version**：適合有**明確版本發佈節奏**的專案（Maven release plugin、semver 嚴謹的團隊）。每次 `mvn release:perform` 或手動 bump `<version>` 之後，新版第一次掃描那一刻成為新基準，後續 commit 都算 New Code，直到下一次版本變更。
  * **常見陷阱**：若 build 流程不更新 `pom.xml` 的 `<version>`（例如長期停留於 `0.0.1-SNAPSHOT`），基準將永遠不動，所有 commit 皆被視為 New Code，Quality Gate 的涵蓋率條件實際上會作用於整個專案歷史，導致 pipeline 持續失敗。缺乏版本管理紀律的專案不建議使用。

* **Number of days**（預設 30）：適合**沒有明確版本發佈、持續交付**型的專案。新 commit 在指定天數內視為 New Code，超過則不再列入 Quality Gate 檢查範圍。基準隨時間滑動，避免 pipeline 被長期累積的舊問題阻塞。
  * **特性**：不依賴版本 bump、設定後幾乎不需維護、Quality Gate 永遠針對近期變更把關，為 CI/CD 持續整合場景下最常見的選擇。

* **Reference branch**：適合**長期分支模型**（GitFlow、環境分支策略）。設 `main` 為 reference 後，於 `test-va` 上掃描時 New Code 為 test-va 相對於 main 的差集（類似 `git diff main..test-va`）。
  * **重要限制**：Community Build 同一個 projectKey 只保留一份分析結果，reference branch 必須**也曾被 SonarQube 掃描過**才能形成有效基準。若僅在開發分支推送分析、reference branch 從未掃過，此選項會退回不精確的判定邏輯，需先確認 reference branch 已納入掃描週期再使用。

**情境建議**：在「Java + Maven、未隨 release 維護 `pom.xml` 版本、僅在單一開發分支執行 SonarQube」的情境下（亦即本文預設情境），建議選 `Is custom → Number of days → 30`。版本基準與分支基準兩種機制在此情境皆無法有效運作，唯有滾動天數窗口能持續產生有意義的 New Code 判定。

> Wizard 提示 `You can change this setting for each project individually at any time in the project administration settings.`——選擇後仍可於個別專案 `Project Settings → New Code` 重新調整，不影響其他已匯入專案。

按 `Create projects` 完成匯入。

##### 匯入完成後

N 個專案都會出現在 SonarQube `Projects` 列表，project key 自動設成 GitLab repo path、主分支從 GitLab 自動取得。**Project Analysis Token 還沒產**——要分別進每個專案的 Analysis Method wizard 完成設定：

* 點進專案（例如 `gen-bi`）→ 引導頁「How do you want to analyze your repository?」選 **`With GitLab CI/CD`**
* 進入 `Analyze your project with GitLab CI` 頁面，分兩大段：
  * **第 1 段：Add environment variables** — 點 `Generate a token` 即時產生 Project Analysis Token（**Expires 建議選 90 天**，理由見下方），並依指示在 GitLab 設 `SONAR_TOKEN` 與 `SONAR_HOST_URL` 兩個 CI/CD Variables。詳細的 Masked / Protected 設定見 [在 GitLab 設 CI/CD Variables](#在-gitlab-設-cicd-variables)
  * 
  * **第 2 段：Create or update the configuration file** — 選擇 build tool（Maven / Gradle / JS/TS & Web / .NET / Python / Other），wizard 會產出對應的 `.gitlab-ci.yml` 片段

> Wizard 給的 yaml 是最小可跑版本；本文 [.gitlab-ci.yml 範例](#gitlab-ciyml-範例maven跳過測試) 額外補上 cache、`GIT_DEPTH=0`、`qualitygate.wait`、`rules:` 等正式環境會用到的細節。

##### Project Analysis Token 的 Expires 怎麼選

`Generate a token` 對話框可選 30 / 60 / 90 天、1 年、或 `No expiration`。各選項的取捨：

| 選項 | 取捨 |
|------|------|
| **30 / 60 天** | 安全性最好，但輪替負擔重——一個季度要動兩到四次，容易忘記、忘了 pipeline 就直接 `Not authorized` 紅 |
| **90 天**（推薦）| 一個季度輪替一次，跟一般公司的安全月、季度檢查節奏對得上；外洩到失效的最大窗口 90 天，可接受 |
| **1 年** | 維運最輕鬆，但外洩到失效最多差 365 天，太久 |
| **No expiration** | **絕對不要選**——一旦外洩就是無限期暴露，沒有時間自然止血 |

> 公司專案建議統一用 90 天，並在密碼管理工具（1Password / Bitwarden / Vault）登記到期日，到期前 7 天提醒 SRE / DevOps 回到 SonarQube 重產一把、更新 GitLab `SONAR_TOKEN` Variable 即可（`.gitlab-ci.yml` 不用改）。

#### 三種 Token 一次釐清

整個 Path B 流程裡其實有**三個不同的 token**，常被搞混。一次列清楚：

| Token | 在哪產 | 給誰用 | 做什麼 |
|-------|--------|--------|--------|
| **Service Account PAT** (`api` scope) | GitLab → 該 group 的 Service accounts | 給 SonarQube **Server** 永久使用 | 列 repo 清單、處理 webhook 等 admin 級 GitLab API 呼叫 |
| **你自己的 GitLab PAT** (`read_api` scope，**選用**) | GitLab → 自己 `User Settings → Access Tokens` | 給 SonarQube **import wizard** 暫時用 | 預設不用——SonarQube 直接用 Service Account PAT 列 repo。只有當你想**覆寫**成自己的權限視角時，才在 onboarding 頁按 `Reset your GitLab personal access token` 貼進去 |
| **Project Analysis Token** | SonarQube → 專案 Analysis Method wizard | 給 GitLab CI **Runner** | 上傳分析結果回 SonarQube（後面 yaml 會用到） |

三個 token 三個方向，記住：**Service Account PAT 是 SonarQube → GitLab 方向；Project Analysis Token 是 GitLab Runner → SonarQube 方向**。

#### CE 用戶的 fallback：開 bot user

GitLab CE 沒有 Service Account 功能，這時改用「Instance admin 開一個普通 user 當 bot」：

1. Instance admin 到 `Admin Area → Users → New user` 開一個帳號（username 例如 `sonarqube-bot`，email 用團隊共用信箱）
2. 邀請該 bot 到要分析的 project / group 為 Reporter
3. 用 bot 帳號登入 → `User Settings → Access Tokens` → 產 token，scope 勾 `api`
4. 把 token 貼到 B.2 的 SonarQube DevOps integration 設定

代價：佔一個 license 席位、bot 帳號能 UI 登入（密碼要妥善保管）、需要 instance admin 才能開人。能用 Service Account 就優先用 Service Account。

---

## 在 GitLab 設 CI/CD Variables

到 GitLab 專案頁面 `Settings → CI/CD → Variables`，新增兩個變數。下表是 SonarQube Analysis Method wizard 的官方建議設定（最寬鬆、最容易跑起來的版本）：

| 變數 | 值 | Masked | Protected |
|------|---|--------|-----------|
| `SONAR_TOKEN` | Analysis Method wizard 中按 `Generate a token` 產生的 Project Analysis Token | 勾 | 不勾 |
| `SONAR_HOST_URL` | wizard 自動帶入（例如 `http://192.168.10.201:9000`） | 不勾 | 不勾 |

各旗標的意義：

* **Masked**：在 Job log 裡會被自動遮罩成 `[MASKED]`，避免 token 不小心被印出來。Token 一定要勾；URL 是公開資訊不勾。
* **Protected**：勾起來時，**只有 protected branch / tag 觸發的 pipeline 才能讀到這個變數**。
  * SonarQube wizard 預設給「不勾」，因為大多數 dev branch（例如 `test-va`、`feature/*`）並非 protected branch；勾了會直接讓 pipeline 抓不到 token、scanner 報 `Not authorized` fail 掉。
  * **想再收緊安全性**：把實際跑掃描的 branch（例如 `test-va`）在 GitLab 設成 protected branch，再把 SONAR_TOKEN 的 Protected 勾起來。這樣其他 feature branch 即使 push 到 GitLab，也偷不到 token。安全 > 便利的工作流走這條。
  * **不收緊也能維持安全**：依預設不勾 Protected，靠 `.gitlab-ci.yml` 的 `rules:` 限制掃描 job 只在指定 branch 觸發（例如 `if: $CI_COMMIT_REF_NAME == "test-va"`），效果類似但約束力不如 protected branch 強。

> 若勾了 Protected 但 pipeline 跑在非 protected branch 上，Job 會看到 `SONAR_TOKEN` 是空的，scanner 報 `Not authorized`。要嘛把那個 branch 設 protected、要嘛取消這個勾。

---

## 設定 pom.xml

Analysis Method wizard 會直接給你三段要加進 `pom.xml` `<properties>` 的 SonarQube 屬性：

```xml
<properties>
  <sonar.projectKey>lndata_data-transform_gen-bi_6c5840f1-9cd2-46c2-844d-3b18b9c629e0</sonar.projectKey>
  <sonar.projectName>gen-bi</sonar.projectName>
  <sonar.qualitygate.wait>true</sonar.qualitygate.wait>
</properties>
```

每一條的意義：

* **`sonar.projectKey`** — 必填，CI 端的 mvn 會用這個 key 對應到 SonarQube Server 上的專案。Path B 的 import flow 會自動產生形如 `<group>_<repo>_<UUID>` 的長字串 key，原樣保留即可（想換成乾淨命名要先在 SonarQube `Project Settings → Update Key` 改、再同步改 pom.xml）。
* **`sonar.projectName`** — Dashboard 顯示用，可中英文。
* **`sonar.qualitygate.wait`** — 讓 scanner 等 Quality Gate 結果出來才退出。寫在 pom.xml 的好處是：
  * `mvn` 命令列不用每次帶 `-Dsonar.qualitygate.wait=true`
  * 本機跑 `mvn sonar:sonar` 也會生效，不只 CI 才生效
  * pom.xml 是 single source of truth，CI yaml 看起來更乾淨

**另外建議：在 `<pluginManagement>` 鎖定 `sonar-maven-plugin` 版本**（wizard 不會主動幫你做這件事，需要自己加）：

避免 CI 抓到不可預期的最新版（撰寫時最新版是 `5.6.0.6792`，請去 [Maven Central](https://central.sonatype.com/artifact/org.sonarsource.scanner.maven/sonar-maven-plugin) 確認）：

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

> **不要寫進 `pom.xml`**：`sonar.token`、`sonar.host.url`——這個檔會 commit 進 git，token 進去就等於洩漏。改用 [GitLab CI/CD Variables](#在-gitlab-設-cicd-variables) 的環境變數處理。

---

## .gitlab-ci.yml 範例（Maven，跳過測試）

### Wizard 給的版本

Analysis Method wizard 第 2 段選 Maven 後，會直接生出這份 yaml：

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

可以直接跑，但有幾處對 Community Build / 跳過測試專案不適合，建議按下節調整。

### 修改後的正式環境版本

```yaml
stages:
  - sonar_scan

sonarqube-check:
  stage: sonar_scan
  tags:
    - lndata-1-docker        # ← 換成你的 GitLab Runner tag
  image: maven:3-eclipse-temurin-17
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
    - if: $CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"
```

### 對 wizard 版本做了哪些修改

| 修改 | 從 → 到 | 理由 |
|------|---------|------|
| 1. cache.paths 移除 `sonar-scanner/` | wizard 多放一行 → 拿掉 | `sonar-scanner/` 是 standalone CLI scanner 才會產的目錄，Maven plugin 不會用到它，留著只是 cache 浪費 |
| 2. script 加 `-DskipTests` | `mvn verify ... sonar:sonar` → 加 `-DskipTests` | 公司專案目前沒有可用的測試（Spring Boot 自動產的 context-load 測試需要 MongoDB / AWS 認證，CI 沒接，會直接 fail） |
| 3. rules 移除 MR 與其他無關 branch | 4 條 → 1 條 | Community Build 不支援 MR / branch analysis，多 branch 推分析會互相覆蓋 Dashboard（理由見 [Branch / MR pipeline 行為](#branch--mr-pipeline-行為)）。只保留實際的「主分析線」一條 |
| 4. 加 `tags` | 沒有 → 加 | 公司用 self-hosted Runner，需要指定 tag；wizard 不知道公司 runner 名所以沒加 |
| 5. allow_failure 切 `false` | `true` → `false`（**接好之後再切**） | 第一階段先依 wizard 預設保留 `true`，跑幾次調整 Quality Gate 後再切 `false` 開始正式把關（見 [漸進採用](#漸進採用先-allow_failure-後切-false)） |

注意 `sonar.qualitygate.wait` **不在 yaml 出現**——它已經寫在 `pom.xml` 的 `<properties>` 裡，CI 命令列不用再帶 `-D` 旗標。

### 逐行解釋

* **`tags: lndata-1-docker`** — 公司專屬 GitLab Runner tag，**不是通用設定**。讀者要換成自己 runner 的 tag，沒設 tag 就拿掉這段。

* **`image: maven:3-eclipse-temurin-17`** — 跟 wizard 預設一致。`sonar-maven-plugin 5.x` 要求 **執行 JRE 21+**，但 plugin 會自動 download JRE 21（auto-provisioning），所以 image 上的 Maven JDK 是 17 沒關係。要省下 auto-provisioning 的時間也可以直接用 `maven:3-eclipse-temurin-21`。

* **`SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"`** — scanner 預設會把下載的 plugin、analyzer 放到 `~/.sonar`，但 GitLab CI 的 cache 機制**只能涵蓋 project workspace 內的路徑**（`${CI_PROJECT_DIR}` 底下），所以要把 user home 改指過來，cache 才管得到。

* **`GIT_DEPTH: "0"`** — GitLab 預設 shallow clone（`GIT_DEPTH=20`），SonarQube 在判斷「這行是新 code 還是舊 code」時要用 SCM blame，歷史不夠就會看到 `Missing blame information` warning，連帶影響 New Code 判定。`0` 表示完整 clone。

* **`cache.key: "sonar-cache-$CI_COMMIT_REF_SLUG"`** — cache 按 branch 分。同一個 branch 重複 commit 可重用 cache（節省 30 ~ 60 秒下載時間），不同 branch 不互相污染。

* **`cache.paths: "${SONAR_USER_HOME}/cache"`** — 只 cache scanner 下載的 plugin / analyzer 子目錄，不 cache 整個 `.sonar`（裡面有暫存檔、報告檔，每次都該重產），也不需要 wizard 多塞的 `sonar-scanner/`。

* **`mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -DskipTests`**：
  * **為何 `verify` 而非 `package`**：`verify` 跑到 verify 階段（包含整合測試會產生的 artifact），確保 `sonar-maven-plugin` 拿得到完整編譯結果。`-DskipTests` 會把 unit + integration test 跳過，但 build 過程仍跑到 verify。
  * **為何 `-DskipTests`**：公司專案目前沒有可用的測試（Spring Boot 自動產的 context-load 測試需要 MongoDB / AWS 認證，CI 環境沒接，會直接 fail）。
  * **為何用完整 plugin coordinate** `org.sonarsource.scanner.maven:sonar-maven-plugin:sonar` 而非短名 `sonar:sonar`：版本可控、避免 Maven 自動拉到不可預期的最新版本（前提是 `pom.xml` 的 `<pluginManagement>` 已經鎖好版本）。
  * **沒有 `-Dsonar.qualitygate.wait=true`**：因為 `sonar.qualitygate.wait` 已經寫在 `pom.xml` 的 `<properties>`，scanner 自動讀。詳見 [`sonar.qualitygate.wait=true` 機制](#sonarqualitygatewaittrue-機制)。

* **`allow_failure: false`** — Quality Gate fail 時 pipeline 也跟著 fail，正式把關。**注意**：第一次接 SonarQube 時建議先用 `true`，等 QG 設成能合理通過後再切 false（見[漸進採用：先 allow\_failure 後切 false](#漸進採用先-allow_failure-後切-false)）。

* **`rules: $CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"`** — 只在 main 的 push 事件跑分析。Community Build 限制下這是最合理的做法（理由見 [Branch / MR pipeline 行為](#branch--mr-pipeline-行為)）。如果你的「主分析線」不叫 main（例如公司用 `test-va`），改成那個 branch 名即可。

---

## Quality Gate 在 CI 的把關

### `sonar.qualitygate.wait=true` 機制

預設行為：scanner 把報告送到 Server 之後就退出，pipeline 永遠 pass——不管 Quality Gate 有沒有過。

加上 `-Dsonar.qualitygate.wait=true` 之後：

* Scanner 持續 polling `/api/qualitygates/project_status` API
* Quality Gate 一旦判定為 Fail → scanner 的 exit code 非零 → pipeline fail
* 預設 timeout 300 秒，可以用 `-Dsonar.qualitygate.timeout=900` 拉長（CE 排隊太久時）

### 跳過測試的專案怎麼處理 Quality Gate

**問題**：

* 你 `-DskipTests` → 沒 coverage 報告 → SonarQube 認知為「new code coverage 0%」
* 預設 Sonar way 第三條 `coverage on new code ≥ 80%` → 永遠 fail
* Pipeline 永遠紅 → 久了大家就把 sonar 當噪音、忽略它，**反而失去把關意義**

三個解法：

**選項一（推薦）：建立自訂 Quality Gate 移掉 coverage**

* 步驟：`Quality Gates → Create` → 取個有意義的名字（例如 `Sonar way (no coverage)`）
* 條件複製預設 Sonar way 但**移除 coverage**：
  * No new issues introduced
  * All new Security Hotspots reviewed
  * Duplicated lines on new code ≤ 3%
* 指派給專案：在你的專案頁面 `Project Settings → Quality Gate` → 選你建的 gate

**選項二：靠 fudge factor 自動忽略（不推薦）**

[官方 Quality Gates 介紹文件](https://docs.sonarsource.com/sonarqube-community-build/quality-standards-administration/managing-quality-gates/introduction-to-quality-gates/) 提到：

> Conditions on coverage are ignored until the number of new lines to cover is at least 20.

意思是：當次 commit 的新增可覆蓋行數 < 20 時，coverage 條件自動跳過。為什麼不推薦：稍微大一點的 PR（> 20 行）就會被觸發，pipeline 會莫名其妙紅起來，你還得回頭解釋為什麼 small commit 過得了、big commit 卻不行。

**選項三：補測試 + JaCoCo（長期方向）**

* 改回 `mvn verify`（不 skip）+ `pom.xml` 加 `jacoco-maven-plugin`
* JaCoCo 預設報告路徑 `target/site/jacoco/jacoco.xml`，`sonar-maven-plugin` **會自動偵測**，不用設 `sonar.coverage.jacoco.xmlReportPaths`
* 但這是「補測試 + 修 context-load 認證」的工程，超出本文範圍。詳見[官方 Java test coverage 文件](https://docs.sonarsource.com/sonarqube-community-build/analyzing-source-code/test-coverage/java-test-coverage/)

### 漸進採用：先 allow_failure 後切 false

實務上不要一上來就把 `allow_failure: false` 跟 `qualitygate.wait=true` 同時打開——你還沒看過分析結果、沒調好 QG 條件，pipeline 一定紅一片。建議節奏：

1. **第一階段**：`allow_failure: true` + `qualitygate.wait=true`
   * Pipeline 不會被擋，但你能看到 QG 結果
   * 跑幾次、看 Dashboard、決定要用預設還是自訂 QG
2. **第二階段**：QG 設成能合理通過之後，切 `allow_failure: false`
   * 正式把關，QG fail 就擋 pipeline

---

## Branch / MR pipeline 行為

[前一篇 sonarqube.md `### Community Build 在分支上的實務行為`](sonarqube.md#community-build-在分支上的實務行為) 已經說明過：Community Build 同一個 `projectKey` 只保留一份分析結果，從不同 branch 推上來會互相覆蓋。

**推薦做法**：用 `rules:` 限制只在「主分析線」跑：

```yaml
rules:
  - if: $CI_COMMIT_REF_NAME == "main" && $CI_PIPELINE_SOURCE == "push"
```

* 如果你公司把 `test-va` 當主分析線（例如先合到 `test-va` 跑整合測試，再合到 `main`），把上面的 `"main"` 改成 `"test-va"` 即可
* `$CI_PIPELINE_SOURCE == "push"` 限定**只在 push event 觸發**——避免 MR pipeline、scheduled pipeline、API trigger 等其他情境跑掉

**想在每個 MR 上看到分析結果**：必須升 Developer Edition 或改用 SonarCloud，沒有免費的迴避方法。硬要在 MR 上跑分析的話，結果會跟 main 互相覆蓋，Dashboard 變得毫無意義。

---

## Cache 與 GIT_DEPTH 細節

`cache` 與 `GIT_DEPTH` 兩段在 [.gitlab-ci.yml 範例](#gitlab-ciyml-範例maven跳過測試) 已經各放一句，這邊把背後的「為什麼」整理在一起：

* **`SONAR_USER_HOME` 為何要指到 `${CI_PROJECT_DIR}` 底下**
  GitLab CI 的 cache 機制只追蹤 project workspace 內的路徑。Scanner 預設用 `~/.sonar`（runner 容器的 home，不在 workspace），cache 抓不到，每次都要重新下載 plugin/analyzer。

* **cache.key 用 `$CI_COMMIT_REF_SLUG` 的取捨**
  按 branch 分 cache：同 branch 重複 commit 可重用、跨 branch 不污染。要更激進共用可以改成固定 key（例如 `"sonar-cache-shared"`），但這在 cache 內容版本不一致時會踩雷。

* **想加 Maven dependency cache**
  ```yaml
  cache:
    paths:
      - "${SONAR_USER_HOME}/cache"
      - .m2/repository/
  ```
  注意 cache 大小，超過幾百 MB 之後上下載 cache 反而比直接下依賴慢。如果 dependency 很多，建議用 GitLab 的 [`MAVEN_OPTS=-Dmaven.repo.local=...`](https://docs.gitlab.com/ee/ci/caching/) 把 local repo 重定向到 `${CI_PROJECT_DIR}/.m2/repository`。

* **`GIT_DEPTH: "0"` 為什麼**
  SonarQube 用 SCM blame 來判斷每行 code 的作者與 commit 時間，shallow clone 會看到 `Missing blame information for X files` warning，影響 New Code 判定（Quality Gate 只看新 code，blame 不準會錯判）。設 0 表示完整 clone。

---

## Gradle 簡短範例

公司主要用 Maven，但 Gradle 專案的接法也類似。`build.gradle` 加 plugin：

```groovy
plugins {
  id "org.sonarqube" version "5.1.0.4882"
}

sonar {
  properties {
    property "sonar.projectKey", "my-gradle-app"
  }
}
```

`.gitlab-ci.yml`：

```yaml
sonarqube-check:
  image: gradle:8.10.0-jdk21
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  cache:
    key: "gradle-sonar-cache-$CI_COMMIT_REF_SLUG"
    paths:
      - "${SONAR_USER_HOME}/cache"
  script:
    - ./gradlew sonar -Dsonar.qualitygate.wait=true
  allow_failure: false
  rules:
    - if: $CI_COMMIT_REF_NAME == "main" && $CI_PIPELINE_SOURCE == "push"
```

差別主要在：image 換成 `gradle`、scanner 不是 maven 那個 plugin 而是 `org.sonarqube` Gradle plugin、指令是 `./gradlew sonar`。其餘 cache、`SONAR_USER_HOME`、`GIT_DEPTH`、`rules` 概念都一樣。Plugin 最新版本去 [Gradle Plugin Portal](https://plugins.gradle.org/plugin/org.sonarqube) 查。

---

## 常見錯誤排查

| 錯誤訊息 | 原因 | 解法 |
|---------|------|------|
| `You're using version of scanner that requires JRE 21+` | image 的 JDK 版本不夠 | 換成 `maven:3.9-eclipse-temurin-21` |
| `Connection refused` / `Unknown host` | `SONAR_HOST_URL` 寫錯，或 GitLab Runner 連不到 SonarQube（內網 firewall） | 確認 URL；在 Runner 上 `curl $SONAR_HOST_URL/api/system/status` 確認連線 |
| `Not authorized` | `SONAR_TOKEN` 沒設、過期、或勾了 Protected 但 branch 不是 protected | 重產 token 或調整 branch 設定 |
| `Coverage is 0% on new code` | 跳過測試的預期結果 | 依[跳過測試的專案怎麼處理 Quality Gate](#跳過測試的專案怎麼處理-quality-gate) 切自訂 QG |
| `Quality Gate timed out` | CE 排隊太久（大專案 / Server 忙） | 加 `-Dsonar.qualitygate.timeout=900` |
| `Missing blame information for X files` | Git history 不夠深 | 確認 `GIT_DEPTH: "0"` 已設 |
| `Project not found` | `sonar.projectKey` 跟 Server 上不一致 | 用 SonarQube Dashboard 上看到的 key 對齊 pom.xml / `-D` 參數 |

---

## 小結

* **三步走完整合**：(1) 在 SonarQube 建專案 + 產 Project Analysis Token (2) 在 GitLab CI/CD Variables 設 `SONAR_HOST_URL` + `SONAR_TOKEN` (3) 套用 [.gitlab-ci.yml 範例](#gitlab-ciyml-範例maven跳過測試) 的 yaml。
* **跳過測試的專案**要建自訂 Quality Gate 移掉 coverage 條件，否則 pipeline 永遠紅、把關意義就消失了。
* **Community Build 不支援分支分析**，只在「主分析線」push 時跑就好；想在每個 MR 看分析結果只能升 Developer Edition / SonarCloud。

下一步：等這套在你公司的 SonarQube 跑穩之後，[第七章 07-sonarqube.md](../GitLab%20CICD/Gitlab-cicd-note-github/07-sonarqube.md) 之後會收錄 GitLab 特化的進階議題（MR pipeline 模式、protected branch 多分支策略等）。
