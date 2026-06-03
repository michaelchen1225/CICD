# gen-bi DevOps Note

> 對象：維運此 pipeline 的 DevOps。記錄 GitLab 設定、基礎設施、設計取捨與 troubleshooting。
> 權威來源是現行 `gen-bi/.gitlab-ci.yml`；本文件解釋「為什麼」與「怎麼維運」。

---

## 1. GitLab 設定先決條件

### 分支 / 合併
- `dev` / `pre-prod` / `master` 為 Protected branch（只允許 Maintainer + CI service account push）。
- Merge method = **Fast-forward merge**（FF-only）。
- Squash 設定：允許但預設 off（feature→dev 可 squash；dev→pre-prod 由 `guard_no_squash_preprod` 強制 off）。
  - GitLab 的 squash 是 project 層單一開關，**無法 per-branch**，所以「feature 可 squash、pre-prod 不可」只能靠 CI guard 區分。

### Tag
- **Protected Tag 規則 `v*`**（Developers + Maintainers 可建立）。CI service account（Developer）能建 `v*` tag。

### CI/CD Variables
| 變數 | 用途 | 設定 |
|---|---|---|
| `GIT_PUSH_TOKEN` | CI push tag / pom bump commit | service account PAT，scope `write_repository`，**Protected + Masked** |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Bedrock | scope 涵蓋目標分支 |
| `SENTRY_DSN` / `SENTRY_AUTH_TOKEN` | Sentry release/deploy | sentry-cli 用 |

> Protected variable 只在 protected branch/tag 可見 → 三個長期分支**必須**是 protected，否則 `prepare_next_release` 取不到 `GIT_PUSH_TOKEN`、bump push 失敗。

### Merge checks
- 確認「**Skipped pipelines are considered successful**」。`pre-prod → master` 的 MR **不會產生 pipeline**（promote-only），若無此設定且要求「Pipelines must succeed」，prod MR 會卡住。

### Runner
- `lndata-1-docker` / `lndata-2-docker`：Docker executor，tag `docker-build`（build / retag / sentry 等 jobs）。
- `lndata-1`：Shell executor（`deploy_dev`，test/dev 機）。
- `lndata-oci-prod`：Shell executor（`deploy_production`，OCI production 機）。
- 部署機器需有：`internalnet` bridge network、port 8090 可用、`sudo docker`（dev=lndata-1、prod=OCI，各自一台）。

---

## 2. 版本與 image tag 契約

| 用途 | 格式 |
|---|---|
| pom.xml | `X.Y.Z-SNAPSHOT` |
| git tag / release image | `vX.Y.Z` |
| dev 可變 image | `:dev` |
| dev 不可變 image | `:dev-<sha8>-<pom版號>`（例 `dev-abc12345-0.1.5-SNAPSHOT`）|

**Producer/Consumer 契約（必須位元一致）：**
- `docker_build_dev`（producer）：sha = `$CI_COMMIT_SHORT_SHA`（前 8 碼）、version = pom 原始值（含 `-SNAPSHOT`）。
- `image_retag_preprod`（consumer）：sha = `git rev-parse origin/dev | cut -c1-8`（同樣前 8 碼）、version = pom 讀值。
  - `cut -c1-8` 刻意對齊 `CI_COMMIT_SHORT_SHA`，避免 `git rev-parse --short`（auto 長度）造成 7 vs 8 不一致而抓不到 image。
- `DEV_BRANCH` 由 `CI_COMMIT_BRANCH` 以 `sed 's|/pre-prod$|/dev|; s|^pre-prod$|dev|'` 推導（`pre-prod` → `dev`），`image_retag_preprod` 與 `guard_no_squash_preprod` 共用此推導。

---

## 3. 關鍵設計取捨（維運時別誤改）

### 3.1 build-once + promote
image 只在 dev push build 一次。`dev → pre-prod` 只 retag（`:dev-<sha>-<版號>` → `:vX.Y.Z`），**不重 build**。digest 必須一致——這是核心驗證點。

### 3.2 Loop 防呆用 `git push -o ci.skip`，**不用 `[skip ci]`**
`prepare_next_release` 的 bump push 帶 `-o ci.skip`，只跳過「這一次 push」的 pipeline。
- **為什麼不用 `[skip ci]` commit message**：`[skip ci]` 會跟著 commit 跑，當 bump commit FF 到 master 時會**連 master push pipeline 一起跳過** → `deploy_production` 不會出現。`-o ci.skip` 讓 commit 保持乾淨，master deploy 正常。
- 韌性：即使 `-o ci.skip` 沒生效，bump push 觸發的 pre-prod pipeline 也只會 idempotent no-op，不會無限迴圈。

