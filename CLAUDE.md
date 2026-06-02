# CLAUDE.md

## claude role

You are a DevOps engineer who is working on a project with a GitLab CI/CD pipeline.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a personal study-notes repository (繁體中文) about CI/CD, with the bulk of the content focused on **GitLab CI/CD**. There is no application code to build or test — the artifacts here are Markdown chapters, screenshot images, and one small `.gitlab-ci.yml` sandbox used to demonstrate concepts in the notes. Treat this as a documentation project: edits are almost always to `.md` files, and the audience is a future reader following a chapter sequence.

## Top-level layout (the "big picture")

The repository has two layers of tables of contents and a sandbox:

- **Root `README.md`** — the master TOC. It is the single entry point and links into every other chapter, including chapters that live deep inside `GitLab CICD/Gitlab-cicd-note-github/`. Spaces in the folder name must be URL-encoded (`GitLab%20CICD`) when writing links from the root README.
- **`01.md`** — the conceptual introduction (what CI/CD is, CI vs CD, common tooling). It sits at the root because it is the prerequisite reading before any GitLab-specific chapter.
- **`GitLab CICD/`** — the GitLab-specific section.
  - `README.md` here is a near-empty stub. The *real* TOC for users is the root `README.md`; do not duplicate chapter listings into this stub.
  - `Gitlab-cicd-note-github/` — the numbered chapter sequence (`01.md`, `02.md`, … `08-cache.md`), with the screenshots they reference collected in the `images/` subfolder (referenced as `images/image-N.png`). Numbered chapters are ordered by their prefix and the order is meaningful (later chapters assume earlier ones). This folder also keeps its own `README.md` index table — update it too when chapters change here.
  - `cases/` — standalone real-world case studies / incident postmortems (e.g. `connector-dockerfile-instantclient-upgrade-incident.md`), deliberately kept out of the numbered chapter sequence. These **are** documentation and belong in the root TOC (under the 案例 / 事故 section).
  - `cicd-tips/` — tips / reference notes (e.g. `pipeline-design-principles.md`) accompanied by a real-world `.gitlab-ci.yml` used as the worked example. The `.md` notes here **are** documentation and belong in the root TOC; the `.gitlab-ci.yml` is only the example artifact (not a TOC entry, and — unlike `cicd-test/` — it is part of this repo, with no nested `.git`).
  - `cicd-test/` — a **nested independent git repository** containing a sample `.gitlab-ci.yml`. It is a sandbox for testing pipeline syntax, not part of the notes themselves. Do not treat its contents as documentation, and do not commit it from the outer repo (it has its own `.git`).
- **`SonarQube/`** — a parallel chapter series on SonarQube integration (`01-sonarqube.md` … `07-token-management.md`, plus `maintainability-considerations.md`). It has its own [`SonarQube/CLAUDE.md`](SonarQube/CLAUDE.md) carrying writing-style rules specific to that series — read it before editing anything under `SonarQube/`. All its chapters are listed in the root `README.md` TOC.

## Authoring conventions worth preserving

- **Language:** all prose is Traditional Chinese (zh-TW). New content should match.
- **Chapter numbering:** new chapters in `Gitlab-cicd-note-github/` follow the `NN-short-slug.md` pattern (e.g. `04-regist-runner.md`). Pick the next free number; do not renumber existing files since the root README links to them by name.
- **Per-chapter TOCs:** longer chapters (e.g. `02.md`, `04-regist-runner.md`, `05-use-case.md`) include their own in-page TOC using GitHub-style anchor links (`[標題](#中文-anchor)`). When editing those chapters, keep the in-page TOC in sync with the headings.
- **Images:** screenshots for the `Gitlab-cicd-note-github/` chapters live in the `Gitlab-cicd-note-github/images/` subfolder and are referenced with the `images/` prefix — `![alt](images/image-3.png)` in Markdown, or `<img src="images/image-14.png" width="...">` when a chapter needs a sized image. Use sequential names (`image.png`, `image-1.png`, …) and add new screenshots with the next free number rather than renaming. (Other folders such as `SonarQube/` may still keep their images beside the `.md`; match whatever the folder you are editing already does.)
- **Cross-chapter links from the root README** must URL-encode the space in `GitLab CICD` as `GitLab%20CICD/...`. In-folder relative links (within `Gitlab-cicd-note-github/`) do not need encoding.

