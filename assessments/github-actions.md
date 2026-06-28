# GitHub Actions — project assessment

Per-repo adoption for [../github-actions.md](../github-actions.md). Last verified: June 2026.

Audited: `irl-planner-pro`, `plugin-skill-hosting`, `trivia`, `linky`, `yt-infographics`, `easy-host-k8s`, `deep-digest-rss`, `start-renovate`, `homepage-v4`, `coffee-diary`, `boardwalk-billionaire`, `cybernight`, `picz`, `picz2`, `video-msg`, `status-tacos`.

| Repo | ci / PR gate | build (ARC) | release | cleanup | actionlint | Notes |
|------|--------------|-------------|---------|---------|------------|-------|
| irl-planner-pro | ✅ `ci.yml` | ✅ SHA pins | ✅ | ✅ | ✅ | Closest full match |
| plugin-skill-hosting | ✅ + e2e | ✅ `@v6` tags | ✅ | ✅ | ❌ | Extra Playwright job; pin SHAs when touching |
| trivia | ✅ + golangci-lint | ✅ | ❌ | ❌ | ❌ | `permissions` omits `packages: write` — OK for `registry.oglimmer.com` + login secrets; no Postgres `services:` in CI |
| linky | ✅ | ✅ | ❌ | ❌ | ❌ | Filters use `server/` + `client/` not `backend/` + `frontend/`; older `@v3`–`@v5` action pins |
| yt-infographics | ✅ | ✅ SHA in build | ❌ | ❌ | ❌ | Go + Svelte filters |
| easy-host-k8s | ❌ | 🔀 hosted runner | ❌ | ❌ | ❌ | **Broken:** `build.yml` runs Maven in `backend/` — directory removed; app is `backend-go/`. Docker on `ubuntu-latest`, not ARC + `oglimmer.sh` |
| deep-digest-rss | 🔀 `pr.yml` PR-only | ✅ 3 components | ❌ | ❌ | ❌ | PR workflow name differs; filters `news-backend`, `news-frontend`, `scraper` |
| start-renovate | ✅ Java + Nuxt | ✅ | ❌ | ❌ | ❌ | Not Vue+Go path filters |
| homepage-v4 | ✅ Nuxt `ci.yml` | ❌ | ❌ | ❌ | ❌ | Lint + typecheck + build only; no ARC `build.yml` |
| coffee-diary | ❌ | ❌ | ❌ | ❌ | ❌ | **No `.github/workflows/`** — README references missing `build.yml` |
| boardwalk-billionaire | ✅ `ci.yaml` | ✅ SHA pins + hook | ❌ | ❌ | ❌ | Filters `client/` + `server/`; optional rename `ci.yaml` → `ci.yml` |
| cybernight | ❌ | ❌ | ❌ | ❌ | ❌ | **No `.github/workflows/`** — add `ci.yml` + ARC `build.yml` |
| picz | ❌ | ❌ | ❌ | ❌ | ❌ | Legacy AWS deploy; no CI |
| picz2 | ❌ | ❌ | ❌ | ❌ | ❌ | **No workflows** — add `ci.yml` + `arc-picz2` `build.yml` |
| video-msg | ⚠️ **stale** | ❌ | ❌ | ❌ | ❌ | `ci.yml` runs Maven in **deprecated** Java `backend/` on `ubuntu-latest`; active code is Go `backend-go/` |
| status-tacos | ❌ | ❌ | ❌ | ❌ | ❌ | **No workflows** — add `ci.yml` + ARC `build.yml` + restart hook |

**Reality check:** only `irl-planner-pro` and `plugin-skill-hosting` ship `release.yml` and `cleanup-images.yml` today. Treat those as the target for ghcr.io apps; do not assume siblings already have them.

**Default branch split:** `main` (irl-planner-pro, deep-digest-rss, start-renovate) vs `master` (trivia, linky, plugin-skill-hosting).

**Org-wide optional workflows (none deployed yet):**

- **Container image scanning** — weekly Trivy over `:latest` registry images, artifacts uploaded, informational exit code.
- **Source dependency scanning** — weekly OSV-Scanner, report artifact + job summary.
- **Manual environment e2e** — `workflow_dispatch` Playwright against dev/stage/prod (complements in-CI e2e on `plugin-skill-hosting`).
- **Per-component PR workflow split** — consider for 3+ deployable surfaces (`deep-digest-rss`, `cybernight` dual-frontend) instead of one monolithic `ci.yml`.

**Disclosed gaps to close (priority):**

- Add `release.yml` + `cleanup-images.yml` to ghcr.io siblings (`plugin-skill-hosting` already has them; extend to others when they tag releases).
- Add `.github/actionlint.yaml` everywhere an `arc-<repo>` label is used (only `irl-planner-pro` today).
- Migrate action pins to full SHAs on repos still on `@v6` / `@v4` tags (`plugin-skill-hosting`, `trivia`, `linky`, `start-renovate`).
- Rewrite `easy-host-k8s` `build.yml` from scratch (remove stale Maven `backend/` job) → ARC + `./oglimmer.sh build --platform auto`.
- Rewrite `video-msg` `ci.yml` to test Go `backend-go/` (not deprecated Java `backend/`); add ARC `build.yml` + restart hook.
- Add `ci.yml` + ARC `build.yml` to `status-tacos`.
- `deep-digest-rss`: consider renaming `pr.yml` → `ci.yml` and adding push-to-main trigger for parity (or document `pr.yml` as intentional PR-only gate).

**Valid deviations (not gaps):**

- Private-registry repos (`trivia`, `start-renovate`, `deep-digest-rss`) use `REGISTRY_USERNAME`/`REGISTRY_PASSWORD` instead of `GITHUB_TOKEN` + `packages: write`.
- `plugin-skill-hosting` e2e job is additive — keep when the repo has Playwright journeys.
- Non–Vue+Go repos legitimately filter different paths (`server`/`client`, `news-*`, Java/Maven, Nuxt).