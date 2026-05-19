# GitLab CI/CD Cache（快取）設定與管理

## 目錄

- [GitLab CI/CD Cache（快取）設定與管理](#gitlab-cicd-cache快取設定與管理)
  - [目錄](#目錄)
  - [為什麼需要 cache](#為什麼需要-cache)
  - [Cache 在 docker executor 的儲存位置](#cache-在-docker-executor-的儲存位置)
  - [`.gitlab-ci.yml` 的 cache 寫法](#gitlab-ciyml-的-cache-寫法)
  - [Cache key 與分支隔離](#cache-key-與分支隔離)
  - [改 runner cache 路徑（host bind mount）](#改-runner-cache-路徑host-bind-mount)
    - [Step 1：建立 host 目錄](#step-1建立-host-目錄)
    - [Step 2：停掉 runner](#step-2停掉-runner)
    - [Step 3：搬遷現有 cache（可選）](#step-3搬遷現有-cache可選)
    - [Step 4：改 config.toml](#step-4改-configtoml)
    - [Step 5：啟動 runner 並驗證](#step-5啟動-runner-並驗證)
  - [如何查 cache 真正落在哪裡](#如何查-cache-真正落在哪裡)
  - [常用維護動作](#常用維護動作)
  - [踩過的坑](#踩過的坑)

---

## 為什麼需要 cache

GitLab CI 的 job 跑在 **每次都從零開始的乾淨環境**（docker executor 每個 job 一個新 container）。沒有 cache 的話，每次 build 都會：

- Maven / Gradle 重新從 central repository 下載所有 dependency
- npm / pnpm 重新下載 `node_modules`
- pip 重新下載 wheels
- Sonar 重新計算 baseline

實測差異（gen-bi 專案）：
| 狀態 | `mvn clean install` 耗時 |
|---|---|
| 無 cache（冷啟動） | 3-5 分鐘（含下載 ~220MB dependency） |
| 有 cache（命中） | 30-90 秒（dependency 直接從本機讀取） |

把 `.m2/repository` 快取下來，pipeline 就能省掉這段下載時間。

---

## Cache 在 docker executor 的儲存位置

GitLab Runner 用 docker executor 時，cache 的位置由 `[runners.docker]` 的 `volumes` 設定決定。

預設寫法是：
```toml
volumes = ["/certs/client", "/cache"]
```

`/cache` 沒綁定 host 路徑 → Runner 會自動建一顆 **named volume**，名稱類似：
```
runner-<token-hash>-cache-<mountpoint-hash>[-protected]
```

例如：
```
runner-d02f81554bd92046047da42b1a326ec2-cache-3c3f060a0374fc8bc39395164f415a70-protected
```

這顆 volume 內部結構：
```
/cache/
└── <group>/
    └── <project>/
        └── <cache-key>/
            └── cache.zip      ← 實際快取內容
```

例如：
```
/cache/lndata/data-transform/gen-bi/maven-cache-gen-bi-protected/cache.zip
```

**缺點**：每次 runner 重註冊 / token 變更 / runner 升級，可能產生**新的 volume hash**，舊 volume 變孤兒，cache 看起來「跑去哪了」很難追。

---

## `.gitlab-ci.yml` 的 cache 寫法

以 gen-bi（Spring Boot + Maven）為例：

```yaml
.maven_cache:
  cache:
    key: maven-cache-gen-bi
    paths:
      - .m2/repository
  variables:
    MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

maven_ci:
  stage: validate
  extends:
    - .maven_cache
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn clean install
```

關鍵點：
1. **`cache.paths`** 指定要快取的目錄（相對於 `$CI_PROJECT_DIR`）
2. **`MAVEN_OPTS`** 強制 Maven 寫到 `$CI_PROJECT_DIR/.m2/repository`，跟 `cache.paths` 對齊
3. **`cache.key`** 決定 cache.zip 的檔名 / 隔離單位

Job 跑起來時 log 會出現：
```
Restoring cache
Checking cache for maven-cache-gen-bi-protected...
Successfully extracted cache       ← cache 命中
```

結束時：
```
Creating cache maven-cache-gen-bi-protected...
.m2/repository: found 3310 matching artifact files and directories
Created cache
```

---

## Cache key 與分支隔離

GitLab 對 protected branch 會**自動加 `-protected` 後綴**，讓 release / main 等 branch 的 cache 跟一般 branch 隔離：

| Branch 類型 | 實際 cache key |
|---|---|
| 一般 branch（如 `feature/xxx`） | `maven-cache-gen-bi` |
| Protected branch（如 `release/*`、`master`） | `maven-cache-gen-bi-protected` |

兩者對應兩個獨立的 `cache.zip`，互不污染。

**若想每個 branch 完全獨立 cache**：

```yaml
cache:
  key: "maven-cache-gen-bi-$CI_COMMIT_REF_SLUG"
  paths:
    - .m2/repository
  fallback_keys:
    - "maven-cache-gen-bi-master"   # 新 branch 第一次跑時從 master 借 cache 暖機
```

代價是 disk 用量會隨活躍 branch 數量線性成長，需要定期清理沒人用的 cache。

---

## 改 runner cache 路徑（host bind mount）

預設 named volume 機制有「重註冊就迷路」的問題。改用 **host bind mount** 把 cache 固定在 host 上某個路徑，從此不論 runner 怎麼變動，cache 永遠在那裡。

以下是實際在 `lndata-1-docker` runner 上做過的步驟：

### Step 1：建立 host 目錄

```bash
sudo mkdir -p /srv/gitlab-runner-docker-executor-cache
sudo chmod 777 /srv/gitlab-runner-docker-executor-cache
```

`chmod 777` 是因為 runner container 內可能用任意 user 寫入，最寬鬆權限避免 permission 問題。

### Step 2：停掉 runner

```bash
sudo systemctl stop gitlab-runner
```

避免搬遷過程中有 job 寫入舊位置。

### Step 3：搬遷現有 cache（可選）

如果想保留既有 cache 避免冷啟動，把舊 named volume 內容拷貝到新位置。先找出有資料的 volume：

```bash
for v in $(sudo docker volume ls -q --filter name=runner); do
  mp=$(sudo docker volume inspect -f '{{.Mountpoint}}' "$v")
  size=$(sudo du -sh "$mp" 2>/dev/null | cut -f1)
  count=$(sudo find "$mp" -type f 2>/dev/null | wc -l)
  echo "$v  files=$count  size=$size"
done
```

挑檔案數和 size 都大的（裡面有 `cache.zip` 的）：

```bash
sudo cp -a /var/lib/docker/volumes/<volume-name>/_data/. \
           /srv/gitlab-runner-docker-executor-cache/
```

`cp -a` 保留權限和時間戳，結尾的 `.` 是把目錄**內容**而非目錄本身複製過去。

驗證：
```bash
sudo find /srv/gitlab-runner-docker-executor-cache -name 'cache.zip' -exec ls -lh {} \;
```

預期看到所有 `cache.zip` 完整列表。

### Step 4：改 config.toml

```bash
sudo cp /etc/gitlab-runner/config.toml /etc/gitlab-runner/config.toml.bak
sudo vim /etc/gitlab-runner/config.toml
```

找到對應 runner 的 `[runners.docker]` 區塊，把：
```toml
volumes = ["/certs/client", "/cache"]
```
改成：
```toml
volumes = ["/certs/client", "/srv/gitlab-runner-docker-executor-cache:/cache"]
```

### Step 5：啟動 runner 並驗證

```bash
sudo systemctl start gitlab-runner
sudo systemctl status gitlab-runner | head -10
```

確認 `active (running)`，然後 retry 一次 `maven_ci` job，看 log：

```
Checking cache for maven-cache-gen-bi-protected...
No URL provided, cache will not be downloaded from shared cache server.
Instead a local version of cache will be extracted.
Successfully extracted cache         ← 從新路徑讀取成功
```

Job 結束後：
```
Creating cache maven-cache-gen-bi-protected...
.m2/repository: found 3353 matching artifact files and directories
Created cache                        ← 寫回新路徑
```

並到 server 上確認檔案 mtime 是剛剛 job 結束的時間：
```bash
sudo ls -lh /srv/gitlab-runner-docker-executor-cache/lndata/data-transform/gen-bi/maven-cache-gen-bi-protected/cache.zip
```

---

## 如何查 cache 真正落在哪裡

不論用 named volume 還是 host bind mount，找 cache 的核心方法：**用 cache key 當關鍵字 grep `cache.zip` 路徑**。

從 job log 拿到 cache key（例如 `maven-cache-gen-bi-protected`），然後：

```bash
# 用 host bind mount 的情況（推薦）
sudo find /srv/gitlab-runner-docker-executor-cache -name 'cache.zip' | grep maven-cache-gen-bi

# 用 named volume 的情況
for v in $(sudo docker volume ls -q --filter name=cache); do
  mp=$(sudo docker volume inspect -f '{{.Mountpoint}}' "$v")
  hit=$(sudo find "$mp" -path "*/maven-cache-gen-bi-protected/cache.zip" 2>/dev/null)
  [ -n "$hit" ] && echo "$v -> $hit"
done
```

---

## 常用維護動作

**列出所有 cache 和大小**：
```bash
sudo find /srv/gitlab-runner-docker-executor-cache -name 'cache.zip' -exec ls -lh {} \;
```

**清掉某個專案/某個 key 的 cache**（強制下次冷啟動）：
```bash
sudo rm -rf /srv/gitlab-runner-docker-executor-cache/lndata/data-transform/gen-bi/maven-cache-gen-bi-protected/
```

**清掉超過 30 天沒動的 cache**（手動 review 後執行）：
```bash
sudo find /srv/gitlab-runner-docker-executor-cache -name 'cache.zip' -mtime +30 -exec ls -lh {} \;
```

**清掉舊的 named volume**（搬遷完成後）：
```bash
sudo docker volume ls -q --filter name=runner | xargs -r sudo docker volume rm
```
已掛載中的 volume 會被 docker 擋下不會誤刪，可放心執行。

---

## 踩過的坑

**1. `MAVEN_OPTS` 沒設好 → cache 沒效果**

如果只寫 `cache.paths: [.m2/repository]` 但沒設 `MAVEN_OPTS=-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository`，Maven 預設會寫到 `~/.m2/repository`（家目錄），跟 `cache.paths` 不一致 → cache 抓不到內容、永遠是空的。

**2. Runner 重註冊後 cache 看起來消失**

重註冊會產生新 token、新 volume hash、新 named volume，舊 cache 變孤兒留在原 volume 內。看起來「cache 不見了」實際是被換到新 volume，舊的還在但不再被讀取。

→ 解法就是改用 host bind mount，徹底跟 token 解耦。

**3. `[runners.cache]` 區段空殼 → log 出現 ERROR 但不影響功能**

`gitlab-runner register` 預設會寫進骨架：
```toml
[runners.cache]
  [runners.cache.s3]
  [runners.cache.gcs]
  [runners.cache.azure]
```

沒設 `Type` 但有 cache 區段 → runner 啟動時試圖載入 distributed cache adapter 失敗：
```
ERROR: Could not create cache adapter
error=cache factory not found: factory for cache adapter "" was not registered
```

是雜訊，fallback 到 local cache 仍然正常運作。要消掉這個錯誤就把整個 `[runners.cache]` 區段刪掉。

**4. `cache.zip` 是覆蓋制，不是累積**

同一個 cache key 永遠只有一份 `cache.zip`，每次 job 結束的 `Created cache` 步驟會直接覆蓋舊版。沒有版本歷史。

如果想強制重置 cache（例如懷疑舊 cache 有問題），改 cache key 的後綴（`maven-cache-gen-bi` → `maven-cache-gen-bi-v2`），舊 `cache.zip` 變孤兒手動 `rm` 即可。
