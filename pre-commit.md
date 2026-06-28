# Pre-commit

Local git hooks catch problems before they reach CI. Fast, auto-fixing checks run on every commit; slow build and test gates run on push.

Per-repo adoption: [assessments/pre-commit.md](assessments/pre-commit.md). Path mapping for non-standard layouts: [assessments/repo-map.md](assessments/repo-map.md).

## Philosophy

- **Mirror CI, don't duplicate it entirely.** Commit hooks run the cheap half of `ci.yml`; pre-push runs the slow gates (build, test).
- **Auto-fix when possible.** `gofmt -w`, `eslint --fix`, trailing-whitespace — fix in place, fail the hook if files changed, then re-stage.
- **Use the project's toolchain.** Go, npm, and helm hooks use `language: system` / `repo: local`, so they pick up the repo's exact versions and config.
- **Pin upstream hook repos.** Set `rev:` on every third-party `repo:` entry.
- **Exclude generated and binary files.** Text-munging hooks must not touch lockfiles, encrypted files, or Helm templates.

## Setup

```bash
pre-commit install --install-hooks            # commit-stage hooks
pre-commit install --hook-type pre-push       # build/test gates
pre-commit run --all-files                    # smoke-test everything
```

Declare both hook types in config:

```yaml
default_install_hook_types: [pre-commit, pre-push]
```

## Standard layout

```
.pre-commit-config.yaml    # hook definitions
.yamllint.yaml             # relaxed rules for YAML (GitHub Actions uses `on:`)
.shellcheckrc              # shellcheck overrides for oglimmer.sh etc.
```

Pair with [github-actions.md](github-actions.md) — actionlint and yamllint cover the same files CI runs.

## Exclude patterns

Top-level `exclude` regex for files hooks must never touch:

```yaml
exclude: |
  (?x)^(
    .*\.cpt$|                              # ccrypt-encrypted files
    .*\.(png|jpe?g|gif|ico|woff2?|ttf|eot)$|
    frontend/package-lock\.json$|          # tool-managed lockfile
    backend/go\.sum$                       # tool-managed checksums
  )$
```

Helm `templates/` are Go-templated YAML, so exclude them from `check-yaml` and `yamllint`. `helm lint` covers the chart instead.

## Hook tiers

### On `commit` — fast hygiene (all repos)

From `pre-commit-hooks` (pin `rev`, e.g. `v6.0.0`):

| Hook | Purpose |
|------|---------|
| `trailing-whitespace` | Strip stray spaces; `--markdown-linebreak-ext=md` preserves MD line breaks |
| `end-of-file-fixer` | Ensure newline at EOF |
| `mixed-line-ending` | Normalize to LF (`--fix=lf`) |
| `check-merge-conflict` | Catch `<<<<<<<` markers |
| `check-case-conflict` | Case-only filename clashes |
| `check-added-large-files` | Block blobs > 1 MB (`--maxkb=1024`) |
| `detect-private-key` | Reject committed PEM keys |
| `check-json` | Valid JSON |
| `check-yaml` | Valid YAML (`--allow-multiple-documents`); exclude Helm templates |

### On `commit` — YAML & workflows (repos with `.github/`)

**yamllint** — `rev: v1.38.0`, args `[--strict, -c, .yamllint.yaml]`:

```yaml
# .yamllint.yaml
extends: relaxed
rules:
  line-length: disable
  document-start: disable
  truthy:
    check-keys: false    # GitHub Actions `on:` is not a boolean
```

**actionlint** — `rev: v1.7.7`. Declare custom ARC runner labels in `.github/actionlint.yaml`.

### On `commit` — shell & secrets

| Hook | Source | Notes |
|------|--------|-------|
| `shellcheck` | `shellcheck-py` | Lints `oglimmer.sh`, hook scripts; `.shellcheckrc` for known false positives |
| `gitleaks` | `gitleaks` | Defence in depth alongside `detect-private-key` |

### On `commit` — Go backend (`repo: local`)

