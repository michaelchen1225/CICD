# SonarQube --- Token 管理

本章補齊 [02-gitlab-ci-integration.md](02-gitlab-ci-integration.md) 未細談的 token 生命週期：四種 token 各自做什麼、產在哪、誰拿來用、該設多長、輪替怎麼做、外洩怎麼處置。Token 是 SonarQube 與 GitLab 之間唯一的信任憑證——一旦外洩，攻擊者可以寫入分析結果污染品質報告，或借 Service Account PAT 讀取整個 group 的 repo——值得獨立一章維護。

## 目錄

- [Token 四件組全景](#token-四件組全景)
- [Service Account PAT](#service-account-pat)
- [Personal PAT (read_api，選用)](#personal-patread_api選用)
- [Project Analysis Token（CI 用的 `SONAR_TOKEN`）](#project-analysis-tokenci-用的-sonar_token)
- [Global / User Analysis Token](#global--user-analysis-token)
- [Expires 策略](#expires-策略)
- [輪替 SOP](#輪替-sop)
- [外洩處置流程](#外洩處置流程)
- [常見誤用與排錯](#常見誤用與排錯)

---

## Token 四件組全景

整合鏈路上會經手四種 token，常被混為一談。先用一張表釐清各自的座標。

| Token | 產在哪 | 誰拿來用 | 權限範圍 | scope | 推薦 Expires |
|-------|--------|----------|----------|-------|-------------|
| **Service Account PAT** | GitLab → group 的 Service accounts | SonarQube **Server** 長期持有 | 所在 group 的 repo 列表、Webhook 等 admin 級 API | `api` | 365 天 |
| **Personal PAT** | GitLab → 自己的 Access Tokens | SonarQube import wizard 一次性 | 個人帳號可見的 repo | `read_api` | 用完即撤銷 |
| **Project Analysis Token** | SonarQube → 該專案 Analysis Method wizard | GitLab CI **Runner** | **單一**專案的 analysis 上傳 | 不適用 | 90 天 |
| **Global Analysis Token** | SonarQube → User → My Account → Security（admin 產） | GitLab CI Runner（跨專案共用時） | **所有**專案的 analysis 上傳 | 不適用 | 90 天 |

方向上要分兩段：

```
GitLab  ─── Service Account PAT / Personal PAT ───►  SonarQube
SonarQube  ──── Project / Global Analysis Token ────►  GitLab Runner (CI)
```

User Token 與 Global / Project Analysis Token 在 SonarQube 端是同一個產生流程的不同類型選項，差別只在「token 帶誰的權限」——詳見 [Global / User Analysis Token](#global--user-analysis-token)。

---

## Service Account PAT

**用途**：SonarQube Server 用這把長期 token 呼叫 GitLab API——列 group 底下的 repo、匯入專案、（付費版才有的）Webhook 與 MR decoration。屬於 server 級設定，產一次後寫入 `Administration → Configuration → DevOps Platform Integrations` 後就不再動。

**為何選 Service Account 而非個人帳號或一般 bot user**

- 不佔 GitLab license 席位（普通 bot user 會吃一個付費席位）
- 無法 UI 登入，token 外洩時攻擊者無法登進 GitLab 操作
- 不綁特定員工——人員流動時不會跟著失效
- group owner 即可建，不需要 instance admin

**產生步驟**：細節見 [02-gitlab-ci-integration.md §B.1](02-gitlab-ci-integration.md#b1-在-gitlab-建立-service-account-並產-pat)。重點摘要：

1. `Settings → Service accounts → Add service account`
2. Service accounts 列表 → 該筆 → `Manage access tokens` → `Add new token`
3. scope **必選 `api`**（`read_api` 不足以匯入新專案）
4. 字串只顯示一次——立刻存進密碼管理工具

**為什麼 scope 要 `api` 而不是 `read_api`**：SonarQube 端有匯入新 repo、建立 webhook 的需求，這些屬於寫操作；`read_api` 雖足夠列 repo，但 wizard 會在後續流程失敗。除非確定**只**用來做唯讀整合，否則一律 `api`。

**CE 環境的替代**：GitLab CE 沒有 Service Account 功能，要由 instance admin 建一個專用 bot user 帳號。代價是佔 license 席位、bot 帳號可 UI 登入（密碼需保管）、需 instance admin 權限。

**推薦 Expires**：365 天。Server 端的整合 token 一年輪替一次，符合大多數企業安全月節奏；不建議「無到期」。

---

## Personal PAT（`read_api`，選用）

**用途**：預設**不需要**。某些 SonarQube wizard 流程會額外要求個人 PAT，用意是讓 wizard 用「執行者本人」的權限視角覆寫 Service Account——例如某個 repo 個人有權限但 Service Account 沒被加進去時的暫時繞道。

**scope**：`read_api` 足夠（只用來讀 repo 清單，不寫）。

**處置原則**：用完即撤銷，不要當常駐 token 用。長期整合一律走 Service Account PAT。

---

## Project Analysis Token（CI 用的 `SONAR_TOKEN`）

這把 token 是本章的重心——CI pipeline 每次跑掃描都靠它。在 GitLab 端被命名為 `SONAR_TOKEN`，與 `SONAR_HOST_URL` 一組成立。

### 它在 GitLab 哪裡

`Project → Settings → CI/CD → Variables`，新增 Variable：

| 欄位 | 值 |
|------|-----|
| **Key** | `SONAR_TOKEN` |
| **Value** | SonarQube wizard 產出的 Project Analysis Token 字串 |
| **Type** | Variable（非 File） |
| **Masked** | 勾 |
| **Protected** | 預設不勾（細節見下） |

旗標的取捨已寫在 [02-gitlab-ci-integration.md §在 GitLab 設 CI/CD Variables](02-gitlab-ci-integration.md#在-gitlab-設-cicd-variables)；本章只強調一點：**Protected 旗標是常見的「`Not authorized`」失敗來源**——非 protected branch（例如 `test-va`、`feature/*`）跑 pipeline 時讀不到，scanner 直接認證失敗。

### 它在 CI 裡做什麼

CI 跑 `mvn ... sonar:sonar` 時，`sonar-maven-plugin` 從環境變數 `SONAR_TOKEN` 讀出字串，當作呼叫 SonarQube Server API 的 Bearer 認證——主要打三類端點：

- `/api/ce/submit`：上傳分析報告給 Compute Engine
- `/api/ce/task`：查報告處理狀態（poll）
- `/api/qualitygates/project_status`：取 Quality Gate 結果（在 `sonar.qualitygate.wait=true` 時 scanner 會 poll 到結果出來）

`SONAR_HOST_URL` 不是 token，但同一個地方設、同樣由 plugin 讀，告訴 plugin 上述 API 要打到哪台 Server。兩者缺一不可。

### 為什麼一定走 Variable，而不寫進 `pom.xml` / `sonar-project.properties`

那兩個檔會 commit 進 git，等同於把 token 公開到所有 repo 讀者面前。即使後來 force-push 移除，git 歷史與 fork、CI cache、第三方靜態分析工具（如 GitGuardian）都可能已經留下副本——必須當作已外洩處理。

### 為什麼是 Project 級而非 Global

最小權限原則：這把 token 只能上傳「該專案」的分析結果，外洩時 blast radius 限於單一專案的 Dashboard 污染。Global 級雖然方便，所有 CI pipeline 共用一把，但外洩時整個 SonarQube 實例上的專案都可能被寫入——詳見下一節。

### 產生步驟

匯入專案後（不論 Path A / Path B），進入該專案的 `How do you want to analyze your repository?` wizard：

1. 選 `With GitLab CI/CD`
2. 第 1 段 `Add environment variables` → 點 `Generate a token`
3. **Expires 選 90 天**（理由見 [Expires 策略](#expires-策略)）
4. 字串只顯示一次——複製到密碼管理工具與 GitLab Variable

---

## Global / User Analysis Token

SonarQube 端的 analysis token 實際上有三種類型：Project、Global、User。Project 已在上一節寫完；這一節釐清剩下兩種，並說明為何 CI 場景一律推薦 Project。

| 類型 | 帶誰的權限 | 跨專案？ | 適合誰 |
|------|-----------|---------|--------|
| **Project Analysis Token** | 該專案 analysis 上傳權限 | 否 | **CI 預設選擇** |
| **Global Analysis Token** | 所有專案 analysis 上傳權限 | 是 | admin 想用一把 token 跑「自動建專案 + 掃描」流程 |
| **User Token** | 該使用者帳號的全部 SonarQube 權限 | 是（人權限所及） | API 探索、個人腳本；**不建議當 CI token** |

三種都在 `User → My Account → Security → Generate Tokens` 同一個介面產，只是建立時選 type。

**為什麼 CI 不用 Global**

- 一把外洩，所有專案都可能被攻擊者上傳偽造的分析結果
- 輪替時影響面是所有 pipeline；Project 級輪替只影響該專案

**為什麼 CI 不用 User Token**

- token 綁特定使用者帳號——使用者離職、停權、權限調整時，CI 會集體變紅，且追根時不易發現是「帳號權限」而非「token 本身」
- 帶的權限超出 analysis 上傳需求（最小權限原則違反）

簡言之：**CI 跑分析 → Project Analysis Token**，例外情境（admin 寫的全自動匯入腳本）才用 Global，User Token 留給本機 API 探索。

---

## Expires 策略

四種 token 用同一張策略表決定到期日，差別只在風險與便利的平衡。

| 選項 | 取捨 | 適用 token |
|------|------|----------|
| 30 / 60 天 | 安全性最好但輪替負擔重，容易忘 | 高敏感專案的 Project Analysis Token |
| **90 天** | 一季輪替一次，與安全月、季度檢查節奏對得上 | **Project / Global Analysis Token 預設** |
| 365 天 | 維運輕鬆，但外洩到失效最大 365 天 | Service Account PAT（一年動一次合理） |
| No expiration | **絕對不要選** | 任何情況都不選 |

到期日登記方式：寫進團隊密碼管理工具的 token 條目，到期前一週由維運提醒重產；不要寫進 repo 任何檔案（包含 README、Wiki、commit message）。

---

## 輪替 SOP

輪替動作四步，順序固定——**先保留舊 token、新 token 驗證通過後才撤舊**，避免出現「新 token 寫錯字、舊 token 已撤銷、pipeline 全紅且無路可退」的中斷期。

1. **重產 token**：到對應介面（GitLab 端 Service accounts / SonarQube 端 Analysis Method wizard）產一把新 token。複製字串。**此時舊 token 仍有效。**
2. **更新 GitLab CI/CD Variable**：覆蓋 `SONAR_TOKEN`（或 Service Account PAT 對應的 SonarQube DevOps integration 設定）的 Value。
3. **觸發一次 dummy pipeline 驗證**：例如改一個 README 空白行 push 一次，確認 scanner job 通過且 SonarQube Dashboard 收到新報告。
4. **撤銷舊 token**：到產生介面把舊 token revoke 掉。

輪替後**不需要改 `.gitlab-ci.yml` 與 `pom.xml`**——這兩個檔只引用 Variable 名稱，不存 token 字串本身。

---

## 外洩處置流程

判斷外洩來源（log 誤印、commit 進 git、screen-share 截到、第三方工具上傳）後，依下列順序處置。**前三步並行**，不要等待。

### 立即動作（並行）

- [ ] **撤銷 token**：
  - Project Analysis Token → 進該專案 `Project Settings → ... → Revoke`（路徑以實機為準；視版本可能在 `Project Settings → Permissions → Project Analysis Tokens` 或專案層 token 管理頁）
  - Service Account PAT → GitLab Service accounts 列表 → 該筆 → `Manage access tokens` → Revoke
  - Global / User Token → SonarQube `User → My Account → Security` → Revoke
- [ ] **清掉 GitLab Variable**：到 `Settings → CI/CD → Variables` 把對應 Variable 的 Value 清空或替換，避免後續 pipeline 還誤用
- [ ] **若 token 已 commit 進 git**：單純 `git revert` 或改 commit history **不夠**——commit 仍存在於 reflog、fork、CI cache、克隆者本機。必須：
  1. 立刻撤銷 token（已在第一步完成）
  2. 用 `git filter-repo` 或 BFG Repo-Cleaner 重寫歷史
  3. force-push 並通知所有協作者重新 clone
  4. **以「已外洩」為前提**接受該 token 的暴露事實，不要僥倖

### 後續動作

- [ ] **重新產一把同型 token**，按 [輪替 SOP](#輪替-sop) 流程接回 CI
- [ ] **查 SonarQube 端異常**：
  - `Administration → Projects → Background Tasks`：看是否有非預期來源的分析任務
  - `Project → Activity`：看是否有不該存在的 scan 記錄
  - Audit Logs：Community Build 是否提供需以實機為準（多數企業 audit log 功能屬付費版）；CE 環境主要靠上述兩個間接訊號
- [ ] **查 GitLab 端異常**（若外洩的是 Service Account PAT）：
  - Service Account 的 `Activity` 頁面，看 API 呼叫來源 IP 與時間
  - 受影響 group 的 repo `Audit Events`（GitLab 版本需 Premium+ 才完整）
- [ ] **事件記錄**：在內部 wiki / ticket 系統留下時間軸、影響範圍、處置動作，作為事後檢討依據

---

## 常見誤用與排錯

- **`Not authorized` 失敗**：多半是 `Protected` 旗標勾起來但 branch 不是 protected branch，或 token 已撤銷／過期。檢查順序：Variable 旗標 → token 是否仍有效 → `sonar.projectKey` 與 Server 專案是否對得上。細節見 [04-quality-gate-and-results.md](04-quality-gate-and-results.md#常見錯誤碼)
- **token 寫進 `pom.xml` / `sonar-project.properties`**：等同 commit 進 git 直接公開。一律走 Variable
- **同一把 Project Analysis Token 在多個專案間共用**：違反最小權限原則，且輪替時要同時改多個 Variable，容易漏改造成部分 pipeline 紅。每個專案產自己的
- **用 User Token 跑 CI**：使用者離職、權限調整時 pipeline 會集體紅。CI 一律用 Project Analysis Token
- **`SONAR_TOKEN` 沒設 Masked**：log 會把 token 字串明碼印出來，等同外洩。Masked 旗標**必勾**
- **Expires 選「No expiration」**：永久 token 沒有自然輪替點，外洩風險隨時間累積。一律設到期日

---

下一步：[maintainability-considerations.md — Maintainability 的長期考量](maintainability-considerations.md)。
