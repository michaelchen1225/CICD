# gen-bi CI/CD 速查表

> 本文件作為日常操作快速參考使用。
>
> 架構設計、實作細節與設計原因請參閱：
> `developer-handbook.md` / `release-manager-handbook.md` / `devops-note.md`
>
> Branch 流程：
> `feature/*` → `dev` → `pre-prod` → `master`（**僅允許 Fast-Forward Merge**）
>
> 版本號僅定義於 `pom.xml` 中專案本身的 `<version>`，格式為：
>
> `X.Y.Z-SNAPSHOT`

---

# Developer

## 開發新功能

```bash
git checkout dev && git pull --ff-only
git checkout -b feature/my-feature
# ... 開發與提交 ...
git push -u origin feature/my-feature
# 建立 MR：feature/my-feature → dev（允許 Squash）→ Merge
```

合併後，`dev` Pipeline 會自動建置並部署最新的 `:dev` Image。

日常開發不需要調整版本號。

---

# Release Manager

## 發版檢查清單

* [ ] Dev Pipeline 全部成功
* [ ] 如有需要，已於 `dev` 更新 MINOR / MAJOR 版本號
* [ ] 沒有開啟中的 `pre-prod → master` MR
* [ ] 已建立並合併 `dev → pre-prod` MR（Squash OFF）
* [ ] `version_guard` 與 `guard_no_squash_preprod` 通過
* [ ] `vX.Y.Z` Tag 與 Release Image 已建立，且 `pre-prod` 版本已自動遞增
* [ ] 已將 `pre-prod` 同步回 `dev`
* [ ] `pre-prod → master` MR 已合併
* [ ] `deploy_production` 已成功執行
* [ ] Sentry 中已存在 Release 與 Production Deployment Event

## 升級 MINOR 或 MAJOR 版本（僅在需要時）

> 版本格式：`MAJOR.MINOR.PATCH`

發版前，於 `dev` 分支修改 `pom.xml` 中專案本身的 `<version>`（不要修改 `<parent>` 裡的版本）：

```xml
<version>0.2.0-SNAPSHOT</version>   <!-- MINOR 升版 -->
<version>1.0.0-SNAPSHOT</version>   <!-- MAJOR 升版 -->
```

PATCH 版號不需手動修改。

每次 Release 完成後，CI 會自動遞增 PATCH。

## 發版：dev → pre-prod

```text
1. 確認 dev Pipeline 全部成功（Registry 中已存在對應 Image）
2. 確認目前沒有開啟中的 pre-prod → master MR
3. 建立 dev → pre-prod MR，設定 Squash = OFF，完成 Review 後合併
4. 等待 pre-prod Pipeline 完成：
   - 建立 Git Tag vX.Y.Z
   - Retag Release Image
   - 自動遞增 pom.xml 版本號
5. 將 pre-prod 同步回 dev（見下節）
```

Release 版本即為 `pom.xml` 中的版本號移除 `-SNAPSHOT` 後的結果。

## 將 pre-prod 同步回 dev（每次發版後必做）

```bash
git checkout dev
git fetch origin
git merge origin/pre-prod   # 拉回 CI 自動產生的版本遞增 Commit
git push origin dev
```

若省略此步驟，下次建立 `dev → pre-prod` MR 時將出現 Fast-Forward 衝突。

解法也是執行上述指令。

## 部署到 Production：pre-prod → master

```text
1. 建立 pre-prod → master MR
   → Code Review
   → Merge

   （此 MR 不會觸發 Pipeline）

2. Master Pipeline 完成後：

   Pipelines
   → deploy_production
   → Run Job

   不需填寫任何變數
   IMAGE_TAG 預設為最新 vX.Y.Z

3. 確認 Sentry 中已存在：
   - Release
   - Production Deployment Event
```

## Rollback

```text
於 Master Pipeline：

deploy_production
→ Run Job

Variables：
IMAGE_TAG = v<previous-version>
```

同一個 Job 會同步記錄 Sentry Release 與 Production Deployment Event，不需執行第二個 Job。

驗證方式：

* 舊版本 Release 應出現新的 Deployment Event
* 最新版本 Release 不應新增 Deployment Event

---

# 五大原則

1. `dev → pre-prod` 合併時必須關閉 Squash；只有 `feature → dev` 允許 Squash。
2. 每次 `dev → pre-prod` 發版完成後，必須立即將 `pre-prod` 同步回 `dev`。
3. 同一時間只能進行一個 Release。當 `pre-prod → master` MR 開啟時，不可再合併新的變更到 `pre-prod`。
4. PATCH 版號由 CI 自動遞增；MINOR 與 MAJOR 必須手動修改 `pom.xml`。
5. `pom.xml` 版本號必須符合 `X.Y.Z-SNAPSHOT` 格式，且移除 `-SNAPSHOT` 後的版本必須大於目前最高的 `v*` Tag。

---

# 版本格式對照

| 用途                      | 格式               | 範例               |
| ----------------------- | ---------------- | ---------------- |
| `pom.xml` 專案版本          | `X.Y.Z-SNAPSHOT` | `0.1.5-SNAPSHOT` |
| Git Tag / Release Image | `vX.Y.Z`         | `v0.1.5`         |
| Rollback 的 `IMAGE_TAG`  | `vX.Y.Z`         | `v0.1.4`         |

---

# 常見問題對照

| 問題                                 | 參考文件                          |
| ---------------------------------- | ----------------------------- |
| `version_guard` 報版本已存在、格式錯誤或版本倒退   | `release-manager-handbook.md` |
| Merge Request 發生 Fast-Forward 衝突   | `release-manager-handbook.md` |
| Push 到 pre-prod 後沒有觸發 Pipeline     | `release-manager-handbook.md` |
| `git push` 連線失敗（could not connect） | `devops-note.md`              |
| Production MR 沒有 Pipeline 或無法合併    | `devops-note.md`              |
| Rollback 未出現在 Sentry               | `release-manager-handbook.md` |
