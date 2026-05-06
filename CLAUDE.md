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
  - `Gitlab-cicd-note-github/` — the numbered chapter sequence (`01.md` … `06-how-to-skip.md`) plus all `image-*.png` screenshots referenced from those chapters. Chapters are ordered by the numeric prefix and the order is meaningful (later chapters assume earlier ones).
  - `cicd-test/` — a **nested independent git repository** containing a sample `.gitlab-ci.yml`. It is a sandbox for testing pipeline syntax, not part of the notes themselves. Do not treat its contents as documentation, and do not commit it from the outer repo (it has its own `.git`).

## Authoring conventions worth preserving

- **Language:** all prose is Traditional Chinese (zh-TW). New content should match.
- **Chapter numbering:** new chapters in `Gitlab-cicd-note-github/` follow the `NN-short-slug.md` pattern (e.g. `04-regist-runner.md`). Pick the next free number; do not renumber existing files since the root README links to them by name.
- **Per-chapter TOCs:** longer chapters (e.g. `02.md`, `04-regist-runner.md`, `05-use-case.md`) include their own in-page TOC using GitHub-style anchor links (`[標題](#中文-anchor)`). When editing those chapters, keep the in-page TOC in sync with the headings.
- **Images:** stored next to the markdown that references them, using sequential names (`image.png`, `image-1.png`, …). Add new screenshots with the next free number rather than renaming.
- **Cross-chapter links from the root README** must URL-encode the space in `GitLab CICD` as `GitLab%20CICD/...`. In-folder relative links (within `Gitlab-cicd-note-github/`) do not need encoding.

## Mandatory rule — keep the root README TOC current

**Whenever a chapter file is added, removed, or renamed under this repository, you must update the `## 目錄` section of the root [`README.md`](README.md) in the same change so it reflects the new state.**

Apply this rule to:

- Any new `*.md` chapter at the repository root (e.g. another conceptual chapter alongside `01.md`).
- Any new `NN-*.md` chapter inside `GitLab CICD/Gitlab-cicd-note-github/`.
- Any rename, deletion, or reorder of the above.

Do **not** add entries for: per-folder `README.md` stubs, `image-*.png` files, the `cicd-test/` sandbox, or files inside the nested `cicd-test/.git`.

When updating the TOC:

1. Open the root `README.md` and locate the `## 目錄` block.
2. Insert/remove/rename the bullet so the displayed order matches the chapter order (root concept chapters first, then GitLab chapters in numeric order, then any 補充 / appendix entries last — mirroring the existing pattern).
3. Use the same bullet style already in the file (`* [標題](path)` with a blank line between entries) and write the link text in Traditional Chinese.
4. URL-encode spaces in paths that traverse `GitLab CICD/` (`GitLab%20CICD/...`).
5. Verify each link resolves to an existing file before finishing.

If a chapter being added is large enough to warrant its own in-page TOC, also add that TOC inside the new chapter (matching the convention in `02.md` / `04-regist-runner.md`).

## Things that are *not* applicable here

There is no build system, package manager, lint config, or test suite in this repository — do not invent commands for them. The only executable artifact is `GitLab CICD/cicd-test/.gitlab-ci.yml`, and it only runs when pushed to a real GitLab instance with a registered runner; it cannot be exercised locally from this repo.
