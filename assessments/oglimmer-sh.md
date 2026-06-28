# oglimmer.sh — project assessment

Per-repo adoption of [../oglimmer-sh.md](../oglimmer-sh.md). Last verified: June 2026.

| Repo | Shape | Registry | Namespace env | Restart hook | `PLATFORM` default | CI `--platform auto` |
|------|-------|----------|---------------|--------------|-------------------|----------------------|
| irl-planner-pro | A full-stack | ghcr.io | `RESTART_NAMESPACE` | ✅ | arm64 | ✅ |
| plugin-skill-hosting | A | ghcr.io | `RESTART_NAMESPACE=default` | ✅ | arm64 | ✅ |
| trivia | A | registry.oglimmer.com | **`K8S_NAMESPACE`** | ✅ | arm64 | ✅ |
| linky | 🔀 client/server | registry.oglimmer.com | `K8S_NAMESPACE` | ✅ | arm64 | ✅ |
| yt-infographics | A | registry.oglimmer.com | `RESTART_NAMESPACE` | ✅ | arm64 | ✅ |
| easy-host-k8s | **C** single | registry.oglimmer.com | — | ⚠️ **kubectl only** | arm64 | ❌ CI broken (see [repo-map.md](repo-map.md)) |
| deep-digest-rss | **B** multi | registry.oglimmer.com | `K8S_NAMESPACE` | ✅ | n/a plain docker | ✅ |
| start-renovate | A | registry.oglimmer.com | `RESTART_NAMESPACE` | ✅ | arm64 | ✅ |
| homepage-v4 | **C** single | registry.oglimmer.com | — | ✅ | arm64 | ❌ no `build.yml` |
| coffee-diary | A | registry.oglimmer.com | — | ⚠️ **kubectl only** | arm64 | ❌ no workflows |
| boardwalk-billionaire | 🔀 A′ client/server | registry.oglimmer.com | `K8S_NAMESPACE` | ✅ | arm64 | ✅ |
| cybernight | A (+ vue\|pixi) | registry.oglimmer.com | — | ⚠️ **kubectl only** | arm64 | ❌ no workflows |
| picz | ❌ | — | — | — | — | ❌ uses `build/` scripts |
| picz2 | 🔀 B-ish (+ worker) | registry.oglimmer.com | — | ⚠️ **kubectl only** | auto default | ❌ no workflows; restarts api **and** worker |
| video-msg | A (`backend-go/`) | registry.oglimmer.com | — | ⚠️ **kubectl only** | **arm64** default | ❌ stale `ci.yml`; `video-msg-fe`/`video-msg-be` |
| status-tacos | A | registry.oglimmer.com | — | ⚠️ **kubectl only** | **multi** default | ❌ no workflows; `status-tacos-frontend`/`status-tacos-backend` |

**Naming splits (valid, document when copying):**

- `RESTART_NAMESPACE` (`irl-planner-pro`, `plugin-skill-hosting`, `start-renovate`, `yt-infographics`) vs `K8S_NAMESPACE` (`trivia`, `linky`, `deep-digest-rss`) — same hook URL shape, just a different variable name.
- Image suffixes: `<repo>-frontend`/`-backend` vs `start-renovate-fe`/`start-renovate-be` vs `news-frontend`/`news-backend`.

**Disclosed gaps:**

- **`easy-host-k8s`:** `restart_deployment` uses kubectl only (no hook), and CI still references the removed `backend/` Maven tree. Rewrite CI before trusting this repo as a template ([repo-map.md](repo-map.md)).
- **`linky`:** a large custom script (~1400 lines) maps `-f`/`-b` to `client`/`server`. It works, but is harder to diff against template A.

**Valid deviations:** `deep-digest-rss` is shape B, with `get_deployments()` returning multiple K8s deployments per component (scraper → `news-scraper` + `news-taggroupper`). It uses plain `docker build` without `PLATFORM` branching, which suits the age of its script.

## Release flow by repo

| Variant | Repos |
|---------|-------|
| Full GitOps (tag push + `release.yml` + `helm-push`) | irl-planner-pro, plugin-skill-hosting |
| Local tag + `execute_build` only | yt-infographics |
| Interactive bump + build, no `helm-push` | start-renovate, coffee-diary (`backend/VERSION`), video-msg (`backend-go/VERSION`), status-tacos (`backend/pom.xml`) |
| No `release` command | deep-digest-rss, linky, trivia, easy-host-k8s |

See [repo-map.md](repo-map.md) for full release variant table.