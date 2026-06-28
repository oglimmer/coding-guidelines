# oglimmer.sh

The repo-root management script for build, push, rollout, release, and local dev. Every deployable project has one. CI calls it from `build.yml`; developers run it locally.

Onboard new repos with the `/setup-arc-build` skill. Per-repo adoption and script shapes: [assessments/oglimmer-sh.md](assessments/oglimmer-sh.md).

## Philosophy

- **One script, one workflow.** Build images, push to registry, restart deployments — don't document three separate manual steps.
- **Works locally and in CI.** kubectl when the developer has cluster access; restart hook when the ARC runner does not.
- **Fail before build, not after.** Validate docker, restart credentials, and platform support up front in `check_prerequisites` / `validate_dependencies`.
- **Never echo secrets.** `RESTART_TOKEN` is redacted in dry-run output (`Bearer ***`).
- **Version lives in one place.** `frontend/package.json` `version` is the semver source of truth; the script propagates it to images, Helm, and git tags. Full rules: [versioning-release.md](versioning-release.md).
- **Shellcheck-clean.** Add `.shellcheckrc` for known false positives (e.g. sourced `.env`).

## What it does

| Concern | Handled by |
|---------|------------|
| Docker build + tag + push | `build` command |
| Build metadata (version, commit, time) | `--build-arg` passed to Dockerfiles |
| Kubernetes rollout | `kubectl rollout restart` or restart hook |
| Semver release | `release` command → commit, tag `v*`, push |
| Helm chart publish | `helm-push` command (OCI → ghcr.io) |
| Local Go backend dev | `start` / `stop` / `status` / `logs` / `test` |
| CI deploy | `.github/workflows/build.yml` invokes `./oglimmer.sh build …` |

Cross-links: [docker.md](docker.md) (Dockerfiles), [github-actions.md](github-actions.md) (`build.yml`), [helm.md](helm.md) (chart versions), [go-backend.md](go-backend.md) (local `start`).

## Two script shapes

Most repos copy one of these templates and customize the names.

### A. Full-stack (Vue + Go + Helm)

Default shape for Vue + Go (or similar two-component) full-stack apps.

```
./oglimmer.sh [COMMAND] [OPTIONS]
```

**Commands:**

| Command | Purpose |
|---------|---------|
| `build` (default) | Build/push selected components, restart deployments |
| `release` | Interactive or `--bump` semver, commit, tag, push, `helm-push` |
| `show` | Print current version from `package.json` |
| `helm-push` | Package and push OCI Helm chart to ghcr.io |
| `start` / `stop` / `status` / `logs` | Local Go backend process |
| `test` | `go test ./...` in `backend/` |

**Component flags:** `-f` / `--frontend`, `-b` / `--backend`, `-a` / `--all` (default when none specified).

**Customize at the top of the file:**

```bash
DEFAULT_REGISTRIES=("ghcr.io/oglimmer")          # or registry.oglimmer.com
DEFAULT_FRONTEND_DEPLOYMENT="<release>-frontend"
DEFAULT_BACKEND_DEPLOYMENT="<release>-backend"
RESTART_NAMESPACE="<k8s-namespace>"
# Image names derived as: $registry/<repo>-frontend, $registry/<repo>-backend
```

### B. Multi-component

For repos with three or more buildable components (e.g. frontend + backend + worker).

```
./oglimmer.sh build [OPTIONS]
```

**Component flags:** per-component (`-f`, `-b`, `-s`, …) or `--all`. At least one flag required.

**Customize with helper functions:**

```bash
get_image_tag()      # component → registry/image:tag
get_build_args()     # component → docker --build-arg string
get_deployments()    # component → one or more K8s deployment names
```

One component can restart **multiple deployments** (e.g. scraper → `news-scraper` + `news-taggroupper`).

Uses `K8S_NAMESPACE` instead of `RESTART_NAMESPACE`. Often adds `-B` / `--build-only` and `-p` / `--push-only` for split phases.

## Standard options (full-stack)

| Flag | Env var | Default | Purpose |
|------|---------|---------|---------|
| `-v` / `--verbose` | `VERBOSE` | `false` | Show docker/curl output |
| `--dry-run` | `DRY_RUN` | `false` | Print commands without running |
| `--no-push` | `PUSH=false` | push on | Build locally only |
| `-n` / `--no-restart` | `RESTART=false` | restart on | Skip rollout (keel variant) |
| `--registries "a,b"` | `DEFAULT_REGISTRIES_ENV` | repo default | Multi-registry push |
| `--platform PLATFORM` | `PLATFORM`, `DOCKER_PLATFORM` | `arm64`* | See platform table |
| `--frontend-deploy NAME` | `FRONTEND_DEPLOYMENT` | repo default | K8s deployment name |
| `--backend-deploy NAME` | `BACKEND_DEPLOYMENT` | repo default | K8s deployment name |
| `--bump major\|minor\|bugfix` | — | interactive | Non-interactive release |

