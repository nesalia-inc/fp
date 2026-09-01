# Release Pipeline — How It Works

**Status:** Active
**Owner:** Engineering
**Last updated:** 2026-08-04
**Audience:** anyone who needs to ship a release, audit the pipeline, or onboard a new maintainer

This document describes the release pipeline **as it actually runs today** — the design rationale and migration history live in [`../plans/release-pipeline.md`](../plans/release-pipeline.md); the GitHub UI and npmjs.com configuration steps live in [`../plans/release-pipeline-github-ui-setup.md`](../plans/release-pipeline-github-ui-setup.md). Read those if you need *why*; read this one if you need *how*.

---

## 1. What the system does

A merge to `main` of a pull request that introduces a Changesets file under `.changeset/` triggers an end-to-end release run that:

1. Detects the new Changeset.
2. Bumps the package version (`pnpm changeset version`).
3. Commits the version bump back to `main`.
4. Builds and tests the package.
5. Runs a smoke test on the built artifact (dynamic ESM import + export presence).
6. Publishes to npm via **OIDC Trusted Publishing**, with **Sigstore provenance attestation**.
7. Creates and pushes the git tag `vX.Y.Z`.
8. Creates a GitHub Release for the tag with auto-generated notes.

Tag pushes (`refs/tags/vX.Y.Z`) and manual `workflow_dispatch` runs follow the same path, skipping only the changeset detection step.

No human touches `NPM_TOKEN` at any point in this flow.

## 2. The single workflow: `.github/workflows/publish.yml`

This file is the **only** workflow that runs npm publish. It is the file registered as the Trusted Publisher on npmjs.com with `workflow filename = publish.yml` and `environment = release`. npm validates the file that contains the `pnpm changeset publish` step; that file must be `publish.yml`, not a caller that dispatches to a reusable workflow.

### 2.1 Triggers

```yaml
on:
  pull_request:
    types: [closed]
    branches: [main]
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Skip publish and tag push (build and test only)'
        type: boolean
        default: false
```

- **`pull_request: closed` (merged into `main`)** — the canonical path. A PR introducing a changeset gets merged and the release runs.
- **`push: tags: 'v*.*.*'`** — hotfix path. A maintainer pushes the tag manually after merging the hotfix branch into `main`.
- **`workflow_dispatch`** — manual path. Useful for re-publishing after fixing a transient error, or for dry-runs.

### 2.2 The single release job

The workflow has one job (`release`) gated on:

```yaml
if: |
  github.event_name == 'workflow_dispatch' ||
  (github.event_name == 'pull_request'
   && github.event.pull_request.merged == true
   && github.event.pull_request.base.ref == 'main') ||
  (github.event_name == 'push'
   && startsWith(github.ref, 'refs/tags/v'))
```

Environment: `release` (named reviewers required by the GitHub environment protection rule). Job-level permissions: `id-token: write`, `contents: write`, `pull-requests: read`.

### 2.3 Step sequence

| # | Step | Always runs? | Purpose |
|---|------|--------------|---------|
| 1 | Checkout `main` with `fetch-depth: 0` | yes | Full history for the changeset diff |
| 2 | Setup pnpm + setup-node v24 | yes | Reproducible build |
| 3 | `npm install -g npm@latest` | yes | Trusted Publishing requires npm ≥ 11.5.1 |
| 4 | `pnpm install --frozen-lockfile` | yes | Reproducible dependency set |
| 5 | Detect changesets | yes | Set `has_changesets` output (see §2.4) |
| 6 | `pnpm changeset version` | only if `has_changesets == true` | Bump the version |
| 7 | Push version bump back to `main` | only if PR-merge path | The bot commits `chore(release): version packages` and pushes |
| 8 | Anti-republish guard | only if `has_changesets == true` | Queries npm for the local version; exits if it already exists |
| 9 | `pnpm build` | only if `has_changesets == true` | Compile |
| 10 | `pnpm test` | only if `has_changesets == true` | Run tests |
| 11 | Smoke test the built artifact | only if `has_changesets == true` | Dynamic ESM import, check exports |
| 12 | `pnpm changeset publish --tag latest` | only if `has_changesets == true` and not `dry_run` | The actual publish to npm |
| 13 | Create git tag `v${VER}` | only if `has_changesets == true` and not `dry_run` | `git tag -a` + `git push origin` |
| 14 | Create GitHub Release | only if `has_changesets == true` and not `dry_run` | `softprops/action-gh-release@v3.0.2` SHA-pinned |