## Mandatory rule — keep the root README TOC current

**Whenever a note `.md` file (chapter, 補充, tip, case study — anything a reader follows) is added, removed, renamed, or reordered *anywhere* in this repository, you must update the `## 目錄` section of the root [`README.md`](README.md) in the same change so it reflects the new state.** Treat "I created / deleted / renamed a note file" and "I touched the root TOC" as one inseparable action — never finish the former without the latter.

This applies to **any** documentation `.md` anywhere under the repo, including:

- Root-level concept chapters (e.g. another one alongside `01.md`).
- Numbered chapters inside `GitLab CICD/Gitlab-cicd-note-github/` (e.g. `08-cache.md`).
- Tip / reference notes inside `GitLab CICD/cicd-tips/`.
- Case-study / incident notes inside `GitLab CICD/cases/` (e.g. `connector-dockerfile-instantclient-upgrade-incident.md`).
- Any chapter inside `SonarQube/`.
- Any new note folder created later under the repo.
- Any rename, deletion, or reorder of the above (keep every note under the `###` section that matches its folder — see the TOC structure below).

Do **not** add TOC entries for (these are not reader-facing notes): per-folder `README.md` stubs, any `CLAUDE.md`, `image-*.png` files, the `.gitlab-ci.yml` example/sandbox files (`cicd-tips/.gitlab-ci.yml` and everything under `cicd-test/`), anything under `.claude/`, or files inside the nested `cicd-test/.git`.

The `## 目錄` is **categorised into `###` sections that mirror the repo's folders** — do NOT flatten it back into one undifferentiated list. The current sections, in order, are:

1. **入門概念** — the root concept chapter (`01.md`).
2. **GitLab CI/CD 筆記** — everything in `Gitlab-cicd-note-github/`: 第一~五章 first (numeric order, the meaningful reading sequence), then the `補充：` entries (skip-CI, cache, sonarqube draft).
3. **CI/CD 設計技巧（cicd-tips）** — notes in `cicd-tips/`.
4. **案例 / 事故（cases）** — real-world case studies / incident postmortems in `cases/`.
5. **SonarQube** — the `SonarQube/` series, `(1)`…`(7)` then `maintainability-considerations.md`.

When updating the TOC:

1. Open the root `README.md` and locate the `## 目錄` block.
2. Put the new entry under the `###` section matching its source folder, in reading order (核心章節 by number, then 補充/案例). If a brand-new note folder is created, add a new `###` section for it in the natural reading position rather than dropping the entry into an unrelated section.
3. Within a section use the bullet style already in the file (`* [標題](path)` with a blank line between entries), link text in Traditional Chinese. Don't repeat the section name in every bullet (under **SonarQube** write `(1) 介紹與建置`, not `補充：SonarQube (1) ...`).
4. URL-encode spaces in paths that traverse `GitLab CICD/` (`GitLab%20CICD/...`).
5. Verify each link resolves to an existing file before finishing.

If a chapter being added is large enough to warrant its own in-page TOC, also add that TOC inside the new chapter (matching the convention in `02.md` / `04-regist-runner.md`).

## Things that are *not* applicable here

There is no build system, package manager, lint config, or test suite in this repository — do not invent commands for them. The only executable artifact is `GitLab CICD/cicd-test/.gitlab-ci.yml`, and it only runs when pushed to a real GitLab instance with a registered runner; it cannot be exercised locally from this repo.
