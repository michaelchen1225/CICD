# CI/CD Pipeline Documentation - gen-bi

> This document is written based on the current `.gitlab-ci.yml`. The YAML file is the source of truth; if there is any inconsistency, update this document accordingly.

## Overview

gen-bi uses a **pom.xml SNAPSHOT-driven, build-once + retag** pipeline running on a four-branch workflow.

Docker images are built **only once** when code is pushed to `dev`. From that point until production deployment, the same image is **promoted via retagging** and is **never rebuilt**.

```text
feature/* ──MR──▶ dev ──MR──▶ pre-prod ──MR──▶ master
                   │            │                │
              build :dev    retag :dev-…      Manual deploy
              deploy to dev → :vX.Y.Z         to production
                            create git tag
                            bump pom PATCH
```

* **dev** — Integration branch. Every push builds a Docker image and deploys it to the development environment.
* **pre-prod** — Release branch. Promotes a dev image into an immutable `vX.Y.Z` release (retag only, no rebuild), creates a Git tag, and bumps `pom.xml` to the next SNAPSHOT version.
* **master** — Production promotion gate. No rebuilds or validations occur here. A manual job deploys the already-built release image to production.

All merges are performed using **fast-forward only (FF-only)**.

---

## Versioning & Tags

> Versioning follows Semantic Versioning (SemVer): `MAJOR.MINOR.PATCH`.

| Purpose                               | Format                      | Example                        |
| ------------------------------------- | --------------------------- | ------------------------------ |
| `pom.xml <version>` (project version) | `X.Y.Z-SNAPSHOT`            | `0.1.5-SNAPSHOT`               |
| Git tag / release image               | `vX.Y.Z`                    | `v0.1.5`                       |
| Mutable dev image                     | `:dev`                      | `:dev`                         |
| Immutable dev image                   | `:dev-<sha8>-<pom-version>` | `:dev-abc12345-0.1.5-SNAPSHOT` |
| `APP_VERSION` / Sentry release (prod) | `X.Y.Z`                     | `0.1.5`                        |
| `APP_VERSION` / Sentry release (dev)  | `<short-sha>`               | `abc12345`                     |

* `sha8` refers to `$CI_COMMIT_SHORT_SHA` (first 8 characters).

* Every push to `dev` produces a new image with two tags:

  1. `:dev` (mutable) — represents the current HEAD of the `dev` branch. Updated and redeployed on every dev push.
  2. `:dev-<sha8>-<pom-version>` (immutable) — represents a specific dev build and includes both the commit SHA and pom version for traceability.

* PATCH versions are automatically bumped by CI after each release from `pre-prod`.

* MINOR and MAJOR versions must be updated manually by developers on the `dev` branch.

> ⚠️ `pom.xml` contains **two** `<version>` elements: one for `spring-boot-starter-parent` and one for the project itself. CI always reads the **project version**, i.e., the `<version>` located after `<artifactId>gen-bi</artifactId>`.

---

## Image Tagging Strategy

```text
Build once on dev push:
  :dev                              (mutable, redeployed on every dev push)
  :dev-<sha8>-<pom-version>         (immutable, source of future promotions)

Promote on pre-prod merge (no rebuild):
  :dev-<sha8>-<pom-version>  ──▶  :vX.Y.Z   (same digest)
```

The digest of `:dev-<sha8>-<version>` and `:vX.Y.Z` **must be identical**.

---

## Workflow Rules

| Source                                     | Pipeline Triggered?                          |
| ------------------------------------------ | -------------------------------------------- |
| MR targeting `dev` / `pre-prod` / `master` | Yes                                          |
| Branch push with an open MR                | No (suppressed to avoid duplicate pipelines) |
| Push to `dev` / `pre-prod` / `master`      | Yes                                          |

> MRs from `pre-prod → master` do **not** run any jobs. The deployment pipeline is triggered only after the MR is merged into `master`.

---

## Runners & Environments

* lndata-1: `192.168.10.201` (development server)
* lndata-2: `192.168.10.202`
* lndata-oci-prod: `152.70.82.144` (public IP)

