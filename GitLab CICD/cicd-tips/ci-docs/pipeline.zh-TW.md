# CI/CD Pipeline 說明文件 - gen-bi

> 本文件依現行 `.gitlab-ci.yml` 撰寫。yaml 為權威來源；若兩者不一致，請修正本文件。

## 概覽

gen-bi 採用 **pom.xml SNAPSHOT 驅動、build-once + retag** 的 pipeline，跑在四分支模型上。
image 只在 `dev` push 時**建置一次**，之後一路到 production 都是**重新打標籤（promote/retag），不會重新 docker build**。

```text
feature/* ──MR──▶ dev ──MR──▶ pre-prod ──MR──▶ master
                   │            │                │
              build :dev    retag :dev-…      手動部署
              部署到 dev      → :vX.Y.Z        到 production
                            建立 git tag
                            bump pom PATCH
```

- **dev** — 整合。每次 push 都會 build image 並部署到 dev 機器。
- **pre-prod** — 切版。把 dev image 提升成不可變的 `vX.Y.Z` release（retag，不重建），建立 git tag，再把 pom.xml bump 到下一個 SNAPSHOT。
- **master** — promote-only 的 prod 閘門。不重新驗證；由一個 manual job 把已建好的 release image 部署到 production。

所有合併都是 **fast-forward only（FF-only）**。

---

## 版號 & tag 

> Versioning 規則：SemVer (MAJOR.MINOR.PATCH)。

| 用途 | 格式 | 範例 |
|---|---|---|
| pom.xml `<version>`（專案）| `X.Y.Z-SNAPSHOT` | `0.1.5-SNAPSHOT` |
| git tag / release image | `vX.Y.Z` | `v0.1.5` |
| dev 可變 image | `:dev` | `:dev` |
| dev 不可變 image | `:dev-<sha8>-<pom版號>` | `:dev-abc12345-0.1.5-SNAPSHOT` |
| `APP_VERSION` / Sentry release（prod）| `X.Y.Z` | `0.1.5` |
| `APP_VERSION` / Sentry release（dev）| `<short-sha>` | `abc12345` |

- `sha8` 即 `$CI_COMMIT_SHORT_SHA`（前 8 個字元）。每次在 dev push 後都會產生一個新的 image，並附帶兩個 tag：
  1. `:dev`（可變） — 代表當前 dev HEAD 的 image。每次 dev push 都會更新這個 tag，並部署到 dev 機器。
  2. `:dev-<sha8>-<pom版號>`（不可變） — 代表「某次 dev push」 build 的 image，tag 中包含 commit sha 和 pom 版號，方便後續追蹤。
   
- PATCH 由 CI 在每次 pre-prod 發版後自動 bump；MINOR/MAJOR 由工程師在 dev 上手動改 pom.xml。


> ⚠️ pom.xml 有**兩個** `<version>`：spring-boot-starter-parent 的版本，以及專案版本。CI 一律讀**專案**那個（`<artifactId>gen-bi</artifactId>` 之後的 `<version>`）。

---

## Image tagging 策略


```text
dev push 時 build 一次：
  :dev                              （可變，每次 dev push 重新部署）
  :dev-<sha8>-<pom版號>             （不可變，retag 來源）

pre-prod merge 時 promote（不重建）：
  :dev-<sha8>-<pom版號>  ──▶  :vX.Y.Z   （同一 digest）
```

`:dev-<sha8>-<版號>` 與 `:vX.Y.Z` 的 **digest 必須相同**

---

## Workflow 規則

| 來源 | 會跑 pipeline 嗎？ |
|---|---|
| MR target 為 `dev` / `pre-prod` / `master` | 會 |
| 分支 push 且該分支有 open MR | 不會（壓制以避免重複 pipeline）|
| push 到 `dev` / `pre-prod` / `master` | 會 |

> `pre-prod → master` 的 MR **不跑任何 job**，只有當 MR merge 回 master 後才會觸發 master 的部署 pipeline。
---

## Runner & environment

- lndata-1：192.168.10.201 (dev 機器)
- lndata-2：192.168.10.202 
- lndata-oci-prod：152.70.82.144 (public IP)

