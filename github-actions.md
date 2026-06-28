# GitHub Actions

How we wire CI, image builds, releases, and registry hygiene in our repos. These are **target** defaults with rationale — adapt names and paths to each project, but keep the shape.

Per-repo adoption: [assessments/github-actions.md](assessments/github-actions.md).

## Philosophy

- **Fast feedback on PRs.** Lint, typecheck, unit tests, and compile checks run on GitHub-hosted runners (`ubuntu-latest`). No Docker builds in CI.
- **Deploy on merge.** Image build + push + rollout runs on push to the default branch, on the in-cluster ARC runner (native arm64, no hosted minutes).
- **Releases are explicit.** Version tags (`v*`) trigger a separate multi-arch release build and GitHub Release — not every push.
- **Least privilege.** Declare `permissions` at workflow scope; grant only what each workflow needs.
- **Explain the why.** Top-of-file and inline comments document non-obvious choices (ARC quirks, restart hook, buildx avoidance).
- **Pin everything.** Actions pinned to full commit SHAs; container images in `services:` pinned to digests where practical.

## Standard layout

Every deployable full-stack repo should have:

```
.github/
  workflows/
    ci.yml              # PR + main: lint, test, build (no images)
    build.yml           # main: build/push images + restart (ARC runner)
    release.yml         # v* tags: multi-arch images + GitHub Release
    cleanup-images.yml  # daily: prune untagged registry versions
  actionlint.yaml       # declare self-hosted runner labels
```

| Workflow | Runner | Trigger | Purpose |
|----------|--------|---------|---------|
| `ci.yml` | `ubuntu-latest` | `push` to default branch, all `pull_request` | Path-filtered lint/test/build gates |
| `build.yml` | `arc-<repo>` | `push` to default branch, `workflow_dispatch` | Build changed components, push `:latest`, restart |
| `release.yml` | `ubuntu-latest` | `push` tags `v*` | Multi-arch images (`:vX.Y.Z` + `:latest`), GitHub Release |
| `cleanup-images.yml` | `ubuntu-latest` | daily cron, `workflow_dispatch` | Delete untagged image versions from ghcr.io |

`build.yml` is optional only for repos that do not ship container images.

### Optional supplementary workflows

Add when the repo needs them — not part of the core quintet:

| Workflow | Runner | Trigger | Purpose |
|----------|--------|---------|---------|
| `e2e.yml` | `ubuntu-latest` | `workflow_dispatch` (environment input) | Playwright/Cypress against dev/stage/prod URLs — not on every PR |
| `security-images.yml` | `ubuntu-latest` | weekly cron, `workflow_dispatch` | Pull `:latest` images, Trivy scan CRITICAL/HIGH, upload JSON + summary artifacts |
| `security-source.yml` | `ubuntu-latest` | weekly cron, `workflow_dispatch` | OSV-Scanner over the repo tree, upload report artifact |

Security scans are **informational by default** (`exit-code: 0` on image scans) so scheduled runs surface debt without blocking merges. Tighten to fail-on-CRITICAL once the baseline is clean.

### Per-component workflow split (alternative layout)

For repos with **three or more independently deployable surfaces** (e.g. `frontend/` + `backend/` + `admin/`, or `frontend/` + `frontend2/` + `backend/`), a single `ci.yml` with `paths-filter` still works — but splitting is valid:

```
.github/workflows/
  pr-ci-backend.yml
  pr-ci-frontend.yml
  pr-ci-<component>.yml   # one per deployable surface
  build.yml               # still one ARC workflow, or one build-upload-<component>.yml each
```

**When to split PR workflows:**

- Each file triggers on `pull_request` only with native `paths:` filters (include the workflow file in every filter).
- Downstream jobs stay simple — no `changes` job required per file.
- Reviewers see only the checks that match their diff.

**When to keep monolithic `ci.yml`:**

- Two-component Vue+Go repos (our default) — one `changes` job + conditional jobs is easier to diff and copy.
- You want push-to-main CI gates in the same file as PR gates.

Regardless of layout, **one ARC `build.yml`** (or per-component build workflows that all call `oglimmer.sh`) remains the deploy path — do not build images on `ubuntu-latest` except in `release.yml` multi-arch.

## `ci.yml` — pull-request gate

### Triggers

```yaml
on:
  push:
    branches: [main]   # match the repo default branch (main or master)
  pull_request:
```

### Concurrency

Cancel superseded PR runs; let main-branch pushes queue.

```yaml
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

### Permissions

```yaml
permissions:
  contents: read