When `has_changesets == false`, the job runs setup through step 5 only and exits green. The intent is to keep the workflow active on every merge (for traceability) while skipping publish on merges that contain no user-visible change.

### 2.4 Changeset detection

```bash
if tag push → always proceed
elif workflow_dispatch → always proceed
elif PR merge → check git diff --name-only HEAD~1 HEAD
  for files matching .changeset/*.md (excluding README.md, config.json)
  if any → has_changesets = true
```

The detection looks at **files added or modified** in the merge commit. A merge that doesn't touch `.changeset/` produces a green run with no publish. This is by design: most merges to `main` are infra-only (workflow changes, doc updates, dependency bumps), and the pipeline should not pretend to publish a release.

### 2.5 Concurrency

```yaml
concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false
```

Single concurrency group per ref. `cancel-in-progress: false` so retried publishes finish rather than getting cancelled mid-publish.

## 3. Security model

### 3.1 Trust chain

```
npm registry
  ↕ OIDC token (id-token, audience = npm:registry.npmjs.org)
GitHub Actions runner
  ↕ GITHUB_TOKEN (per-run, scoped to repo)
publish.yml + softprops/action-gh-release
```

- **No long-lived secrets.** No `NPM_TOKEN` anywhere — not in repo secrets, not in org secrets, not on developer machines.
- **OIDC tokens are short-lived** (minutes), minted per workflow run, scoped to the OIDC `aud` claim configured by npm.
- **GitHub Actions SHA pinning.** Every third-party action (`actions/checkout`, `pnpm/action-setup`, `actions/setup-node`, `softprops/action-gh-release`) is pinned by commit SHA in `publish.yml`. No tag-based refs.

### 3.2 Gates that must all close before a publish

1. **The PR that introduced the changeset has been merged into `main`.** A direct push to `main` is rejected by branch protection (`Include administrators: ON`, no bypass list).
2. **The merge commit contains a changeset file.** Otherwise the job exits green without publishing.
3. **The environment `release` has approved the run.** Required reviewers configured on the GitHub environment.
4. **The anti-republish guard sees the version as not yet on npm.** Prevents double-publish on retries and accidental re-runs.
5. **The build, the test suite, and the smoke test all pass.**
6. **`pnpm changeset publish` succeeds against npm** with OIDC provenance.

### 3.3 What we don't use, and why

