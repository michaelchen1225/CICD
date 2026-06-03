# gen-bi CI/CD Cheat Sheet

> This document is intended as a quick operational reference. For architecture, design rationale, and implementation details, see:
> `developer-handbook.md` / `release-manager-handbook.md` / `devops-note.md`.
>
> Branch workflow:
> `feature/*` → `dev` → `pre-prod` → `master` (**FF-only**).
>
> Versioning is defined solely in the project's `<version>` element within `pom.xml`, using the format `X.Y.Z-SNAPSHOT`.

---

## Developer

### Developing a New Feature

```bash
git checkout dev && git pull --ff-only
git checkout -b feature/my-feature
# ... develop and commit ...
git push -u origin feature/my-feature
# Open MR: feature/my-feature → dev (Squash allowed) → merge
```

After the merge, the `dev` pipeline automatically builds and deploys the `:dev` image. No version changes are required during normal development.

---

## Release Manager

### Release checklist

* [ ] Dev pipeline is green
* [ ] MINOR / MAJOR version updated in `dev` (if required)
* [ ] No open `pre-prod → master` MR
* [ ] MR `dev → pre-prod` created, Squash OFF, merged
* [ ] `version_guard` and `guard_no_squash_preprod` passed
* [ ] `vX.Y.Z` tag and release image created; pre-prod version bumped
* [ ] pre-prod synced back into dev
* [ ] MR `pre-prod → master` merged
* [ ] `deploy_production` executed successfully
* [ ] Sentry contains both the release and the production deployment event

### Bump MINOR or MAJOR Version (only when needed)

> Version format: MAJOR.MINOR.PATCH

Before releasing, update the project's `<version>` in `pom.xml` on the `dev` branch (not the version inside `<parent>`):

```xml
<version>0.2.0-SNAPSHOT</version>   <!-- MINOR bump -->
<version>1.0.0-SNAPSHOT</version>   <!-- MAJOR bump -->
```

PATCH versions do not need to be updated manually. CI automatically increments the PATCH version after each release.

### Release: dev → pre-prod

```text
1. Verify the dev pipeline is green (image already exists in the registry)
2. Verify there is no open pre-prod → master MR
3. Open an MR from dev → pre-prod, set Squash = OFF, review, and merge
4. Wait for the pre-prod pipeline to finish:
   tag vX.Y.Z + image retag + pom version bump
5. Sync pre-prod back to dev (see below)
```

The release version is the `dev` branch version in `pom.xml` with `-SNAPSHOT` removed.

### Sync pre-prod Back to dev (required after every release)

```bash
git checkout dev
git fetch origin
git merge origin/pre-prod   # Pull back the CI-generated version bump commit
git push origin dev
```

If skipped, the next dev → pre-prod MR will encounter a fast-forward conflict; the resolution is the same commands.

### Deploy to Production: pre-prod → master

```text
1. Open an MR from pre-prod → master → code review → merge
   (This MR does not trigger a pipeline)

2. After the master pipeline completes:
   Pipelines → deploy_production → Run job
   (Leave variables empty; IMAGE_TAG defaults to the latest vX.Y.Z)

3. Verify that Sentry contains both:
   - the release
   - the production deployment event
```

### Rollback

```text
In the master pipeline:
   deploy_production → Run job

   Variables:
   IMAGE_TAG = v<previous-version>

The same job also marks Sentry (release + production deploy event) — no second job to run.
```

Validation criteria:

* The older release receives a new deployment event.
* The latest release does not receive an additional deployment event.

---

## Five Golden Rules

1. Always disable Squash when merging `dev → pre-prod`; Squash is allowed only for `feature → dev`.
2. Always sync pre-prod back to `dev` immediately after every `dev → pre-prod` merge.
3. Only one release may be in progress at a time. While a `pre-prod → master` MR is open, no additional changes may be merged into `pre-prod`.
4. PATCH versions are automatically incremented by CI. MINOR and MAJOR versions must be updated manually in `pom.xml`.
5. The pom version must follow the format `X.Y.Z-SNAPSHOT`, and the resulting version (after removing `-SNAPSHOT`) must be greater than the highest existing `v*` tag.

---

## Version Format Reference

| Purpose                       | Format           | Example          |
| ----------------------------- | ---------------- | ---------------- |
| `pom.xml` project `<version>` | `X.Y.Z-SNAPSHOT` | `0.1.5-SNAPSHOT` |
| Git tag / release image       | `vX.Y.Z`         | `v0.1.5`         |
| Rollback `IMAGE_TAG` value    | `vX.Y.Z`         | `v0.1.4`         |

---

## Troubleshooting Reference

| Issue                                                                           | Reference                |
| ------------------------------------------------------------------------------- | ------------------------ |
| `version_guard` reports existing version, invalid format, or version regression | release-manager-handbook |
| Fast-forward conflict in MR                                                     | release-manager-handbook |
| No pipeline triggered after push to pre-prod                                    | release-manager-handbook |
| `git push` connectivity issues (could not connect)                              | devops-note              |
| No pipeline for production MR / unable to merge                                 | devops-note              |
| Rollback not recorded in Sentry                                                 | release-manager-handbook |
