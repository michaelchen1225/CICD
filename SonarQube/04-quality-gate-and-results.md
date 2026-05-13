# SonarQube --- Quality Gate 與掃描結果判讀

Pipeline 接好之後，重點變成「Dashboard 顯示的數字怎麼讀、Quality Gate 沒過怎麼辦、跳過測試的專案怎麼活下來」。本篇處理**看懂結果 + 把 Quality Gate 調成符合自己團隊現實的標準**。

## 目錄

- [看懂 SonarQube Dashboard](#看懂-sonarqube-dashboard)
- [Quality Gate 在 CI 的把關](#quality-gate-在-ci-的把關)
- [跳過測試的專案怎麼處理 Quality Gate](#跳過測試的專案怎麼處理-quality-gate)
- [Issues 與 Security Hotspots 的處理流程](#issues-與-security-hotspots-的處理流程)
- [Quality Gate 測試實戰](#quality-gate-測試實戰)
- [Branch / MR pipeline 行為](#branch--mr-pipeline-行為)
- [Cache 與 GIT_DEPTH 細節](#cache-與-git_depth-細節)
- [Gradle 簡短範例](#gradle-簡短範例)
- [常見錯誤排查](#常見錯誤排查)

---

## 看懂 SonarQube Dashboard

### Quality Gate Passed / Failed 是什麼

Overview 頂端的綠勾「Passed」或紅叉「Failed」反映**最近一次掃描針對 New Code 的判定結果**。不看 Overall Code、不看歷史、只看「這次掃描相對 New Code 基準的差量」是否符合 QG 全部條件。

- **Passed**：所有條件全過（包含被 fudge factor 自動跳過的）
- **Failed**：至少一條沒過

### New Code vs Overall Code

| 分頁 | 範圍 | Quality Gate 看不看 |
|------|------|---------------------|
| **New Code** | 自 New Code 基準以來新增/修改的部分 | **看**——QG 條件全作用於此 |
| **Overall Code** | 整個 codebase 的當前狀態 | **不看**——只給人類看現況 |

呼應 Clean as You Code 哲學：不糾結既有技術債，只要新寫的 code 乾淨，整體品質自然會慢慢變好。

實際情境例子：

- Overall Code 顯示 `Coverage 0%, on 2.4k lines to cover` —— 全 codebase 沒測試，紅圈
- New Code 顯示 `Coverage -, on 0 lines to cover` —— 沒新增可覆蓋的 code，QG 不擋
- → Quality Gate Passed

**Overall Code 的紅燈不會擋 pipeline，只會擋你良心**。

### Sonar way 的四個條件

預設 QG「Sonar way」（標記為 `Default` + `Built-in`，不可直接編輯）內建四個條件，全部歸在 **Conditions on New Code**：

| 條件（Web UI 精確措辭）| 判定邏輯 |
|----------------------------|----------|
| **New code has 0 issues** | New Code 的 issue 數量必須等於 0（任何 severity、任何 software quality 都算）|
| **All new security hotspots are reviewed** | New Code 上所有 Security Hotspot 必須被人工 review（標為 Acknowledged / Fixed / Safe）|
| **New code has sufficient test coverage** | Coverage ≥ **80.0%** |
| **New code has limited duplications** | Duplicated Lines (%) ≤ **3.0%** |

注意第一條是「count = 0」而非「rating ≥ A」——任何新 issue 不分嚴重程度都會擋，這個門檻比看 rating 還嚴格。

> 條件數值會因版本略有差異，以 Web UI **Quality Gates → Sonar way** 顯示的為準。Sonar way 不能改，要客製化得按右上 `Copy`。

### fudge factor 與「假性 pass」

[官方文件](https://docs.sonarsource.com/sonarqube-community-build/quality-standards-administration/managing-quality-gates/introduction-to-quality-gates/) 寫明的兩條規則：

> - The conditions on duplication are ignored until the number of new lines is at least 20.
> - The conditions on coverage are ignored until the number of new lines to cover is at least 20.

| 被影響的條件 | 觸發跳過的門檻 |
| --- | --- |
| Duplication 相關 | 當次掃描「新增行數」< 20 |
| Coverage 相關 | 當次掃描「新增可覆蓋行數」< 20 |

**小 commit（< 20 行）下，duplication 跟 coverage 條件直接被忽略、不參與 QG 判定**。設計目的是避免 typo 修正、註解調整因比例失真（例 1 行重複 / 1 行佔比 100%）被擋。

> fudge factor 是 instance-wide 預設啟用，所有新專案都會套用；Project Administrator 可在專案層級覆寫關閉。

要讓 fudge factor 不再遮蓋實況，兩條路：

1. 等實際 PR 累積到新增 / 修改 ≥ 20 行，duplication / coverage 條件就會醒過來
2. 建自訂 QG **移掉 coverage 條件**，承認「這個專案不靠 coverage 把關」，改用其他三條擋

**第一次掃描為什麼幾乎都是 Passed**：四條條件中 `New code has 0 issues` 與 `All new security hotspots are reviewed` 屬 vacuously pass（無 New Code 即無新 issue / hotspot）；coverage 被 fudge factor 跳過；duplication 同樣以 0% 通過。要驗證 QG 是否真在運作，最快是依[§模擬實際情境](#模擬實際情境) 推一個 ≥ 20 行的 commit。

---

## Quality Gate 在 CI 的把關

預設行為：scanner 把報告送到 Server 之後就退出，pipeline 永遠 pass——不管 QG 有沒有過。

加上 `-Dsonar.qualitygate.wait=true`（或寫在 pom.xml `<properties>`）之後：

- Scanner 持續 polling `/api/qualitygates/project_status`
- QG fail → scanner exit code 非零 → pipeline fail
- 預設 timeout 300 秒，可用 `-Dsonar.qualitygate.timeout=900` 拉長（CE 排隊久時）

---

## 跳過測試的專案怎麼處理 Quality Gate

**問題**：`-DskipTests` → 沒 coverage 報告 → 一旦 New Code 累積到 ≥ 20 行可覆蓋程式碼，SonarQube 看到「new code coverage 0%」→ Sonar way 第三條 `coverage on new code ≥ 80%` 永遠 fail → pipeline 永遠紅 → 久了大家當噪音，**反而失去把關意義**。

三個解法：

| 解法 | 取捨 |
|------|------|
| **(推薦) 建自訂 QG 移掉 coverage** | QG 條件變 Issues / Hotspots / Duplications 三條，沒測試也能 meaningful 把關 |
| 靠 fudge factor 自動忽略 | 不推薦：稍大的 PR（> 20 行）就觸發、pipeline 莫名其妙紅起來，還得解釋為何小 commit 過、大 commit 不過 |
| 補測試 + JaCoCo（長期）| 改回 `mvn verify` + 加 `jacoco-maven-plugin`，預設報告路徑 `target/site/jacoco/jacoco.xml` 會被自動偵測。但是「補測試 + 修 context-load 認證」的工程，超出本文範圍 |

### 建立自訂 Quality Gate

兩種起手式：

| 起手式 | 適用情境 |
| --- | --- |
| **A. 複製 Sonar way 後修改** | 大多數條件想沿用預設、只想動一兩條（例如移掉 coverage） |
| **B. 從零建立空 QG** | 想完全照自家標準（例如 [PoC-Gate / Product-Gate](03-quality-gate-design.md)），不希望被預設條件混淆 |

**方式 A：複製 Sonar way 後修改**

跳過測試的最常見情境：

1. `Quality Gates` → 點 `Sonar way` → 右上 `Copy`（不是 `Edit`，Sonar way 不能直接改）→ 取名例如 `Sonar way (no coverage)`
2. 進入新 QG → 找 `New code has sufficient test coverage` → 右側垃圾桶 `Delete`

剩下三條：`New code has 0 issues` / `All new security hotspots are reviewed` / `New code has limited duplications`。

**方式 B：從零建立空 QG**

當條件設計跟 Sonar way 差異大時（典型例：PoC-Gate / Product-Gate）。

1. `Quality Gates` → 左側 `Create` → 輸入名稱 → `Save`
2. 點 `Add Condition`，依序填三個欄位：
   - **Where?**：選 `On New Code`（遵循 Clean as You Code）
   - **Quality Gate fails when**：選 metric（`Security Rating`、`Coverage`、`Duplicated Lines (%)`、`Security Hotspots Reviewed`…）
   - **Operator + Value**：Rating 類選 `is worse than` + A/B/C/D；比例類選 `is less than` / `is greater than` + 數值

範例（[PoC-Gate](03-quality-gate-design.md#qg-1poc-gate)）：

| # | 條件 | Where | Operator | Value |
| --- | --- | --- | --- | --- |
| 1 | Security Rating | On New Code | is worse than | C |
| 2 | Reliability Rating | On New Code | is worse than | C |
| 3 | Maintainability Rating | On New Code | is worse than | D |

從零建立的空 QG 預設沒任何條件，所以「不加 = 不檢查」。

**指派給專案（A / B 共用）**

目標專案 → 左側 `Project Settings → Quality Gate` → 從 `Default` 切到 `Always use a specific Quality Gate` → 下拉選 → `Save`。

> 想設為所有專案的全域預設可回到 `Quality Gates → 該 QG → Set as default`，但只在「絕大多數專案都套同一組標準」時才這樣做。

---

## Issues 與 Security Hotspots 的處理流程

### Issues 分類

每個 issue 有三個維度的屬性：

**Software Quality**（一個 issue 可能同時影響多個 quality，每個各自有 severity）：

| Software Quality | 意義 | 範例 |
|------------------|------|------|
| **Security** | 可被惡意利用的問題 | SQL injection、未驗證輸入、硬編碼密碼 |
| **Reliability** | 程式執行時會出錯 | NullPointerException 風險、未關閉的 resource |
| **Maintainability** | 影響可讀性 / 可維護性但不影響執行 | 過度複雜的函式、重複 code、Dead code |

**Severity**：Blocker / High / Medium / Low / Info（高到低）。

**Code attribute (Clean Code Attribute)**：分四大類（**Adaptability / Consistency / Intentionality / Responsibility**），每類再細分子屬性（例 `Intentionality | Not clear`）。屬性說明 issue 違反了乾淨程式的哪一面向，**不參與 QG 條件判定**，但 review 時有助於理解規則動機。

`Issues` 分頁可用 Software Quality / Severity / Scope / Status / Assignee / File 多維度過濾。

### Issues 的處置動作

詳情頁左下 `Open ⌄` 下拉切換狀態：

| 狀態 | 意義 | 對 QG 的影響 |
| --- | --- | --- |
| **Open** | 預設、剛被偵測到 | 計入 |
| **Accept** | 知道這個 issue 但決定不立即修 | **不再計入** |
| **False Positive** | 分析結果不對、誤報 | **不再計入** |

> Overview 頁的 `Accepted issues` 即標為 `Accept` 的數量。修好程式碼後下次掃描 SonarQube 會自動轉為 `Closed`，不需手動。

標 `False Positive` / `Accept` 需要 `Administer Issues` 權限——避免有人為了讓 pipeline 過綠把所有 issue 都標 False Positive。

### 工程師處理一個 Issue 的標準流程

1. **鎖定子集**：`Issues` 列表勾 `Issues in new code`、`Software Quality` 看 `Security`/`Reliability`、`Severity` 先選 `Blocker` + `High`
2. **讀懂 issue**：點標題進詳情頁，看四處：
   - 左上規則連結（如 `java:S1710`）→ SonarSource 規則文件，含 compliant / non-compliant 範例
   - 右側 `Software qualities impacted`：影響的 quality 與 severity
   - 右側 `Code attribute`：規則保護的 Clean Code 面向
   - 三個 tab：`Why is this an issue?` / `How can I fix it?` / `More info`（CWE / OWASP 等外部引用）
3. **三選一處置**：
   | 路徑 | 何時用 | 動作 |
   | --- | --- | --- |
   | (a) 改程式碼修掉 | 規則確實點出實際問題 | 依 `How can I fix it?` 修改 |
   | (b) 標 `False Positive` | 分析誤判、規則在此 context 不適用 | comment 寫明為什麼是誤報 |
   | (c) 標 `Accept` | 是真問題但這次不修（時程、優先級） | comment 寫明後續處理計畫 |

   (b)/(c) 都會留 audit trail（誰、何時、為什麼）
4. **路徑 (a) 推上 git**：pipeline 重跑 → 修好的 issue 在新掃描中自動轉為 `Fixed`

   ![alt text](image-1.png)
5. **確認 QG 變綠**：`New Code` 分頁的 `New issues` 降到目標值、QG 從 Failed 變 Passed、GitLab pipeline 從紅變綠

### Security Hotspots 的特殊流程

Security Hotspot **不等於** Vulnerability：

- **Vulnerability**：SonarQube 確定是漏洞 → 直接列為 Issue → 修就對了
- **Security Hotspot**：看到敏感程式碼模式（如 `Math.random()`、SSL 設定、處理使用者上傳檔名），**無法判斷在你的 context 下是否真為漏洞** → 需人工判斷

舉例：`Math.random()` 在「產生遊戲動畫亂數」context 下沒問題，在「產生 session token」context 下是嚴重漏洞——機器看不出差別。

**Hotspot 四個平行狀態**（對應 `Security Hotspots` 頁面四個分頁，互相切換）：

| 狀態 | 意義 |
| --- | --- |
| **To review** | 預設，等人 review |
| **Acknowledged** | 看過了、知道有風險，目前接受 |
| **Fixed** | 已修掉風險 |
| **Safe** | review 過、確認在此 context 下不是真風險 |

額外兩個分組維度：**Review priority**（High / Medium / Low，非 severity）與 **Category**（`SQL Injection`、`DoS`、`Weak Cryptography`…）。預設按 Review priority 由高到低分組。

Sonar way 第二條要求 New Code 上的 Hotspot 100% 從 `To review` 移到其他三個狀態之一。流程：

1. 進 `Security Hotspots`，從 `To review` 依 Review priority 由 High 看起
2. 點開 hotspot 看五個 tab：`Where is the risk?` / `What's the risk?` / `Assess the risk?` / `How can I fix it?` / `Activity`
3. 右上 `Review` 按鈕選 `Acknowledged` / `Fixed` / `Safe`，寫 comment

> Overview 的 Security Hotspots 評等與 review 完成度直接掛鉤——`4, E` 即 4 個全在 `To review`、評等被打到 E；逐個 review 後評等回升到 A。Hotspot 被 review 過跟「程式有沒有真的修」是兩件事——SonarQube 在意的是「人有沒有看過並做出判斷」。

---

## Quality Gate 測試實戰

### 漸進採用：先 allow_failure 後切 false

實務上不要一上來就把 `allow_failure: false` 跟 `qualitygate.wait=true` 同時打開——還沒看過分析結果、沒調好 QG 條件，pipeline 一定紅一片。建議節奏：

1. **第一階段**：`allow_failure: true` + `qualitygate.wait=true`。Pipeline 不被擋，但能看到 QG 結果（job log 印 `QUALITY GATE STATUS: PASSED/FAILED`）。跑幾次、看 Dashboard、決定要用預設還是自訂 QG
2. **第二階段**：QG 設成能合理通過後切 `allow_failure: false` 正式把關

切換時機：連續 3~5 次 push QG Passed，且每次都是「真的 pass」（非 fudge factor 跳過）→ 可以切了。

### 模擬實際情境

要真的看到 QG 從 pass 切換成 fail：

1. repo 加一個新 Java 檔，內容塞 30 行以上的 method（**且都是可覆蓋的**——不能只是 imports、空行、註解）
2. push 到分析線（例 `test-va`）→ 觸發 pipeline
3. SonarQube 掃完 → 進 `New Code` 分頁應看到：
   - `New Code Lines: 30+`
   - `Coverage on New Code: 0.0%`（紅）
   - QG **Failed**（如果還用預設 Sonar way）

這時候 pipeline——`allow_failure: true` 黃色警告但不擋；`false` 紅色失敗。看到這個結果之後，再切自訂 QG 移掉 coverage，重跑就會 pass。

---

## Branch / MR pipeline 行為

Community Build 同一個 `projectKey` 只保留一份分析結果，不同 branch 推會互相覆蓋。**推薦做法**用 `rules:` 限制只在主分析線跑：

```yaml
rules:
  - if: $CI_COMMIT_BRANCH == "test-va" && $CI_PIPELINE_SOURCE == "push"
```

- 公司把 `test-va` 當主分析線（先合 `test-va` 跑整合測試，再合 `main`）
- `$CI_PIPELINE_SOURCE == "push"` 限定只在 push event 觸發——避免 MR / scheduled / API trigger pipeline 跑掉

**想在每個 MR 上看到分析結果**：必須升 Developer Edition、改用 SonarCloud，或接受非官方 plugin 的代價（見 [06-mr-pr-integration.md](06-mr-pr-integration.md)）。硬要在 MR 上跑分析會讓結果與主分析線互相覆蓋，Dashboard 變得毫無意義。

---

## Cache 與 GIT_DEPTH 細節

- **`SONAR_USER_HOME` 為何指到 `${CI_PROJECT_DIR}` 底下**：GitLab CI cache 只追蹤 workspace 內的路徑。Scanner 預設用 `~/.sonar`（runner 容器 home，不在 workspace），cache 抓不到，每次重新下載 plugin / analyzer

- **cache.key 用 `$CI_COMMIT_REF_SLUG`**：按 branch 分 cache，同 branch 重複 commit 可重用、跨 branch 不污染。更激進共用可改固定 key，但 cache 內容版本不一致時會踩雷

- **想加 Maven dependency cache**：

  ```yaml
  cache:
    paths:
      - "${SONAR_USER_HOME}/cache"
      - .m2/repository/
  ```

  注意超過幾百 MB 後上下載 cache 反而比直接下依賴慢。dependency 多時建議用 `MAVEN_OPTS=-Dmaven.repo.local=${CI_PROJECT_DIR}/.m2/repository` 把 local repo 重定向

- **`GIT_DEPTH: "0"` 的必要性**：GitLab Runner 預設只 clone 最近 20 個 commits（shallow clone）以加速；SonarQube 需靠 `git blame` 取得每行 code 的最後修改時間，藉此判定 New Code vs Overall Code。

  舉例：專案 200 commits、New Code 基準 = 50 個 commits 前——
  - `GIT_DEPTH: 20`：blame 缺失，scanner 印 `Missing blame information for X files`，整檔可能被誤判為 New Code（或反之）
  - `GIT_DEPTH: 0`：完整 clone，blame 完整、New Code 判定準確

  錯判的具體影響：原本應該只看 38 行新增 code 的 QG，可能因 blame 缺失被當成「整個 6.4k 行檔案都是新的」→ Issues / Coverage 數字爆炸、QG 莫名其妙 fail。代價是 clone 變慢（大型 repo 多幾十秒），但對 SonarQube 必要

---

## Gradle 簡短範例

公司主要用 Maven，Gradle 接法類似。`build.gradle`：

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
    - if: $CI_COMMIT_BRANCH == "test-va" && $CI_PIPELINE_SOURCE == "push"
```

差別：image 換成 `gradle`、scanner 用 `org.sonarqube` Gradle plugin、指令是 `./gradlew sonar`。其餘 cache、`SONAR_USER_HOME`、`GIT_DEPTH`、`rules` 概念都一樣。Plugin 最新版去 [Gradle Plugin Portal](https://plugins.gradle.org/plugin/org.sonarqube) 查。

---

## 常見錯誤排查

| 錯誤訊息 | 原因 | 解法 |
|---------|------|------|
| `You're using version of scanner that requires JRE 21+` | image JDK 版本不夠 | 換成 `maven:3.9-eclipse-temurin-21` |
| `Connection refused` / `Unknown host` | `SONAR_HOST_URL` 寫錯或 Runner 連不到（內網 firewall）| 確認 URL；在 Runner 上 `curl $SONAR_HOST_URL/api/system/status` |
| `Not authorized` | `SONAR_TOKEN` 沒設、過期、勾了 Protected 但 branch 不是 protected、或 `sonar.projectKey` 對不上 | 重產 token、調 branch 設定，並確認 pom.xml projectKey 與 Server 一致 |
| `Coverage is 0% on new code` | 跳過測試的預期結果 | 切自訂 QG 移掉 coverage 條件 |
| `Quality Gate timed out` | CE 排隊太久 | `-Dsonar.qualitygate.timeout=900` |
| `Missing blame information for X files` | Git history 不夠深 | 確認 `GIT_DEPTH: "0"` 已設 |
| `Project not found` | `sonar.projectKey` 跟 Server 不一致 | 用 Server 上看到的 key 對齊 pom.xml |

下一步：[05-quality-profile.md — Quality Profile](05-quality-profile.md)。