| Tag | Executor | 使用的 job |
|---|---|---|
| `docker-build` | Docker（`lndata-1-docker` / `lndata-2-docker`）| 所有 build / test / retag / sentry job |
| `lndata-1` | Shell（dev 機器）| `deploy_dev` |
| `lndata-oci-prod` | Shell（production 機器，OCI）| `deploy_production` |

---

## Mixins

- **`.validate_rules`** — MR 進 `dev|pre-prod`，或 push 到 `dev`（且 `pom.xml`/`src` 有變動）。master 刻意排除（promote-only）。
- **`.dev_push_rules`** — push 到 `dev`（`pom.xml`/`src`/`Dockerfile`/`docker-compose.yml` 有變動）。
- **`.preprod_push_rules`** — push 到 `pre-prod`。
- **`.master_push_rules`** — push 到 `master`。
- **`.maven_cache`** — `.m2/repository` 快取（key `maven-cache-gen-bi`）。
- **`.ci_helpers`** — pre-prod git job 共用的 `before_script`。把 `gitlab.com` 釘成 IPv4 寫進 `/etc/hosts`（見[網路韌性](#網路韌性)），並定義 `retry`、`git_net`、`pom_version`、`REMOTE` 與 git 身分。
- **`.deploy_base`** — `deploy_dev` + `deploy_production` 共用：`AWS_REGION` / `BEDROCK_MODEL_ID` / `BEDROCK_EMBEDDING_MODEL` 變數與 port 8090 檢查 `before_script`。兩個 job 都 `extends` 它且**不自帶 `before_script`**（extends 對 `before_script` 是整段覆蓋、非合併，自帶會蓋掉這道 guard）。job 各自的差異——`IMAGE_TAG` 來源、`SENTRY_ENVIRONMENT`、以及 prod 才有的 AWS precheck / Sentry 標記——留在各自 job。

---

## Stages

```text
validate → maven_build → sentry_release → docker_build
         → image_retag → prepare_release → deploy → sentry_mark
```

`sentry_release` 排在 `docker_build` 之前，讓 dev image 烘進已知的 `SENTRY_RELEASE`。
所有非部署 job 為 `interruptible: true`；部署與帶副作用的 retag job 設 `interruptible: false`。

> interruptible ：當同一 pipeline 被新的 commit 觸發時，舊 pipeline 中還沒開始的 job 會被自動取消（cancel）。這對於頻繁 push 的 dev 分支特別有用，可以節省 CI 資源。

---

## Jobs

### `validate`

#### `maven_ci`
- **觸發：** `.validate_rules`（MR 進 dev/pre-prod、push 到 dev）。
- **image：** `maven:3.9-eclipse-temurin-21` + `docker:24-dind`（Testcontainers/MongoDB，透過 `TESTCONTAINERS_HOST_OVERRIDE: docker`）。
- `mvn clean install`。

#### `sonar_scan`
- **觸發：** `.validate_rules`。`allow_failure: true`（POC 階段）。
- 跑 SonarQube scanner；MR 時帶 pull-request 參數，否則用 `-Dsonar.branch.name`。

#### `version_guard`
- **觸發：** MR target 為 `pre-prod`。
- **image：** `alpine:3`。讀專案 pom 版號，做三項驗證，失敗時各印完整 debug 步驟：
  1. 格式必須 `X.Y.Z-SNAPSHOT`。
  2. tag `vX.Y.Z` 不可已存在（抓「上次發版後忘了 sync dev」）。
  3. 不可 regression — 版號須 ≥ 現存最高 `v*` tag。

#### `guard_no_squash_preprod`
- **觸發：** push 到 `pre-prod`（`.ci_helpers` + `.preprod_push_rules`）。
- 防止 dev squash merge 到 pre-prod，此 job 在 `validate`，所以 squash 會擋掉後續 retag/tag 階段。失敗時印出 reset + 重新 merge 的修復步驟。

### `maven_build`

#### `maven_build_dev`
- **觸發：** push 到 `dev`。`mvn clean package -DskipTests`。
- Artifact：`target/gen-bi-*.jar`（1 小時）。`needs: maven_ci`（optional — CI-only 改動時會被略過）。

### `sentry_release`

#### `sentry_release_dev`
- **觸發：** push 到 `dev`。建立/set-commits/finalize Sentry release `gen-bi@<short-sha>`。

#### `sentry_release_prod`
- **觸發：** push 到 `master`。讀最新 `vX.Y.Z` tag → 建立/set-commits/finalize `gen-bi@X.Y.Z`。

### `docker_build`

#### `docker_build_dev`
- **觸發：** push 到 `dev`。
- **image：** `docker:24` + `docker:24-dind`。
- 產生一個 image 附帶兩個 tag： `:dev`（可變）&  `:dev-<sha8>-<pom版號>`。
- `needs: maven_build_dev`（artifacts）+ `sentry_release_dev`。

### `image_retag`（僅 pre-prod push）

#### `image_retag_preprod`
- **觸發：** push 到 `pre-prod`。`interruptible: false`。
- `resource_group: preprod_release`（與 `prepare_next_release` 共用）—— 讓整個發版動作在多條並行的 pre-prod pipeline 間序列化。否則兩條 pipeline 會同時通過「tag 不存在」檢查,接著競爭 `:vX.Y.Z` 的 docker push（last-writer-wins）與 git tag push。
- **image：** `docker:24` + `docker:24-dind`。
- 讀 pom 版號 → `TAG=vX.Y.Z`。
- **Idempotent：** 若 git tag 已存在，完全跳過 retag/tag。
- 否則：來源 sha 直接取 pre-prod tip 的 `$CI_COMMIT_SHORT_SHA`。FF-only model 下 pre-prod tip 就是被合併的那個 dev commit，等於 `docker_build_dev` 用的同一個 short sha，**不需** fetch `origin/dev`（那會與並行的 dev push 競態、可能推到較新且未 review 的 image）。pull `:dev-<sha8>-<版號>`（短 retry 吸收 registry 複寫延遲）；**抓不到就 fail，絕不 fallback 到可變的 `:dev`**（會破壞 build-once / promote 已驗證 artifact）。成功則 retag → `:vX.Y.Z` 並推送，最後建立並 push git tag `vX.Y.Z`。

### `prepare_release`（僅 pre-prod push）

#### `prepare_next_release`
- **觸發：** push 到 `pre-prod`。`interruptible: false`。
- `resource_group: preprod_release`（與 `image_retag_preprod` 同一 group）—— 連 pom bump/push 也序列化,兩條並行 pre-prod pipeline 不會同時從同一個 `RAW` bump、競爭 branch push。
- **image：** `alpine:3`。
- `needs: image_retag_preprod`。
- 算出 `NEXT = X.Y.(Z+1)-SNAPSHOT`，bump pom.xml 專案 `<version>`，commit 乾淨訊息，
  並用 **`-o ci.skip`** push（只跳過這次 push 的 pipeline —— commit 內**不放** `[skip ci]`，所以之後 FF 到 master 時仍會觸發 master 部署 pipeline）。

### `deploy`

#### `deploy_dev`
- **觸發：** push 到 `dev`。
- **Runner：** `lndata-1`。
- `interruptible: false`、`resource_group: dev_deploy`。`extends: .deploy_base`（繼承 region/Bedrock 變數與 port 8090 guard）。
- `before_script` 檢查 port 8090（繼承自 `.deploy_base`；見 [Port 檢查](#port-檢查deploy-before_script)）。
- pull `:dev`、寫 env file、`docker compose up -d --force-recreate`。
  `APP_VERSION` / `SENTRY_RELEASE` 用 short-sha；`SENTRY_ENVIRONMENT=dev`。
- `needs: docker_build_dev`。

#### `deploy_production`
- **觸發：** push 到 `master`，**`when: manual`**(手動觸發)。
- **Runner：** `lndata-oci-prod`。
- `interruptible: false`、`resource_group: production_deploy`。`needs: sentry_release_prod`。`extends: .deploy_base`（繼承 region/Bedrock 變數與 port 8090 guard,與 `deploy_dev` 共用）。
- `IMAGE_TAG` 預設使用最新 `vX.Y.Z` tag。若需指定特定版本可在手動觸發時使用變數覆寫。
- `before_script` 檢查 port 8090（繼承自 `.deploy_base`）。pull `:$IMAGE_TAG`、用 compose 部署、`APP_VERSION` = 去掉 `v` 的版號。
- 把實際部署的 tag 寫進純文字 artifact `deployed-image-tag.txt`（不用 dotenv —— manual re-run 會在 dotenv 上傳時衝突），作為 audit 記錄。
- **部署後在同一個 job 內標記 Sentry**（已無獨立的 `sentry_mark_prod`）：`SENTRY_RELEASE=gen-bi@<版號>`（去 `v`）→ 以 `docker run getsentry/sentry-cli` 跑 `releases new` → `finalize` → `deploys ... -e production`。`releases new` 為 idempotent：forward 時 release 已由 `sentry_release_prod` 建好（no-op），rollback 到沒發布過的版本則會建立 release 讓 deploy event 掛上（解掉 "Release not found"）。標記放在此 job，所以 rollback（重跑這個 manual job）會在同一次執行標記 Sentry——重跑已完成 pipeline 的 job 不會帶動另一個獨立標記 job。

### `sentry_mark`

#### `sentry_mark_dev`

- **觸發：** push 到 `dev`。
- `sentry-cli releases deploys gen-bi@<short-sha> new -e dev`。
- `needs: deploy_dev`。

> Production deploy 的標記已併入 `deploy_production`（見上），因此沒有 `sentry_mark_prod` job。

## Rollback 流程

1. 在 GitLab UI 上找到最新的 production deploy pipeline，重新手動觸發 `deploy_production`，覆寫 `IMAGE_TAG` 為要 rollback 的版本（例如 `v0.1.4`）。

   這一個 `deploy_production` job 會在部署後同步於 Sentry 標記 rollback（release + production deploy event），不需再跑第二個 job；`releases new` 為 idempotent，rollback 到沒 forward 發布過的版本也能正確記錄（不會 "Release not found"）。

---

## 潛在風險與設計取捨

> 本節記錄目前 pipeline 中刻意的取捨與已知限制，供維護者理解「為什麼這樣設計」以及日後強化的方向。

### 刻意的設計取捨

| 項目 | 取捨 | 緣由 / 緩解 |
|---|---|---|
| `image_retag_preprod` 來源 sha 取 `$CI_COMMIT_SHORT_SHA`，抓不到不可變 image 即硬失敗 | 依賴 FF-only 不變式（pre-prod tip == 被合併的 dev commit），由 `guard_no_squash_preprod` 保證。`:dev-<sha8>-<版號>` 短 retry 後仍抓不到就 abort，不 fallback | build-once / promote 已驗證 artifact：絕不推可變的 `:dev`（可能已被後續 dev push 蓋掉）或從競態的 `origin/dev` HEAD 解析出的 image。不可變 tag 不存在代表該發版 commit 從未 build image——是該停下的真錯誤 |
| `guard_no_squash_preprod` 為事後偵測 | 在 pre-prod push 後才檢查是否 squash，而非事前鎖定 | GitLab 的 squash / merge-method 是 project 層單一開關，無法 per-target-branch。要同時「feature→dev 可 squash、dev→pre-prod 不可」只能靠此 CI guard 區分；它在 validate stage，fail 會擋掉後續 retag/tag |
| `sonar_scan` 設 `allow_failure: true` | SonarQube 失敗不擋 pipeline | POC 階段刻意放行；品質門檻穩定後應收緊為 blocking |
| Sentry 標記分散在兩個 job | `sentry_release_prod` 與 `deploy_production` 都會跑 `releases new` + `finalize` | 刻意的冪等安全網：`deploy_production` 重複建立 release，是為了讓 rollback 到「從未 forward 發過的版本」也能掛上 deploy event（解 "Release not found"）。兩者不完全重複——`set-commits --auto` 只在 `sentry_release_prod`，deploy event 只在 `deploy_production` |
| deploy job 使用 `sudo docker` | `deploy_dev` / `deploy_production` 跑在 shell executor，CI user 有無密碼 sudo docker（等同 root）| shell executor 直接部署的固有性質。實際控制點：master 為 protected branch、`deploy_production` 為 `when: manual`，能上 prod 的人有限 |
| `GIT_PUSH_TOKEN` 拼進 push URL | `REMOTE=https://oauth2:${GIT_PUSH_TOKEN}@...`，以 inline URL push | token 為 Masked 變數，log 會遮罩；且為 inline push（未 `git remote add`），不會出現在 `git remote -v`。可強化：改用 `-c http.extraHeader` 把 token 移出 URL（defense-in-depth，非急迫）|
| deploy 收尾 `docker image prune -f` | 清掉 host 上所有 dangling image | 只刪 untagged（dangling），不是 `-a` / `docker system prune`，不會動到別服務在用的 tagged image。prod 機只跑 gen-bi；共用的 lndata-1 建議日後加 `--filter` 收窄 |

### 已知限制與後續強化

- deploy 後沒有健康檢查：`docker compose up -d` 回傳 0 不代表容器真的健康，新版若 crash-loop，job 仍可能綠燈。後續可在 deploy 收尾加 `curl .../actuator/health` gate，失敗才人工 rollback。
- 無自動 rollback：rollback 為手動（重跑 `deploy_production` 覆寫 `IMAGE_TAG`）。對目前團隊規模可接受，但應由健康檢查失敗來提示。
- pom 版號萃取邏輯有三份：`.ci_helpers` 的 `pom_version()`、`version_guard` 內嵌 awk、`docker_build_dev` 內嵌 awk。pom 結構若改需同步三處。（後兩者未 extend `.ci_helpers`，無法直接共用；若改為共用會連帶吃進 IPv4 pin 與 `bind-tools`。）
- `maven_ci` 與 `maven_build_dev` 重複編譯：dev push 時兩者都編譯同一份 source（前者含測試、後者 skipTests），約多花 1–2 分鐘。可讓 `maven_ci` 直接輸出 jar artifact 供 `docker_build_dev` 取用；屬效能、非正確性。

---

## 網路韌性

CI job 容器到 `gitlab.com` 的 IPv6 路徑間歇性不通，所以容器內的 `git fetch`/`push` 連 `gitlab.com:443` 會在約 2 秒失敗。
`.ci_helpers` 用 `dig`（`bind-tools`）把 `gitlab.com` 釘成 IPv4 寫進 `/etc/hosts`，讓 git/curl 不再走壞掉的路徑。

> 所有 git 網路操作另外都包了 `retry` + `git_net` 。

---

## Port 檢查（deploy `before_script`）

| 情況 | 結果 |
|---|---|
| port 8090 空閒 | 繼續 |
| 由 `docker-proxy` 持有、容器是 `gen-bi` | 通過（將被重建）|
| 由 `docker-proxy` 持有、非相關容器 | ERROR — 中止 |
| 由非 Docker 程序持有 | ERROR — 中止 |

偵測：`sudo ss -tlnp | grep ':8090'`。

---

## 部署時的環境變數

| 變數 | 定義位置 |
|---|---|
| `GIT_PUSH_TOKEN` | CI/CD variable（Protected + Masked）—— pre-prod 的 git tag/bump push 使用 |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | CI/CD masked variables |
| `AWS_REGION` / `BEDROCK_MODEL_ID` / `BEDROCK_EMBEDDING_MODEL` | `.deploy_base` mixin `variables`（兩個 deploy job 共用） |
| `SENTRY_DSN` / `SENTRY_AUTH_TOKEN` | CI/CD variables（sentry-cli）|
| `GENBI_IMAGE` / `APP_VERSION` / `SENTRY_RELEASE` | 由 deploy job 計算 |

gen-bi 加入共用的 `internalnet` bridge 網路；deploy job 在其不存在時自動建立。

---

## 檔案結構

```text
gen-bi/
  .gitlab-ci.yml          # pom.xml 驅動、build-once + retag 的 pipeline
  Dockerfile              # Runtime image，複製 target/gen-bi-*.jar
  docker-compose.yml      # gen-bi 服務，port 8090，image 由 ${GENBI_IMAGE} 指定
  pom.xml                 # 專案 <version> 為 release 來源（X.Y.Z-SNAPSHOT）
  ci-docs/
    pipeline.md           # 英文版
    pipeline.zh-TW.md     # 本文件
  src/
    main/resources/
      application.properties
      application-local.properties
```
