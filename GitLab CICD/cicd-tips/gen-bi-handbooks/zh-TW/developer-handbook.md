# gen-bi 工程師工作手冊

> 若只想快速上手，請先閱讀 **Cheat Sheet**（`cheat-sheet.md`）。

## 1. Branch 架構

```text
feature/* 或 fix/*
    │  MR（CI：maven_ci + sonar_scan + Code Review）
    ▼
   dev          每次 Push 自動建置並部署 :dev
    │  MR dev → pre-prod（Release）
    ▼
  pre-prod      Release 版本快照與 Code Review Gate（無實際部署環境）
    │  MR pre-prod → master
    ▼
  master        Production 部署入口
```

`dev` 是日常開發的整合分支。

絕大部分情況下，你只需要建立指向 `dev` 的 Merge Request，由 Reviewer 完成審查後合併。

除非有特殊情況需要在 `pre-prod` 或 `master` 進行 Hotfix，否則不要直接修改這兩個分支。若確實需要，請先與團隊溝通。

> `dev → pre-prod` 與 `pre-prod → master` 的合併及版本管理由 Release Manager 負責。詳細流程請參閱 `release-manager-handbook.md`。

---

## 2. 日常開發流程：feature → dev

```bash
git checkout dev
git pull --ff-only
git checkout -b feature/my-feature

# ... 開發 ...

git push origin feature/my-feature

# 建立 MR：feature/my-feature → dev
```

* MR Pipeline 會執行：

  * `maven_ci`（Build + Testcontainers 測試）
  * `sonar_scan`（程式碼品質與弱點分析）

* `feature → dev` 的 MR 允許使用 Squash Merge，因為 Feature Branch 屬於一次性分支，合併後即可刪除。

* 合併至 `dev` 後，Dev Push Pipeline 會自動：

  * 建置 Docker Image

  * Push 以下 Tag：

    ```text
    :dev
    :dev-<sha>-<version>
    ```

  * 執行 `deploy_dev`，使用 `:dev` Image 部署到開發環境

開發環境永遠使用最新的 `:dev` Image（Mutable Tag）。

日常開發不需要管理版本號。

---

## 3. 版本管理

專案版本定義於 `pom.xml` 的 `<version>` 欄位，格式如下：

```text
X.Y.Z-SNAPSHOT
```

（不包含 `v` 前綴，遵循 Maven 慣例）

版本管理完全由 Release Manager 負責，開發者不需自行修改版本號。

* PATCH：每次 Release 後由 CI 自動遞增
* MINOR / MAJOR：由 Release Manager 於發版前決定並調整
* 發版完成後，Release Manager 會將新版號同步回 `dev`

因此，執行 `git pull` 後看到 `pom.xml` 的版本號更新屬於正常現象。

發版時，CI 會移除 `-SNAPSHOT` 並加上 `v` 前綴，產生 Git Tag 與 Release Image Tag：

```text
vX.Y.Z
```

---

## 4. Merge 策略

| Merge 流向          | 策略        | 說明                                   |
| ----------------- | --------- | ------------------------------------ |
| `feature/* → dev` | 允許 Squash | 保持 `dev` 歷史乾淨，Feature Branch 可於合併後刪除 |

`dev → pre-prod` 與 `pre-prod → master` 採用 **FF-only** 策略，並由 Release Manager 執行。

詳細流程請參閱 `release-manager-handbook.md`。

---

## 5. Image Tag 對照表

| Tag                         | 建立時機                       | 用途                                 |
| --------------------------- | -------------------------- | ---------------------------------- |
| `:dev`                      | 每次 Push 至 `dev`            | Mutable Tag，供開發環境使用                |
| `:dev-<sha8>-<pom-version>` | 每次 Push 至 `dev`            | Immutable Tag，作為 Release Retag 的來源 |
| `:vX.Y.Z`                   | `dev → pre-prod` Release 時 | Immutable Tag，永久保留，供 Production 使用 |

---

## 6. 不應該做的事情

* 不要手動建立 Git Tag（Release 時由 CI 自動建立）。
* 不要手動修改或 Retag Image。
* 不要直接 Push 到 `dev`、`pre-prod` 或 `master`，一律透過 Merge Request。
* 不要在 `dev → pre-prod` Merge 時使用 Squash。
* 不要自行修改專案版本號，版本管理由 Release Manager 負責。
