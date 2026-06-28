# Docker — project assessment

Per-repo adoption for [../docker.md](../docker.md). Last verified: June 2026.

| Repo | BE Dockerfile | FE Dockerfile | compose | .dockerignore | Notes |
|------|---------------|---------------|---------|---------------|-------|
| irl-planner-pro | ✅ distroless multi-stage | ✅ node→nginx | ✅ PG digest | ✅ both | Target |
| plugin-skill-hosting | ✅ | ✅ | ✅ | ✅ both | Matches target |
| trivia | ✅ | ✅ | ⚠️ PG tag not digest | ❌ missing | **Gap:** add `.dockerignore` on both components |
| linky | 🔀 `server/` | 🔀 `client/` | 🔀 **MariaDB** | partial | Not Postgres; paths differ |
| yt-infographics | ⚠️ single-stage alpine | 🔀 Svelte | ❌ | ❌ | **Gap:** no compose; BE not distroless; ldflags on `main` not `buildinfo` |
| easy-host-k8s | 🔀 `backend-go/` | n/a | 🔀 `docker-compose.yml` | — | Server-rendered HTML; CI still points at removed `backend/` Maven — see [repo-map.md](repo-map.md) |
| deep-digest-rss | 🔀 Java `news-backend/` | 🔀 `news-frontend/` | partial | partial | + Python `scraper/` third image |
| start-renovate | 🔀 Java JAR image | 🔀 Nuxt node-server | ❌ | partial BE only | Pattern B |
| homepage-v4 | n/a | ✅ Nuxt generate→nginx | ❌ | ✅ | Pattern A; CI uses `build`, Docker uses `generate` |
| coffee-diary | ⚠️ alpine multi-stage | ✅ node→nginx | ✅ MariaDB compose | ⚠️ FE only | Not distroless BE; `VITE_GIT_COMMIT` build-arg on FE |
| boardwalk-billionaire | ✅ JAR multi-stage | ✅ node→nginx | ❌ | ✅ both | `server/` + `client/` paths; Temurin 25 |
| cybernight | ✅ JAR (BuildKit cache) | ✅ vue nginx; **also `frontend2/`** | ⚠️ `mariadb:latest` tag | ❌ | **Gap:** unpinned `nginx:latest`; `npm i` not `ci` in FE Dockerfile |
| picz | ✅ Gradle JAR | ✅ `Dockerfile-prod` + Traefik compose | ⚠️ traefik:latest | ❌ | Legacy; S3 via s3fs sidecar |
| picz2 | ✅ Maven JAR | ✅ nginx FE | ✅ MariaDB + MinIO compose | ✅ root `.dockerignore` | Multi-service local stack |
| video-msg | 🔀 Go alpine+ffmpeg | ✅ nginx FE | ⚠️ DB-only `compose.yml` + `docker-compose.local.yml` | ❌ | Active BE is `backend-go/`; deprecated Java `backend/Dockerfile` remains |
| status-tacos | ✅ JAR multi-stage | ✅ nginx FE | ✅ MariaDB compose + teams mock | ❌ | FE Dockerfile comments out lint/tsc/vitest in image build |

**Disclosed gaps:** `trivia` missing `.dockerignore`; `yt-infographics` needs `renovate.json` + compose story; `easy-host-k8s` should stop building images in GitHub Actions directly ([github-actions.md](github-actions.md) assessment).

**Valid deviations:** `deep-digest-rss` uses a three-component layout, documented as shape B in [../oglimmer-sh.md](../oglimmer-sh.md) rather than this doc's `backend/`/`frontend/` paths.