- **No tag protection ruleset.** The `github-actions[bot]` identity that pushes tags from the workflow is not allow-listable on GitHub's tag protection rules. Adding such a rule would block the release pipeline with HTTP 403 (observed). Defense in depth comes from environment protection, branch protection, and the anti-republish guard, not from tag protection.
- **No reusable workflows.** Earlier iterations dispatched to `_publish-*.yml`. npm Trusted Publishing validates the **workflow file containing the publish step** — with reusable workflows, the OIDC `workflow_ref` claim pointed to the reusable file and surfaced as a misleading `E404 Not Found` (npm/cli #9088). The fix was to inline every publish step into `publish.yml`.
- **No `NPM_TOKEN` fallback.** npm CLI 11.5.1+ prefers OIDC and falls back to tokens only if OIDC is unavailable. We don't provide a token, so there's no fallback path. OIDC must work — and it does.

### 3.4 Provenance attestation

Every publish under this pipeline produces a Sigstore-signed SLSA provenance attestation. Verification:

```bash
npm view @deessejs/fp@1.1.0 dist.attestations.provenance
# { 'predicateType': 'https://slsa.dev/provenance/v1' }
```

The attestation is built automatically by npm because:

- The package is public.
- The publish used OIDC (Trusted Publishing).
- `packages/fp/package.json` declares `publishConfig.provenance: true`.

Users can verify with `npm audit signatures`.

## 4. Day-to-day operations

### 4.1 Cutting a release

A developer wants to ship a feature.

1. **Branch off `main`.** (No `dev` or `staging` branch in this pipeline.)

   ```bash
   git checkout -b feat/add-result-flatten
   ```

2. **Make the change.** Touch `packages/fp/src/**` as needed.

3. **Add a Changeset.** This is the user-facing signal that a release is needed:

   ```bash
   pnpm changeset
   ```

   Interactive prompt: pick `@deessejs/fp`, pick the bump type (`patch`, `minor`, `major`), write a one-line user-facing summary. This writes `.changeset/<random-name>.md`.

4. **Open a PR against `main`.** Add a reviewer. CI runs (`ci.yml`).

5. **Reviewer merges.** The merge triggers `publish.yml`. Within ~2 minutes:
   - npm has the new version with provenance.
   - `main` has the version-bump commit.
   - `vX.Y.Z` tag exists on `main`.
   - A GitHub Release exists for the tag.

That's it. No `npm login`, no `npm version`, no `npm publish` on a developer machine.

### 4.2 Cutting a hotfix

1. **Branch off `main`.**

   ```bash
   git checkout -b hotfix/fix-typo
   ```

2. **Apply the fix.** Update `packages/fp/src/**` as needed.

3. **Bump the version manually.** This is the one place where Changesets is bypassed: the hotfix is too urgent to wait for a PR review.

   ```bash
   # In packages/fp/package.json, edit "version" to e.g. "1.1.1".
   git add packages/fp/package.json
   git commit -m "hotfix: bump to 1.1.1 for the critical fix"
   ```

4. **Open a PR against `main`.** One reviewer (required by branch protection).

5. **After the PR merges, push the tag manually.** The hotfix path is triggered by `git push --tags`, not by a PR merge (because the merge alone doesn't carry the version bump information as a Changeset).

   ```bash
   git tag -a v1.1.1 -m "Hotfix v1.1.1"
   git push origin v1.1.1
   ```

   The push triggers `publish.yml` on the tag path. The job runs the anti-republish guard (which fails if 1.1.1 already exists), then publishes.

6. **Back-merge the hotfix.** After the hotfix ships, merge `main` (which now contains both the fix and the version bump commit) back into `dev` and any other long-lived branches. In this monorepo, there are none, so the back-merge is usually unnecessary.

### 4.3 Manual dry-run

To validate that the pipeline is healthy without burning a version number:

1. GitHub → Actions → "Release" → "Run workflow" → branch `main` → check "Skip publish and tag push" → Run.

This triggers `publish.yml` with `dry_run: true`. The job runs setup, builds, tests, and smoke tests, then exits without publishing. The run reports green; nothing is published.

### 4.4 Re-running after a transient failure

If the workflow fails after step 8 (anti-republish guard) or later:

1. Don't `gh run rerun` blindly. The anti-republish guard will fail again because the version is now on npm.
2. Either accept the partial state (npm has the version but the GitHub Release was not created) and create the GitHub Release manually with `gh release create vX.Y.Z --target <commit-sha>` — or fix the underlying issue and re-push a no-op commit to `main` to trigger a fresh run.

The CI exit status of the failed step tells you where it broke:

- **Anti-republish guard failed** → publish already happened upstream. Verify on npm, then complete the release manually.
- **Smoke test failed** → the build is broken. Investigate, fix, re-run.
- **Publish failed (e.g. `E404`)** → npm rejected the OIDC. Verify the Trusted Publisher configuration and the workflow filename match on npmjs.com.
- **Create GitHub Release failed (e.g. `HTTP 500`)** → GitHub API outage. Re-run later or create the release manually with `gh release create`.

## 5. Branch strategy

This pipeline does **not** use a `staging` branch or a `dev` branch. The earlier design that did (`main <- staging <- dev`) was replaced by the current monorepo-style pattern because it introduced a chicken-and-egg problem: the changeset-version workflow couldn't open a "Version Packages" PR from staging to main without producing a cross-branch diff that GitHub rejected. The current design assumes:

- **Direct PRs to `main`** for every change. No `staging` intermediate.
- **Branch protection on `main`** with PR required + 1 reviewer + linear history. Main is the only branch that matters.
- **`feat/*`, `fix/*`, `hotfix/*`** as branch naming conventions, all targeting `main`.

If multi-stage integration becomes necessary in the future (e.g. for long-running refactors that shouldn't land on `main` daily), reintroduce a `staging` branch — but expect to redesign the workflow to handle the cross-branch flow again. Today the design does not support it.

## 6. Trust boundary with npm

The pipeline talks to npm via three surfaces:

- **Trusted Publisher** — registered on npmjs.com as GitHub Actions, workflow filename `publish.yml`, environment `release`, allowed action `npm publish`.
- **Package publishing access** — `Require two-factor authentication and disallow tokens` on the npmjs.com `/access` page.
- **OIDC audience** — implicit; npm CLI sends `audience = npm:registry.npmjs.org` and GitHub Actions returns the OIDC token.

Once the Trusted Publisher is registered, **all other tokens are noise**. Delete them.

## 7. What this pipeline does not do

- **Multi-package monorepo publishing.** The current `pnpm changeset publish` invocation publishes whatever Changesets found in the merge. With multiple packages, that works as-is — but the workflow doesn't have package-specific guards (e.g. a separate anti-republish check per package). For now, `@deessejs/fp` is the only published package in this repo.
- **Pre-release cycles (`next`, `beta`, `rc`).** Not implemented. Adding them requires a `release.yml` variant with `pnpm changeset pre enter next` or similar, plus a separate npm Trusted Publisher entry pointing to that workflow — which is currently impossible because npm allows only one Trusted Publisher per package.
- **`pkg.pr.new` or other canary hosting.** The canary path was removed when the pipeline switched to the monorepo senior pattern. Today, the only pre-publish safety net is the GitHub Release workflow run itself, which is observable but doesn't expose artifacts to consumers. If canary hosting is needed, `pkg.pr.new` is the 2026 senior choice — see [`../plans/release-pipeline.md`](../plans/release-pipeline.md) §8.1.2 for the deferred migration plan.
- **Automated rollback.** If a published version turns out to be broken, the recovery is manual: revert the publish commit on `main`, push a tag re-publish, or `npm unpublish` (within the 72h window). The pipeline has no automatic undo.

## 8. Local debugging

You cannot test this pipeline locally in any meaningful way — OIDC requires GitHub Actions' OIDC provider. What you *can* do locally:

- **Verify a build before opening a PR**: `pnpm build`, `pnpm test`, then inspect `packages/fp/dist/` to confirm the artifact is publishable.
- **Dry-run a changeset locally**: `pnpm changeset status` shows pending changesets; `pnpm changeset version --snapshot canary` produces a snapshot version locally that you can `npm pack` and inspect.
- **Reproduce a publish failure locally**: with `NPM_CONFIG_PROVENANCE=true` and a long-lived `NODE_AUTH_TOKEN` set, you can `npm publish` from your machine to see the same error messages without consuming the OIDC path.

For pipeline debugging, the GitHub Actions run log is the source of truth. `gh run view <id> --log` or the web UI.

## 9. Auditing a past release

```bash
# 1. Find the release commit and tag.
git log --oneline | grep -E "chore\(release\): version packages"
git tag --list "v1.*"

# 2. Verify the tag's commit and provenance on npm.
curl -s https://registry.npmjs.org/@deessejs/fp/1.1.0 | jq '.dist.attestations.provenance'

# 3. Find the original PR that introduced the changeset.
git log --diff-filter=A --name-only --pretty=format:"%h %s" -- '.changeset/*.md'
# Pick the commit, then look up the PR in the GitHub UI.

# 4. Find the workflow run.
gh run list --workflow "Release" --branch main --limit 10
# Filter by date and commit to find the matching run.

# 5. Verify the GitHub Release.
gh release view v1.1.0
```

## 10. References

- **Plan** (design rationale, migration history): [`../plans/release-pipeline.md`](../plans/release-pipeline.md)
- **UI / npmjs.com setup** (one-time manual steps): [`../plans/release-pipeline-github-ui-setup.md`](../plans/release-pipeline-github-ui-setup.md)
- **npm Trusted Publishing docs**: <https://docs.npmjs.com/trusted-publishers/>
- **npm/cli #9088** (misleading 404 on Trusted Publishing failures): <https://github.com/npm/cli/issues/9088>
- **Changesets docs**: <https://github.com/changesets/changesets>
- **GitHub OIDC for Actions**: <https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect>