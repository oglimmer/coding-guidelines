# Helm — project assessment

Per-repo adoption for [../helm.md](../helm.md). Last verified: June 2026.

| Repo | Chart | `helm/argocd/` | SealedSecret location | Ingress / paths | Notes |
|------|-------|----------------|----------------------|-----------------|-------|
| irl-planner-pro | ✅ | ✅ | `helm/argocd/*-sealed-secret.yaml` | ✅ API before SPA | Target |
| plugin-skill-hosting | ✅ | ✅ | `helm/argocd/` | ✅ | Sibling reference |
| trivia | ✅ | ❌ | in-chart `templates/sealed-secret.yaml` | ✅ | **Gap:** no argocd/ apps |
| linky | ✅ | ❌ | in-chart templated | ✅ | MySQL app, not Postgres |
| yt-infographics | ✅ | ❌ | in-chart | ✅ | Single backend deployment |
| easy-host-k8s | ⚠️ single app | ❌ | `secret.yaml.template` plaintext | partial | **Gap:** no split FE/BE chart |
| deep-digest-rss | 🔀 multi-deploy | ❌ | in-chart + `helm/seal-secrets.sh` | Traefik middleware | Scraper + taggroupper deployments |
| start-renovate | ✅ | ❌ | `helm/sealed-app-secret.yaml` at root | ✅ | Java + Nuxt images in one chart |
| coffee-diary | ✅ | ❌ | in-chart `templates/sealed-secret.yaml` | ✅ | Split FE/BE deployments; external MariaDB |
| boardwalk-billionaire | ✅ | ❌ | cert-manager TLS (no app SealedSecret) | ✅ `/ws` first | Stateless backend; no bundled DB |
| cybernight | ✅ | ❌ | in-chart SealedSecret | ✅ `/ws` + OAuth paths | External MariaDB; image tags `latest` in values |
| picz | ❌ | ❌ | — | — | **No Helm** — AWS deploy in `build/` |
| picz2 | 🔀 **5 workloads** | ❌ | SealedSecret + configmap | ✅ api paths | api, worker, frontend, **tusd**, retention CronJob |
| video-msg | ✅ `video-msg` | ❌ | `helm/sealed-database-secret.yaml` | ✅ `/api` before SPA | Backend PVC for `video-storage` (50Gi default) |
| status-tacos | ✅ chart at **`helm/`** root | ❌ | `helm/sealed-database-secret.yaml` | ✅ `/api` before SPA | cert-manager TLS; external MariaDB |

**Disclosed gaps:** Only `irl-planner-pro` and `plugin-skill-hosting` use the `helm/argocd/{app,secret-app,sealed-secret}` layout; migrate the other repos when adopting GitOps. Bundled-Postgres repos without a Renovate major freeze: `trivia`, `plugin-skill-hosting` (see [renovate.md](renovate.md) assessment).

**Valid deviations:** `deep-digest-rss` chart models four workloads (frontend, backend, scraper, taggroupper) — one chart, many Deployments, not the two-component template.