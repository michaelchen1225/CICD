# CLAUDE.md — SonarQube 子目錄

此檔僅適用於 `SonarQube/` 底下的文件編輯，補充根目錄 `CLAUDE.md` 的通用守則。

---

## 使用者對「寫文章」的明確要求

以下是在這個系列的編寫過程中使用者反覆指出、必須遵守的原則。違反任一條都會被退回重寫。

### 1. 是寫文章，不是跟使用者對話

* **禁用第二人稱對話口吻**：不要寫「你看截圖就知道」「我會選」「我們來做」這種句子
* **改用第三人稱分析語氣**：「Sonar way 四條條件全部列在 X 底下」「實務上的驗證方式是 Y」
* 「我」「你」「我們」原則上不出現在正文敘述中（操作步驟的祈使句例外，例如「在 Web UI 點 `Create`」）

### 2. 不引用文件裡沒有的素材

* 文章目前**沒有截圖**——任何 `[XX 截圖](...)` 連結、`如圖所示`、`從截圖可見` 都不能寫
* 截圖位置可保留 `![alt](placeholder)` 由使用者後補；正文敘述要能在無圖狀態下自洽
* 不要把單次掃描看到的具體數字（例如「13 個 issue / 4 個 hotspot / 0% coverage」）當成通則寫進文章——那是個案，要寫成一般化敘述

### 3. 不要亂掰、不要看圖說故事

* 使用者多次點名兩種失敗：
  * **亂掰**：對機制只憑直覺猜測（例：把「第一次掃描沒有 New Code 基準」當成事實寫，但實際上機制因 New Code definition 而異）
  * **看圖說故事**：只是把 Web UI 上的字照抄一遍當作說明
* 處置方式：
  * 寫機制前先決定「哪些是可驗證的事實」「哪些是隨配置變動的細節」，後者要明說「精確機制不在文中細究」
  * 寫 UI 操作時要分析其背後機制（例如 New Code definition 三個選項各自的差異與適用情境），不是逐字翻譯畫面

### 4. UI / 文案要與實際版本一致

* 所有 Web UI 文字（Quality Gate 條件名稱、按鈕文字、狀態名稱）以使用者實機 SonarQube Community Build 上看到的為準
* 寫之前如不確定，先請使用者提供截圖或文字確認；不要憑記憶或舊版文件寫
* 已知**容易踩到舊版文案**的地方（每次寫到都要再確認）：
  * Sonar way 四條條件名稱（目前實機是 `New code has 0 issues` / `All new security hotspots are reviewed` / `New code has sufficient test coverage` / `New code has limited duplications`，**不是**舊版的 `No new issues are introduced` 等）
  * Issue 分類用 MQR 模式（Security / Reliability / Maintainability + Blocker / High / Medium / Low / Info），**不是**舊版的 Bug / Vulnerability / Code Smell + Critical / Major / Minor
  * Issue Resolution 是 `Fixed / False positive / Accepted`（`Accepted` 取代舊版 `Won't Fix`）
  * Hotspot Review 狀態是 `Acknowledged / Fixed / Safe`

### 5. 引用官方文件要忠實、完整

* 引用官方文件時抄完整段，不要只抄半句後自行延伸
* 已踩過的坑：fudge factor 規則同時涵蓋 **duplication** 與 **coverage** 兩條，曾只寫 coverage 那條就被退回
* 引用時保留英文原文 + 中文整理表，方便讀者對照

### 6. 重寫時的範圍

* 使用者指定「重寫這段」就重寫**這段**——不要趁機改其他段
* 使用者切文章（例如砍 line 407 之後）就**嚴格切**，不要為了「銜接更順」自行保留模糊地帶

---

## SonarQube 領域上特別注意的點

### MQR vs Standard Experience（最重要）

* 從 SonarQube 10.8+ 起，Issues 預設採 **Multi-Quality Rule (MQR) 模式**：以 Software Qualities（Security / Reliability / Maintainability）+ Severity（Blocker / High / Medium / Low / Info）分類
* 舊版 Standard Experience 仍存在但是 legacy：以 Issue Type（Bug / Vulnerability / Code Smell）+ Severity（Blocker / Critical / Major / Minor / Info）分類
* 寫到 Issues 章節**先確認使用者實機是哪一種模式**——使用者目前的環境是 MQR
* 線索：Web UI 上看到 `Accepted issues` 字樣表示是 MQR；舊版會寫 `Won't Fix`

### Quality Gate 的判定範圍

