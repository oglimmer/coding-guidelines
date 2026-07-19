---
name: coding-guidelines
description: >-
  Apply, review against, or scaffold from the house coding guidelines (16 concept
  docs, bundled with this skill) covering Go / Java Spring / Vue / Nuxt services,
  Postgres from Go or Spring, Docker, Helm, GitHub Actions, Renovate, pre-commit,
  observability, MCP, oglimmer.sh, testing and versioning. Use when the user asks
  to follow "our/house conventions", to build or review code in those stacks, or
  asks which guideline applies to a repo. Routes to the right doc(s) instead of
  dumping all of them.
trigger: /coding-guidelines
---

# /coding-guidelines

The conventions are **bundled with this skill** as one markdown file per concept under
`references/`, so they work on any host with no checkout required. Read them with paths
relative to this file — e.g. `references/go-backend.md`.

Do **not** read all 16 docs. Work out what the task touches, read only those. That
routing is the whole job of this skill.

## The three questions (answer them first, in order)

1. **What are we doing?** Scaffolding new · adding to existing · reviewing/auditing ·
   answering "which guideline applies / how do we do X".
2. **Which layers?** Detect stack markers (below), then read only the matching docs.
3. **Is this a known repo?** If `~/dev/coding-guidelines/assessments/` exists locally,
   check it for this repo's path mapping and known gaps — those override the generic
   guidance. It is not bundled here (it is an internal inventory); skip this step when
   the directory is absent.

Guidelines state **what we build toward**. When editing an existing repo, prefer matching
that repo's current style over imposing the target wholesale — surface the gap, don't
silently rewrite.

## Stack detection

Run at the repo root to see which layers are present, then read only those docs.

```bash
echo "repo: $(basename "$PWD")"
[ -f go.mod ] || [ -d backend-go ] || ls */go.mod >/dev/null 2>&1 && echo "→ references/go-backend.md"
ls **/pom.xml */build.gradle* 2>/dev/null | head -1 | grep -q . && echo "→ references/java-spring-backend.md"
[ -f nuxt.config.ts ] || ls */nuxt.config.* >/dev/null 2>&1 && echo "→ references/nuxt-frontend.md"
{ [ -f vite.config.ts ] || ls */vite.config.* >/dev/null 2>&1; } && echo "→ references/vue-frontend.md (confirm Vue, not Svelte)"
ls **/Dockerfile Dockerfile 2>/dev/null | head -1 | grep -q . && echo "→ references/docker.md + references/oglimmer-sh.md"
[ -d helm ] || ls **/Chart.yaml >/dev/null 2>&1 && echo "→ references/helm.md"
[ -f oglimmer.sh ] && echo "→ references/oglimmer-sh.md"
[ -d .github/workflows ] && echo "→ references/github-actions.md"
[ -f renovate.json ] || [ -f .github/renovate.json ] && echo "→ references/renovate.md"
[ -f .pre-commit-config.yaml ] && echo "→ references/pre-commit.md"
grep -rqli 'modelcontextprotocol\|mcp' --include=go.mod --include=pom.xml . 2>/dev/null && echo "→ references/mcp.md (confirm it's an MCP server)"
grep -rqli 'prometheus\|micrometer\|/metrics' . 2>/dev/null && echo "→ references/observability.md"
grep -rqls --include=go.mod 'jackc/pgx\|lib/pq' . && echo "→ references/postgres-for-golang.md"
grep -rqls --include=pom.xml --include='build.gradle*' 'org.postgresql' . && echo "→ references/postgres-for-spring.md"
```

Note: **MySQL/MariaDB Go services** diverge from `go-backend.md`'s Postgres/pgx
assumptions — read the doc but expect sqlx/driver differences, and skip
`postgres-for-golang.md` entirely. **Svelte** has no frontend guideline — skip
`vue-frontend.md`.

## Topic → doc map

All paths are relative to this skill's directory.

