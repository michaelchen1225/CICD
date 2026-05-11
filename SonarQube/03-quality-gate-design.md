# SonarQube Quality Gate 設計：PoC 與 Product 兩階段

本篇大致介紹一 Quality Gate 的設計範例 (純理論)，實際操作細節請參考 [04-quality-gate-and-results.md](04-quality-gate-and-results.md)。

> 針對不同產品階段設計兩組 Quality Gate（QG），用 SonarQube Community Build 26.4（MQR mode）內建可設定的條件落地。

## 官方依據

- [Metric definitions](https://docs.sonarsource.com/sonarqube-server/10.8/user-guide/code-metrics/metrics-definition/) — 所有 rating 的判定門檻（Security / Reliability / Maintainability / Security Review）皆來自此頁，包含 MQR mode 下的 severity 對照與 Maintainability 的 Tech Debt Ratio 區間。
- [Quality Gates](https://docs.sonarsource.com/sonarqube-server/10.8/instance-administration/analysis-functions/quality-gates) — Quality Gate 條件可設定範圍、`On New Code` / `On Overall Code` 區分。

## 設計原則

- **只對 New Code 卡 QG**：遵循官方 Clean as You Code 原則。舊技術債用 Overall Code 看趨勢、不卡 pipeline。
- 
- **Rating 的判定是「最嚴重那一顆」**：不是計數。例如 Security Rating B 代表「最嚴重只到 Low」，而不是「允許某個數量的 issue」。
- 
- **Hotspot Reviewed 一定要顯式設定**：不設條件不代表不檢查，預設要求 100%。

## 三個 Software Qualities 的官方定義

引自官方文件 [Software qualities](https://docs.sonarsource.com/sonarqube-server/10.8/core-concepts/software-qualities)：

> "High quality code contributes to software that is secure, reliable, and maintainable. These three aspects, security, reliability, and maintainability, are called *software qualities* in SonarQube and they contribute to the long-term value of your software."

| Software Quality | 官方定義（原文） | 中文整理 |
| --- | --- | --- |
| **Security** | "Security is the protection of your software from unauthorized access, use, or destruction." | 保護軟體免於未授權存取、使用或破壞 |
| **Reliability** | "Reliability is a measure of how your software is capable of maintaining its level of performance under stated conditions for a stated period of time." | 衡量軟體在指定條件下、指定時間內維持效能的能力 |
| **Maintainability** | "Maintainability refers to the ease with which you can repair, improve and understand software code." | 衡量修復、改進、理解程式碼的容易程度 |

## Severity 等級官方定義

同樣來自上面那篇官方文件，MQR mode 下 issue 的五個 severity 等級：

| Severity | 官方說明 |
| --- | --- |
| **Blocker** | Issues with significant probability of severe unintended consequences requiring immediate fixes, including production crashes and security vulnerabilities |
| **High** | Issues with substantial application impact needing prompt resolution |
| **Medium** | Issues with moderate impact |
| **Low** | Issues with minimal impact |
| **Info** | Issues with no expected application impact; informational only |

## 實用的核心 Metrics

設 Quality Gate 之前，先認識幾個常用 metric 的 key 與意義會有幫助——QG 條件設定面板下拉選的就是這些。完整清單見官方 [Metric definitions](https://docs.sonarsource.com/sonarqube-server/10.8/user-guide/code-metrics/metrics-definition)，下面挑出實務上最常用到的：

### Rating 類（QG 條件最常用）

| Metric | Key（MQR） | 用途 |
| --- | --- | --- |
| Security Rating on New Code | `new_software_quality_security_rating` | 看新 code 最嚴重的 Security issue severity，設 QG 卡資安 |
| Reliability Rating on New Code | `new_software_quality_reliability_rating` | 看新 code 最嚴重的 Reliability issue severity，設 QG 卡 bug |
| Maintainability Rating on New Code | `new_software_quality_maintainability_rating` | 看新 code 的 Tech Debt Ratio 對應 A–E 等級 |
| Security Review Rating on New Code | `new_security_review_rating` | 看 Hotspot Reviewed 比例 |

### 計數類（追蹤絕對數量、做趨勢圖）

| Metric | Key（MQR） | 用途 |
| --- | --- | --- |
| Security issues / new | `software_quality_security_issues` / `new_software_quality_security_issues` | Security issue 總數 / 新增數 |
| Reliability issues / new | `software_quality_reliability_issues` / `new_software_quality_reliability_issues` | Reliability issue 總數 / 新增數 |
| Maintainability issues / new | `software_quality_maintainability_issues` / `new_software_quality_maintainability_issues` | Code Smell 總數 / 新增數 |
| 各 severity 計數 | `software_quality_blocker_issues` / `_high_issues` / `_medium_issues` / `_low_issues` / `_info_issues` | 依 severity 拆開看，做嚴重度分布圖 |
| Accepted issues | `accepted_issues` / `new_accepted_issues` | 被工程師判定為 Accept 的 issue 數，反映「個案豁免」的累積量 |

### 技術債類（Maintainability 的量化）

| Metric | Key | 用途 |
| --- | --- | --- |
| Technical Debt | `software_quality_maintainability_remediation_effort`（= `sqale_index`） | 修完所有 Maintainability issue 的總分鐘數，趨勢觀察用 |
| Technical Debt on New Code | `new_software_quality_maintainability_remediation_effort`（= `new_technical_debt`） | 新 code 帶入的 debt 分鐘數，做 sprint / release 報告用 |
| Technical Debt Ratio | `software_quality_maintainability_debt_ratio`（= `sqale_debt_ratio`） | 上面那個除以「重寫成本」的比值，對應 Maintainability Rating |

### 覆蓋率與重複類（QG 常見條件）

| Metric | Key | 用途 |
| --- | --- | --- |
| Coverage on New Code | `new_coverage` | 設 QG「新 code 測試覆蓋率 ≥ N%」 |
| Lines to cover on New Code | `new_lines_to_cover` | 新增的可覆蓋行數；fudge factor 看的就是這個（< 20 自動跳過 coverage 條件） |
| Duplicated Lines (%) on New Code | `new_duplicated_lines_density` | 設 QG「新 code 重複行比例 < N%」 |
| Duplicated Blocks on New Code | `new_duplicated_blocks` | 重複的程式碼區塊數量（至少 100 tokens 才算一塊） |

### 規模與複雜度類（不設 QG，但 Dashboard / 趨勢圖會用）

| Metric | Key | 用途 |
| --- | --- | --- |
| Lines of Code | `ncloc` | 有效程式碼行數（不含空白與註解），是 Tech Debt Ratio 公式的分母基礎 |
| New Lines | `new_lines` | 新增實體行數（含空白） |
| Cyclomatic Complexity | `complexity` | 控制流路徑數，量化邏輯分支 |
| Cognitive Complexity | `cognitive_complexity` | 評估「讀起來有多難懂」，比 Cyclomatic 更貼近人類感受 |

> **MQR 與 Standard Experience 的 metric key 並存**：官方文件兩種都列。例如 `bugs` / `vulnerabilities` / `code_smells` 是 Standard Experience 的 key，仍可使用但已是 legacy。新做 dashboard 或 API 串接建議用 `software_quality_*` 前綴的 MQR key。

## 評級標準一覽

四個 rating 都採用 A（最好）到 E（最差）的五級制，但**判定邏輯各自不同**：Security / Reliability 看 issue 的最高 severity；Maintainability 看 Tech Debt Ratio；Security Review 看 Hotspot Reviewed 比例。下面數值門檻全部來自 [Metric definitions](https://docs.sonarsource.com/sonarqube-server/10.8/user-guide/code-metrics/metrics-definition/)。

### Security Rating（MQR mode）

依當前範圍內**最嚴重那一個** Security issue 的 severity 決定。

| Rating | 判定條件 |
| --- | --- |
| A | 0 個 issue（或僅 Info） |
| B | 至少 1 個 **Low** |
| C | 至少 1 個 **Medium** |
| D | 至少 1 個 **High** |
| E | 至少 1 個 **Blocker** |

### Reliability Rating（MQR mode）

判定邏輯與 Security 完全相同，只是看 Reliability 類 issue。

| Rating | 判定條件 |
| --- | --- |
| A | 0 個 issue（或僅 Info） |
| B | 至少 1 個 **Low** |
| C | 至少 1 個 **Medium** |
| D | 至少 1 個 **High** |
| E | 至少 1 個 **Blocker** |

### Maintainability Rating（sqale_rating）

依 **Technical Debt Ratio** 判定。下面三個概念是層層相依：先有 Technical Debt，再算出 Ratio，最後對應到 Rating。內容引自官方 [Metric definitions — Maintainability](https://docs.sonarsource.com/sonarqube-server/10.8/user-guide/code-metrics/metrics-definition#maintainability)。

#### (1) Technical Debt

> "The technical debt is the sum of the maintainability issue remediation costs. An issue remediation cost is the effort (in minutes) evaluated to fix the issue. It is taken over from the effort assigned to the rule that raised the issue."
>
> "An 8-hour day is assumed when the technical debt is shown in days."

換句話說：

- Technical Debt = Σ（所有 Maintainability issue 的 remediation cost，單位：分鐘）
- 每條 SonarQube 規則本身會預先標註「修一次大約要幾分鐘」（rule effort），issue 的 remediation cost 就直接沿用該值。
- 顯示成「天」時是以 **1 天 = 8 小時** 換算。

#### (2) Technical Debt Ratio

> "The technical debt ratio is the ratio between the cost to develop the software and the technical debt (the cost to fix it). It is calculated based on the following formula:
>
> `sqale_debt_ratio` = technical debt /(cost to develop one line of code * number of lines of code)
>
> Where the cost to develop one line of code is predefined in the database (by default, 30 minutes)."

公式拆解：

```text
sqale_debt_ratio = technical_debt / (cost_per_line × lines_of_code)
```

- `cost_per_line` 預設 **30 分鐘 / 行**。可在 **Administration → Configuration → General settings → Technical Debt** 的 `Development cost` 欄位調整（property 名稱 `sonar.technicalDebt.developmentCost`，單位：分鐘）。詳見官方 [Metrics parameters](https://docs.sonarsource.com/sonarqube-server/10.8/instance-administration/analysis-functions/metrics-parameters)。
- 分母代表「重新開發一遍這個專案估計要花的時間」。
- 分子（technical_debt）跟分母同單位（分鐘）相除得到無單位比值，再以 % 呈現。

官方提供的範例：

| 項目 | 數值 |
| --- | --- |
| Technical debt | 122,563 分鐘 |
| Number of lines of code | 63,987 行 |
| Cost to develop one line of code | 30 分鐘 |
| **Technical debt ratio** | **6.4%** |

驗算：`122563 / (30 × 63987) = 122563 / 1919610 ≈ 6.38%`。

白話解讀：**「修完所有技術債的工時，相當於重寫整個專案工時的 6.38%。」** 注意 ratio 不是「6.38% 的程式碼是爛 code」，而是時間/成本比；重寫一遍要 ~4,000 工作天，修完現有 issue 要 ~255 工作天，兩者相除。對應到 rating 是 **B**（5%–10%）。

#### (3) Maintainability Rating

> "The default Maintainability rating scale (`sqale_rating`) is:
>
> - A ≤ 5% to 0%
> - B ≥ 5% to <10%
> - C ≥ 10% to <20%
> - D ≥ 20% to < 50%
> - E ≥ 50%
>
> You can define another maintainability rating grid."

| Rating | Technical Debt Ratio |
| --- | --- |
| A | 0% – ≤ 5% |
| B | ≥ 5% – < 10% |
| C | ≥ 10% – < 20% |
| D | ≥ 20% – < 50% |
| E | ≥ 50% |

> Maintainability rating **不看 severity**——所有 Maintainability issue 的 remediation cost 加總後換算為 Ratio。這是它與 Security / Reliability 的關鍵差別。
>
> 上表是預設值。Administration 介面可以自訂一組新的 grid（"You can define another maintainability rating grid."）。

### Security Review Rating

依 **Hotspot Reviewed 比例**判定。Hotspot 必須被人工 Review 後標記為 `Acknowledged` / `Fixed` / `Safe` 才算 reviewed；`To review` 狀態不計入。

| Rating | Hotspot Reviewed 比例 |
| --- | --- |
| A | 100% |
| B | ≥ 90% |
| C | ≥ 70% |
| D | ≥ 50% |
| E | < 50% |

## QG 1：`PoC-Gate`

定位：能跑、不爆。允許技術債，但擋掉嚴重 bug。

| # | 條件（Metric） | 範圍 | Operator | 門檻 |
| --- | --- | --- | --- | --- |
| 1 | Security Rating | On New Code | is worse than | **C** |
| 2 | Reliability Rating | On New Code | is worse than | **C** |
| 3 | Maintainability Rating | On New Code | is worse than | **D** |

### PoC-Gate 各條件設計理由

- **Security Rating ≤ C**：PoC 重點在驗證功能而非上線，允許 Medium 等級的 Security issue（例如未做完整輸入清洗的次要欄位）可以先用、後修；但 High / Blocker（可被利用的漏洞）即使 PoC 也該擋——一旦演示環境被滲透就直接結案。
- 
- **Reliability Rating ≤ C**：PoC 程式跑壞可以重啟，允許 Medium bug（例如部分 race condition、未處理的次要 exception）；但 High（NPE 在主流程、resource leak）會讓 demo 直接掛掉，仍須擋。
- 
- **Maintainability Rating ≤ D**：PoC 階段預期會大量改寫、推翻重來，技術債本來就高。D（< 50%）是保留一道底線——超過 50%（E）代表「修完現有 debt 比重寫一半專案還貴」，這時連 PoC 都該停下來檢討。
- **不設 Coverage**：PoC 通常沒寫測試（甚至刻意不寫，因為需求還在變），強制 70% 會逼工程師寫沒意義的 test，反而拖慢驗證速度。
- 
- **不設 Duplicated Lines**：PoC 常複製貼上嘗試不同實作，DRY 不是這階段重點。
- 
- **不設 Hotspots Reviewed**：PoC 通常不上 public 環境，Security Hotspot 等 production 化時再一次審。但要**顯式設為「不檢查」**——預設值是 100%，不設 = 卡死。

## QG 2：`Product-Gate`

定位：上線品質基準。新寫的 code 高標準，舊技術債不卡。

| # | 條件（Metric） | 範圍 | Operator | 門檻 |
| --- | --- | --- | --- | --- |
| 1 | Security Rating | On New Code | is worse than | **A** |
| 2 | Reliability Rating | On New Code | is worse than | **B** |
| 3 | Maintainability Rating | On New Code | is worse than | **B** |
| 4 | Coverage | On New Code | is less than | **70.0%** |
| 5 | Duplicated Lines (%) | On New Code | is greater than | **3.0%** |
| 6 | Security Hotspots Reviewed | On New Code | is less than | **100%** |

### Product-Gate 各條件設計理由

- **Security Rating = A**：Security 問題的後果是**外部攻擊面**（SQL injection、XSS、權限繞過），即使 Low 等級也可能是真實漏洞（例如弱加密演算法）。上線後修補成本遠高於開發時擋掉，**不留情**。
- 
- **Reliability Rating ≤ B**：Reliability 是會「跑壞」的 bug。抓 A 會卡到很多誤判跟邊角情境（例如某些 lint 認為 `equals` 沒處理 null 是 Low bug，但呼叫處已保證非 null），導致工程師疲於 suppress。抓 B 允許工程師針對 Low 用 `Accept` / `False Positive` 個案處置，但 Medium 以上（已可能在 production 出狀況）一律修。
- 
- **Maintainability Rating ≤ B**：抓 A（< 5%）對「新寫的」程式太嚴，重構過程中暫時超過很正常。抓 B（5–10%）給工程師合理緩衝，但不允許新 code 一寫就帶 20% 債（那是 C 的範圍）——半年後重構成本會爆炸。
- 
- **Coverage ≥ 70%**：業界普遍共識是「新 code 70% 起跳」。抓 80% 以上會逼工程師為了數字寫沒意義的測試（例如 getter / setter），70% 強迫覆蓋主要邏輯但留彈性給邊角。
- 
- **Duplicated Lines < 3%**：3% 是 Sonar way 預設值，也是業界公認的「複製貼上警戒線」。超過代表開始累積維護負擔（同一段邏輯要改多處）。
- 
- **Hotspots Reviewed = 100%**：Hotspot 不是 issue，是「需要人工判斷風險」的點。不審 = 不知道風險。漏一個就足以讓 production 出資安事件。且**不設這條，Security Review Rating 會鎖死在 E**（< 50% reviewed），間接影響 Overall Security 評等。

### 為什麼 Product 不抓 AAA

> 三條 rating 都抓 A 表面上最嚴，實際上工程師會疲於 suppress Low issue，QG 變成「裝飾」而不是「品質工具」。
>
> Security 因為**後果不對稱**（一個漏洞可能造成資安事件），值得抓 A；Reliability / Maintainability 抓 B 留一格容忍空間，讓工程師有判斷餘地，QG 才會被認真看待。**A / B / B 是針對三種失敗代價量身設計**：Security 零容忍、Reliability 給 Low 緩衝、Maintainability 給合理債務空間。

## 設定步驟

兩個 QG 設定流程相同：

1. 頂部導覽列 → **Quality Gates**
2. 左側 → **Create** → 輸入名稱（`PoC-Gate` 或 `Product-Gate`）
3. 進入新建的 QG → **Add Condition** → 選 `On New Code` → 選 Metric → 填 Operator + Value
4. 重複加完所有條件
5. 在對應的 Project → **Project Settings → Quality Gate** 指派

## 與 PM 原始草案的差異

| PM 原本 | 調整為 | 原因 |
| --- | --- | --- |
| Product Maintainability **C**（debt 11–20%） | **B**（< 10%） | C 對「新 code」偏鬆，新寫的 code 拿 B 不難 |
| Product Hotspots Reviewed「再討論」 | **100%** | 不設等於擋不到；不審 Hotspot 會讓 Security Rating 鎖在 E |
| PoC Maintainability **E**（< 50%） | **D**（worse than D） | E 本身就是 ≥50%，「允許到 E」＝實際上沒擋 |
| PM 用 Minor / Major / Critical 描述 bug | 改用 Low / Medium / High | MQR mode 詞彙；Minor ≈ Low、Major ≈ Medium、Critical ≈ High |

## SonarQube 掃不到的自訂規則

PM 提到的這幾條，內建規則庫無法處理，建議拉到 pipeline 另一個 stage 用 **ArchUnit** 落地：

| 自訂規則 | 落地方式 |
| --- | --- |
| Controller 不可直接呼叫 Repository | ArchUnit 架構測試 |
| 敏感欄位必須加密 | 自訂 annotation + ArchUnit |
| SQL 執行必須經過 guard | 自訂 annotation + Code Review |

技術上可以寫 SonarQube 自訂規則 plugin（sonar-java API），但要打 jar、改 scanner image、長期維護，ROI 不高。pipeline 開獨立 `architecture-check` stage 跑 ArchUnit 與 SonarQube QG 平行擋，兩邊各做擅長的事，訊息也清楚。