```

CI never writes packages, releases, or repo contents.

### Path filtering

Always start with a `changes` job on `ubuntu-latest` using [`dorny/paths-filter`](https://github.com/dorny/paths-filter). Downstream jobs `needs: changes` and `if: needs.changes.outputs.<component> == 'true'`.

Include the workflow file itself in every filter so a CI change re-runs the affected jobs:

```yaml
filters: |
  backend:
    - 'backend/**'
    - '.github/workflows/ci.yml'
  frontend:
    - 'frontend/**'
    - '.github/workflows/ci.yml'
```

Add more components (e2e, docs, infra) as the repo grows. Mirror the same component boundaries in `build.yml`.

### Backend job

- `defaults.run.working-directory: backend`
- `actions/setup-go` with `go-version-file: backend/go.mod` and `cache-dependency-path: backend/go.sum`
- Steps, in order: `gofmt` (fail if unformatted), `go vet`, `go build`, `go test -race`
- Use a `services.postgres` block when integration tests need a database. Pin the image to a digest:

  ```yaml
  services:
    postgres:
      image: postgres:16@sha256:fe03a7605299a34ddf5e4f285dff78c3d7190a576b3c6b46f2fcff69f4bffd54
      env:
        POSTGRES_USER: <app>
        POSTGRES_PASSWORD: <app>
        POSTGRES_DB: <app>
      ports: ['5432:5432']
      options: >-
        --health-cmd "pg_isready -U <app>"
        --health-interval 10s
        --health-timeout 5s
        --health-retries 5
  ```

- Pass the test DB URL via `env` (e.g. `IRL_TEST_DATABASE_URL` / `TEST_DATABASE_URL`). Tests should self-skip when the variable is unset so local `go test` without Postgres still works.

### Frontend job

- `defaults.run.working-directory: frontend`
- `actions/setup-node` with an explicit `node-version` (or `.nvmrc`), `cache: npm`, `cache-dependency-path: frontend/package-lock.json`
- Steps, in order: `npm ci`, lint, typecheck, test, build
- Never `npm install` — always `npm ci` for reproducible CI.

### Pin actions

Pin every `uses:` to a full commit SHA with a version comment:

```yaml
- uses: actions/checkout@9c091bb21b7c1c1d1991bb908d89e4e9dddfe3e0 # v7
```

Renovate's `helpers:pinGitHubActionDigests` keeps these current.

## `build.yml` — image build & deploy

Runs on the per-repo ARC scale set (`runs-on: arc-<repo>`). See [Self-hosted ARC runners](#self-hosted-arc-runners) for onboarding.

### Triggers

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
```

### Concurrency

One build per ref; cancel superseded pushes on the same branch.

```yaml
concurrency:
  group: build-${{ github.ref }}
  cancel-in-progress: true
```

For repos that split build into per-component workflows, use **per-component groups** so a frontend push does not cancel an in-flight backend build:

```yaml
concurrency:
  group: build-frontend-${{ github.ref }}
  cancel-in-progress: true
```

### Permissions

```yaml
permissions:
  contents: read
  packages: write
```

### Path filtering

Same `dorny/paths-filter` shape as `ci.yml`, but **also** include shared build inputs in every component filter:

```yaml
filters: |
  backend:
    - 'backend/**'
    - 'oglimmer.sh'
    - '.github/workflows/build.yml'
  frontend:
    - 'frontend/**'
    - 'oglimmer.sh'
    - '.github/workflows/build.yml'
```

A change to `oglimmer.sh` or `build.yml` must rebuild all components that depend on it.

### Build job steps

Per changed component:

1. `actions/checkout` (SHA-pinned)
2. `docker/login-action` — registry `ghcr.io`, `username: ${{ github.actor }}`, `password: ${{ secrets.GITHUB_TOKEN }}`
3. `./oglimmer.sh build -<flag> --platform auto --verbose` with `RESTART_TOKEN` in `env`

```yaml
- name: Build, push & restart backend
  env:
    RESTART_TOKEN: ${{ secrets.RESTART_TOKEN }}
  run: ./oglimmer.sh build -b --platform auto --verbose
```

**Always pass `--platform auto` in the workflow**, even if `oglimmer.sh` defaults to something else. On ARC (arm64 + dind), `auto` selects plain `docker build` + `docker push`. Buildx's docker-container driver loses its push session against the registry ingress (`no active session … context deadline exceeded`).

The runner has no `kubectl`. Rollout goes through the restart hook (`https://restart.oglimmer.com/restart/<ns>/<deployment>`) using `RESTART_TOKEN`. Alternative: `--no-restart` + keel watching the registry — pick one per repo; default to the restart hook.

### Registry credentials

| Secret | Used by | Source |
|--------|---------|--------|
| `GITHUB_TOKEN` | `ghcr.io` login in workflow | Automatic; needs `packages: write` |
| `RESTART_TOKEN` | `oglimmer.sh` restart hook | Cluster secret `restart-hook-secret` / key `token` |
| `REGISTRY_USERNAME` / `REGISTRY_PASSWORD` | Private `registry.oglimmer.com` repos | Cluster secret `oglimmerregistrykey` |