| Concern | Doc | Read when |
|---|---|---|
| Go HTTP service (chi, pgx, migrations, auth, jobs) | `references/go-backend.md` | `go.mod` present |
| Spring Boot API | `references/java-spring-backend.md` | `pom.xml`/`build.gradle` present |
| Postgres from Go (JSONB, FTS, upsert, SKIP LOCKED, pgxpool) | `references/postgres-for-golang.md` | Go repo + Postgres |
| Postgres from Spring (Hibernate 6 JSON, upsert, Flyway, Hikari) | `references/postgres-for-spring.md` | Spring repo + Postgres |
| Vue 3 SPA (Vite, Pinia, fetch client, styling) | `references/vue-frontend.md` | `vite.config` + Vue |
| Nuxt 4 (A: static site · B: Spring SPA) | `references/nuxt-frontend.md` | `nuxt.config` present |
| MCP server layered on a backend | `references/mcp.md` | Repo exposes MCP tools |
| Container images (multi-stage, pinned bases, `.dockerignore`) | `references/docker.md` | Builds images |
| Kubernetes deploy (chart layout, `helm/argocd/`, sealed secrets) | `references/helm.md` | Deploys to k8s |
| Prometheus metrics into the cluster stack | `references/observability.md` | Exposes `/metrics` |
| CI/CD quintet (`ci`, `build`, `release`, `cleanup-images`, `actionlint`) | `references/github-actions.md` | `.github/workflows/` |
| Dependency automation | `references/renovate.md` | Has dependencies (see also `/renovate-config`) |
| Pre-commit trio (`.pre-commit-config`, `.yamllint`, `.shellcheckrc`) | `references/pre-commit.md` | Vue+Go+Helm full stack |
| Tests (Go/Java/npm unit, Playwright e2e) | `references/testing.md` | Ships tests |
| `oglimmer.sh` build/deploy/restart shape | `references/oglimmer-sh.md` | Has `oglimmer.sh` (see also `/setup-arc-build`) |
| Version bumps, tags, release artifacts | `references/versioning-release.md` | Ships versioned artifacts |

`references/README.md` carries the stack-routing table (which docs combine for a given
stack) — start there for "what do I read for a Vue + Go + Helm app".

## Postgres: the two environments

Go and Spring services run against a **direct Postgres** in the k3s cluster and behind a
**transaction-mode PgBouncer** in the company environment, from one build. Both Postgres
docs open with a `§0` stating the rule: write PgBouncer-compatible code, never require
PgBouncer. Set the pooler-safe options unconditionally (Go: `QueryExecModeExec` plus
disabled statement/description caches on `cfg.ConnConfig`; Spring: `prepareThreshold=0`) —
they are inert on a direct connection. Session-scoped state is the whole incompatibility
surface: `LISTEN`, session advisory locks, bare `SET`, and session temp tables are
unavailable to any service that must run in both. Never deploy PgBouncer to k3s to "match"
the company setup — compatibility is a property of the code, not the deployment.

## Cross-references inside the docs

The bundled docs link to each other by bare filename (`[go-backend.md](go-backend.md)`),
which resolves correctly inside `references/`. They also link to `assessments/<name>.md`
for per-repo adoption detail — **those are not bundled** and resolve only on a host with
the `~/dev/coding-guidelines` checkout. Treat a missing `assessments/` as "no repo-specific
overrides available", not an error.

## Known repos (only with the local checkout)

When `~/dev/coding-guidelines/assessments/` is present and `basename $PWD` matches one of
these, read its row in `assessments/repo-map.md` (paths, DB, registry, deploy) **and** its
column in the `assessments/README.md` adoption matrix (⚠️/❌ = a known gap to respect or fix).

`irl-planner-pro` · `plugin-skill-hosting` · `trivia` · `linky` · `yt-infographics` ·
`easy-host-k8s` · `deep-digest-rss` · `start-renovate` · `homepage-v4` · `coffee-diary` ·
`boardwalk-billionaire` · `cybernight` · `picz` · `picz2` · `video-msg` · `status-tacos`

Fastest full-stack reference: **`irl-planner-pro`** (only repo matching every guideline).
Closest Vue+Go sibling to copy from: **`plugin-skill-hosting`**.

## Sibling skills (hand off, don't reimplement)

- **`/renovate-config`** — generate & validate a `renovate.json`.
- **`/setup-arc-build`** — onboard a repo to the local self-hosted ARC (build.yml, runner
  scale set, `oglimmer.sh` restart hook, secrets).
- **`go-vue-fullstack`** skill — deeper Go+Vue architecture patterns with the *why*.

## When editing the guidelines themselves

The source of truth is the `~/dev/coding-guidelines` repo; `references/` here is a
published copy. Change the doc there, then re-upload it to this skill so the two do not
drift. A concept doc and its `assessments/<same-name>.md` are a pair — update both, and
refresh the adoption matrix in `assessments/README.md` if the change affects what repos
match. Keep concept docs repo-agnostic; keep repo-specific facts in `assessments/`.
`assessments/` files carry a "Last verified" date — bump it when you re-check against disk.
