# SonarQube --- MR / PR 整合（用 community-branch-plugin 解鎖）

[02-gitlab-ci-integration.md](02-gitlab-ci-integration.md) 與 [04-quality-gate-and-results.md](04-quality-gate-and-results.md) 都點到一條限制：Community Build **官方**不支援 branch analysis 與 MR decoration，所以 `.gitlab-ci.yml` 的 `rules:` 只能鎖在 `test-va` push、MR pipeline 完全不跑 SonarQube——開發者要等程式碼合進 `test-va` 才看得到分析結果。

這篇處理的問題是：**Community Build 上有沒有方法讓 MR 階段也能拿到 SonarQube 的 inline comment 與 Quality Gate 把關？**

社群第三方 plugin [mc1arke/sonarqube-community-branch-plugin](https://github.com/mc1arke/sonarqube-community-branch-plugin)（LGPL-3.0）能在 Community Build 上解鎖這兩項功能，效果接近 Developer Edition 的原生體驗。本文用 SonarQube Community Build **v26.4.0** 為環境基準，搭配 plugin **v26.4.0**（兩者 major.minor 對齊是必要條件）整理整套導入流程，並把第三方 plugin 的風險寫在最前面，避免讀者「先裝再後悔」。

## 目錄

- [SonarQube --- MR / PR 整合（用 community-branch-plugin 解鎖）](#sonarqube-----mr--pr-整合用-community-branch-plugin-解鎖)
  - [目錄](#目錄)
  - [Plugin 解開了什麼、要付出什麼代價](#plugin-解開了什麼要付出什麼代價)
    - [解開的功能](#解開的功能)
    - [四條風險](#四條風險)
    - [導入治理建議](#導入治理建議)
  - [從官方 image 切換/安裝 plugin](#從官方-image-切換安裝-plugin)
    - [改 `docker-compose.yml`](#改-docker-composeyml)
    - [切換前必做 1：把既有 DB 資料對齊新 volume 配置](#切換前必做-1把既有-db-資料對齊新-volume-配置)
      - [官方 compose 留下的 volume 問題](#官方-compose-留下的-volume-問題)
      - [檢查目前部署狀態](#檢查目前部署狀態)
      - [狀態 A 救援：資料在匿名 volume 上](#狀態-a-救援資料在匿名-volume-上)
      - [狀態 B 遷移：把 PGDATA 搬到 `postgresql_data`](#狀態-b-遷移把-pgdata-搬到-postgresql_data)
      - [⚠️ 期間絕對不要做的事](#️-期間絕對不要做的事)
    - [切換前必做 2：拿掉舊的 `sonarqube_extensions` volume](#切換前必做-2拿掉舊的-sonarqube_extensions-volume)
      - [⚠️ 還是要叮嚀一次](#️-還是要叮嚀一次)
    - [切換後容器與資料的行為](#切換後容器與資料的行為)
    - [啟動後驗證](#啟動後驗證)
    - [替代方案：在官方 image 上手動掛 plugin](#替代方案在官方-image-上手動掛-plugin)
  - [步驟二：確認 GitLab ALM Integration](#步驟二確認-gitlab-alm-integration)
    - [2.1 設定 SonarQube 端的 GitLab DevOps Platform Integration](#21-設定-sonarqube-端的-gitlab-devops-platform-integration)
      - [02 章 Path B 走過的人怎麼確認（多數讀者）](#02-章-path-b-走過的人怎麼確認多數讀者)
      - [設定步驟（沒設過 ALM Integration 的人）](#設定步驟沒設過-alm-integration-的人)
    - [2.2 專案層級綁定 Pull Request Decoration（plugin 新增）](#22-專案層級綁定-pull-request-decorationplugin-新增)
  - [步驟三：修改 `.gitlab-ci.yml`](#步驟三修改-gitlab-ciyml)
    - [3.1 `rules:` 從一條擴成兩條](#31-rules-從一條擴成兩條)
    - [3.2 `sonar.pullrequest.*` 要不要顯式傳](#32-sonarpullrequest-要不要顯式傳)
    - [3.3 `sonar.qualitygate.wait=true` 仍然生效](#33-sonarqualitygatewaittrue-仍然生效)
  - [步驟四：端到端驗證](#步驟四端到端驗證)
  - [](#)
  - [裝 plugin 之後 New Code definition 怎麼設](#裝-plugin-之後-new-code-definition-怎麼設)
  - [不裝 plugin 的 fallback](#不裝-plugin-的-fallback)
  - [常見問題與升版](#常見問題與升版)

---

## Plugin 解開了什麼、要付出什麼代價

### 解開的功能

裝上 plugin 後，Community Build 取得以下原本要付費才有的能力：

- **多分支 Dashboard 並存**：不同 branch 的分析結果各自獨立儲存，不再像 [01-sonarqube.md `### Community Build 在分支上的實務行為`](01-sonarqube.md#community-build-在分支上的實務行為) 描述的「同 projectKey 互相覆蓋」
- **MR 上自動 inline comment**：分析完成後 SonarQube 以 Bot 身份在 GitLab MR 介面留言，列出 Quality Gate 狀態、新 issue 數、coverage 等
- **`sonar.pullrequest.key / branch / base` 三件組可用**：scanner 帶這三個參數時會建立 PR analysis（不是 branch analysis），結果進入 SonarQube UI 的 `Pull Requests` tab
- **UI 多出兩個 tab**：每個專案多了 `Branches` 與 `Pull Requests` 切換頁，可以分別檢視各分支與 MR 的分析

### 四條風險

1. **非 SonarSource 官方維護**
   Plugin README 開頭就寫 *"This plugin is not maintained or supported by SonarSource"*。SQ 官方論壇上遇到問題時，工作人員通常不會協助 plugin 相關 bug。

2. **升 Developer/Enterprise Edition 時可能 lose data**
   Plugin 文件警告：*"migrating to commercial editions may result in data loss"*。原因是 plugin 把 branch / PR 分析結果寫進 DB 的方式不一定相容於商業版的 schema。如果未來打算升 commercial，要先評估資料遷移風險。

3. **SonarQube 升版必須同步升 plugin**
   Plugin 與 SQ 的 major.minor 版本**必須對齊**——例如 plugin v26.4.x 配 SQ v26.4.x；錯版可能讓 SQ 整個起不來。Plugin 通常會在 SQ 新版釋出後幾天內跟上，但「先升 SQ 才發現 plugin 還沒對應版本」是常見踩雷情境。

4. **LGPL-3.0 授權**
   公司 legal / security policy 是否接受第三方 LGPL plugin 進 production 環境，由團隊自評。Plugin 不修改 SQ 原始碼，但會在 SQ 容器內掛 Java agent 改變執行行為。

### 導入治理建議

- 先在 PoC 環境跑一段時間（建議至少 2 週），確認 plugin 與專案規模相容、無 crash 後再 promote 到 production
- 在 GitLab / 內部 Wiki 記錄「目前 SQ 版本 ↔ plugin 版本」對應表，避免升版時錯位
- 預留退場路徑（[常見問題與升版](#常見問題與升版)有說明）：拔 plugin 不會弄壞 main branch 資料，但 branch / PR 歷史會「不可見」

---

## 從官方 image 切換/安裝 plugin

雖然第一篇用的已經是包含 plugin 的 image，不過這邊還是提一下如果你用的是官方 image 或 [docker compose](https://github.com/SonarSource/docker-sonarqube/tree/master/example-compose-files/sq-with-postgres)，如何替換以及會踩到的坑。

### 改 `docker-compose.yml`

如果以前用的是官方 `sonarqube:community`。除了換 image，順便把兩個 volume 寫法的潛在地雷修掉——對齊 [plugin 作者的 reference compose](https://github.com/mc1arke/sonarqube-community-branch-plugin/blob/master/docker-compose.yml) 的做法：

```yaml
services:
  sonarqube:
    image: mc1arke/sonarqube-with-community-branch-plugin:26.4.0.121862-community
    container_name: sonarqube
    depends_on:
      db:
        condition: service_healthy
    read_only: true
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: admin
      SONAR_JDBC_PASSWORD: admin
    volumes:
      # 不持久化 /opt/sonarqube/extensions —— plugin 已烤進 image
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_temp:/opt/sonarqube/temp
    tmpfs:
      - /tmp:size=256M,mode=1777
    ports:
      - "9000:9000"

  db:
    image: postgres:17
    container_name: sonarqube-db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin
      POSTGRES_DB: sonar
    volumes:
      # 雙 volume：第二行明確掛在 PGDATA，搶在 image VOLUME directive 之前
      - postgresql:/var/lib/postgresql
      - postgresql_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  sonarqube_data:
  sonarqube_logs:
  sonarqube_temp:
  postgresql:
  postgresql_data:
```

要記住的四件事：

- **預建 image 已內掛 Java agent**：`SONAR_WEB_JAVAADDITIONALOPTS` 與 `SONAR_CE_JAVAADDITIONALOPTS` 在 image build 時就已塞好 `-javaagent:./extensions/plugins/sonarqube-community-branch-plugin-XX.jar=web` 與 `=ce` 兩段參數，docker-compose 不需要再加環境變數

- **版本對齊規則**：image tag 完整對應 upstream SQ 版本（含 build 號）。Plugin 26.4.x 系列的 image 大約是 `26.4.0.121862-community` 這種格式，建議在 [Docker Hub mc1arke/sonarqube-with-community-branch-plugin](https://hub.docker.com/r/mc1arke/sonarqube-with-community-branch-plugin/tags) 確認當下要用的 tag 後再寫死

- **`sonarqube_extensions` volume 拿掉**：plugin / driver 隨 image 烤入，持久化這個目錄反而會踩 named-volume override bug（已存在的 volume 蓋住新 image 的內容，導致升 plugin 後 plugin jar「不見」、SQ 起不來）。代價是 Marketplace UI 額外安裝的 plugin 不會持久化——對「只用 community-branch-plugin」的環境不痛。既有部署若原本有這顆 volume，下一節 [切換前必做 2](#切換前必做-2拿掉舊的-sonarqube_extensions-volume) 會處理

- **db 改成雙 volume 寫法**：第二行 `postgresql_data:/var/lib/postgresql/data` 明確把 named volume 掛在 PGDATA 位置，搶在 postgres image 的 `VOLUME /var/lib/postgresql/data` directive 之前——匿名 volume 根本不會被建立、container 重建時資料安全。第一行 `postgresql:/var/lib/postgresql` 順便捕捉父目錄雜物（`.psql_history` 等）。既有部署若原本只有單行寫法（不論掛在父目錄或 PGDATA），下一節 [切換前必做 1](#切換前必做-1把既有-db-資料對齊新-volume-配置) 會處理遷移

**改完 yaml 先不要 `docker compose up -d`**——接下來兩節各有一個既有部署必做的前置處理，跳過會丟資料或讓 SQ 起不來。

### 切換前必做 1：把既有 DB 資料對齊新 volume 配置

新 yaml 改用雙 volume 寫法（`postgresql:/var/lib/postgresql` + `postgresql_data:/var/lib/postgresql/data`）。直接 `docker compose up -d` 會發生什麼，取決於原 yaml 的 db volume 寫法——三種狀態各自的處置不同，**選錯會直接丟資料**。

#### 官方 compose 留下的 volume 問題

> 以下假設使用的是 SonarSource 官方 `sq-with-postgres` compose 範本。

官方範本中的 PostgreSQL service 只有掛載：

```yaml
volumes:
  - postgresql:/var/lib/postgresql
```

但 PostgreSQL 官方 image 本身在 Dockerfile 內宣告了：

```dockerfile
ENV PGDATA=/var/lib/postgresql/data
VOLUME /var/lib/postgresql/data
```

由於 image 已經宣告 `VOLUME /var/lib/postgresql/data`，若 compose 沒有明確覆蓋該路徑，Docker 會在 container 建立時自動建立一個匿名 volume 掛到 `PGDATA`。

問題在於：官方 compose 掛的是父目錄 `/var/lib/postgresql`，並未覆蓋 `/var/lib/postgresql/data`，因此實際資料仍會寫入匿名 volume。匿名 volume 與 container 綁定，當執行：

```bash
docker compose down
docker compose up -d
```

container 重建後，Docker 會配置新的匿名 volume，舊資料則變成 dangling volume。對 SonarQube 而言，結果等同於 PostgreSQL 被初始化成全新的空資料庫。

因此本文的 compose 額外補上：

```yaml
volumes:
  - postgresql:/var/lib/postgresql
  - postgresql:/var/lib/postgresql/data
```

當 external named volume 明確掛載到 `PGDATA` 後，會覆蓋 image 內建的 `VOLUME` 宣告，Docker 不再建立匿名 volume；即使 container 重建，PostgreSQL 資料仍會保留於同一個 named volume 中。

#### 檢查目前部署狀態

```bash
docker inspect sonarqube-db --format '{{json .Mounts}}' | jq
```

對照三種狀態：

| 狀態 | inspect 看到的 mount | 處置 |
|---|---|---|
| **A**：原始 bug（單行掛父目錄） | 一個 named at `/var/lib/postgresql` + 一個 hex 名稱的匿名 volume at `/var/lib/postgresql/data` | 走「狀態 A 救援」——資料在匿名 volume，先搬出來 |
| **B**：單行掛 PGDATA | 只有一個 named at `/var/lib/postgresql/data` | 走「狀態 B 遷移」——資料在 `postgresql` named volume，要搬到 `postgresql_data` |
| **C**：已是雙 volume | 兩個 named，分別在父目錄與 PGDATA | 沒事，直接做 [切換前必做 2](#切換前必做-2拿掉舊的-sonarqube_extensions-volume) |

#### 狀態 A 救援：資料在匿名 volume 上

舊 PGDATA 還在 dangling 匿名 volume 上，把它搬到新 yaml 期望的 `postgresql_data` named volume：

```bash
# 1. 停 stack（保留所有 volume，包含 dangling）
docker compose down

# 2. 找出舊 PGDATA 所在的匿名 volume
sudo bash -c '
for vol_dir in /var/lib/docker/volumes/[0-9a-f]*; do
  data="$vol_dir/_data"
  if [ -f "$data/PG_VERSION" ]; then
    echo "$(basename $vol_dir)  pgver=$(cat $data/PG_VERSION)  size=$(du -sh $data 2>/dev/null | cut -f1)  mtime=$(stat -c %y $data | cut -d. -f1)"
  fi
done | sort -k4
'
# 挑出 pgver 與 db image 對得上、mtime 最接近上次正常用 SQ 的那一個 hex 名稱
# 假設叫 <OLD_HEX>

# 3. 驗證候選 volume 真的是 SonarQube 的（避免救錯 DB）
docker run --rm -d --name pg-recovery-test \
  -v <OLD_HEX>:/var/lib/postgresql/data \
  postgres:17
sleep 5
docker exec pg-recovery-test psql -U admin -l
docker exec pg-recovery-test psql -U admin -d sonar -c \
  "SELECT count(*) FROM projects WHERE qualifier='TRK';"
docker stop pg-recovery-test

# 4. 建新的 postgresql_data，從舊匿名 volume 搬資料進去
docker volume create <project>_postgresql_data
docker run --rm \
  -v <OLD_HEX>:/source:ro \
  -v <project>_postgresql_data:/dest \
  postgres:17 \
  sh -c "cp -a /source/. /dest/"

# 5. 清空舊的 postgresql named volume（新 yaml 它是父目錄角色，起步該是空的）
docker volume rm <project>_postgresql
docker volume create <project>_postgresql

# 6. 接著做切換前必做 2（清舊 sonarqube_extensions volume），才 up -d
```

第 4 步用 read-only 掛舊 volume 來複製，舊匿名 volume 原封不動；萬一後續出狀況，舊 hex 還在、可以重來。

#### 狀態 B 遷移：把 PGDATA 搬到 `postgresql_data`

舊 yaml 寫 `postgresql:/var/lib/postgresql/data`，所以 **`postgresql` named volume 內容就是 PGDATA**。新 yaml 期望 `postgresql_data` 才是 PGDATA、`postgresql` 退居父目錄角色——需要做一次 volume 內容遷移：

```bash
# 1. 停 stack
docker compose down

# 2. 建新的 postgresql_data，從 postgresql 搬資料進去
docker volume create <project>_postgresql_data
docker run --rm \
  -v <project>_postgresql:/source:ro \
  -v <project>_postgresql_data:/dest \
  postgres:17 \
  sh -c "cp -a /source/. /dest/"

# 3. 清空 postgresql volume（新 yaml 它是父目錄角色）
docker volume rm <project>_postgresql
docker volume create <project>_postgresql

# 4. 接著做切換前必做 2，才 up -d
```

第 2 步是 read-only 來源 + 新建目標，舊 `postgresql` 內容會在第 3 步才被清掉，這之前如果搬錯都可以中斷重來。

#### ⚠️ 期間絕對不要做的事

- **不要用 `docker compose down -v`**：會把 *所有* named volume 一起砍；狀態 A 的 dangling 匿名 volume 也可能被 prune——救援機會直接歸零

- 遷移未完成前不要跑 `docker volume prune`：dangling volume 是狀態 A 的救援素材，prune 後就回不來

- 不要直接砍 `<OLD_HEX>` 或舊 `postgresql` volume：等驗證完新 DB 起得來、SonarQube UI 上專案完整可見之後再清

### 切換前必做 2：拿掉舊的 `sonarqube_extensions` volume

新 yaml 已經把 `/opt/sonarqube/extensions` 的 mount 移除——plugin jar 直接從 image 載入、不再走 named volume。既有部署若原本有 `sonarqube_extensions` 這顆 named volume，它會在這次 yaml 改動後變成 **orphan**（沒人引用）。技術上不影響運作，但留著有兩個副作用：

- **占空間**：volume 的內容（舊版本的 plugin、driver、staging 檔）一直留在 disk 上
- **語意混淆**：未來看 `docker volume ls` 會看到一顆找不到對應 yaml 引用的 volume

更重要的是，**原本踩過 named-volume override bug** 的 SQ（舊 yaml 有 `sonarqube_extensions` mount + 切過 image），這顆 volume 內可能還塞著舊 plugin 的 staging 檔案。雖然新 yaml 不掛它了，留著只是垃圾。建議在 `docker compose up -d` **之前**清掉，順手做個衛生處理：

```bash
# 1. 停 stack（保留其他 volume）
docker compose down

# 2. 確認 volume 名稱（通常是 <project>_sonarqube_extensions）
docker volume ls | grep sonarqube

# 3. 只刪 extensions 這顆
docker volume rm <project>_sonarqube_extensions
```

刪掉後 `docker volume ls` 就乾淨了。新 yaml 不引用這個名字，後續 `up -d` 也不會把它建回來。

> **沒踩過 override bug 的部署**（例如從 community-branch-plugin image 全新部署）通常不會有這顆 volume，這節可以略過。

#### ⚠️ 還是要叮嚀一次

- **不要用 `docker compose down -v`**：會把 *所有* named volume 砍掉，包含 PostgreSQL 的資料 volume——專案、Quality Gate、token、issue 歷史會**全部消失**。要砍就點名 `sonarqube_extensions`

### 切換後容器與資料的行為

執行 `docker compose up -d` 後（已先做完上一節的 volume 清理），Compose 偵測到 `image:` 變更，行為是 **recreate**（不是 restart）：

1. 舊容器 `docker stop` → `docker rm`
2. 拉新 image（首次下載約 1 GB）
3. 用新 image 建立新容器並啟動，plugin jar 直接從 image 內 `/opt/sonarqube/extensions` 路徑載入（沒掛 named volume，不會有 override 問題）

資料保留情況：

| Volume | 內容 | 本次切換後 |
|---|---|---|
| `sonarqube_data` | Elasticsearch indices、analysis cache | 保留 |
| `sonarqube_logs` | 日誌 | 保留 |
| `postgresql_data` | PGDATA（專案、issues、user、token、ALM config） | 保留（從舊 volume 遷移或全新建立） |
| `postgresql` | postgres user home 雜物（`.psql_history` 等） | 保留 |

關鍵在於專案、Quality Gate 設定、token、歷史分析結果**全部存在 PostgreSQL 的 PGDATA 裡**，新 yaml 用 `postgresql_data` named volume 接住——只要前面切換前必做 1 的遷移正確，「應用層資料」完全保留。Plugin / driver 則跟著 image 走、不持久化，所以「升 plugin = 換 image」一步到位、不會留垃圾。

Downtime 預期：從停舊容器到 SQ 完整啟動（含內建 Elasticsearch 起 indices）大約 **1–3 分鐘**。期間 CI 上若剛好觸發 sonar job 會 fail，建議挑離峰時段切換。

額外要注意一件版本鎖定的事：原 yaml 寫 `image: sonarqube:community` 沒帶版本號，每次 `docker compose pull` 都可能拉到當下最新；換成 `mc1arke/...:26.4.0.121862-community` 之後版本被鎖死。鎖死本身是好事（避免 plugin 與 SQ 版本錯位），但代價是未來升 SQ 需要手動改 yaml 並同步升 plugin。

### 啟動後驗證

依序檢查：

- `docker compose logs -f sonarqube` 看到 `SonarQube is operational`

- 登入 SonarQube → `Administration > Marketplace > Installed`，列表中應出現 `Community Branch Plugin`，版本與 SQ 對齊

- 開原有的任一專案，左側選單應多出 `Branches &Pull Requests` 這個 tab（原生 Community Build 沒這兩個）：
![alt text](image-2.png)

- 原專案的 Dashboard 內容與 issue 數應完全保留——若 SonarQube 顯示「沒有任何專案」，多半是 [切換前必做 1](#切換前必做-1把既有-db-資料對齊新-volume-配置) 的遷移漏做或做錯，回頭比對 `docker inspect sonarqube-db` 的 mount 是否真的指到含資料的 volume

### 替代方案：在官方 image 上手動掛 plugin

若公司 policy 不接受第三方預建 image，可以用官方 `sonarqube:community` 為基底自行掛 plugin：

1. 從 [plugin release 頁](https://github.com/mc1arke/sonarqube-community-branch-plugin/releases) 下載對應版本的 `sonarqube-community-branch-plugin-XX.jar`
2. 放進掛載到 `/opt/sonarqube/extensions/plugins/` 的 volume
3. 在容器啟動參數加上 `SONAR_WEB_JAVAADDITIONALOPTS` 與 `SONAR_CE_JAVAADDITIONALOPTS`，內容為 `-javaagent:./extensions/plugins/sonarqube-community-branch-plugin-XX.jar=web` 與 `=ce`
4. 新版 plugin 還會要求替換 `web/` 目錄為 plugin 提供的 `sonarqube-webapp.zip`——這步驟在預建 image 已替你做完

本文後續步驟不重複此路徑的細節；想走這條的讀者可直接參考 [plugin README](https://github.com/mc1arke/sonarqube-community-branch-plugin#installation)。

---

## 步驟二：確認 GitLab ALM Integration

Plugin 裝好後 scanner 已能跑 PR analysis，但要讓 **SonarQube 主動 POST comment 回 GitLab MR**，還需要兩件事：

1. SonarQube 端設好 GitLab DevOps Platform Integration（sonarQube admin 層級 config，存 PAT 用來呼叫 GitLab API）
  
2. 在專案層級啟用 Pull Request Decoration、綁定 GitLab project ID（這個區塊是 plugin 才會出現）

第一件事 [02 章 Path B](02-gitlab-ci-integration.md#path-b--import-from-gitlabdevops-平台匯入) 走過匯入流程的環境已經設好；若 02 章是走 Path A 手動建專案、或這次部署從頭開始，這節要補設。GitLab 端的 Service Account 與 PAT 建立步驟在 02 章 [B.1](02-gitlab-ci-integration.md#b1-在-gitlab-建立-service-account-並產-pat) 與 [CE 用戶的 fallback：開 bot user](02-gitlab-ci-integration.md#ce-用戶的-fallback開-bot-user) 都寫過，本節不重複；scope 維持 `api` 就足以涵蓋 MR commenting。

### 2.1 設定 SonarQube 端的 GitLab DevOps Platform Integration

#### 02 章 Path B 走過的人怎麼確認（多數讀者）

若 02 章是走 Path B 匯入過專案，這筆 configuration 應該早就存在——直接到 `Administration > Configuration > General Settings > DevOps Platform Integrations > GitLab`，確認 `GitLab Configuration` 區塊已列出對應的 configuration（例如 `Company GitLab`）、按 `Check Configuration` 仍是綠勾即可，不必重設。`api` scope 已涵蓋 MR commenting，**token 也不必重產**。

#### 設定步驟（沒設過 ALM Integration 的人）

到 `Administration > Configuration > General Settings > DevOps Platform Integrations > GitLab > Create configuration`：

- `Configuration name`：自取一個 identifier，例 `Company GitLab`，之後專案層級會引用
- `GitLab API URL`：自架 GitLab 的 `https://gitlab.example.com/api/v4`（SaaS 則填 `https://gitlab.com/api/v4`）
- `Personal Access Token`：貼上 02 章 [`三種 Token 一次釐清`](02-gitlab-ci-integration.md#三種-token-一次釐清) 表格中的 Service Account PAT（scope `api`）。若 GitLab 端 PAT 還沒生成，先到 02 章 [B.1](02-gitlab-ci-integration.md#b1-在-gitlab-建立-service-account-並產-pat) 把帳號與 token 準備好再回來
- 按 `Save configuration`

存檔後同一頁面下方按 `Check Configuration`——綠勾就過。若紅叉，依序排查：

- URL 結尾有沒有 `/api/v4`
- PAT scope 是否包含 `api`
- Service Account / bot user 在目標 GitLab project 上至少 `Reporter`

### 2.2 專案層級綁定 Pull Request Decoration（plugin 新增）

到目標專案 `Project Settings > General Settings > DevOps Platform Integration`：

- `Configuration name`：選 2.1 那個 configuration（例 `Company GitLab`）
- `Project ID`：填 GitLab 對應 project 的**數字 ID**（在 GitLab project 首頁的 Project ID 欄位看得到，**不是** path 字串 `group/repo`）

存檔後 SonarQube 該專案才知道「PR analysis 結果要 POST 去哪個 GitLab project」。透過 [02 章 Path B](02-gitlab-ci-integration.md#path-b--import-from-gitlabdevops-平台匯入) 匯入的專案，Project ID 通常已經自動帶好（plugin 偵測得到既有的 GitLab binding），只需要確認一下；走 Path A 手動建專案的則要自己填。

---

## 步驟三：修改 `.gitlab-ci.yml`

把 [02 章正式版 yaml](02-gitlab-ci-integration.md#修改後的正式環境版本) 拿出來改 `rules:` 區塊；其他欄位（image、cache、`GIT_DEPTH`、tags）不動。

### 3.1 `rules:` 從一條擴成兩條

原本：

```yaml
rules:
  - if: $CI_COMMIT_BRANCH == "test-va" && $CI_PIPELINE_SOURCE == "push"
```

改成：

```yaml
rules:
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  - if: $CI_COMMIT_BRANCH == "test-va" && $CI_PIPELINE_SOURCE == "push"
```

兩條規則各自的用意：

| Rule | 觸發時機 | SonarQube 端的結果 |
|---|---|---|
| 第一條 | MR 開啟 / 更新時的 MR pipeline | 進 `Pull Requests` tab，建立 PR analysis、自動在 MR 留 comment |
| 第二條 | 合併進 `test-va` 後的 push pipeline | 進 `Branches` tab 的 `test-va`，更新該 branch dashboard |

兩條規則彼此**不互相覆蓋**——plugin 讓 PR analysis 與 branch analysis 分開儲存，這是它與 [01-sonarqube.md `### Community Build 在分支上的實務行為`](01-sonarqube.md#community-build-在分支上的實務行為) 描述的「同 projectKey 互相覆蓋」最大的不同。

實務上的副作用：一次完整 MR 流程（開 MR → push 修改 → merge）會觸發 **兩次** scanner——MR 階段一次、合併後一次。CI 耗時翻倍是 plugin 路線必然的成本。

### 3.2 `sonar.pullrequest.*` 要不要顯式傳

GitLab 預定義環境變數會被 SonarScanner 5.x+ 自動讀取對應到 `sonar.pullrequest.*`，理論上不必顯式傳。兩種寫法都列：

**寫法 A（推薦）：依賴 auto-detection**

腳本一行就夠：

```yaml
script:
  - mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -DskipTests
```

Scanner 在 MR pipeline 跑時會自動偵測到 `CI_MERGE_REQUEST_IID` 等變數並轉成 `sonar.pullrequest.key`，在 `test-va` push pipeline 跑時則走 branch analysis。

**寫法 B：顯式傳（debug / 不信任 auto-detection 時用）**

```yaml
script:
  - |
    if [ "$CI_PIPELINE_SOURCE" = "merge_request_event" ]; then
      mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -DskipTests \
        -Dsonar.pullrequest.key=$CI_MERGE_REQUEST_IID \
        -Dsonar.pullrequest.branch=$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME \
        -Dsonar.pullrequest.base=$CI_MERGE_REQUEST_TARGET_BRANCH_NAME
    else
      mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -DskipTests
    fi
```

三個參數的對應：

- `sonar.pullrequest.key` ← `$CI_MERGE_REQUEST_IID`（MR 在 GitLab 上的數字 ID，scanner 用這個對到 SQ 的 PR analysis）
- `sonar.pullrequest.branch` ← `$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME`（source branch 名稱）
- `sonar.pullrequest.base` ← `$CI_MERGE_REQUEST_TARGET_BRANCH_NAME`（target branch 名稱，通常是 `test-va`）

實務建議：先用寫法 A 跑跑看，若 PR analysis 沒正確建立再切寫法 B 確認。

### 3.3 `sonar.qualitygate.wait=true` 仍然生效

[02 章已把 `sonar.qualitygate.wait=true` 寫進 pom.xml](02-gitlab-ci-integration.md#設定-pomxml)，這設定在 MR pipeline 也會吃到——意思是 MR pipeline 跑出來的 Quality Gate 狀態**就是 job 的 pass/fail 結果**。配合 `allow_failure: false`，QG fail 時 MR pipeline 紅、MR merge 被擋（若專案設了 `Pipelines must succeed`），是 plugin 路線上「擋住 bad code」的真正機制——不必依賴開發者主動去點 MR comment 才知道有問題。

---

## 步驟四：端到端驗證

從一個 dummy MR 走到底：

1. **MR 開啟**：建立 feature branch，改一行 code，開 MR 對 `test-va`
   - GitLab pipeline 出現 sonar job
   - SonarQube `Pull Requests` tab 看到新 PR analysis 條目
   - MR 介面下方應出現 SonarQube bot 的 comment，內容包含 Quality Gate 狀態與 issue 數
2. **推一個有問題的 commit**：故意改一行會被 sonar way 抓的 code（例如未使用變數），push
   - SonarQube 標記新 issue
   - MR 上 bot comment 更新（issue 數變化）
   - Quality Gate fail → MR pipeline 變紅
   - MR 介面顯示「pipeline failed」狀態
3. **修掉問題 → 重 push**：QG pass → pipeline 綠
4. **合併 MR**：
   - `test-va` push pipeline 跑第二次 sonar
   - SonarQube `Branches` tab 看到 `test-va` 結果更新
   - `Pull Requests` tab 該 MR 標為「已合併」

若第 1 步 bot comment 沒出現，依序排查：

- SonarQube logs：有沒有 `Pull request decoration failed` 字樣
- 步驟二 2.2 的 ALM Integration 是否在當時設好
- 步驟二 2.3 的 Project ID 是否正確（最常見錯：填了 path 字串而不是數字 ID）
- Bot 帳號是否在該 GitLab project 上有 Reporter 權限

例如下圖，先推一個有問題的 MR：

![alt text](image-3.png)
> reviewer 也會收到 email 通知，提醒他去看 SonarQube comment 裡的 issue

成功修復後：

![alt text](image-4.png)
---

## 裝 plugin 之後 New Code definition 怎麼設

Plugin 解開的不只是 MR decoration，**New Code 計算的精準度也改善了**。

裝 plugin 前（[02–04 章預設情境](04-quality-gate-and-results.md#看懂專案-dashboard)）：建議 `Previous version` 或 `Number of days`，因為 Community Build 沒有原生 branch 概念，Reference branch 行為不穩定。

裝 plugin 後：

- **MR analysis 時**：scanner 收到 `sonar.pullrequest.base`（自動偵測為 `test-va`），plugin 會把 New Code 定義為「MR 相對於 base branch 的差異」——這是 Developer Edition 的原生行為，現在 Community Build 也能用
- **branch analysis 時**：sonar 沿用專案層級 New Code definition 設定

**實務建議：保留 `Previous version`**。Plugin 在 MR 上下文會自動切換到「相對 base branch」的算法，不需要為 plugin 改動 New Code definition 設定。只有「`test-va` branch analysis 時想用 reference branch 對比 `main`」這種特殊需求，才需要在專案層級改 New Code definition。

---

## 不裝 plugin 的 fallback

若公司 policy 不接受第三方 plugin、或暫時不想承擔升 commercial 的 data loss 風險，仍有兩條路徑：

- **方案 A：維持 [02 章 yaml 原樣](02-gitlab-ci-integration.md#修改後的正式環境版本)**——`rules:` 只鎖 `test-va` push，MR 階段不跑 SonarQube。代價是合併後才知道分析結果
- **方案 C：MR 也用同 projectKey 跑，接受 Dashboard 失真**——`rules:` 加上 `merge_request_event` 那條，但因為沒有 plugin，PR analysis 結果會與 branch analysis 互相覆蓋（[01-sonarqube.md L67 的同 projectKey 機制](01-sonarqube.md#community-build-在分支上的實務行為)）。要避免 New Code 基準失真，需把專案層級 New Code definition 改為 `Reference branch = test-va`

兩條 fallback 都不在本章主推範圍，列在這裡讓無法走 plugin 路線的讀者有所選擇。

---

## 常見問題與升版

**Plugin 沒啟動 / SQ 起不來**

- log 出現 `Error opening zip file or JAR manifest missing` 加 `agent library failed Agent_OnLoad: instrument` → 舊 yaml 有 `sonarqube_extensions` mount，named volume override 蓋住 image 內 plugin jar。新 yaml 已移除這個 mount，徹底解決；既有部署若還沒套用新 yaml，請參考 [切換前必做 2：拿掉舊的 `sonarqube_extensions` volume](#切換前必做-2拿掉舊的-sonarqube_extensions-volume)
- 切換 image 後 SQ 起來但「所有專案不見了」 → db volume mount path bug 觸發。救援與遷移步驟見 [切換前必做 1：把既有 DB 資料對齊新 volume 配置](#切換前必做-1把既有-db-資料對齊新-volume-配置)
- 容器拉不到 image：檢查 tag 是否確實存在於 [Docker Hub](https://hub.docker.com/r/mc1arke/sonarqube-with-community-branch-plugin/tags)（少數 SQ build 號可能延後幾天才出對應 plugin image）
- SQ 起得來但 plugin 沒載入：`Administration > Marketplace > Installed` 看不到 plugin → 通常是用了「自己掛 plugin」的路徑但 `SONAR_WEB_JAVAADDITIONALOPTS` / `SONAR_CE_JAVAADDITIONALOPTS` 沒設或路徑錯，見 [替代方案：在官方 image 上手動掛 plugin](#替代方案在官方-image-上手動掛-plugin)
- log 出現 `Could not find class` 之類錯誤：plugin 版本與 SQ 版本錯位，回頭核對 [Plugin Release 頁](https://github.com/mc1arke/sonarqube-community-branch-plugin/releases) 的對應關係

**MR 沒有出現 SonarQube comment**
- 9 成是步驟二 2.3 的 Project ID 填錯（填了 path 字串）
- Bot user 在該 GitLab project 沒 Reporter 權限
- `Administration > ALM Integrations` 點 `Check Configuration` 重新驗證

**SonarQube 升版的流程**
1. 先查 plugin 的 release 頁，確認對應的 SQ 版本有 plugin release
2. 若有：先升 plugin（換 image tag），確認 plugin 在新版 SQ 上正常運作
3. 若無：等 plugin release 出來再升，**不要** 先升 SQ
- 規範化做法：在內部維護「SQ 版本 ↔ plugin 版本 ↔ image tag」對應表

**退場：移除 plugin 改回官方 image**
- 把 image 改回 `sonarqube:community`，`docker compose up -d`
- branch 與 PR 的歷史分析資料會「不可見」——資料還在 PostgreSQL，但官方 SQ 不認 plugin 寫入的 schema 部分
- main branch 的分析結果保留，Quality Gate、token、user 也都保留
- 若想徹底清乾淨，需要進 DB 手動清 plugin 寫入的表——這部份 plugin 文件沒明說，不建議輕易執行
