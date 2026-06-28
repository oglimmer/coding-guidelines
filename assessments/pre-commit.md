# Pre-commit — project assessment

Per-repo adoption for [../pre-commit.md](../pre-commit.md). Last verified: June 2026.

| Repo | Config | yamllint | shellcheck | Coverage vs target |
|------|--------|----------|------------|-------------------|
| irl-planner-pro | ✅ full | ✅ | ✅ | Target |
| plugin-skill-hosting | ⚠️ minimal | ❌ | ❌ | gofmt, vet, build, eslint, vue-tsc — all on **commit**; no actionlint/gitleaks/helm |
| trivia | ⚠️ minimal | ❌ | ❌ | Same pattern as plugin-skill-hosting; older `pre-commit-hooks@v5` |
| linky | ❌ | ❌ | ❌ | **Gap:** no pre-commit |
| yt-infographics | ❌ | ❌ | ❌ | **Gap:** no pre-commit |
| easy-host-k8s | ❌ | ❌ | ❌ | **Gap:** no pre-commit |
| deep-digest-rss | 🔀 Java | ❌ | ❌ | gitleaks + trufflehog + spotless + `news-frontend` lint — not Vue+Go shape |
| start-renovate | 🔀 Java+Nuxt | ❌ | ❌ | spotless, eslint, vitest, helm-lint loop — not Vue+Go shape |
| coffee-diary | ❌ | ❌ | ❌ | **Gap:** no pre-commit |
| boardwalk-billionaire | ❌ | ❌ | ❌ | **Gap:** no pre-commit |
| cybernight | 🔀 Java+Vue | ❌ | ❌ | gitleaks + trufflehog + spotless + eslint + typecheck (vue + frontend2); helm-lint |
| picz | 🔀 Gradle+Vue | ❌ | ❌ | gitleaks + trufflehog + **Gradle** spotless + eslint |
| picz2 | 🔀 Maven+Vue | ❌ | ❌ | gitleaks + trufflehog + spotless + eslint + typecheck; helm-lint |
| video-msg | 🔀 Java on deprecated BE | ❌ | ❌ | gitleaks + trufflehog + spotless (`backend/`) + eslint; helm-lint; **no Go hooks** |
| status-tacos | 🔀 Maven+Vue | ❌ | ❌ | gitleaks + trufflehog + spotless + eslint + typecheck; helm-lint |

**Disclosed gaps:** roll the `irl-planner-pro` pre-commit stack out to `plugin-skill-hosting` and `trivia` first, since they share the same stack. Add at least shellcheck plus `oglimmer.sh` to every repo that has an `oglimmer.sh`. Java/Spring repos (`deep-digest-rss`, `start-renovate`) need their own hook set — do not force Vue+Go hooks there.

**Valid deviation:** smaller repos may keep all hooks on `commit` until test runtime forces a pre-push split (`plugin-skill-hosting`, `trivia` today).