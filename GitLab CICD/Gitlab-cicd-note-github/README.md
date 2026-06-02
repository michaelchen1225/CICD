# GitLab CI/CD 筆記

個人在實務專案中整理的 GitLab CI/CD 操作筆記，內容圍繞 `.gitlab-ci.yml` 撰寫、自架 Runner、優化技巧與整合工具。

## 目錄

| 編號 | 主題 | 說明 |
|---|---|---|
| [01](01.md) | GitLab CI/CD 入門 | 前置作業、`.gitlab-ci.yml` 基本結構、stage / job 概念 |
| [02](02.md) | Stage 失敗行為 | 前面 stage 失敗時，後面 stage 的執行邏輯 |
| [03](03.md) | 優化篇 | Pipeline 優化技巧 |
| [04](04-regist-runner.md) | 註冊自架 Runner | 在自己的 server 上註冊 GitLab Runner |
| [05](05-use-case.md) | 使用情境 | 實際 pipeline 設計案例 |
| [06](06-how-to-skip.md) | 跳過 CI/CD | commit message 控制是否觸發 pipeline |
| [07](07-sonarqube.md) | SonarQube 整合 | 把 SonarQube 接入 GitLab CI pipeline |
| [08](08-cache.md) | Cache 設定與管理 | `.gitlab-ci.yml` cache 寫法、改 runner cache 路徑、查詢與維護 |

> 截圖統一收在 [`images/`](images/) 子資料夾。

## 相關

* CI/CD 實戰事故與案例總結已獨立至 [`../cases/`](../cases/) 資料夾。