```yaml
- id: gofmt
  entry: gofmt -s -w
  language: system
  files: ^backend/.*\.go$

- id: go-vet
  entry: bash -c 'cd backend && go vet ./...'
  language: system
  files: ^backend/.*\.go$
  pass_filenames: false
```

`gofmt -s -w` rewrites in place. If the hook changes files, the commit fails — re-stage the formatted code and commit again.


### On `commit` — Vue/TS frontend (`repo: local`)

```yaml
- id: eslint
  entry: bash -c 'cd frontend && npm run lint:fix'
  files: ^frontend/.*\.(ts|tsx|js|jsx|vue)$

- id: vue-tsc
  entry: bash -c 'cd frontend && npm run typecheck'
  files: ^frontend/.*\.(ts|tsx|vue)$
```

ESLint runs on commit with fix enabled. Typecheck can run on commit or pre-push depending on speed — prefer pre-push when `vue-tsc` is slow.

### On `pre-push` — slow gates

Mark with `stages: [pre-push]`:

| Hook | Command |
|------|---------|
| `go-build` | `cd backend && go build ./...` |
| `go-test` | `cd backend && go test ./...` (DB tests self-skip without env var) |
| `vue-tsc` | `cd frontend && npm run typecheck` |
| `vitest` | `cd frontend && npm run test` |
| `helm-lint` | `helm lint helm/<chart>` |

Pre-push mirrors the full `ci.yml` pipeline, so a green push usually means a green CI run.

### On `commit` — Helm

```yaml
- id: helm-lint
  entry: helm lint helm/<chart-name>
  files: ^helm/<chart-name>/
  pass_filenames: false
```

## Multi-component repos

When a repo ships more than one deployable surface (`frontend/` + `backend/` + `admin/`, or `frontend/` + `frontend2/` + Java `backend/`), define **one hook block per component**, each with its own `files:` prefix and `working-directory`:

```yaml
# Frontend
- id: frontend-lint
  entry: bash -c 'cd frontend && npm run lint'
  files: ^frontend/
  pass_filenames: false

# Backend (Go)
- id: backend-test
  entry: bash -c 'cd backend && go test ./...'
  files: ^backend/
  pass_filenames: false

# Second frontend (e.g. Pixi client)
- id: frontend2-type-check
  entry: bash -c 'cd frontend2 && npm run type-check'
  files: ^frontend2/
  pass_filenames: false
```

Mirror each component's CI job step order (lint → typecheck → test → build). A change under `admin/` must not run `frontend/` hooks, and vice versa — `files:` prefixes enforce this locally the same way `paths:` filters do in CI.

If CI is split into per-component workflow files, pre-commit blocks should still map 1:1 to those files' check steps.

## Minimal config (smaller repos)

Not every repo needs the full stack on day one. A minimal config (gofmt, vet, build, eslint, vue-tsc all on commit) is fine for a smaller repo with fast tests. Move to the commit/pre-push split as the test suite grows.

Always keep hygiene hooks, secret scanning, and whatever matches your `ci.yml`.

## New-repo checklist

1. Copy `.pre-commit-config.yaml` from a sibling repo with the same stack.
2. Adjust `files:` patterns and `working-directory` paths to match layout.
3. Add `.yamllint.yaml` and `.github/actionlint.yaml` if the repo has workflows.
4. Add `.shellcheckrc` if the repo has `oglimmer.sh`.
5. Run `pre-commit install --install-hooks` and `pre-commit install --hook-type pre-push`.
6. Run `pre-commit run --all-files` and fix anything that fails.
7. Confirm hook coverage matches [github-actions.md](github-actions.md) `ci.yml` steps.

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Run full test suite on every commit | `stages: [pre-push]` for slow hooks |
| Use `npm install` in hooks | Project scripts (`npm run lint`) assume deps are installed |
| Lint Helm templates with `check-yaml` | Exclude `helm/**/templates/`; use `helm lint` |
| Leave hooks unpinned (`rev: master`) | Pin `rev` to a tag or SHA |
| Skip `pre-commit install` in onboarding | Document setup in the repo README |
| Duplicate CI logic differently | Mirror `ci.yml` step order and commands |