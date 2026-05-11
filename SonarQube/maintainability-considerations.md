# Maintainability 的長期考量

> 補充 [03-quality-gate-design.md](03-quality-gate-design.md)：探討 Maintainability rating 在 New Code / Overall Code 上的差異、技術債的長期累積監控、AI 輔助時代的參數調整。

## 一、New Code 上的 ratio 計算範圍

官方 [Metric definitions](https://docs.sonarsource.com/sonarqube-server/10.8/user-guide/code-metrics/metrics-definition) 對 New Code 上的 Maintainability metric 只給了文字定義，**沒有明寫公式分母用的是哪個 LOC**：

| Metric | 官方定義 |
| --- | --- |
| `new_technical_debt` | "A measure of effort to fix the maintainability issues raised for the first time on new code." |
| `new_software_quality_maintainability_rating` | "The rating related to the value of the technical debt ratio on new code." |

從 SonarQube 一貫的 `new_*` metric 慣例推論，分子分母應該**都限縮到 New Code 範圍**：

```text
new_sqale_debt_ratio ≈ 新增 maintainability debt / (cost_per_line × 新增 LOC)
```

否則 6 萬行的舊專案新增 50 行帶 30 分鐘 debt，ratio = 30 / (30 × 60000) ≈ 0.0017%，永遠是 A，QG 等於沒設——這顯然不是設計意圖。

合理的解讀是：**「修完這次 PR 帶進來的 Code Smell，要花的工時佔『重寫這段新 code』工時的幾 %」**。

對應到不同 PR 規模的感受：

| 情境 | 分母 LOC | 6 分鐘 debt 對應的 ratio |
| --- | --- | --- |
| Overall（全專案 63,987 行） | 63,987 | 0.0003% → A |
| New Code（PR 新增 200 行） | 200 | 1.0% → A |
| New Code（PR 新增 20 行） | 20 | 10% → C |

所以對 New Code 設 Maintainability Rating 條件實際上**相當嚴**——小 PR 帶一點 debt 就可能掉到 B/C，這正是 Clean as You Code 想要的效果。

## 二、容忍 B 的長期累積問題

New Code QG 設 `Maintainability Rating ≤ B` 的意思是：每次 PR 都可以**合法**帶入少量 debt（< 10% ratio）。單次無感，但 100 次 PR 累積後 Overall Code 上就會出現可觀的 debt。

更關鍵的是：**New Code 視窗（預設 30 天或上次 release 之後）一過，這些 debt 就「沉澱」變成 Overall Code 的一部分，不再被 QG 擋。**

這是 Clean as You Code 原則的盲點——它擋得住「一次性的爛 code」，擋不住「每次都合法、長期慢性累積」。

## 三、Overall Code 是即時更新的

關鍵事實：**Overall Code 的指標每次掃描都會重新計算**，反映 HEAD 的當前狀態。SonarQube 不是只記第一次掃描的結果。

| 範圍 | 每次掃描的行為 |
| --- | --- |
| **Overall Code** | 重新掃整個 codebase，所有指標（issue 數、debt、ratio、rating）即時反映當前 HEAD |
| **New Code** | 相對於 New Code baseline 計算差異，baseline 由 New Code definition 決定 |

直觀例子：今天掃描 Overall Tech Debt = 255 天，隔週修掉 50 天的 debt 再掃，Overall Tech Debt 會變成 205 天，Maintainability Rating 也可能從 B 回到 A，**Activity tab** 折線圖會記錄這次下降。

可能造成「只記第一次」誤解的兩個來源：

1. **Issue 的 Introduction date**：每個 issue 記錄它第一次被偵測到的時間（用來判斷是否屬於 New Code），這是 issue 屬性，不是 Overall 指標的更新邏輯。
2. **New Code baseline**：baseline 一旦設定，在下個 version 發佈前是固定的；但 Overall Code 跟 baseline 無關，永遠跟 HEAD 同步。

## 四、長期技術債的監控策略

光靠 New Code QG 不夠，需要對 Overall Code 做**趨勢監控**——不是設 QG，而是定期 review。

### 應該追蹤的三個 Overall 指標

| 指標 | 看什麼 | 警訊 |
| --- | --- | --- |
| **Technical Debt（絕對值）** | 趨勢圖：是否單調上升 | 連續 3 個月只增不減 |
| **Maintainability Rating on Overall Code** | 等級是否劣化 | A → B、B → C 的跨級劣化 |
| **Technical Debt Ratio on Overall Code** | 是否逼近下一級門檻 | 例如從 4.x% 爬到 4.9%，快掉 B |

工具：專案頁 → **Activity** tab 提供每次掃描的歷史折線圖，是判斷趨勢最直接的方式。

### 判斷「合理累積」的準則

光看絕對值有沒有變大太簡化，更務實的判準是**債務累積速度 vs 程式碼成長速度**：

- **健康**：Tech Debt 跟 LOC 同比例成長 → ratio 持平，rating 不變
- **警告**：Tech Debt 成長快過 LOC → ratio 上升，遲早跨級
- **危險**：LOC 沒長、Tech Debt 在長 → 純粹在現有程式碼上堆 smell（通常是 hotfix 趕工後遺症）

### 為什麼不直接對 Overall Code 設 QG

理論上可以設「Overall Maintainability Rating ≤ B」，但實務上會踩兩個雷：

1. **接舊專案直接 fail**：很多 legacy 專案 Overall 一開始就是 D/E，QG 永遠紅，工程師就會直接無視 QG。
2. **共業問題**：Overall 是全團隊歷史累積，當前 PR 作者修不動別人的舊 code，卡住他的 PR 不公平。

所以 Overall Code 適合**監控**（dashboard + 定期 review），不適合**強制阻擋**（QG）。

### 推薦的混合策略

| 層級 | 機制 | 標準 |
| --- | --- | --- |
| **每次 PR** | New Code QG 卡 | Maintainability Rating ≤ B |
| **每個 Sprint** | Tech Lead review Overall 趨勢 | Ratio 持平或下降 |
| **每個 Release** | 強制償債 sprint | 拉回 Overall A，或至少消化該 release 新增 debt 的 50% |
| **每年** | Architecture review | 評估是否對特定模組做大重構 |

一句話結論：**New Code QG 保證「不變壞」，Overall Code 趨勢監控保證「有在變好」。前者卡 pipeline，後者進 sprint review。**

## 五、AI 輔助時代下 `cost_per_line` 的調整

`cost_per_line` 預設 30 分鐘/行 是 pre-AI 時代的估算。現在 Copilot / Claude 普及後，實際每行開發成本明顯下降——但 ratio 公式分母沒變，算出來的 Tech Debt Ratio 會比實際樂觀，rating 也偏好看。

### 三種調整方向

**方向 1：下修 `cost_per_line`**

| 情境 | 建議值 |
| --- | --- |
| 重度 AI 輔助 | 10–15 分鐘/行 |
| 部分 AI 輔助 | 20 分鐘/行 |
| 不用 AI | 維持 30 分鐘 |

副作用：SonarQube 的 rule effort（分子那邊每條 issue 的修補分鐘數）是內建寫死的，沒有跟著 AI 時代下修。分母動、分子不動，會讓 ratio 比真實的「修債務 ÷ 重寫」更高，反向失真。

**方向 2：保留預設，收緊 rating grid**

直接在 Administration → Configuration → Maintainability rating grid 改門檻：

| Rating | 預設 | AI 時代建議 |
| --- | --- | --- |
| A | ≤ 5% | ≤ 3% |
| B | < 10% | < 6% |
| C | < 20% | < 12% |
| D | < 50% | < 30% |
| E | ≥ 50% | ≥ 30% |

邏輯：AI 讓修債務也變快、開發也變快，兩邊抵消後 ratio 數字不一定要變，但「同樣 ratio 的容忍度」應該下降。

**方向 3（推薦）：兩邊都不動，改看絕對值與趨勢**

AI 輔助下，真正有意義的指標不是 Ratio，而是「**絕對 Technical Debt 的成長速度**」。

理由：

- Ratio 受兩個都被 AI 影響的參數左右，數字本身的可比性下降
- 修債務的門檻變低後，重點從「能不能修」變成「願不願意每個 PR 順手清」
- 不如在 QG 加一條 `Maintainability Issues on New Code = 0`（新 code 不准帶任何 Code Smell），讓 AI 紅利直接體現在「每次都乾淨」而非「比例好看」

### 配置選擇對照

| 目標 | 配置 |
| --- | --- |
| 數字接近現實 | `cost_per_line = 15`、rating grid 不動 |
| 維持與舊專案可比性 | `cost_per_line = 30`、改用方向 3 的新 QG 條件 |
| 最嚴格 | `cost_per_line = 15` + rating grid 收緊一半 |

實務上**方向 3 最划算**——不用動 instance-wide 設定（影響所有專案）、不用解釋為什麼數字突然惡化，而是在 QG 層級加一條「New Code 0 個 Maintainability issue」，把 AI 紅利直接綁進新 code 品質要求。
