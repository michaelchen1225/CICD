# 如何 commit 時跳過 CI/CD ?

有時候，某些 commit 並不需要觸發 CI/CD，以下有兩種方式跳過 CI/CD：

* 在 commit message 中加入 `[ci skip]` 或 `[skip ci]` 字樣

```bash
git commit -m “<commit massage>” -m “[skip ci]”
```

* 在 push 時加入 `-o ci.skip`：

```bash
git push -o ci.skip
```

雖然兩者效果類似，但**關鍵差異**在於 skip 標記「住在哪裡」：`[skip ci]` 寫在 **commit message** 裡，會跟著 commit 一起被合併；而 `-o ci.skip` 是 **push 當下的選項**，不會寫進 commit。

以下是一個之前踩過的坑。

* **前提：** feature 開 MR 回 dev，採 FF only（fast-forward only）。feature 的最後一條 commit 由腳本自動產生，為了不讓它再觸發一次 pipeline，在那條 commit message 加上 `[skip ci]`。預期 merge 回 dev 後，會照常觸發 dev 的 pipeline。

* **問題：** dev 的 pipeline 竟然也一起被跳過了。

```text
feature  A──B  "… [skip ci]"   ← 只想跳過 feature 這次的 pipeline
            ╲
             ╲  MR → dev（FF only：不產生新 commit，B 原封併入）
dev   …───────B  "… [skip ci]"   ✗ dev 的 pipeline 也被跳過
                （skip 標記寫在 commit 裡 → 跟著 commit 一起進了 dev）
```

* **原因：** FF only 合併不會產生新的 merge commit，而是把 feature 那條「帶 `[skip ci]`」的 commit **原封不動**接到 dev 頂端。skip 標記既然寫在 commit message 裡，自然跟著 commit 進了 dev，於是 dev 的 pipeline 也被跳過。

* **解法：**
  * **方法 1（可放寬 FF only 時）：用 squash merge** —— 把 feature 的 commit 壓成一條**全新**的 commit 回 dev，新 commit 的 message 乾淨、不含 skip 標記，pipeline 正常觸發。
  * **方法 2（FF only 為強制要求時）：改用 `git push -o ci.skip`** —— skip 只綁定「這一次 push」，不寫進 commit，所以 FF 併回 dev 後 dev 的 pipeline 照常跑：

```text
feature  A──B  "…"   ← commit message 乾淨；推送時 git push -o ci.skip（skip 只綁這次 push）
            ╲
             ╲  MR → dev（FF only）
dev   …───────B  "…"   ✓ dev 的 pipeline 正常觸發（commit 裡沒有 skip 標記）
```
