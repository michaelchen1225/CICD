# gen-bi DevOps Notes

> Audience: DevOps engineers responsible for maintaining this pipeline.
> This document records GitLab configuration, infrastructure requirements, design decisions, and troubleshooting guidance.
> The authoritative source is the current `gen-bi/.gitlab-ci.yml`; this document explains the reasoning behind the implementation and how to operate it.

---

## 1. GitLab Prerequisites

### Branches / Merge Configuration

* `dev`, `pre-prod`, and `master` must be configured as **Protected Branches** (only Maintainers and the CI service account may push).
* Merge method: **Fast-forward merge (FF-only)**.
* Squash merge: enabled but disabled by default.

  * `feature → dev`: squash allowed.
  * `dev → pre-prod`: squash prohibited and enforced by `guard_no_squash_preprod`.
* GitLab squash settings are configured at the project level and **cannot be configured per branch**. Therefore, the distinction between "feature branches may squash" and "pre-prod merges may not" is enforced through CI validation.

### Tags

* Configure a **Protected Tag** rule matching `v*` (Developers and Maintainers may create tags).
* The CI service account (Developer role) must be able to create `v*` tags.

### CI/CD Variables

| Variable                                      | Purpose                                      | Configuration                                                             |
| --------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------- |
| `GIT_PUSH_TOKEN`                              | CI tag creation and pom version bump commits | Service account PAT with `write_repository` scope, **Protected + Masked** |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Bedrock access                               | Scope must include target branches                                        |
| `SENTRY_DSN` / `SENTRY_AUTH_TOKEN`            | Sentry release and deployment tracking       | Used by `sentry-cli`                                                      |

> Protected variables are only available on protected branches and tags. Therefore, all three long-lived branches (`dev`, `pre-prod`, `master`) **must** be protected. Otherwise, `prepare_next_release` will be unable to access `GIT_PUSH_TOKEN`, causing version bump pushes to fail.

### Merge Checks

Verify that **"Skipped pipelines are considered successful"** is enabled.

The `pre-prod → master` MR does **not** generate a pipeline (promote-only workflow). If this setting is disabled while "Pipelines must succeed" is enabled, production MRs will become blocked.

### Runners

* `lndata-1-docker` / `lndata-2-docker`

  * Docker executor
  * Tagged with `docker-build`
  * Used for build, retag, Sentry, and related jobs

* `lndata-1`

  * Shell executor
  * Runs `deploy_dev`
  * Development/test environment

* `lndata-oci-prod`

  * Shell executor
  * Runs `deploy_production`
  * OCI production environment

Deployment hosts must provide:

* `internalnet` Docker bridge network
* Port `8090`
* `sudo docker` access
* One development host (`lndata-1`)
* One production host (OCI)

---

## 2. Versioning and Image Tag Contract

| Purpose                 | Format                                                           |
| ----------------------- | ---------------------------------------------------------------- |
| `pom.xml`               | `X.Y.Z-SNAPSHOT`                                                 |
| Git tag / release image | `vX.Y.Z`                                                         |
| Mutable dev image       | `:dev`                                                           |
| Immutable dev image     | `:dev-<sha8>-<pom-version>` (e.g. `dev-abc12345-0.1.5-SNAPSHOT`) |

### Producer / Consumer Contract (Must Match Bit-for-Bit)

#### Producer: `docker_build_dev`

* SHA: `$CI_COMMIT_SHORT_SHA` (first 8 characters)
* Version: raw `pom.xml` value (including `-SNAPSHOT`)

#### Consumer: `image_retag_preprod`

* SHA: `git rev-parse origin/dev | cut -c1-8`
* Version: read directly from `pom.xml`

The explicit `cut -c1-8` is intentional to guarantee alignment with `CI_COMMIT_SHORT_SHA`.

Do **not** replace it with:

```bash
git rev-parse --short
```

because Git may automatically choose a different abbreviation length, resulting in mismatches (e.g. 7 vs 8 characters) and causing image lookups to fail.

### DEV_BRANCH Derivation

`DEV_BRANCH` is derived from `CI_COMMIT_BRANCH` using:

```bash
sed 's|/pre-prod$|/dev|; s|^pre-prod$|dev|'
```

Examples:

```text
pre-prod      → dev
team/pre-prod → team/dev
```

This logic is shared by both:

* `image_retag_preprod`
* `guard_no_squash_preprod`

---

## 3. Critical Design Decisions (Do Not Change Lightly)

### 3.1 Build Once, Promote Later

Images are built exactly once on `dev` pushes.

`dev → pre-prod` performs only a retag operation:

```text
:dev-<sha>-<version>
          ↓
      :vX.Y.Z
```

No rebuild occurs.

Image digests must remain identical. This is a core validation point of the release process.

### 3.2 Prevent Loops with `git push -o ci.skip` (Not `[skip ci]`)

`prepare_next_release` performs the version bump commit using:

```bash
git push -o ci.skip
```

This skips only the pipeline associated with that specific push.

Why not use `[skip ci]` in the commit message?