Set repo secrets from the cluster; never commit or echo them. `oglimmer.sh` must accept kubectl (local dev) **or** `RESTART_TOKEN` (CI) — not require kubectl unconditionally.

## `release.yml` — tagged releases

Triggered when the release script pushes a `v*` tag. Builds **multi-arch** (`linux/amd64`, `linux/arm64`) images on a hosted runner with QEMU + buildx, tags them `:v<version>` and `:latest`, and creates a GitHub Release with auto-generated notes.

Everyday `:latest` builds stay in `build.yml` (arm64-only, ARC). `release.yml` exists for reproducible, version-pinned, multi-arch artifacts. Semver sources, UI display, and tag rules: [versioning-release.md](versioning-release.md).

```yaml
on:
  push:
    tags: ['v*']

permissions:
  contents: write
  packages: write
```

Extract the version from the tag:

```yaml
- id: version
  run: echo "version=${GITHUB_REF_NAME#v}" >> "$GITHUB_OUTPUT"
```

Use `docker/build-push-action` with GHA cache (`cache-from: type=gha`, `cache-to: type=gha,mode=max`) and bake build metadata (`VERSION`, `GIT_COMMIT`, `BUILD_TIME`) via `build-args`.

Finish with `softprops/action-gh-release` and `generate_release_notes: true`.

### Scheduled rebuild (optional)

Add a weekly `schedule` + `workflow_dispatch` on `release.yml` (or a dedicated `rebuild-release.yml`) to rebuild the **latest `v*` tag** without a new git push. Useful when base images or scanner baselines change but application code did not:

```yaml
on:
  push:
    tags: ['v*']
  schedule:
    - cron: '0 9 * * 5'   # e.g. Friday 09:00 UTC
  workflow_dispatch:
```

Resolve the tag in a step: use `github.ref_name` on tag push; on schedule/dispatch, `git describe --tags "$(git rev-list --tags --max-count=1)"` and `git checkout` that tag before building.

## `e2e.yml` — manual environment tests (optional)

Keep browser e2e **out of the PR critical path** when tests hit shared environments or are slow. Run on demand:

```yaml
on:
  workflow_dispatch:
    inputs:
      target:
        description: Target environment
        type: choice
        options: [dev, stage, production]
        default: dev

permissions:
  contents: read
```

Map `inputs.target` to `UI_HOST` / `API_HOST` via `fromJSON` or a small lookup table in `env:`. Cache `~/.cache/ms-playwright` keyed on `package-lock.json`; run `playwright install --with-deps` only on cache miss. Upload `playwright-report/` as an artifact (`retention-days: 30`, `if: always()`).

Repos that already run Playwright in `ci.yml` (e.g. plugin-skill-hosting) can add this for post-deploy smoke without duplicating PR coverage.

## `security-images.yml` — container scan (optional)

Weekly scan of images already in the registry:

```yaml
on:
  schedule:
    - cron: '0 6 * * 0'
  workflow_dispatch:

permissions:
  contents: read
  security-events: write   # if uploading SARIF later
```

Use a `matrix` over image names/tags. Pull each image, run Trivy (`format: table` for logs, `format: json` for artifacts). Upload `trivy-results-<component>.json` and a markdown summary per matrix leg. Default `exit-code: 0` until the team opts into gating.

## `security-source.yml` — dependency scan (optional)

```yaml
on:
  schedule:
    - cron: '0 0 * * 0'
  workflow_dispatch:

permissions:
  contents: read
```

Install a pinned OSV-Scanner release binary, `osv-scanner scan source --recursive .`, upload the report, append to `$GITHUB_STEP_SUMMARY`. `continue-on-error: true` on the scan step — same informational default as image scans.

## `cleanup-images.yml` — registry hygiene

ArgoCD image-updater (and similar) pins deployments by digest. Every `:latest` push leaves the previous digest as an untagged version. Without cleanup these pile up forever.

```yaml
on:
  schedule:
    - cron: '0 4 * * *'
  workflow_dispatch:

permissions:
  packages: write
```

Use `dataaxiom/ghcr-cleanup-action` with `delete-untagged: true`. List every package name for the repo. Tagged versions (`:latest`, `:v*`) are kept.

## Self-hosted ARC runners

