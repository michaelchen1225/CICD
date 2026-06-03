# gen-bi Release Manager Handbook

> The release manager is responsible for version management, releases, code reviews, and production deployments.
>
> Branch workflow:
> `feature/*` → `dev` → `pre-prod` → `master` (**FF-only**).

---

## 1. Releasing: dev → pre-prod

The project version is defined in the project's `<version>` element in `pom.xml`, using the format:

```text
X.Y.Z-SNAPSHOT
```

The release manager is responsible for deciding version changes.

The next release version is simply the `dev` branch version with `-SNAPSHOT` removed. (PATCH has already been automatically incremented by CI after the previous release.)

* **PATCH**: Automatically incremented by CI after each release. No manual action required.
* **MINOR / MAJOR**: Must be updated manually in the project's `<version>` before releasing.

Example:

```xml
<version>0.2.0-SNAPSHOT</version>   <!-- MINOR bump -->
<version>1.0.0-SNAPSHOT</version>   <!-- MAJOR bump -->
```

> `pom.xml` contains two `<version>` elements:
>
> * The Spring Boot version inside `<parent>` (must not be modified).
> * The project's own `<version>` located after `<artifactId>gen-bi</artifactId>` (this is the version that should be updated).

### Release Procedure

1. If a MINOR or MAJOR version bump is required, update the project version in `dev`.
2. Verify that the `dev` pipeline is green and that the image `:dev-<sha>-<version>` already exists in the registry(gen-bi -> Deploy -> Container registry).
3. Verify that there is no open `pre-prod → master` merge request.
4. Open an MR from `dev → pre-prod`, set **Squash = OFF**, review, and merge.

After the merge, the pre-prod push pipeline automatically performs the following:

| Job                       | Action                                                                                                      |
| ------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `guard_no_squash_preprod` | Verifies that the merge was not squashed                                                                    |
| `image_retag_preprod`     | Retags `:dev-<sha>-<version>` to `:vX.Y.Z` and creates Git tag `vX.Y.Z` (no rebuild)                        |
| `prepare_next_release`    | Increments PATCH in `pre-prod`, commits, and pushes using `-o ci.skip` to avoid triggering another pipeline |

> **Heads-up** — Before merging, confirm that the dev pipeline is green and :dev-<sha>-<version> is already in the registry.
> 
>image_retag_preprod retags :dev-<sha>-<version> — an immutable image tied to the exact commit being released — to :vX.Y.Z.
>
>If that image cannot be found, the job falls back to the mutable :dev tag and prints WARN: ... falling back to :dev. At that point, the published vX.Y.Z may not correspond to the reviewed commit, since :dev can be overwritten by any subsequent push.
>
>If you see this warning, verify that the image digest matches what was reviewed, or wait for the dev pipeline to go green and re-cut the release.

### After the Release: Sync pre-prod Back to dev

CI pushes a version-bump commit to `pre-prod`. This commit does not exist in `dev`.

If it is not synced back, the next `dev → pre-prod` MR will encounter a fast-forward conflict.

After the release pipeline finishes, run:

```bash
git checkout dev
git fetch origin
git merge origin/pre-prod   # Pull back the CI-generated version bump commit
git push origin dev
```

If a fast-forward conflict has already occurred, the resolution is the same. The conflict is usually limited to the version number in `pom.xml`.

---

## 2. Scheduling Rule: Only One Release at a Time

While a `pre-prod → master` MR is open, no additional changes may be merged into `pre-prod`.

### Why?

A workflow rule suppresses branch pipelines when a branch already has an open merge request.

Since all release jobs run on the pre-prod push pipeline, merging additional changes into `pre-prod` while a `pre-prod → master` MR is open will silently prevent release creation.

No error message will be shown.

### Correct Sequence

1. Complete `dev → pre-prod`.
2. Wait for the release pipeline to finish:

   * Git tag created
   * Release image created
   * Version bumped
3. Open the `pre-prod → master` MR.
4. After the master MR is merged (or closed), begin the next `dev → pre-prod` release cycle.

### Recovery Procedure

If a pre-prod pipeline was skipped:

1. Close the open `pre-prod → master` MR.
2. Re-run the pipeline or push a new commit to `pre-prod`.
3. Wait for the release pipeline to complete.
4. Re-open the `pre-prod → master` MR.

---

## 3. Version Guard (`version_guard`)

`version_guard` runs on `dev → pre-prod` merge requests and performs three validations.

| Error                    | Meaning                                                                           | Resolution                                                           |
| ------------------------ | --------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Invalid version format   | The project version is not in `X.Y.Z-SNAPSHOT` format                             | Update the version in `dev`                                          |
| Version already released | Tag `vX.Y.Z` already exists, usually because `dev` was not synced with `pre-prod` | Merge `origin/pre-prod` back into `dev` and reopen the MR            |
| Version regression       | Version is lower than the highest existing release tag                            | Increase the version in `dev` so it exceeds the highest existing tag |

All three failure scenarios include detailed troubleshooting instructions directly in the CI logs.

---

## 4. Deploying to Production: pre-prod → master

1. Open an MR from `pre-prod → master`, perform code review, and merge.

   * This MR intentionally does not run a pipeline.
   * `master` is promotion-only and does not rerun `maven_ci` or `sonar_scan`.

2. After the merge, the master push pipeline performs:

   * `sentry_release_prod` creates release `gen-bi@X.Y.Z`
   * `deploy_production` (manual job)

     * Open Pipelines
     * Click **Run job**
     * No variables are required
     * `IMAGE_TAG` automatically defaults to the latest `vX.Y.Z`
     * The same job marks the production release and deploy event in Sentry (there is no separate `sentry_mark_prod`)

---

## 5. Rollback

```text
GitLab → CI/CD → Pipelines → master pipeline
→ deploy_production → Run job
→ Variables: IMAGE_TAG = v<previous-version>
→ Run
```

`deploy_production` marks Sentry itself, so the rollback is recorded by that same job — there is no second job to run. The Sentry release is created idempotently, so rolling back to a version that was never released forward still records correctly (no "Release not found").

### Validation

* The older release shows a new deployment event.
* The latest release does not receive an additional deployment event.

---

## 6. Idempotency and Safe Re-runs

Both `image_retag_preprod` and `prepare_next_release` are idempotent and safe to rerun manually.

### `image_retag_preprod`

* If the Git tag already exists, retagging and tag creation are skipped.

### `prepare_next_release`

* Compares the remote `pre-prod` version in `pom.xml`.
* If the version has already been bumped, the job exits without changes.

If a release job fails due to networking issues or other transient problems, simply rerun the failed job. No duplicate tags or duplicate version bumps will be created.

---

## 7. Release Checklist

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