Because the commit itself would later fast-forward into `master`, causing the master push pipeline to be skipped as well. In that scenario:

* `deploy_production` would never appear.
* Production deployment would be blocked.

Using `-o ci.skip` keeps the commit clean while preserving normal production deployment behavior.

Resilience:

Even if `-o ci.skip` fails to take effect, the resulting pre-prod pipeline is idempotent and will not create an infinite loop.

### 3.3 `master` Is Promote-Only

Neither `.validate_rules` nor `version_guard` applies to `master`.

Therefore:

* `pre-prod → master` MRs do not rerun `maven_ci`.
* `master` pushes do not rerun `sonar_scan`.

Production quality gates rely on:

* Human review
* Previously validated release artifacts

Deployment is triggered through the manual `deploy_production` job on master push.

### 3.4 Split Release Logic into Two Jobs

Release processing is intentionally divided into:

1. `image_retag_preprod`

   * Docker 24 + DinD
   * Image retagging and Git tag creation

2. `prepare_next_release`

   * Alpine
   * Pure Git version bump

The version bump push is kept outside the DinD environment to avoid Docker networking instability and to allow independent retries.

### 3.5 Network Resilience (`.ci_helpers`)

#### IPv4 `/etc/hosts` Pinning (Primary Fix)

The job container occasionally encounters unroutable IPv6 paths when accessing GitLab.

Symptoms:

```text
git fetch
git push
```

fail after approximately two seconds with:

```text
Could not connect to server
```

Docker registry traffic through DinD is unaffected.

`.ci_helpers` resolves GitLab's IPv4 address via `dig` (`bind-tools`) and injects it into `/etc/hosts`, forcing Git and curl to use IPv4.

#### `git_net()`

```bash
git -c http.connectTimeout=30000
```

This is not intended to solve the IPv6 routing problem. It only acts as a safeguard against genuinely slow connections.

#### `retry <max> <delay> <cmd>`

All Git and Docker network operations are wrapped with retries.

Retrying is safe because re-pushing the same tag or commit is effectively a no-op.

#### Known Limitation

The runner helper container performs repository cloning before `before_script` executes.

Because IPv4 pinning is applied later, it does not affect that initial clone operation.

If cloning itself is impacted by IPv6 routing issues, the fix must be applied at the runner level (e.g. disabling IPv6 or adjusting DNS configuration).

---

## 4. `$CI_OPEN_MERGE_REQUESTS` Suppression Trap

Workflow rule:

```yaml
$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
  when: never
```

prevents duplicate branch and MR pipelines.

Side effect:

If a branch already has an open MR, push pipelines on that branch are suppressed.

### Consequence

When `pre-prod` has an open MR (for example, `pre-prod → master`), newly merged commits into `pre-prod` will not trigger the release pipeline.

### Mitigation

The release process assumes only one active release at a time (see `release-manager-handbook`, Section 2).

If release pipelines must continue running while a `pre-prod → master` MR remains open, this workflow rule will need to be adjusted carefully to avoid duplicate pipelines on other branches.

---

## 5. Troubleshooting

| Symptom                                                   | Cause / Resolution                                                                                                                                                                               |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `git ... could not connect` fails after ~2 seconds        | GitLab IPv6 routing issue. IPv4 pinning in `.ci_helpers` should resolve it. If failures persist, check connectivity with `curl -v https://gitlab.com` and `dig gitlab.com` from the runner host. |
| No pipeline triggered after a pre-prod push               | `pre-prod` has an open MR (see Section 4). Close the MR and retry.                                                                                                                               |
| `version_guard` rejects a valid version                   | An abnormally high historical tag exists (often from testing). Remove the incorrect `v*` tag.                                                                                                    |
| `image_retag_preprod` cannot find `:dev-<sha>-<version>`  | The dev build may still be running (race condition). The job retries and eventually falls back to `:dev`. Alternatively, the source commit may have skipped image generation.                    |
| Production MR cannot be merged because no pipeline exists | Enable "Skipped pipelines are considered successful" (see Section 1).                                                                                                                            |
| Rollback is not recorded in Sentry                        | Deployment tracking is integrated into `deploy_production`. Re-run the deployment job and verify that the production runner can pull `getsentry/sentry-cli` and access the `SENTRY_*` variables. |
| Version bump commit triggered a second pre-prod pipeline  | `-o ci.skip` was not honored. Verify GitLab accepted the push option or implement a workflow-rule-based fallback.                                                                                |

---

## 6. Job Migration Reference

### Current Jobs

```text
maven_ci
sonar_scan
version_guard
guard_no_squash_preprod
maven_build_dev
sentry_release_dev
sentry_release_prod
docker_build_dev
image_retag_preprod
prepare_next_release
deploy_dev
deploy_production
sentry_mark_dev
```

`deploy_production` includes built-in Sentry production deployment tracking.

### Jobs That Should No Longer Exist

```text
maven_release
image_retag_release
version_check
release/*
tag-triggered release jobs
gen-bi/VERSION
```

If any of the above still appear in pipelines, the migration is incomplete.
