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