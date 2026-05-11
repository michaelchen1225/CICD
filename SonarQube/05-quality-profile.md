# SonarQube --- Quality Profile

[前一篇](04-quality-gate-and-results.md) 講完 Quality Gate 怎麼判結果，這篇補上另一個容易跟 QG 混淆的概念：**Quality Profile (QP)**。多數團隊（包含目前的 Java 專案）其實沿用內建 Sonar way QP 就夠用，所以本篇以「概念釐清 + 何時才需要動手」為主，操作步驟點到為止。

## Quality Profile vs Quality Gate

兩者在 SonarQube 流程裡的位置不同：

```
code → 套用 Quality Profile → 產生 issue / hotspot → 套用 Quality Gate → pass / fail
```

| 元件 | 作用階段 | 決定什麼 | 範圍 |
| --- | --- | --- | --- |
| **Quality Profile** | 掃描階段 | 「該掃哪些規則」——哪些規則啟用、severity 覆寫 | 按**語言**一份（Java 一份、JS 一份） |
| **Quality Gate** | 判定階段 | 「結果要符合哪些條件」——issue 數、coverage、duplication、hotspot 是否讓 pipeline pass | 按**專案**一份 |

一句話：**QP 決定看到什麼 issue，QG 決定看到的 issue 能不能讓 pipeline 過**。

## Sonar way（內建 QP）的角色

每種語言安裝時 SonarQube 都會帶一份 `Sonar way` QP，標記 `Default` + `Built-in`：

* Scanner 在掃描時依專案語言**自動套用該語言當前指派的 QP**；未個別指派就用 `Default` 的那份
* 與 Quality Gate 的 Sonar way 一樣，built-in QP 不可直接編輯規則；要改必須 `Copy` 一份再修
* QP 是「按語言」隔離的——客製 Java QP 不會影響同一 server 上 JS 專案的掃描

對多數團隊而言，Sonar way 收錄的規則集合已涵蓋 SonarSource 認定的關鍵規則，**不必額外客製就能用**。本文也建議：先沿用預設、累積足夠的掃描樣本再決定要不要動 QP，避免一開始就為了消滅噪音而停掉關鍵規則。

## 什麼時候才需要客製 QP

下列情境才有動手的價值，否則維持預設：

* **同一條規則在這個專案 context 下持續產生雜訊**（例如 `java:S1192` 字串字面量重複在某些 DSL/SQL 場景天天觸發），且不是個案誤判
* **內部 coding style 與 Sonar way 預設衝突**——例如團隊有自己的命名規範
* **想對某類規則整批調整 severity**——例如把所有 `Maintainability | Low` 統一降為 `Info`

如果只是「這一個 issue 不算」這種**個案豁免**，正確作法是在 Issue 上標 `False Positive` / `Accept`（見 [04-quality-gate-and-results.md §Issues 的處置動作](04-quality-gate-and-results.md#issues-的處置動作)），而不是停用整條規則。**規則性的噪音改 QP、個案誤判改 Issue 狀態**——兩者分工清楚。

## 客製 QP 的操作流程（概覽）

實機需要時的路徑，不展開細節：

1. `Quality Profiles`（頂部導覽列）→ 找到語言下的 `Sonar way` → 右上角 `Copy` → 命名（例如 `Java way (relaxed)`）
2. 進入複製出來的 QP → 在規則列表用 filter 找到目標規則（可用規則 ID 如 `java:S1192` 直接搜）→ 切換 `Activate` / `Deactivate` 或調整 severity
3. 指派：`Projects → 該專案 → Project Settings → Quality Profiles` → 該語言那一行從 `Use the default` 切到 `Always use a specific quality profile` → 選自訂 QP → `Save`

<!-- ![alt](placeholder-qp-copy.png) -->

> 規則改動只影響「下次掃描」，**不會回追既有 issue**——已存在的 issue 要等對應程式碼被改動後才會重新評估。

## 與 Quality Gate 的協同

調整 pipeline 把關鬆緊度時，先判斷問題層級：

* **規則本身要不要篩** → 改 QP（例如「不想看到 `java:S1192` 的 issue」）
* **結果門檻要不要鬆** → 改 QG（例如「跳過測試的專案不想被 coverage 卡」，見 [03 §跳過測試的專案怎麼處理 Quality Gate](04-quality-gate-and-results.md#跳過測試的專案怎麼處理-quality-gate)）

兩者通常一起調，不是二選一。但實務上多數團隊在 SonarQube 導入初期**只需要動 QG**（移掉 coverage、調 rating），QP 維持 Sonar way 即可；等累積幾個月掃描樣本、看出哪些規則對團隊真的是雜訊，再回頭精修 QP。

---

## 小結

* QP 與 QG 是不同階段的設定：QP 在掃描階段決定產出哪些 issue，QG 在判定階段決定 issue 能不能讓 pipeline 過
* QP 按語言隔離、QG 按專案指派
* Sonar way QP 對多數團隊已足夠，**沒有強需求就不要動**——個案誤判用 Issue 狀態解，規則性噪音才考慮客製 QP
* 完整 QG 設計範例見 [Quality Gate 設計（PoC / Product）](03-quality-gate-design.md)
