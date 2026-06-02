# Oracle Instant Client 升級事故總結

> **TL;DR**：CI 處裡壓縮檔的時候記得加 `-o`（force overwrite），不然如果新版本的壓縮檔內容跟舊版本有衝突，`unzip` 會跳出互動式 prompt 詢問是否覆寫，導致 Docker build 在非互動環境下失敗。

## 背景

印尼同事在 commit `7d1bb059`（_Update Oracle Instant Client version to 23.26.2 in Dockerfile_）將 `data-transform-core/connector/Dockerfile` 中的 Oracle Instant Client 從 **21.13.0** 升級到 **23.26.2**。

該 commit 只改了兩個 `curl` 下載 URL 的版本號，沒動其他地方，CI 在 `docker_build` 階段失敗。

---

## 錯誤訊息（節錄）

```
#6 [2/5] RUN apt-get update && apt-get install -y --no-install-recommends ...
    && unzip -q /tmp/ic-basic.zip -d /opt/oracle
    && unzip -q /tmp/ic-tools.zip -d /opt/oracle ...

25.76 replace /opt/oracle/META-INF/MANIFEST.MF? [y]es, [n]o, [A]ll, [N]one, [r]ename:  NULL
25.76 (EOF or read error, treating as "[N]one" ...)

ERROR: failed to solve: process "/bin/sh -c apt-get update && ..." did not complete successfully: exit code: 1
```

---

## 根本原因

### 問題 1：`unzip` 互動 prompt 導致 build 失敗（這次直接踩到）

- 23.26.2 的 `basiclite` 與 `tools` 兩個 zip **頂層都包含 `META-INF/MANIFEST.MF`**。
- 兩個 zip 都解到同一個 `/opt/oracle` 目錄，第二個解壓時遇到檔案衝突，`unzip` 跳出互動式 prompt 詢問是否覆寫。
- Docker build 是非互動環境（沒有 stdin），`unzip` 讀到 EOF 後以 exit code 1 中止 → 整層 build 失敗。
- 舊版 21.13.0 的 zip 內容是放在版本化子目錄（`instantclient_21_13/`），頂層沒有衝突檔案，所以同樣的 `unzip -q` 寫法以前不會炸。

### 問題 2：`ORACLE_HOME` 路徑指向舊版本目錄（會造成 runtime 失敗，本次未觸及）

- Instant Client 解壓出的目錄名是版本化的：
  - 21.13.0 → `instantclient_21_13`
  - 23.26.2 → `instantclient_23_26`
- Dockerfile 中：
  ```dockerfile
  ENV ORACLE_HOME=/opt/oracle/instantclient_21_13
  ENV PATH="${ORACLE_HOME}:${PATH}"
  ENV LD_LIBRARY_PATH="${ORACLE_HOME}"
  ```
  原作者只改 URL，沒同步更新 `ORACLE_HOME`。
- 即使解壓成功，container 啟動後：
  - `sqlldr` 不在 `PATH` → SQL*Loader 任務報 `command not found`
  - `libclntsh.so` 不在 `LD_LIBRARY_PATH` → 動態庫載入失敗
- 比 build 失敗更難 debug，因為 build 過了，問題只在 runtime 才浮現。

---

## 修復方式

`data-transform-core/connector/Dockerfile`：

```diff
-    && unzip -q /tmp/ic-basic.zip -d /opt/oracle \
-    && unzip -q /tmp/ic-tools.zip -d /opt/oracle \
+    && unzip -oq /tmp/ic-basic.zip -d /opt/oracle \
+    && unzip -oq /tmp/ic-tools.zip -d /opt/oracle \
...
-ENV ORACLE_HOME=/opt/oracle/instantclient_21_13
+ENV ORACLE_HOME=/opt/oracle/instantclient_23_26
```

- `unzip -o`：force overwrite，不再 prompt。
- `ORACLE_HOME`：指向新版本解壓出的目錄。

### 驗證指令（build 完後執行）

```bash
docker run --rm <connector-image> ls /opt/oracle
docker run --rm <connector-image> sqlldr -help
```

確認：
1. `/opt/oracle` 下的目錄名與 `ORACLE_HOME` 設定一致（推測是 `instantclient_23_26`，需實機確認）。
2. `sqlldr -help` 能印出 usage（而非 `command not found` 或缺 `.so` 錯誤）。

---

## 教訓 / 後續預防

1. **Oracle Instant Client 版本升級不是改兩個 URL 就好**，必須同步檢查：
   - 兩個 `curl` URL 的版本號
   - `unzip` 是否帶 `-o`（防 zip 內容衝突）
   - `ORACLE_HOME` 目錄名是否更新
2. **CI build 過 ≠ runtime 沒事**。Dockerfile 的 ENV 路徑錯誤不會在 build 階段被發現，需要 runtime smoke test。
3. 已在 [`data-transform-core/ci-docs/pipeline.md`](../OneDrive/%E6%A1%8C%E9%9D%A2/Lndata/project/Fusion/data-transform-core/ci-docs/pipeline.md) 新增「Upgrading the Oracle Instant Client version」小節，把這三個檢查點明確寫進文件，避免下次有人踩同樣的雷。

---

## 涉及檔案

- `data-transform-core/connector/Dockerfile` — 修復
- `data-transform-core/ci-docs/pipeline.md` — 新增升級指引段落

## 涉及 commit

- 出錯來源：`7d1bb059` Update Oracle Instant Client version to 23.26.2 in Dockerfile
