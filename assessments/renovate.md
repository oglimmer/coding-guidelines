# Renovate — project assessment

Per-repo adoption for [../renovate.md](../renovate.md). Last verified: June 2026.

| Repo | `renovate.json` | Standard baseline | Postgres major freeze | Notes |
|------|-----------------|-------------------|----------------------|-------|
| irl-planner-pro | ✅ | ✅ | ✅ | Closest full match + bundled-DB override |
| plugin-skill-hosting | ✅ | ✅ | ❌ | **Gap:** chart ships bundled Postgres — add freeze rule |
| trivia | ✅ | ✅ | ❌ | **Gap:** `helm/trivia` has Postgres — add freeze rule |
| linky | ✅ | ✅ | n/a | External DB |
| yt-infographics | ❌ | — | — | **Gap:** add standard config |
| easy-host-k8s | ✅ | ✅ | n/a | |
| deep-digest-rss | ✅ | ✅ | n/a | No bundled Postgres in chart |
| start-renovate | ✅ | ✅ | n/a | Extra `frontend/renovate.json` for UI fixtures only |
| homepage-v4 | ✅ | ✅ | n/a | Nuxt/npm only |
| coffee-diary | ✅ | ✅ | n/a | npm + gomod + docker |
| boardwalk-billionaire | ✅ | ✅ | n/a | npm + maven + docker |
| cybernight | ✅ | ✅ | n/a | npm (×2 frontends) + maven + docker |
| picz | ✅ | ✅ | n/a | npm + **gradle** + docker |
| picz2 | ✅ | ✅ | n/a | npm + maven + docker |
| video-msg | ✅ | ✅ | n/a | npm + gomod + maven (deprecated BE) + docker |
| status-tacos | ✅ | ✅ | n/a | npm + maven + docker |

**Disclosed gaps:** add a `renovate.json` to `yt-infographics`. Copy the Postgres major-freeze `packageRule` from `irl-planner-pro` into any repo whose Helm chart bundles Postgres (`trivia`, `plugin-skill-hosting`).