# gen-bi Release Manager 手冊

> Release Manager 負責版號管理、發版、Code Review 與 Production 部署。
>
> Branch 流程：
> `feature/*` → `dev` → `pre-prod` → `master`（**僅允許 Fast-Forward Merge**）

---

## 1. 發版：dev → pre-prod

專案版本定義於 `pom.xml` 中的 `<version>`，格式如下：

```text
X.Y.Z-SNAPSHOT
```

Release Manager 負責決定版本號的調整。

下一個 Release 版本就是將 `dev` 分支上的版本號移除 `-SNAPSHOT` 後得到的版本。（PATCH 版號已於前一次發版後由 CI 自動遞增。）

* **PATCH**：每次發版後由 CI 自動遞增，不需人工操作。
* **MINOR / MAJOR**：發版前需手動修改專案的 `<version>`。

範例：

```xml
<version>0.2.0-SNAPSHOT</version>   <!-- MINOR 升版 -->
<version>1.0.0-SNAPSHOT</version>   <!-- MAJOR 升版 -->
```

> `pom.xml` 中有兩個 `<version>`：
>
> * `<parent>` 裡的 Spring Boot 版本（不可修改）
> * `<artifactId>gen-bi</artifactId>` 後面的專案版本（需修改的是這個）

### 發版流程

1. 若需要進行 MINOR 或 MAJOR 升版，先在 `dev` 更新專案版本號。
2. 確認 `dev` Pipeline 全部成功，且 Registry 中已存在對應映像檔 `:dev-<sha>-<version>`（gen-bi → Deploy → Container Registry）。
3. 確認目前沒有開啟中的 `pre-prod → master` Merge Request。
4. 建立 `dev → pre-prod` MR，設定 **Squash = OFF**，完成 Review 後合併。

合併完成後，`pre-prod` 的 Push Pipeline 會自動執行以下工作：

| Job                       | 動作                                                                         |
| ------------------------- | -------------------------------------------------------------------------- |
| `guard_no_squash_preprod` | 驗證此次 Merge 未使用 Squash                                                      |
| `image_retag_preprod`     | 將 `:dev-<sha>-<version>` 重新標記為 `:vX.Y.Z`，並建立 Git Tag `vX.Y.Z`（不重新建置 Image） |
| `prepare_next_release`    | 自動遞增 PATCH 版號，Commit 並 Push（使用 `-o ci.skip` 避免觸發新 Pipeline）                |



---

> **注意：merge 前請確認 dev pipeline 已綠燈、`:dev-<sha>-<version>` 已在 registry。**
>
> `image_retag_preprod` 的作用是將 `:dev-<sha>-<version>`（綁定本次發版 commit 的不可變 image）retag 成 `:vX.Y.Z`。
>
>若該 image 找不到，job 會 fallback 到可變的 `:dev` tag，並印出 `WARN: ... falling back to :dev`。此時發出的 `vX.Y.Z` 可能不是被 review 的那個 commit，因為 `:dev` 隨時可能被後續的 push 覆蓋。
>
>看到該 WARN 時，請核對 image digest 是否符合預期，或等 dev pipeline 綠燈後重新 cut。

### 發版後：將 pre-prod 同步回 dev

CI 會自動向 `pre-prod` 推送一個版本遞增 Commit，此 Commit 不存在於 `dev`。

若未同步回來，下次建立 `dev → pre-prod` MR 時會產生 Fast-Forward 衝突。

發版 Pipeline 完成後執行：

```bash
git checkout dev
git fetch origin
git merge origin/pre-prod   # 拉回 CI 自動產生的版號遞增 Commit
git push origin dev
```

如果已經發生 Fast-Forward 衝突，解法也是相同的。通常衝突只會出現在 `pom.xml` 的版本號。

---

## 2. 排程規則：同一時間只能有一個 Release

當 `pre-prod → master` 的 MR 開啟期間，不允許再將新的變更合併進 `pre-prod`。

### 為什麼？

Workflow Rule 會在某個分支已存在開啟中的 Merge Request 時，抑制該分支的 Pipeline。

由於所有 Release Job 都執行於 `pre-prod` Push Pipeline，因此若在 `pre-prod → master` MR 尚未關閉時繼續合併新的變更到 `pre-prod`，Release 流程會被靜默跳過。

不會顯示任何錯誤訊息。

### 正確流程