* **Quality Gate 只判定 New Code**——`Conditions on New Code` 底下那幾條才會擋 pipeline
* `Conditions on Overall Code` 在預設 Sonar way 上是空的；Overall Code 上的紅燈（既有 issue、低 coverage）**不會**讓 Quality Gate fail
* 寫到「Passed / Failed 為什麼」時必須先講這個前提，否則所有後續分析都會偏掉

### fudge factor 兩條規則必須一起寫

* Duplication 條件：當次掃描「新增行數」< 20 時被忽略
* Coverage 條件：當次掃描「新增可覆蓋行數」< 20 時被忽略
* fudge factor 是 instance-wide 預設啟用、專案管理員可在專案層級覆寫
* 寫「第一次掃描為什麼 pass」時，這兩條是構成「假性 pass」的關鍵之一，不能只提 coverage

### Sonar way 不可改

* Sonar way 是內建 Quality Gate，無法直接編輯——要客製化必須 `Copy` 一份再改
* 條件數值（80% / 3%）會因版本略有差異，文章要附「請以 Web UI 顯示為準」的警語
* 自訂 QG 移除 coverage 條件是「跳過測試的專案」最常見的解法，務必詳細寫步驟

### Community Build 的限制

* 不支援 Branch analysis 與 MR decoration——非 main 分支會混在一起寫入同一份分析
* 文章裡建議的 `rules:` 寫法（只在特定分支跑 SonarQube）是 Community Build 下的合理變通，不是「最佳實踐」
* 不要寫「在 MR 上會看到 SonarQube 評論」這種話，那是 Developer Edition+ 才有
* **例外**：若團隊接受第三方 plugin，可用 [`06-mr-pr-integration.md`](./06-mr-pr-integration.md) 介紹的 `mc1arke/sonarqube-community-branch-plugin` 解鎖 branch analysis 與 PR decoration——但要明寫非官方、升 commercial 可能 lose data 等風險

### Token 三件組

* **Service Account PAT**（GitLab 端，scope: `api`）：給 SonarQube 用來匯入 repo / 建專案
* **Personal PAT**（GitLab 端，scope: `read_api`，可選）：個人在 SonarQube 端的身份綁定
* **Project Analysis Token**（SonarQube 端）：CI pipeline 跑 scanner 時用的 `SONAR_TOKEN`
* 三者用途不可混用；寫整合步驟時必須分清楚是哪一個

### GitLab Service Account vs Bot User

* Service Account 是 GitLab Premium+ 功能（16.x+），CE 沒有
* CE 環境的替代是「建一個專用 bot user 帳號」，行為類似但占一個 license 名額
* 寫之前先確認使用者的 GitLab 版本/方案

### Java 專案：scanner / JDK 版本

* SonarScanner for Maven 5.x 要求執行環境有 **JRE 21+**（會自動 provision，但前提是 maven image 有夠新的 JDK）
* 範例 yaml 用 `maven:3.9-eclipse-temurin-21`——如果改 image 版本，正文解釋也要同步改

### 跳過測試的專案怎麼活下來

* 三個方向：自訂 Quality Gate 移除 coverage（推薦） / 倚賴 fudge factor（不可靠） / 補測試 + JaCoCo（長期）
* 倚賴 fudge factor 不推薦的理由要寫清楚：commit 一旦 ≥ 20 行就被觸發，pipeline 會「莫名其妙」紅起來

---

## 文件結構約定

* `01-sonarqube.md` — 介紹與建置
* `02-gitlab-ci-integration.md` — 整合 GitLab CI/CD（前置作業 + Variables + pom.xml + .gitlab-ci.yml 範例）
* `03-quality-gate-design.md` — Quality Gate 設計（PoC / Product 兩階段範例 + 評級標準）
* `04-quality-gate-and-results.md` — Quality Gate 與掃描結果判讀（Dashboard / Issues / Hotspots / 自訂 QG / 測試實戰）
* `05-quality-profile.md` — Quality Profile 介紹（QP vs QG 差異、Sonar way 角色、何時才需要客製）
* `06-mr-pr-integration.md` — MR/PR 整合（用 `mc1arke/sonarqube-community-branch-plugin` 解鎖 PR decoration）
* `maintainability-considerations.md` — Maintainability 的長期考量（New Code vs Overall、AI 時代調整）

跨檔連結用相對路徑（`01-sonarqube.md#xxx`），不要寫絕對 URL。新增章節時記得同步更新根目錄 `README.md` TOC（規則見根目錄 CLAUDE.md）。