- **One scale set per repo.** `oglimmer` is a personal GitHub User (no user-level runners), so each repo gets `arc-<repo>` in `~/dev/global-install-k8s/actions-runner-controller/k8s/manifest.yml`.
- **Cluster facts:** arm64 (Raspberry Pi), `minRunners: 0` (cold start per job), dind sidecar, pinned to node `k8s-node27`.
- **Operational docs:** `~/dev/global-install-k8s/actions-runner-controller/k8s/howto.md`.
- **Onboarding skill:** `/setup-arc-build` (adds scale set, `oglimmer.sh` restart path, `build.yml`, GitHub secrets).

Declare custom labels in `.github/actionlint.yaml` so local lint passes:

```yaml
---
self-hosted-runner:
  labels:
    - arc-<repo>
```

## Cross-cutting rules

### Workflow file header comment

Every workflow starts with a `#` block explaining what it does, what triggers it, which runner it uses, and how it relates to the other workflows.

### `permissions` block

Always present at workflow scope. Never rely on the default `GITHUB_TOKEN` permissions.

### No secrets in logs

`run:` scripts and `oglimmer.sh` must never echo tokens. Pass secrets only via `env:` or `with:`.

### `workflow_dispatch`

Add to `build.yml` and `cleanup-images.yml` so operators can trigger manually without a push.

### Service container images

Pin to a digest in CI when the image is stable (Postgres, Redis). Renovate's `docker:pinDigests` can keep these updated.

### Job `defaults`

Set `working-directory` at the job level, not repeated per step.

### Job timeouts

Set `timeout-minutes` on every job. PR gates: 10–15 min; ARC builds: 20–30 min; e2e: 30 min. Prevents hung runners (especially self-hosted) from blocking the scale set indefinitely.

### Quality gates before image push

`ci.yml` on PR + main is the primary gate. On merge, `build.yml` may **re-run the cheap checks** for the component being built (lint, typecheck, unit tests) immediately before `oglimmer.sh build` — especially when frontend Dockerfiles skip those steps. Belt-and-suspenders, not a substitute for `ci.yml`.

### `if:` on filtered jobs

```yaml
if: needs.changes.outputs.backend == 'true'
```

Use the string `'true'` — `paths-filter` outputs are strings, not booleans.

### Mirror pre-commit

Fast checks in pre-commit should mirror the cheap half of `ci.yml`; slow build/test gates run on `pre-push`. See each repo's `.pre-commit-config.yaml`.

## Linting & validation

Every repo with workflows should lint them locally:

| Tool | Config | Purpose |
|------|--------|---------|
| [actionlint](https://github.com/rhysd/actionlint) | `.github/actionlint.yaml` | Workflow syntax, expression types, runner labels |
| [yamllint](https://github.com/adrienverge/yamllint) | `.yamllint.yaml` | YAML structure (`truthy.check-keys: false` for `on:`) |

Hook both in `.pre-commit-config.yaml`. actionlint shells out to shellcheck for `run:` blocks when shellcheck is installed.

Exclude Helm Go-templated YAML from yamllint — it is not valid YAML.

## Dependency updates (Renovate)

Extend at minimum:

```json
"helpers:pinGitHubActionDigests"
```

Group non-major action updates; automerge minor/patch/pin/digest for actions alongside other low-risk deps.

## New-repo checklist

1. Add `ci.yml` with path filters matching the repo layout (or per-component `pr-ci-*.yml` if 3+ deployable surfaces).
2. If the repo ships images: onboard ARC (`arc-<repo>`), add `build.yml`, set `RESTART_TOKEN` (+ registry secrets if not using ghcr.io + `GITHUB_TOKEN`).
3. Add `release.yml` if the repo uses semver tags and multi-arch releases.
4. Add `cleanup-images.yml` if images are pushed to ghcr.io.
5. Add `.github/actionlint.yaml` with the ARC label.
6. Add actionlint + yamllint to pre-commit.
7. Enable Renovate `pinGitHubActionDigests`.
8. Set `timeout-minutes` on every workflow job.
9. Consider `security-images.yml` + `security-source.yml` once the repo ships to production.
10. Consider `e2e.yml` (`workflow_dispatch`) if Playwright against live environments is needed post-deploy.
11. Smoke-test: `gh workflow run build.yml` → `gh run watch`; job stuck `queued` ⇒ listener/token issue on the cluster.

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Build Docker images in `ci.yml` | `build.yml` on ARC, or `release.yml` for tags |
| Use `runs-on: self-hosted` without a named scale set | `runs-on: arc-<repo>` |
| Use buildx docker-container driver on ARC | `--platform auto` → plain docker |
| Require `kubectl` in CI | Restart hook or keel |
| Use `@v6` tags without SHA in new workflows | Full SHA + `# v6` comment |
| Grant `permissions: write-all` | Minimal per-workflow `permissions` |
| Skip path filtering | `changes` job + conditional downstream jobs |
| Run full CI on docs-only PRs | Tight `paths-filter` definitions |