| Tag               | Executor                                       | Used By                             |
| ----------------- | ---------------------------------------------- | ----------------------------------- |
| `docker-build`    | Docker (`lndata-1-docker` / `lndata-2-docker`) | Build, test, retag, and Sentry jobs |
| `lndata-1`        | Shell (development server)                     | `deploy_dev`                        |
| `lndata-oci-prod` | Shell (production server on OCI)               | `deploy_production`                 |

---

## Mixins

* **`.validate_rules`** — MR targeting `dev` or `pre-prod`, or push to `dev` with changes in `pom.xml` or `src`. `master` is intentionally excluded (promotion-only).
* **`.dev_push_rules`** — Push to `dev` with changes in `pom.xml`, `src`, `Dockerfile`, or `docker-compose.yml`.
* **`.preprod_push_rules`** — Push to `pre-prod`.
* **`.master_push_rules`** — Push to `master`.
* **`.maven_cache`** — Maven cache for `.m2/repository` (key: `maven-cache-gen-bi`).
* **`.ci_helpers`** — Shared `before_script` for pre-prod Git operations. Pins `gitlab.com` to IPv4 in `/etc/hosts` (see [Network Resilience](#network-resilience)), and defines `retry`, `git_net`, `pom_version`, `REMOTE`, and Git identity.
* **`.deploy_base`** — Shared config for `deploy_dev` + `deploy_production`: the `AWS_REGION` / `BEDROCK_MODEL_ID` / `BEDROCK_EMBEDDING_MODEL` variables and the port-8090 guard `before_script`. Both jobs `extends` it and leave their own `before_script` absent (extends replaces `before_script` wholesale, so defining one per job would shadow the guard). Job-specific bits — `IMAGE_TAG` source, `SENTRY_ENVIRONMENT`, and the prod-only AWS precheck / Sentry marking — stay in each job.

---

## Stages

```text
validate → maven_build → sentry_release → docker_build
         → image_retag → prepare_release → deploy → sentry_mark
```

`sentry_release` runs before `docker_build` so that the dev image can embed a known `SENTRY_RELEASE`.

All non-deployment jobs are configured with `interruptible: true`.

Deployment and side-effect jobs (e.g., retagging) use `interruptible: false`.

> `interruptible` means that when a newer commit triggers another pipeline, pending jobs from older pipelines are automatically cancelled. This is especially useful on frequently updated branches such as `dev`, reducing unnecessary CI consumption.

---

## Jobs

### `validate`

#### `maven_ci`

* **Trigger:** `.validate_rules` (MR into dev/pre-prod or push to dev)
* **Image:** `maven:3.9-eclipse-temurin-21` + `docker:24-dind`
* Uses Testcontainers/MongoDB via `TESTCONTAINERS_HOST_OVERRIDE: docker`
* Executes:

```bash
mvn clean install
```

#### `sonar_scan`

* **Trigger:** `.validate_rules`
* `allow_failure: true` (POC phase)
* Runs SonarQube analysis
* Uses pull request parameters for MRs, otherwise uses `-Dsonar.branch.name`

#### `version_guard`

* **Trigger:** MR targeting `pre-prod`
* **Image:** `alpine:3`

Reads the project version from `pom.xml` and validates:

1. Format must be `X.Y.Z-SNAPSHOT`
2. Tag `vX.Y.Z` must not already exist (detects forgotten sync-back to dev after a release)
3. No version regression — version must be greater than or equal to the highest existing `v*` tag

Detailed debugging instructions are printed on failure.

#### `guard_no_squash_preprod`

* **Trigger:** Push to `pre-prod`
* Uses `.ci_helpers` + `.preprod_push_rules`

Prevents squash-merging `dev` into `pre-prod`.

Because this job runs during `validate`, any squash merge is blocked before reaching the retag/tag stages.

Failure output includes reset and re-merge instructions.

---

### `maven_build`

#### `maven_build_dev`

* **Trigger:** Push to `dev`

Runs:

```bash
mvn clean package -DskipTests
```

Artifacts:

```text
target/gen-bi-*.jar
```

Retention: 1 hour

`needs: maven_ci` (optional, skipped when only CI files change)

---

### `sentry_release`

#### `sentry_release_dev`

* **Trigger:** Push to `dev`

Creates, sets commits for, and finalizes:

```text
gen-bi@<short-sha>
```

#### `sentry_release_prod`

* **Trigger:** Push to `master`

Reads the latest `vX.Y.Z` tag and creates/finalizes:

```text
gen-bi@X.Y.Z
```

---

### `docker_build`

#### `docker_build_dev`

* **Trigger:** Push to `dev`
* **Image:** `docker:24` + `docker:24-dind`

Builds one image with two tags:

```text
:dev
:dev-<sha8>-<pom-version>
```

Needs:

* `maven_build_dev`
* `sentry_release_dev`

---

### `image_retag` (Pre-prod Only)

#### `image_retag_preprod`

* **Trigger:** Push to `pre-prod`
* `interruptible: false`
* `resource_group: preprod_release` — shared with `prepare_next_release`, so the whole release serializes across concurrent pre-prod pipelines (otherwise two pipelines both pass the "tag absent" check and race the `:vX.Y.Z` docker push and the git-tag push).
* **Image:** `docker:24` + `docker:24-dind`

Workflow:

1. Read version from `pom.xml`
2. Generate `TAG=vX.Y.Z`

Idempotent behavior:

* If Git tag already exists, retagging and tag creation are skipped entirely.

Otherwise:

1. Derive the source SHA from the pre-prod tip — `$CI_COMMIT_SHORT_SHA`. Under the FF-only model the pre-prod tip is the exact dev commit that was merged, so this is the same short SHA `docker_build_dev` used. (No `origin/dev` lookup: that would race a concurrent dev push and could promote a newer, un-reviewed image.)

2. Pull the immutable image (a short retry absorbs registry replication lag):

```text
:dev-<sha8>-<version>
```

3. If it still cannot be pulled, **fail the job** — never fall back to the mutable `:dev`. Promoting `:dev` would break build-once / promote-a-verified-artifact (it may already have been overwritten by a later dev push). A missing immutable tag means the release commit never built an image.

4. Retag and push:

```text
:vX.Y.Z
```

5. Create and push Git tag:

```text
vX.Y.Z
```

---

### `prepare_release` (Pre-prod Only)

#### `prepare_next_release`

* **Trigger:** Push to `pre-prod`
* `interruptible: false`
* `resource_group: preprod_release` — same group as `image_retag_preprod`; serializes the pom bump/push so two concurrent pre-prod pipelines can't both bump from the same `RAW` and race the branch push.
* **Image:** `alpine:3`

Needs:

```text
image_retag_preprod
```

Workflow:

1. Calculate:

```text
NEXT = X.Y.(Z+1)-SNAPSHOT
```

2. Update project `<version>` in `pom.xml`
3. Commit with a clean commit message
4. Push using:

```bash
git push -o ci.skip
```

This skips only the current pipeline and does **not** insert `[skip ci]` into the commit message.

As a result, the commit still triggers the deployment pipeline when later fast-forwarded into `master`.

---

### `deploy`

#### `deploy_dev`

* **Trigger:** Push to `dev`
* **Runner:** `lndata-1`
* `interruptible: false`
* `resource_group: dev_deploy`
* `extends: .deploy_base` — inherits the region/Bedrock variables and the port-8090 guard `before_script`.

`before_script` validates port `8090` (inherited from `.deploy_base`; see [Port Validation](#port-validation-deploy-before_script)).

Deployment:

1. Pull `:dev`
2. Generate environment file
3. Run:

```bash
docker compose up -d --force-recreate
```

Environment:

```text
APP_VERSION      = short-sha
SENTRY_RELEASE   = short-sha
SENTRY_ENVIRONMENT = dev
```

Needs:

```text
docker_build_dev
```

#### `deploy_production`

* **Trigger:** Push to `master`
* **when:** `manual`
* **Runner:** `lndata-oci-prod`
* `extends: .deploy_base` — inherits the region/Bedrock variables and the port-8090 guard `before_script` (shared with `deploy_dev`).

Configuration:

```text
interruptible: false
resource_group: production_deploy
```

Needs:

```text
sentry_release_prod
```

Behavior:

* Defaults to the latest `vX.Y.Z` tag
* Can override `IMAGE_TAG` during manual execution

Deployment:

1. Validate port 8090
2. Pull `:$IMAGE_TAG`
3. Deploy using Docker Compose
4. Set:

```text
APP_VERSION = X.Y.Z
```

Writes the deployed image tag into:

```text
deployed-image-tag.txt
```

A plain-text artifact is used instead of dotenv because manual re-runs can cause dotenv upload conflicts. It is kept as an audit record of the actually-deployed tag.

After deploying, this same job marks the production release and deploy event in Sentry (there is no separate `sentry_mark_prod` job):

1. `SENTRY_RELEASE=gen-bi@<version>` (the `v`-stripped deployed tag)
2. Via `docker run getsentry/sentry-cli`: `releases new` → `releases finalize` → `releases deploys ... -e production`

`releases new` is idempotent: for a forward deploy the release already exists (created by `sentry_release_prod`); for a rollback to a version that was never released forward it creates the release so the deploy event can attach. Marking lives in this job so a rollback (a re-run of this manual job) marks Sentry in the same run — re-running a finished pipeline's job would not cascade to a separate marking job.

---

### `sentry_mark`

#### `sentry_mark_dev`

* **Trigger:** Push to `dev`

Runs:

```bash
sentry-cli releases deploys gen-bi@<short-sha> new -e dev
```

Needs:

```text
deploy_dev
```

> Production deploys are marked inside `deploy_production` itself (see above); there is no separate `sentry_mark_prod` job.

---

## Rollback Procedure

1. Locate the most recent production deployment pipeline in GitLab.
2. Re-run `deploy_production` manually and override `IMAGE_TAG` with the desired rollback version, for example:

```text
v0.1.4
```

The same `deploy_production` job records the rollback in Sentry (release + production deploy event) — there is no second job to run. Because `releases new` is idempotent, rolling back to a version that was never released forward still records correctly (no "Release not found").

---

## Known Risks & Design Tradeoffs

> This section records the **deliberate tradeoffs** and **known limitations** in the current pipeline, so maintainers understand *why* it is built this way and where it can be hardened later.

### Deliberate Tradeoffs

| Item | Tradeoff | Rationale / Mitigation |
|---|---|---|
| `image_retag_preprod` derives the source SHA from `$CI_COMMIT_SHORT_SHA` and hard-fails if the immutable image is missing | Relies on the FF-only invariant (pre-prod tip == the merged dev commit), enforced by `guard_no_squash_preprod`. If `:dev-<sha8>-<version>` can't be pulled after a short retry, the job aborts rather than falling back | Build-once / promote-a-verified-artifact: never promote the mutable `:dev` (a later dev push may have overwritten it) or an image resolved from a racing `origin/dev` HEAD. A missing immutable tag means the release commit never built an image — a real error worth stopping on. |
| `guard_no_squash_preprod` is after-the-fact | Checks for a squash only after the pre-prod push, not as a pre-merge lock | GitLab's squash / merge-method setting is project-wide, with no per-target-branch policy. Allowing "squash for feature→dev but not dev→pre-prod" is only expressible via this CI guard. It runs in `validate`, so a failure blocks the later retag/tag stages. |
| `sonar_scan` uses `allow_failure: true` | A SonarQube failure does not block the pipeline | Intentional during the POC phase; tighten to blocking once the quality gate stabilizes. |
| Sentry marking split across two jobs | Both `sentry_release_prod` and `deploy_production` run `releases new` + `finalize` | Intentional idempotent safety net: `deploy_production` re-creates the release so a rollback to a never-forward-released version can still attach a deploy event (fixes "Release not found"). They are not fully redundant — `set-commits --auto` is only in `sentry_release_prod`, the deploy event only in `deploy_production`. |
| Deploy jobs use `sudo docker` | `deploy_dev` / `deploy_production` run on shell executors where the CI user has passwordless sudo docker (root-equivalent) | Inherent to deploying directly from a shell executor. Actual controls: `master` is a protected branch and `deploy_production` is `when: manual`, so who can reach prod is limited. |
| `GIT_PUSH_TOKEN` embedded in the push URL | `REMOTE=https://oauth2:${GIT_PUSH_TOKEN}@...`, pushed via an inline URL | The token is a Masked variable (redacted in logs) and the push is inline (no `git remote add`), so it never appears in `git remote -v`. Can be hardened by moving the token out of the URL via `-c http.extraHeader` (defense-in-depth, not urgent). |
| `docker image prune -f` at end of deploy | Removes all dangling images on the host | Only removes untagged (dangling) images — not `-a` / `docker system prune` — so it won't touch tagged images other services use. The prod host runs only gen-bi; on the shared lndata-1, consider adding a `--filter` later. |

### Known Limitations & Future Hardening

* **No post-deploy health check:** `docker compose up -d` returning 0 does not mean the container is healthy; a crash-looping new version can still leave the job green. Add a `curl .../actuator/health` gate at the end of deploy and roll back manually on failure.
* **No automatic rollback:** Rollback is manual (re-run `deploy_production` with an `IMAGE_TAG` override). Acceptable for the current team size, but it should be prompted by a failed health check.
* **pom version extraction exists in three places:** `pom_version()` in `.ci_helpers`, the inline awk in `version_guard`, and the inline awk in `docker_build_dev`. A change to the pom structure must be synced in all three. (The latter two don't extend `.ci_helpers`, so they can't reuse it directly; making them do so would also pull in the IPv4 pin and `bind-tools`.)
* **`maven_ci` and `maven_build_dev` compile twice:** On a dev push both compile the same source (the former with tests, the latter `-DskipTests`), costing ~1–2 extra minutes. `maven_ci` could emit the jar artifact for `docker_build_dev` to consume. Performance only, not correctness.

---

## Network Resilience

CI job containers occasionally experience broken IPv6 connectivity to `gitlab.com`.

As a result, `git fetch` and `git push` to `gitlab.com:443` may fail after roughly two seconds.

`.ci_helpers` uses `dig` (from `bind-tools`) to resolve an IPv4 address for `gitlab.com` and writes it into `/etc/hosts`, ensuring Git and curl avoid the faulty route.

> All Git network operations are additionally wrapped with `retry` and `git_net`.

---

## Port Validation (`deploy before_script`)

| Condition                                         | Result                                |
| ------------------------------------------------- | ------------------------------------- |
| Port 8090 is free                                 | Continue                              |
| Owned by `docker-proxy` and container is `gen-bi` | Allowed (container will be recreated) |
| Owned by `docker-proxy` but unrelated container   | ERROR — abort                         |
| Owned by a non-Docker process                     | ERROR — abort                         |

Detection command:

```bash
sudo ss -tlnp | grep ':8090'
```

---

## Deployment Environment Variables

| Variable                                                      | Defined In                                                                          |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `GIT_PUSH_TOKEN`                                              | CI/CD variable (Protected + Masked) — used for pre-prod tag and version bump pushes |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`                 | CI/CD masked variables                                                              |
| `AWS_REGION` / `BEDROCK_MODEL_ID` / `BEDROCK_EMBEDDING_MODEL` | `.deploy_base` mixin `variables` (shared by both deploy jobs)                       |
| `SENTRY_DSN` / `SENTRY_AUTH_TOKEN`                            | CI/CD variables (used by sentry-cli)                                                |
| `GENBI_IMAGE` / `APP_VERSION` / `SENTRY_RELEASE`              | Computed by deployment jobs                                                         |

gen-bi joins the shared `internalnet` bridge network.

Deployment jobs automatically create the network if it does not already exist.

---

## Directory Structure

```text
gen-bi/
  .gitlab-ci.yml          # pom.xml-driven build-once + retag pipeline
  Dockerfile              # Runtime image, copies target/gen-bi-*.jar
  docker-compose.yml      # gen-bi service, port 8090, image defined by ${GENBI_IMAGE}
  pom.xml                 # Project version source (X.Y.Z-SNAPSHOT)
  ci-docs/
    pipeline.md           # English version
    pipeline.zh-TW.md     # This document
  src/
    main/resources/
      application.properties
      application-local.properties
```