### 3.3 `master` 是 promote-only
`.validate_rules` 與 `version_guard` 都**不含 master** → `pre-prod → master` MR 與 master push 不重跑 maven_ci/sonar。prod 靠 human review + 既有 release artifact 把關。deploy 走 master push 的 manual `deploy_production`。

### 3.4 release jobs 拆兩個 job
`image_retag_preprod`（docker:24 + DinD，做 retag + tag）與 `prepare_next_release`（alpine，純 git bump）分開。bump 的 git push 移出 DinD 環境，避開 DinD 的網路不穩，且失敗可獨立重跑。

### 3.5 網路韌性（`.ci_helpers`）
- **IPv4 `/etc/hosts` pin（主要修法）**：job container 連 gitlab.com 的 IPv6 路徑間歇不可路由，git fetch/push 約 2 秒就失敗（`Could not connect to server`）；docker→registry（走 DinD）不受影響。`.ci_helpers` 以 `dig`（`bind-tools`）解出 gitlab.com 的 IPv4 寫入 `/etc/hosts`，強制 git/curl 走 IPv4，避開壞掉的 IPv6 路徑。
- `git_net()`：`git -c http.connectTimeout=30000`。**connectTimeout 無法解決上述「連不上」**（那是 routing 問題，由 IPv4 pin 解決），僅作為「真的很慢的連線」的廉價保險。
- `retry <max> <delay> <cmd>`：所有 git/docker 網路操作的外層重試。重推同 tag/commit 是 no-op，retry 安全。
- 盲點：runner-helper 的自動 clone 跑在 `before_script` **之前**的另一個 helper container，IPv4 pin 不覆蓋該階段；若該 clone 命中 IPv6 問題，需在 runner 層處理（停用 IPv6 或設 DNS）。

---

## 4. `$CI_OPEN_MERGE_REQUESTS` 壓制陷阱

workflow rule `$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS → when: never`（避免 branch+MR 重複 pipeline）會在「分支有 open MR」時壓掉該分支的 push pipeline。

- **後果**：`pre-prod` 有 open MR（如 pre-prod→master）時，新 merge 進 pre-prod **不會跑 release pipeline**。
- **緩解**：流程上「一次只處理一個 release」（見 release-manager-handbook §2）。若要讓「pre-prod 有 open MR 時仍能重跑 release」，需調整此規則（小心別讓 dev/master 產生重複 pipeline）。

---

## 5. Troubleshooting

| 症狀 | 原因 / 處理 |
|---|---|
| `git ... could not connect`（約 2 秒失敗）| gitlab.com IPv6 路徑不通；由 `.ci_helpers` 的 IPv4 `/etc/hosts` pin 解決（見 §3.5）。持續失敗 → 上 runner 機器 `curl -v https://gitlab.com`、`dig gitlab.com` 查對外網路與 DNS |
| pre-prod push 沒跑 pipeline | pre-prod 有 open MR（§4）。關閉 MR 再 push/re-run |
| `version_guard` regression 擋住合理版號 | 有殘留的異常高版號 tag（例如測試遺留未刪）。刪掉異常 `v*` tag |
| `image_retag_preprod` 抓不到 `:dev-<sha>-<版號>` | dev pipeline 還沒 build 完（race）；job 會 retry 後 fallback `:dev`。或 dev 是 CI-only commit 沒觸發 build |
| prod MR 無 pipeline 卡住無法 merge | 開啟「Skipped pipelines considered successful」（§1）|
| rollback 沒被 Sentry 記錄 | 標記已併入 `deploy_production`（部署後自行 `releases new`→`deploys`），重跑該 job 即會記錄；確認 prod runner 能 pull `getsentry/sentry-cli` 且 `SENTRY_*` 變數可見 |
| bump commit 觸發了第二條 pre-prod pipeline | `-o ci.skip` 未生效；確認 push option 被 GitLab 接受，或改用 workflow rule 備案 |

---

## 6. 新舊 job 對照（確認舊 pipeline 已被取代）

**現行 jobs**：`maven_ci`、`sonar_scan`、`version_guard`、`guard_no_squash_preprod`、`maven_build_dev`、`sentry_release_dev`、`sentry_release_prod`、`docker_build_dev`、`image_retag_preprod`、`prepare_next_release`、`deploy_dev`、`deploy_production`（部署後內建 Sentry production 標記）、`sentry_mark_dev`。

**應已消失**：`maven_release`、`image_retag_release`、`version_check`、任何 `release/*` 或 tag-triggered release job、`gen-bi/VERSION` 檔。
