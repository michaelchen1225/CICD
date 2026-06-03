# gen-bi Developer Handbook

> For a quick start, see **Cheat Sheet** (`cheat-sheet.md`).

## 1. Branch Structure

```text
feature/* or fix/*
    │  MR (CI: maven_ci + sonar_scan + code review)
    ▼
   dev          Every push automatically builds/deploys :dev
    │  MR dev → pre-prod (release)
    ▼
  pre-prod      Version snapshot + code review gate (no deployment environment)
    │  MR pre-prod → master
    ▼
  master        Production deployment trigger
```

`dev` is the integration branch for day-to-day development. In most cases, you will only create MRs targeting `dev`, which will then be reviewed and merged by a code reviewer.

Do not make changes directly to `pre-prod` or `master` unless there is a special situation requiring a hotfix on those branches. If that happens, communicate with the team first.

> Merges and version management for `dev → pre-prod` and `pre-prod → master` are handled by the release manager. See `release-manager-handbook.md`.

---

## 2. Daily Workflow: feature → dev

```bash
git checkout dev
git pull --ff-only
git checkout -b feature/my-feature
# ... development ...
git push origin feature/my-feature
# Create MR: feature/my-feature → dev
```

* The MR pipeline runs `maven_ci` (build + Testcontainers tests) and `sonar_scan` (code quality and vulnerability analysis).
* Squash merge is allowed for `feature → dev` MRs, since feature branches are disposable and can be deleted after merge.
* After merging into `dev`, the dev push pipeline automatically:

  * Builds the Docker image
  * Pushes `:dev` and `:dev-<sha>-<version>`
  * Deploys to the development environment (`deploy_dev`) using the `:dev` image

The development environment always runs the latest `:dev` image (mutable). No version management is required during normal development.

---

## 3. Versioning

The project version is defined in the `<version>` field of `pom.xml` using the format:

```text
X.Y.Z-SNAPSHOT
```

(no `v` prefix, following Maven conventions).

Version management (MINOR/MAJOR increments and PATCH progression) is handled exclusively by the release manager. Developers should not modify the version themselves.

* PATCH: Automatically incremented by CI after a release.
* MINOR / MAJOR: Determined and updated by the release manager before a release.
* After a release, the release manager will sync the updated version back to `dev`. Seeing changes to the version in `pom.xml` after a `git pull` is expected.

During release, CI removes the `-SNAPSHOT` suffix and adds the `v` prefix to create both the Git tag and the release image tag:

```text
vX.Y.Z
```

---

## 4. Merge Strategy

| Merge             | Strategy       | Notes                                                    |
| ----------------- | -------------- | -------------------------------------------------------- |
| `feature/* → dev` | Squash allowed | Keeps dev history clean; feature branches are disposable |

`dev → pre-prod` and `pre-prod → master` are **FF-only** and are performed by the release manager. See `release-manager-handbook.md`.

---

## 5. Image Tag Reference

| Tag                         | Created When                    | Purpose                                              |
| --------------------------- | ------------------------------- | ---------------------------------------------------- |
| `:dev`                      | Every push to `dev`             | Mutable, used by the development environment         |
| `:dev-<sha8>-<pom-version>` | Every push to `dev`             | Immutable, source image for release retagging        |
| `:vX.Y.Z`                   | During `dev → pre-prod` release | Immutable, permanently retained, used for production |

---

## 6. Things You Should Not Do

* Do not create Git tags manually (CI creates them during release).
* Do not modify image tags manually.
* Do not push directly to `dev`, `pre-prod`, or `master`; always use Merge Requests.
* Do not use Squash when merging `dev → pre-prod`.
* Do not change project versions yourself; version management is handled by the release manager.