1. 完成 `dev → pre-prod`。

2. 等待 Release Pipeline 執行完成：

   * Git Tag 建立完成
   * Release Image 建立完成
   * 版本號遞增完成

3. 建立 `pre-prod → master` MR。

4. 待 Master MR 合併（或關閉）後，再開始下一輪 `dev → pre-prod` 發版流程。

### 復原流程

如果 `pre-prod` Pipeline 被跳過：

1. 關閉目前開啟中的 `pre-prod → master` MR。
2. 重新執行 Pipeline，或向 `pre-prod` Push 一個新的 Commit。
3. 等待 Release Pipeline 完成。
4. 重新開啟 `pre-prod → master` MR。

---

## 3. 版本檢查（`version_guard`）

`version_guard` 會在 `dev → pre-prod` Merge Request 中執行，並進行三項驗證。

| 錯誤                       | 意義                                          | 解法                                    |
| ------------------------ | ------------------------------------------- | ------------------------------------- |
| Invalid version format   | 專案版本不符合 `X.Y.Z-SNAPSHOT` 格式                 | 在 `dev` 修正版本號                         |
| Version already released | `vX.Y.Z` Tag 已存在，通常是因為 `dev` 未同步 `pre-prod` | 將 `origin/pre-prod` 合併回 `dev`，重新建立 MR |
| Version regression       | 版本號低於目前最高 Release Tag                       | 提高 `dev` 上的版本號                        |

上述三種錯誤都會在 CI Log 中提供詳細排查說明。

---

## 4. 部署到 Production：pre-prod → master

1. 建立 `pre-prod → master` MR，完成 Code Review 後合併。

   * 此 MR 故意不執行 Pipeline。
   * `master` 僅用於 Promotion，不會重新執行 `maven_ci` 或 `sonar_scan`。

2. 合併後，Master Push Pipeline 會執行：

   * `sentry_release_prod` 建立 Release：`gen-bi@X.Y.Z`
   * `deploy_production`（手動 Job）

     * 開啟 Pipelines
     * 點擊 **Run Job**
     * 不需填寫任何變數
     * `IMAGE_TAG` 預設為最新的 `vX.Y.Z`
     * 此 Job 會同時在 Sentry 建立 Production Deployment Event（不再需要額外的 `sentry_mark_prod`）

---

## 5. Rollback

```text
GitLab → CI/CD → Pipelines → master pipeline
→ deploy_production → Run job
→ Variables: IMAGE_TAG = v<previous-version>
→ Run
```

`deploy_production` 會自行標記 Sentry，因此 Rollback 也由同一個 Job 完成記錄，不需額外執行第二個 Job。

Sentry Release 建立採用冪等（idempotent）方式，即使回滾到一個從未正式部署過的版本，也能正確建立紀錄，不會出現「Release not found」。

### 驗證方式

* 舊版 Release 應出現新的 Deployment Event。
* 最新版 Release 不應新增 Deployment Event。

---

## 6. 冪等性（Idempotency）與安全重跑

`image_retag_preprod` 與 `prepare_next_release` 均具備冪等性，可安全手動重跑。

### `image_retag_preprod`

* 若 Git Tag 已存在，會直接跳過 Retag 與 Tag 建立動作。

### `prepare_next_release`

* 會比對遠端 `pre-prod` 中的 `pom.xml` 版本號。
* 若版本已遞增完成，Job 會直接結束，不做任何修改。

若 Release Job 因網路問題或其他暫時性錯誤失敗，直接重新執行失敗的 Job 即可，不會產生重複 Tag 或重複版本遞增。

---

## 7. Release Checklist

* [ ] Dev Pipeline 全部成功
* [ ] 若需要，已於 `dev` 更新 MINOR / MAJOR 版本號
* [ ] 沒有開啟中的 `pre-prod → master` MR
* [ ] 已建立並合併 `dev → pre-prod` MR（Squash OFF）
* [ ] `version_guard` 與 `guard_no_squash_preprod` 通過
* [ ] `vX.Y.Z` Tag 與 Release Image 已建立，且 `pre-prod` 版本已自動遞增
* [ ] 已將 `pre-prod` 同步回 `dev`
* [ ] `pre-prod → master` MR 已合併
* [ ] `deploy_production` 已成功執行
* [ ] Sentry 中已存在 Release 與 Production Deployment Event 紀錄
