---
name: coding-guidelines
description: >-
  Apply, review against, or scaffold from the house coding guidelines (14 concept
  docs + assessments of 16 audited repos) that live in ~/dev/coding-guidelines.
  Use when the user asks to follow "our/house conventions", build or review Go /
  Java Spring / Vue / Nuxt code, set up Docker / Helm / GitHub Actions / Renovate /
  pre-commit / observability / MCP / oglimmer.sh / versioning, or asks which
  guideline applies to a repo. Routes to the right doc(s) instead of dumping all of them.
trigger: /coding-guidelines
---

# /coding-guidelines

The canonical conventions for how we build software live as one markdown file per
concept in **`~/dev/coding-guidelines/`** (this skill's home repo). Adoption, path
mappings, gaps, and copy-from repos live in **`~/dev/coding-guidelines/assessments/`**.

Do **not** read all 14 docs. Figure out what the task touches, read only those, and
— when working inside one of the 16 audited repos — layer in that repo's path mapping
and known gaps. That routing is the whole job of this skill.

## The three questions (answer them first, in order)

1. **What are we doing?** Scaffolding new · adding to existing · reviewing/auditing ·
   answering "which guideline applies / how do we do X".
2. **Which repo?** Match `basename $PWD` against the Known repos table below. If it's a
   known repo, its path names and gaps override the generic guidance. If it's new/unknown,
   detect the stack (step 3) and treat these docs as the target to build toward.
3. **Which layers?** Detect stack markers, then read only the matching guideline docs.

Guidelines state **what we build toward**. When editing an existing repo, prefer matching
that repo's current style over imposing the target wholesale — surface the gap, don't
silently rewrite. Reference-repo copy sources are listed per topic in
`assessments/README.md`.

## Stack detection

Run this at the repo root to see which layers are present, then read only those docs.

```bash
G=~/dev/coding-guidelines
echo "repo: $(basename "$PWD")"
[ -f go.mod ] || [ -d backend-go ] || ls */go.mod >/dev/null 2>&1 && echo "→ go-backend.md"
ls **/pom.xml */build.gradle* 2>/dev/null | head -1 | grep -q . && echo "→ java-spring-backend.md"
[ -f nuxt.config.ts ] || ls */nuxt.config.* >/dev/null 2>&1 && echo "→ nuxt-frontend.md"
{ [ -f vite.config.ts ] || ls */vite.config.* >/dev/null 2>&1; } && echo "→ vue-frontend.md (confirm Vue, not Svelte)"
ls **/Dockerfile Dockerfile 2>/dev/null | head -1 | grep -q . && echo "→ docker.md + oglimmer-sh.md"
[ -d helm ] || ls **/Chart.yaml >/dev/null 2>&1 && echo "→ helm.md"
[ -f oglimmer.sh ] && echo "→ oglimmer-sh.md"
[ -d .github/workflows ] && echo "→ github-actions.md"
[ -f renovate.json ] || [ -f .github/renovate.json ] && echo "→ renovate.md"
[ -f .pre-commit-config.yaml ] && echo "→ pre-commit.md"
grep -rqli 'modelcontextprotocol\|mcp' --include=go.mod --include=pom.xml . 2>/dev/null && echo "→ mcp.md (confirm it's an MCP server)"
grep -rqli 'prometheus\|micrometer\|/metrics' . 2>/dev/null && echo "→ observability.md"
```

Note: **MySQL/MariaDB Go services** (linky, easy-host-k8s, coffee-diary) diverge from
`go-backend.md`'s Postgres/pgx assumptions — read the doc but expect sqlx/database driver
differences. **Svelte** (yt-infographics) has no frontend guideline — skip `vue-frontend.md`.

## Topic → doc map

| Concern | Doc | Read when |
|---|---|---|
| Go HTTP service (chi, pgx, migrations, auth, jobs) | `go-backend.md` | `go.mod` present |
| Spring Boot API | `java-spring-backend.md` | `pom.xml`/`build.gradle` present |
| Vue 3 SPA (Vite, Pinia, fetch client, styling) | `vue-frontend.md` | `vite.config` + Vue |
| Nuxt 4 (A: static site · B: Spring SPA) | `nuxt-frontend.md` | `nuxt.config` present |
| MCP server layered on a backend | `mcp.md` | Repo exposes MCP tools |
| Container images (multi-stage, pinned bases, `.dockerignore`) | `docker.md` | Builds images |
| Kubernetes deploy (chart layout, `helm/argocd/`, sealed secrets) | `helm.md` | Deploys to k8s |
| Prometheus metrics into the cluster stack | `observability.md` | Exposes `/metrics` |
| CI/CD quintet (`ci`, `build`, `release`, `cleanup-images`, `actionlint`) | `github-actions.md` | `.github/workflows/` |
| Dependency automation | `renovate.md` | Has dependencies (see also `/renovate-config`) |
| Pre-commit trio (`.pre-commit-config`, `.yamllint`, `.shellcheckrc`) | `pre-commit.md` | Vue+Go+Helm full stack |
| Tests (Go/Java/npm unit, Playwright e2e) | `testing.md` | Ships tests |
| `oglimmer.sh` build/deploy/restart shape | `oglimmer-sh.md` | Has `oglimmer.sh` (see also `/setup-arc-build`) |
| Version bumps, tags, release artifacts | `versioning-release.md` | Ships versioned artifacts |

Start any stack-routing question at `README.md` (Stack routing table) or
`assessments/repo-map.md` (per-repo path mapping).

## Known repos (path mapping + gaps override the generic docs)

When `basename $PWD` matches one of these, read its row in
`assessments/repo-map.md` (paths, DB, registry, deploy) **and** its column in the
`assessments/README.md` adoption matrix (⚠️/❌ = a known gap to respect or fix).

`irl-planner-pro` · `plugin-skill-hosting` · `trivia` · `linky` · `yt-infographics` ·
`easy-host-k8s` · `deep-digest-rss` · `start-renovate` · `homepage-v4` · `coffee-diary` ·
`boardwalk-billionaire` · `cybernight` · `picz` · `picz2` · `video-msg` · `status-tacos`

Fastest full-stack reference: **`irl-planner-pro`** (only repo matching every guideline).
Closest Vue+Go sibling to copy from: **`plugin-skill-hosting`**. Full copy-source table:
`assessments/README.md` → "Reference repos by topic".

## Sibling skills (hand off, don't reimplement)

- **`/renovate-config`** — generate & validate a `renovate.json`.
- **`/setup-arc-build`** — onboard a repo to the local self-hosted ARC (build.yml, runner
  scale set, `oglimmer.sh` restart hook, secrets).
- **`go-vue-fullstack`** skill — deeper Go+Vue architecture patterns with the *why*.

## When editing this guidelines repo itself

If the task is to change a guideline (not apply one): the concept doc and its
`assessments/<same-name>.md` are a pair — update both, and refresh the adoption matrix /
known-gaps list in `assessments/README.md` if the change affects what repos match. Keep
concept docs repo-agnostic; keep repo-specific facts in `assessments/`. `assessments/`
files carry a "Last verified" date — bump it when you re-check against disk.
