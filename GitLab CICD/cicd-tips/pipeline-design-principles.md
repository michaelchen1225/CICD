# CI/CD Pipeline 設計原理：設計腳本時要考慮的種種因素

> 本篇以 [`.gitlab-ci.yml`](./.gitlab-ci.yml)(gen-bi 專案的 pipeline)為實例,把它寫得好的地方抽象成**可轉移的設計原理**。

## 目錄

* [前言：把 pipeline 當成「會被別人接手的程式」](#前言把-pipeline-當成會被別人接手的程式)

* [一、可讀性與意圖：讓人三分鐘看懂](#一可讀性與意圖讓人三分鐘看懂)

* [二、結構與重用：不要複製貼上](#二結構與重用不要複製貼上)

* [三、正確性與冪等：可以安全重跑](#三正確性與冪等可以安全重跑)

* [四、效能與快取：別讓 runner 做白工](#四效能與快取別讓-runner-做白工)

* [五、供應鏈與發版：上線的就是測過的](#五供應鏈與發版上線的就是測過的)

* [六、並行控制與安全：保護共享資源與機密](#六並行控制與安全保護共享資源與機密)

* [七、韌性：面對不可靠的網路與基礎設施](#七韌性面對不可靠的網路與基礎設施)

* [結語：設計前的檢查清單](#結語設計前的檢查清單)

---

## 前言：把 pipeline 當成「會被別人接手的程式」

CI/CD 腳本最容易被當成「能跑就好」的黏合膠水,結果是:半年後沒人敢動、出錯了不知道為什麼、改一個值要找五個地方。

好的 pipeline 設計其實和好的程式設計是同一套價值觀:**可讀、可重用、可重跑、可觀測、可回滾**。本篇把它拆成下面七個面向——先在這裡把每一項「在問什麼、為什麼重要」講清楚;對哪一項有興趣,點標題就能跳到對應章節看 `.gitlab-ci.yml` 的實例拆解。

1. **[意圖與可讀性](#一可讀性與意圖讓人三分鐘看懂)** —— 把 pipeline 當成會被別人接手的程式:開頭幾行就講清楚核心策略與分支模型,每個非顯然的決策(尤其踩過的坑)都留一行「為什麼」,失敗訊息要可操作,跨 job 的隱性約定要寫成明確 CONTRACT。讓接手者三分鐘看懂,而不是逆向工程整個檔案。

2. **[結構與重用](#二結構與重用不要複製貼上)** —— 同一段 rules / cache / shell 出現第二次,就抽成 `.` 開頭的隱藏 job 用 `extends` 組合;善用「安全預設 + 例外覆寫」與「`before_script` 當共用函式庫」。目標是改一處、全部生效,而不是複製貼上五份、日後各自改到走樣。

3. **[正確性與冪等](#三正確性與冪等可以安全重跑)** —— 任何會「切 tag / 推 commit / 部署」的 job 都要能安全重跑:先檢查狀態再動作,用權威的遠端狀態(而非會說謊的本地 checkout)做判斷,解析設定檔時對準真正想要的欄位,push 回自己分支時主動防 CI 觸發迴圈。

4. **[效能與快取](#四效能與快取別讓-runner-做白工)** —— 用 DAG(`needs`)讓 job 一就緒就並行,依「跟分支有沒有關係」分層設計快取 key,每個 job 用剛好夠用的 image / runner,別讓 runner 重複做白工。

5. **[供應鏈與發版](#五供應鏈與發版上線的就是測過的)** —— 上線的二進位就是測過的那一個:image 只建一次,之後靠重新貼標籤(retag)往上游送,並留一個不可變的追溯點;萬一抓不到那個驗證過的版本,寧可直接失敗,也不退回會一直被覆寫的 `:dev`。

6. **[並行控制與安全](#六並行控制與安全保護共享資源與機密)** —— 動到共享資源(機器、分支、git tag、雲端資源)的 job,用 `resource_group` 做跨 pipeline 互斥、用 `interruptible: false` 防中途被砍;機密的暴露面(process list、log、磁碟、子行程)壓到最小。

7. **[韌性與回滾](#七韌性面對不可靠的網路與基礎設施)** —— 外部網路呼叫會失敗,用通用 `retry` 包住(前提是操作冪等),把已知的 infra 缺陷連同根因註解寫進腳本;發版從第一天就設計好「一鍵回滾」的路徑與稽核記錄。

---

## 一、可讀性與意圖：讓人三分鐘看懂

### 1. 開頭就宣告整條 pipeline 的心智模型

腳本最上方先用一段註解講清楚「核心策略 + 分支模型 + 詳細文件在哪」,讓接手者讀幾行就懂全貌,不必逆向工程整個檔案。

```yaml
# Driven by pom.xml SNAPSHOT versioning; build-once on dev, promote/retag on pre-prod.
# Branch model: feature/* → dev → pre-prod → master (FF-only)
# Design & details: see ci-docs/pipeline.md
```

見 [`.gitlab-ci.yml` L1-7](./.gitlab-ci.yml#L1-L7)。**要考慮的因素**:讀者第一眼需要的是「為什麼這樣設計」,而不是「第一個 job 是什麼」。長篇設計另存一份文件,YAML 裡只留夠用的指引。

### 2. 註解寫「為什麼」,尤其是踩過的坑

顯然的事不用註解,**非顯然的決策**才要。最有價值的註解是「我們踩過這個坑,所以這樣寫」——它能阻止後人「好心」把看似多餘的程式碼刪掉。

* 為什麼 stage 要這樣排:`sentry_release` 刻意排在 `docker_build` 前,好讓建出來的 image 內建一個確定的 release id(見 [L12](./.gitlab-ci.yml#L12))。
* 為什麼用這個值而非那個:retag 來源直接用 `$CI_COMMIT_SHORT_SHA`(FF-only 下,pre-prod 分支的最新 commit(也就是 tip)就是剛被合併進來的那個 dev commit,兩端讀同一個 GitLab 變數、字串天生對得上),而不是去解析 `origin/dev` HEAD——後者會跟並行的 dev push 競態、可能 promote 到還沒被 review 的新 image(見 [L495-504](./.gitlab-ci.yml#L495-L504))。
* 某個設定其實沒用但保留的理由:`http.connectTimeout` 對「連不上」無效,只當慢連線的廉價防護(見 [L139-141](./.gitlab-ci.yml#L139-L141))。

**要考慮的因素**:每一個「為什麼不是更直覺的那種寫法」的地方,都欠一行註解。

### 3. 錯誤訊息要「可操作」,不是只 `exit 1`

當驗證失敗時,印出**發生什麼 + 根因推測 + 逐步修復指令**,讓開發者照著做就能自救,而不是回頭問人。

```bash
echo "ERROR: Version $VERSION has already been released (tag $TAG exists)."
echo "This usually means 'dev' was not updated after the last pre-prod release."
echo "Debug steps:"
echo "  1. Check what version pre-prod is currently on: ..."
echo "  2. Pull the bumped pom.xml from pre-prod into dev: ..."
```

見 `version_guard` [L250-268](./.gitlab-ci.yml#L250-L268)。**要考慮的因素**:CI 失敗訊息的讀者是「當下被卡住、很焦慮的開發者」。把你 debug 時會做的事直接寫進訊息。

### 4. 把 job 之間的隱性約定寫成明確的 CONTRACT

當兩個 job 必須對某個格式達成一致(例如 A 產生的 tag 名稱要被 B 重建),就明白標註雙方的契約,避免任一邊單獨改動而破裂。

```yaml
# CONTRACT: image_retag_preprod rebuilds this exact string from the
# dev HEAD sha + pre-prod pom.xml — both sides must format it identically:
#   sha     = $CI_COMMIT_SHORT_SHA (first 8 chars of the commit)
#   version = raw pom <version>, SNAPSHOT kept
```

見 [L429-432](./.gitlab-ci.yml#L429-L432)、[L503-504](./.gitlab-ci.yml#L503-L504)。**要考慮的因素**:跨 job 共享的「字串格式 / 檔名 / tag 規則」是隱形耦合,要在兩端都註明,否則改一邊就壞。

---

## 二、結構與重用：不要複製貼上

### 5. 安全預設 + 例外覆寫

先用 `default:` 把「對多數 job 安全且省資源」的設定當基準,再**只在少數危險 job** 覆寫。比起每個 job 都寫一遍,既乾淨又不易漏。

```yaml
default:
  interruptible: true        # 多數 job:新 pipeline 可取消舊的,省 runner
# 部署 / 有副作用的 job 個別覆寫成 interruptible: false
```

見 [L21-22](./.gitlab-ci.yml#L21-L22)。**要考慮的因素**:哪個值「對 90% 的 job 是對的」?把它設成預設,讓例外顯眼。

### 6. 用 `extends` / 隱藏 job 抽出規則、快取、helper

把「觸發規則」「快取設定」「shell helper」抽成 `.` 開頭的隱藏 job,再用 `extends` 組合。觸發條件集中一處維護,一改全改。

```yaml
.dev_push_rules:   { rules: [...] }
.maven_cache:      { cache: [...], variables: { MAVEN_OPTS: "..." } }

maven_build_dev:
  extends: [.maven_cache, .dev_push_rules]   # 正交組合:快取 + 規則
```

見 [L41-106](./.gitlab-ci.yml#L41-L106)、[L341](./.gitlab-ci.yml#L341)。兩個 deploy job 共用的設定——AWS region、Bedrock 模型變數,以及「部署前先檢查 port 8090 有沒有被別的東西佔住」那段 `before_script`——也抽成 `.deploy_base`,`deploy_dev` 和 `deploy_production` 各自 `extends` 它(見 [L78-98](./.gitlab-ci.yml#L78-L98))。

> ⚠️ **`extends` 有個陷阱**:它對 `before_script` 這類「清單型」設定是**整段覆蓋、不是合併**。所以一旦某個 job `extends` 了帶 `before_script` 的 mixin(像這裡的 deploy),它自己就不能再寫 `before_script`,否則會把繼承來的那段整個蓋掉(見 `.deploy_base` 開頭的提醒註解 [L72-77](./.gitlab-ci.yml#L72-L77))。

**要考慮的因素**:同一段 `rules` 或 `cache` 在第二個 job 出現時,就該抽成 mixin;並把「必須同進同退」的設定綁在同一個 mixin 裡(例如快取路徑 `.m2/repository`,和指向它的 `-Dmaven.repo.local`,綁在一起就不會只改到一半)。

### 7. 用 `before_script` 當共用 shell 函式庫

利用「`before_script` 和 `script` 在同一個 shell 執行」的特性,把共用函式定義在 `before_script`,它們就會自動帶進每個繼承此 mixin 的 job——等於做出可共用的函式庫,卻不需要額外檔案。

```yaml
.ci_helpers:
  before_script:
    - |
      retry()       { ... }          # 通用重試
      git_net()     { git -c http.connectTimeout=30000 "$@"; }
      pom_version() { awk '...' }     # 解析 pom 版本
```

見 [L111-153](./.gitlab-ci.yml#L111-L153)。**要考慮的因素**:多個 job 重複的 shell 邏輯(解析、重試、設定),抽成函式集中維護。

---

## 三、正確性與冪等：可以安全重跑

### 8. 有副作用的操作一律設計成「可安全重跑」

CI 會因為網路、被取消、手動 retry 而重跑。任何「切 tag / 推 commit / 部署」的 job,都要先想「如果這步成功了一半、或整個再跑一次,會怎樣?」

```bash
if git rev-parse "refs/tags/$TAG" >/dev/null 2>&1; then
  echo "INFO: tag $TAG already exists — skipping retag (idempotent re-run)."
else
  ... 切 tag、推 image ...
fi
```

見 `image_retag_preprod` [L489-526](./.gitlab-ci.yml#L489-L526)。**要考慮的因素**:重跑會不會重複切 tag、重複部署、推壞分支?用「先檢查狀態、再決定要不要動作」,讓重跑變成「什麼都不做」(no-op)。

### 9. 用「權威來源」而非「本地狀態」做決策

job 的 checkout 可能是過時的某個 commit,本地檔案會誤導判斷。要決定「該不該做」時,去問**遠端的真實狀態**(remote tip / git tag),而不是看本地的工作目錄(working copy)。

```bash
# 本地 pom 永遠是 bump 前的版本,會誤判;改比對 remote 分支 tip
retry 3 15 git_net fetch origin "$CI_COMMIT_BRANCH"
git show "FETCH_HEAD:pom.xml" > "$remote_pom"
REMOTE_VERSION=$(pom_version "$remote_pom")
if [ "$REMOTE_VERSION" = "$NEXT" ]; then echo "already bumped, skip"; exit 0; fi
```

見 `prepare_next_release` [L565-581](./.gitlab-ci.yml#L565-L581)。**要考慮的因素**:我用來做判斷的資料,是不是這個 job 環境下「會說謊」的版本?對未知/異常狀態,寧可不動也不要覆寫([L577-581](./.gitlab-ci.yml#L577-L581))。

### 10. 解析資料時要對準「真正想要的那一個」

從設定檔抽值時,小心同名欄位。pom.xml 有兩個 `<version>`(父 POM 一個、專案一個),直接 `grep -m1` 會抓錯。先用錨點鎖定再抓。

```awk
/<artifactId>gen-bi<\/artifactId>/ { found=1; next }
found && /<version>/ { gsub(/.*<version>|<\/version>.*/, ""); print; exit }
```

見 [L145-150](./.gitlab-ci.yml#L145-L150)。**要考慮的因素**:我抓的欄位在檔案裡唯一嗎?不唯一就要先定位上下文(anchor),並把這個易錯點寫成註解 + 抽成共用函式重用。

### 11. 主動避免 CI 觸發迴圈

當 job 會 push 回觸發它的分支(例如自動 bump 版本),要防止無限迴圈。但要理解不同 skip 機制的「作用範圍」。

```bash
# 用 push option 只跳過「這一次 push」的 pipeline,
# 而不是把 [skip ci] 寫進 commit message(那會一路跟著 commit 到 master,
# 連帶把該跑的正式 pipeline 也跳掉)
retry 3 15 git_net push -o ci.skip "$REMOTE" HEAD:$CI_COMMIT_BRANCH
```

見 [L587-590](./.gitlab-ci.yml#L587-L590)。**要考慮的因素**:這個 job 會不會 push 而觸發自己?用哪種 skip——`push -o ci.skip`(只跳這次)還是 `[skip ci]`(跟著 commit 跑)——取決於這個 commit 之後會不會合併到其他需要跑 pipeline 的分支。

---

## 四、效能與快取：別讓 runner 做白工

### 12. 依「用途」分層設計快取 key

要怎麼設快取的 key,取決於一個問題:**這份快取的內容,在不同分支之間能不能共用?** 能共用,就用固定的 key、所有分支命中同一份;不能共用,就讓 key 跟著分支變、各分支各存一份:

| 快取對象 | key 策略 | 原因 |
|---|---|---|
| Maven 依賴(`.m2`) | **固定 key** `maven-cache-gen-bi` | 依賴與分支無關,所有分支共用以最大化命中 |
| Sonar 掃描快取 | **per-branch** `sonar-cache-$CI_COMMIT_REF_SLUG` | 掃描狀態與分支相關,不該共用 |
| Docker layer | `--cache-from` 先 `pull` 舊 image | 無狀態 runner 上重建 layer 的唯一辦法 |

見 [L100-106](./.gitlab-ci.yml#L100-L106)、[L184-190](./.gitlab-ci.yml#L184-L190)、[L418-424](./.gitlab-ci.yml#L418-L424)。

Docker 端還有一個進階技巧:`--build-arg CACHE_BUSTER="$(date +%s)"` 故意讓某個 build-arg 每次都變,**精準地讓某一層之後不被快取**,同時保留它前面的 layer 快取。

**要考慮的因素**:每一份要快取的東西,先問「它跟分支有關嗎?」決定 key;再問「無狀態 runner 上怎麼把上一次的成果帶進來?」

### 13. 用 DAG(`needs`)加速,並用 `optional` 容錯

用 `needs` 讓 job 一旦依賴就緒就開跑,不必等整個 stage;再用 `optional: true` 處理「上游可能被 `changes` 過濾跳過」的情形,避免下游卡死。

```yaml
needs:
  - job: maven_ci
    optional: true            # maven_ci 被 changes 過濾跳過時,下游照樣能跑
  - job: maven_build_dev
    artifacts: true           # 精確指定只取它的 artifact
```

見 [L350-352](./.gitlab-ci.yml#L350-L352)、[L439-442](./.gitlab-ci.yml#L439-L442)。**要考慮的因素**:哪些 job 其實沒有先後依賴、可以提早並行?上游被條件跳過時,下游該卡住還是該繼續?

### 14. 把工作放到「對的 image / runner」上

每個 job 用剛好夠用的環境,而不是一律用最重的。把容易出問題的操作隔離到專屬環境。

* git-only 的版本 bump → 輕量 `alpine:3`,不需要 DinD(見 [L550](./.gitlab-ci.yml#L550))。
* 需要建 image → `docker:24` + DinD service(見 [L409-412](./.gitlab-ci.yml#L409-L412))。
* 部署 → 帶特定 tag 的自架 runner(`lndata-1` / `lndata-oci-prod`)。

並刻意把「易失敗的 gitlab.com git push」拆到輕量 job,遠離笨重的 DinD job(見 [L532-533](./.gitlab-ci.yml#L532-L533))。**要考慮的因素**:這個 job 真正需要什麼能力?把不同性質(重 build vs 純 git vs 部署)的工作分開,各用合適環境。

---

## 五、供應鏈與發版：上線的就是測過的

### 15. Build-once、promote 已驗證的 artifact

最重要的供應鏈原則:**測過的那個二進位,就是上線的那個二進位**。image 只在 dev 建置一次,之後一路到正式環境都只「重新貼標籤」、不再重新編譯,避免「測的是這一份、上線的卻是另一份重編出來的」。

```
dev push    → 建 image,推 :dev 與 :dev-<sha>-<version>
pre-prod    → 只 docker tag + push 成 :vX.Y.Z(不重 build)
master      → promote-only,連 maven/sonar 都不重跑
```

見 [L444-458](./.gitlab-ci.yml#L444-L458) 的設計說明,與 `.validate_rules` 對 master 的排除註解 [L42-44](./.gitlab-ci.yml#L42-L44)。

而且要 **fail closed**(寧可整個失敗,也不退而用一個不確定的版本):retag 的來源固定是不可變的 `:dev-<sha>-<version>`(其中 `sha` 取 `$CI_COMMIT_SHORT_SHA`;FF-only 下 pre-prod 的最新 commit 就是那個 dev commit)。萬一這個 image 在 registry 還沒出現,短暫重試後就**直接失敗**,絕不退回會變動的 `:dev`——`:dev` 每次 dev push 都會被覆寫,退回它等於可能把還沒 review 過的 image 推上線,破壞「只用驗證過的 image」這個保證(見 [L507-516](./.gitlab-ci.yml#L507-L516))。

**要考慮的因素**:你的「測試環境的產物」和「正式環境的產物」是同一個嗎?如果是分別 build,兩者就可能不一致;promote 時也要確保抓到的是「那個 immutable 版本」,而不是會變動的 latest。

### 16. mutable + immutable tag 並存

同時推一個「會動的 tag」和一個「永遠不變的 tag」,兼顧方便與可追溯。

* `:dev`(mutable)——永遠指最新,方便引用。
* `:dev-<short-sha>-<version>`(immutable)——固定不變的追溯點,當作 promote 的來源,版本字串讓人在 registry 一眼看出對應哪次發版。

見 [L426-438](./.gitlab-ci.yml#L426-L438)。**要考慮的因素**:你需要「總是最新」還是「精確指定某一版」?答案通常是「都要」,所以兩個 tag 都推。

### 17. 設計回滾路徑

發版機制要從第一天就考慮「怎麼退回去」。正式部署設成手動,且部署的版本可被覆寫成任意舊 tag。

```yaml
deploy_production:
  when: manual
  # IMAGE_TAG 不寫死,預設取最新 vX.Y.Z tag;
  # 在 Run job 對話框 override(例如 IMAGE_TAG=v0.1.4)即可回滾
```

見 [L632-646](./.gitlab-ci.yml#L632-L646)、[L649-661](./.gitlab-ci.yml#L649-L661)。

另外兩個延伸做法:回滾時在**同一個 job 內**順手重新標記 Sentry,靠 `releases new` 的冪等性補建可能還不存在的舊版本(見 [L701-722](./.gitlab-ci.yml#L701-L722));部署完也把「實際上線的那個 tag」(包含手動回滾時填入的版本)寫進 `deployed-image-tag.txt` 當 artifact 保存,留下「這次到底部署了哪一版」的記錄(見 [L693-696](./.gitlab-ci.yml#L693-L696)、[L724-727](./.gitlab-ci.yml#L724-L727))。

**要考慮的因素**:出事要回到上一版,需要幾個步驟?能不能「一鍵指定某個舊版本重跑」?事後查得到當時上線的是哪一版嗎?

---

## 六、並行控制與安全：保護共享資源與機密

### 18. `resource_group` + `interruptible` 保護共享資源

兩個機制管不同層面,對「部署」這類動作通常成對使用:

| 設定 | 防的是 | 沒有它會怎樣 |
|---|---|---|
| `resource_group` | 兩個 job **同時跑**(跨 pipeline) | 並行部署互相覆寫、port 衝突、新舊版上線順序錯亂 |
| `interruptible: false` | 跑到一半**被新 pipeline 取消** | 容器部署到一半被砍,半成品狀態 |

```yaml
deploy_dev:
  resource_group: dev_deploy   # 跨 pipeline 互斥:一次只一個在部署
  interruptible: false         # 部署中不准被打斷
```

見 deploy 的 [L601-602](./.gitlab-ci.yml#L601-L602)、[L639-640](./.gitlab-ci.yml#L639-L640)。

**不只是部署**:發版用的 `image_retag_preprod` 和 `prepare_next_release` 也共用同一個 `resource_group: preprod_release`(見 [L467](./.gitlab-ci.yml#L467)、[L549](./.gitlab-ci.yml#L549))。否則兩條時間相近的 pre-prod pipeline 會雙雙通過「tag 還不存在」的檢查,接著搶著推同一個 `:vX.Y.Z`、git tag 和 bump pom——把「整段發版流程」綁進同一個 group,才能讓它們乖乖排隊、一次只跑一條。

**要考慮的因素**:這個 job 動到的「共享資源」是什麼(一台機、一個分支、一個 git tag、一個雲端資源)?兩個 pipeline 同時動它會不會出事?`stages`/`needs` 只在單一 pipeline 內排序,**跨 pipeline 的互斥要靠 `resource_group`**。

### 19. 最小化機密(secret)的暴露

機密的暴露面要壓到最小:不要 inline 出現在指令、用完即清、登入後盡快登出。

* secret 寫進 `mktemp` 產生的暫存 env file 再餵給 compose,用完 `rm -f`,而非散在指令列(見 [L614-626](./.gitlab-ci.yml#L614-L626))。
* `docker login` → `pull` → 立刻 `docker logout`,縮短憑證留存時間(見 [L609-611](./.gitlab-ci.yml#L609-L611))。
* 上線前檢查必要機密是否為空,並提示「variable scope / Protected 設定要對得上分支」這個常見坑(見 [L663-667](./.gitlab-ci.yml#L663-L667))。

**要考慮的因素**:機密在哪些地方會被看到(process list、log、磁碟、子行程環境)?每一處都想辦法縮短或消除暴露。

---

## 七、韌性：面對不可靠的網路與基礎設施

### 20. 通用 retry + 把網路呼叫收斂到單一入口

外部網路操作會間歇失敗。準備一個通用的 `retry`,並把所有同類的網路呼叫都收斂到同一個函式(只留一個統一的「進出口」,single seam),日後要加逾時、proxy 設定,只要改這一處。

```bash
retry() {              # retry <max> <delay> <cmd...>
  max=$1; delay=$2; shift 2; n=1
  while true; do
    "$@" && return 0
    [ "$n" -ge "$max" ] && { echo "ERROR: failed after $max attempts"; return 1; }
    n=$((n + 1)); sleep "$delay"
  done
}
git_net() { git -c http.connectTimeout=30000 "$@"; }   # 所有 git 網路呼叫的唯一入口
```

見 [L127-142](./.gitlab-ci.yml#L127-L142)。**要考慮的因素**:重試前先確認操作是**冪等**的(重推同 tag/commit 是 no-op),否則 retry 會造成重複副作用。

### 21. 把已知的基礎設施缺陷,連同根因註解一起寫進腳本

當基礎設施有你無法立刻修的缺陷(例如某條網路路徑間歇不通),在腳本裡做 workaround,並**完整記錄根因**,免得後人以為是多餘程式碼而刪掉。

```bash
# CI 容器到 gitlab.com 的 IPv6 路徑間歇不通,git fetch/push 約 2s 就失敗。
# 解析出 IPv4 並釘進 /etc/hosts,強迫走 IPv4。
GLIP=$(dig +short A "$CI_SERVER_HOST" | grep -E '^[0-9.]+$' | head -1)
[ -n "$GLIP" ] && echo "$GLIP $CI_SERVER_HOST" >> /etc/hosts
```

見 [L114-125](./.gitlab-ci.yml#L114-L125)。**要考慮的因素**:治本(修 infra)做不到時的治標方案,要附上「為什麼需要它」,並在治本後能被安全移除。

---

## 結語：設計前的檢查清單

設計前的檢查清單,就是開頭那七個面向——它們不是寫完才回頭檢查,而是**動手前**就該逐一自問的提問:[意圖與可讀性](#一可讀性與意圖讓人三分鐘看懂)、[結構與重用](#二結構與重用不要複製貼上)、[正確性與冪等](#三正確性與冪等可以安全重跑)、[效能與快取](#四效能與快取別讓-runner-做白工)、[供應鏈與發版](#五供應鏈與發版上線的就是測過的)、[並行控制與安全](#六並行控制與安全保護共享資源與機密)、[韌性與回滾](#七韌性面對不可靠的網路與基礎設施)。

把這七個面向當成設計腳本的固定起點,大多數「能跑但脆弱」的 pipeline 問題,都能在寫之前就避開。