\* **CI must pass `--platform auto` explicitly** — see [Platform](#platform-build-mode) below.

If `--no-push` is set with restart enabled, the script auto-disables restart (nothing new to roll out).

## Platform build mode

| `--platform` | Docker behaviour | Use when |
|--------------|------------------|----------|
| `auto` | Plain `docker build` + `docker push` (no buildx driver) | **ARC CI** (arm64 dind) |
| `arm64` | buildx with `--platform linux/arm64` | Local cross-build to arm64 |
| `amd64` | buildx with `--platform linux/amd64` | Local cross-build to amd64 |
| `multi` | buildx multi-arch push | Release workflow (`release.yml`), not ARC |

On the in-cluster ARC runner, buildx's docker-container driver loses its push session against the registry ingress (`no active session … context deadline exceeded`). `auto` avoids buildx entirely.

**Rule:** `build.yml` always passes `--platform auto --verbose`, regardless of the script's `PLATFORM` default.

## Build metadata

Read once per build from `frontend/package.json` + git:

```bash
app_version=$(grep '"version"' frontend/package.json | …)
git_commit=$(git rev-parse --short HEAD)
build_time=$(date -u +%Y-%m-%dT%H:%M:%SZ)
```

Pass to Docker:

| Component | Build args |
|-----------|------------|
| Frontend | `VITE_APP_VERSION`, `VITE_GIT_COMMIT`, `VITE_BUILD_TIME` |
| Backend | `VERSION`, `GIT_COMMIT`, `BUILD_TIME` |

Must match what the Dockerfiles expect ([docker.md](docker.md)).

## Restart behaviour

Rollout is triggered after a successful push for each built component.

```
restart_deployment <deployment-name>
  ├─ kubectl available?  → kubectl rollout restart deployment/<name> -n <namespace>
  └─ else RESTART_TOKEN? → POST <RESTART_HOOK_URL>/<namespace>/<deployment>
                           Authorization: Bearer $RESTART_TOKEN
```

Defaults:

```bash
RESTART_HOOK_URL="${RESTART_HOOK_URL:-https://restart.oglimmer.com/restart}"
RESTART_NAMESPACE="${RESTART_NAMESPACE:-<app-namespace>}"   # some older scripts use K8S_NAMESPACE — same hook path
```

Normalize on `RESTART_NAMESPACE` in new scripts; older repos may still use `K8S_NAMESPACE` ([assessments/oglimmer-sh.md](assessments/oglimmer-sh.md)).

**Prerequisite check** (before build):

```bash
if restart requested && ! kubectl && ! RESTART_TOKEN; then
  exit 1  # "Install kubectl, set RESTART_TOKEN, or pass --no-restart"
fi
```

| Environment | Restart path |
|-------------|--------------|
| Developer laptop with kubeconfig | kubectl |
| ARC CI runner | restart hook + `RESTART_TOKEN` secret |
| Keel-watched registry | `--no-restart` in `build.yml`; keel rolls out on new digest |

Never log or dry-run-print the actual token.

## Release flow (`release` command)

**Full GitOps release** (ghcr.io apps with `release.yml` + `helm-push`):

1. Read current version from `frontend/package.json`.
2. Bump semver (interactive `select` or `--bump major|minor|bugfix`).
3. `npm version <new> --no-git-tag-version` in `frontend/`.
4. Update `helm/<chart>/Chart.yaml` `version` and `appVersion`.
5. `git add` changed files → `git commit -m "Release vX.Y.Z"` → `git tag -a vX.Y.Z`.
6. `git push origin HEAD` + `git push origin vX.Y.Z`.
7. Tag push triggers `release.yml` (multi-arch images on hosted runner).
8. `helm-push` — package chart, `helm registry login` via `gh auth token`, push to `oci://ghcr.io/oglimmer`.

**Partial variant:** commit, tag, `execute_build` only — no `git push`, no `release.yml`, no `helm-push` (intentional for some private apps).

**No release command:** shape B multi-component scripts and single-image shape C often omit `release`.

Never hand-edit versions in `Chart.yaml` and `package.json` separately — use the scripted release where implemented.

Release variants by repo: [assessments/repo-map.md](assessments/repo-map.md).

## `helm-push`

```bash
helm package helm/<chart> -d <tmpdir>
helm push <chart>-<version>.tgz oci://ghcr.io/oglimmer
```

Authenticate with `gh auth token | helm registry login ghcr.io`. Chart version read from `Chart.yaml`.

## Local dev commands (Go backends)

| Command | Behaviour |
|---------|-----------|
| `start` | `ensure_backend_env` → `go build` → `nohup` to `/tmp/<app>.pid` |
| `stop` | Kill PID from pidfile |
| `status` | Check pidfile |
| `logs` | `tail -f /tmp/<app>.log` |
| `test` | `cd backend && go test ./...` |

`ensure_backend_env` creates `backend/.env` on first run with `JWT_SECRET=$(openssl rand -hex 32)` and `AUTH_MODE=password` — only when the file is missing. Document that Postgres must still be running (`compose up`).

`load_backend_env` sources `backend/.env` before start. Shellcheck: disable `SC1091` in `.shellcheckrc` for the sourced file.

## CI integration

`build.yml` pattern ([github-actions.md](github-actions.md)):

```yaml
- name: Log in to ghcr.io
  uses: docker/login-action@…
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build, push & restart backend
  env:
    RESTART_TOKEN: ${{ secrets.RESTART_TOKEN }}
  run: ./oglimmer.sh build -b --platform auto --verbose
```

For `registry.oglimmer.com` repos, login uses `REGISTRY_USERNAME` / `REGISTRY_PASSWORD` secrets instead.

Include `oglimmer.sh` in every `paths-filter` component block so script changes rebuild all images.

**GitHub repo secrets** (from cluster — see `/setup-arc-build`):

| Secret | Purpose |
|--------|---------|
| `RESTART_TOKEN` | Restart hook bearer token |
| `REGISTRY_USERNAME` / `REGISTRY_PASSWORD` | Private registry login (non-ghcr.io) |
| `GITHUB_TOKEN` | ghcr.io login (automatic with `packages: write`) |

## Script structure (full-stack template)

Every full-stack `oglimmer.sh` follows this skeleton:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. SCRIPT_DIR, defaults (registries, deployment names, dirs)
# 2. Option defaults (BUILD_*, VERBOSE, DRY_RUN, PUSH, RESTART, PLATFORM)
# 3. Logging helpers (log_info, log_error, log_verbose)
# 4. execute_cmd() — honours DRY_RUN and VERBOSE
# 5. show_help()
# 6. parse_args() — command dispatch + flag parsing + image array build
# 7. check_prerequisites() — docker, restart creds, release tools, buildx for multi
# 8. build_image() — platform routing (buildx vs plain docker)
# 9. restart_deployment() / restart_via_kubectl() / restart_via_hook()
# 10. execute_build() — metadata, build components, restart
# 11. execute_release() — bump, commit, tag, push, helm-push
# 12. Dev commands (start/stop/status/logs/test/helm-push)
# 13. main() — parse → help → dev → prerequisites → build|release
```

Keep `main` thin. Put repo-specific names only in the defaults block and image-name derivation — not scattered through build logic.

## Registries

| Registry | Use when | Auth in CI |
|----------|----------|------------|
| `ghcr.io/oglimmer` | Public OSS-style apps on GitHub | `GITHUB_TOKEN` |
| `registry.oglimmer.com` | Private cluster apps | `REGISTRY_USERNAME` / `REGISTRY_PASSWORD` |

Image naming convention: `<registry>/<project>-frontend` and `<registry>/<project>-backend`. Some repos shorten suffixes (`-fe`/`-be`) or use component-specific names — document in the script defaults.

## New-repo checklist

1. Copy `oglimmer.sh` from an existing org repo with the closest shape (see [assessments/repo-map.md](assessments/repo-map.md)).
2. Set `DEFAULT_*_DEPLOYMENT`, `RESTART_NAMESPACE`, registry, and image name suffixes.
3. Add component flags if the repo has more than frontend + backend.
4. Implement restart hook path (`restart_via_hook`) if not already present.
5. Wire `check_prerequisites` to accept kubectl **or** `RESTART_TOKEN`.
6. Add `.shellcheckrc` if sourcing `.env`.
7. Add `shellcheck` to pre-commit ([pre-commit.md](pre-commit.md)).
8. Add `.github/workflows/build.yml` with `--platform auto` ([github-actions.md](github-actions.md)).
9. Set GitHub secrets from cluster (`/setup-arc-build`).
10. Smoke-test: `./oglimmer.sh build -a --dry-run` locally; `gh workflow run build.yml` in CI.

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Require kubectl unconditionally | kubectl **or** `RESTART_TOKEN` |
| Echo `RESTART_TOKEN` in logs/dry-run | Redact as `Bearer ***` |
| Default `PLATFORM=arm64` in CI without override | `build.yml` passes `--platform auto` |
| Use buildx on ARC for everyday pushes | Plain docker via `--platform auto` |
| Hand-bump `package.json` and `Chart.yaml` | `./oglimmer.sh release` |
| Skip restart after push in CI without keel | Restart hook or explicit keel setup |
| Hard-code registry in `build.yml` | Login in workflow; registry in `oglimmer.sh` defaults |
| `docker build` without build-args | Pass version/commit/time every build |
| Forget `oglimmer.sh` in paths-filter | Include in every component filter |
| Duplicate docker build logic in CI | Single entry point: `./oglimmer.sh build` |
| Leave `set +e` / missing `pipefail` | `set -euo pipefail` at top |