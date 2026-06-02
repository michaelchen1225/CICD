# CI/CD NOTE

> 以 GitLab CI/CD 為主

## 先備知識

* Git (至少了解到分支合併的觀念)

## 目錄

> 依資料夾分四類:**入門概念** → **GitLab CI/CD 筆記** → **CI/CD 設計技巧** → **SonarQube**。

### 入門概念

* [什麼是 CI/CD](./01.md)

### GitLab CI/CD 筆記

核心章節依序閱讀(後面章節假設你已讀過前面);補充與案例可視需要查閱。

* [第一章：GitLab CI/CD](./GitLab%20CICD/Gitlab-cicd-note-github/01.md)

* [第二章：語法整理 (持續更新)](./GitLab%20CICD/Gitlab-cicd-note-github/02.md)

* [第三章：優化 CI/CD 設定檔](./GitLab%20CICD/Gitlab-cicd-note-github/03.md)

* [第四章：Gitlab Runner --- 註冊 & 除錯](./GitLab%20CICD/Gitlab-cicd-note-github/04-regist-runner.md)

* [第五章：使用場景](./GitLab%20CICD/Gitlab-cicd-note-github/05-use-case.md)

* [補充：如何 commit 時跳過 CI/CD ?](./GitLab%20CICD/Gitlab-cicd-note-github/06-how-to-skip.md)

* [補充：Cache（快取）設定與管理](./GitLab%20CICD/Gitlab-cicd-note-github/08-cache.md)

* [補充：SonarQube 整合 GitLab CI/CD（草稿）](./GitLab%20CICD/Gitlab-cicd-note-github/07-sonarqube.md)

* [案例：Oracle Instant Client 升級事故總結](./GitLab%20CICD/Gitlab-cicd-note-github/connector-dockerfile-instantclient-upgrade-incident.md)

### CI/CD 設計技巧（cicd-tips）

以實戰 `.gitlab-ci.yml` 為範例,拆解可轉移的 pipeline 設計原理。

* [Pipeline 設計原理（實戰範例拆解）](./GitLab%20CICD/cicd-tips/pipeline-design-principles.md)

### SonarQube

* [(1) 介紹與建置](./SonarQube/01-sonarqube.md)

* [(2) 整合 GitLab CI/CD (Java)](./SonarQube/02-gitlab-ci-integration.md)

* [(3) Quality Gate 設計（PoC / Product）](./SonarQube/03-quality-gate-design.md)

* [(4) Quality Gate 與掃描結果判讀](./SonarQube/04-quality-gate-and-results.md)

* [(5) Quality Profile](./SonarQube/05-quality-profile.md)

* [(6) MR / PR 整合（解鎖 Community Build 的 branch / PR decoration）](./SonarQube/06-mr-pr-integration.md)

* [(7) Token 管理](./SonarQube/07-token-management.md)

* [Maintainability 的長期考量](./SonarQube/maintainability-considerations.md)

## 待處理
