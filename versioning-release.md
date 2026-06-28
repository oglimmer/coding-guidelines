# Versioning & release

How we number releases, bump versions, tag git, build images, publish charts, and show build info in the UI. Cross-links: [oglimmer-sh.md](oglimmer-sh.md) (`release` command), [github-actions.md](github-actions.md) (`build.yml` / `release.yml`), [helm.md](helm.md) (chart versions), [docker.md](docker.md) (build-args).

Per-repo release variants and gaps: [assessments/repo-map.md](assessments/repo-map.md) (release flow table).

## Philosophy

- **One semver source of truth per repo.** Pick a canonical file; the release script propagates from there. Never hand-edit the same version in three places.
- **Tags are explicit.** Production semver lives in git as annotated tags `vX.Y.Z`. Everyday deploys use `:latest`; tagged releases also get `:vX.Y.Z` (and often refresh `:latest`).
- **Build metadata is not semver.** Git commit and build timestamp travel with the binary for support and debugging — they do not replace `package.json` / `Chart.yaml` version fields.
- **Show both sides during rolling deploys.** Frontend and backend versions can differ briefly after a partial rollout; the UI should display each component's own build info, not assume they match.
- **Release is a deliberate act.** Bumping semver, committing, tagging, and pushing are grouped in `oglimmer.sh release` (or equivalent) — not silent side effects of `build`.

## Three concepts (do not conflate)

| Concept | Example | Where it lives | User-facing? |
|---------|---------|----------------|--------------|
| **App semver** | `1.4.2` | `frontend/package.json`, `Chart.yaml`, tag `v1.4.2` | Yes — changelog, support tickets |
| **Image tag** | `:latest`, `:v1.4.2` | Container registry | Operators / GitOps |
| **Build metadata** | `abc1234`, `2026-06-28T12:00:00Z` | `-ldflags`, `VITE_*` env, `/api/version` | Yes — footer / about popover |

`:latest` is a **moving pointer**, not a version. Clusters that pin by digest (ArgoCD image-updater, `imagePullPolicy: Always` + rollout restart) still need semver tags or chart `appVersion` for human-readable releases.

## Source of truth by stack

| Stack | Canonical semver | Also updated on release | Notes |
|-------|------------------|-------------------------|-------|
| Vue + Go + Helm (default) | `frontend/package.json` `version` | `helm/<chart>/Chart.yaml` `version` + `appVersion` | Target: `irl-planner-pro` |
| Vue + Go, no helm-push | `frontend/package.json` | `backend/VERSION` optional | `coffee-diary` uses `backend/VERSION` as primary bump input |
| Vue + Java Spring | `frontend/package.json` | `backend/pom.xml` (`mvn versions:set`) | Post-release SNAPSHOT bump on backend is common |
| Go `backend-go/` only path | `backend-go/VERSION` + `frontend/package.json` | — | `video-msg` — align both in `release` |
| Nuxt / static | `package.json` at app root | `helm/Chart.yaml` if chart exists | `homepage-v4` |
| Java-only image name | `pom.xml` `<version>` | — | Extract with `xmlstarlet` in tag-triggered CI if needed |

**Rule:** `oglimmer.sh` (or the release workflow) owns the propagation list. Developers run `./oglimmer.sh show` (or `release` preview) to see current values before bumping.

### Helm chart sync

On every release that publishes a chart, bump **both** fields together:

```yaml
version: 1.4.2      # chart package version
appVersion: "1.4.2" # deployed application version
```

`version` is the OCI chart artifact version; `appVersion` is what operators read in `helm list` / ArgoCD. Keep them equal unless you have a deliberate chart-only change (rare — document why).

### SNAPSHOT convention (Java / VERSION-file backends)

After a release build, some repos reset the backend to a development suffix:

```
1.4.2        → release tag + images
1.4.3-SNAPSHOT → next dev cycle on main
```

Commit the SNAPSHOT bump as a **separate commit** after the release tag and build — never tag SNAPSHOT versions. Frontend `package.json` may stay at the released semver until the next bump, or move to `-SNAPSHOT` — pick one per repo and stick to it (`status-tacos`, `video-msg`, `coffee-diary` use backend SNAPSHOT; Vue+Go ghcr repos typically do not).

## Semver rules

- **Format:** `MAJOR.MINOR.PATCH` — no leading `v` in files; `v` prefix **only** on git tags (`v1.4.2`).
- **Bump mapping** in `oglimmer.sh release`:
  - `major` → incompatible API / schema breaking changes
  - `minor` → backwards-compatible features
  - `bugfix` / `patch` → backwards-compatible fixes
- **Initial version:** `0.1.0` for new apps; `0.0.x` only for pre-1.0 experiments.
- **Pre-1.0:** `0.y.z` may break APIs — still tag releases for reproducibility.
- **Do not** encode environment (`1.2.3-staging`) in semver. Use image tags, Helm values, or ingress hostnames for environment.

## Release flow variants

Not every repo implements the full GitOps path. Pick one variant and document it in the repo README.

| Variant | When | Steps | CI |
|---------|------|-------|-----|
| **Full GitOps** | ghcr.io apps with Helm OCI publish | bump files → commit → tag `v*` → **push tag** → `release.yml` multi-arch → `helm-push` | `release.yml` on tag push |
| **Tag push + ARC build** | Private registry, no chart publish | bump → commit → tag → push → `build.yml` or manual `oglimmer.sh build` | `build.yml` on main; optional tag workflow |
| **Local tag + build** | Small / single-node deploy | bump → commit → tag → `execute_build` only (no remote tag push) | None or manual |
| **Interactive bump + build** | `registry.oglimmer.com` kubectl repos | bump → commit → tag → local build+push+restart; optional SNAPSHOT follow-up | Often no `release.yml` |
| **None** | Legacy / manual ops | Ad-hoc tags or floating `:latest` only | Document as gap |

Reference implementations: **Full GitOps** — `irl-planner-pro`, `plugin-skill-hosting`. **Interactive bump + build** — `start-renovate`, `coffee-diary`, `video-msg`, `status-tacos`.

## `oglimmer.sh release`

Standard shape (Vue + Go + Helm):

```bash
./oglimmer.sh release              # interactive major/minor/bugfix
./oglimmer.sh release --bump minor # non-interactive
./oglimmer.sh show                 # print current version(s) without releasing
```

**Full GitOps sequence** (inside `execute_release`):

1. Read current semver from `frontend/package.json`
2. Bump per operator choice
3. Write `package.json` + `package-lock.json`
4. Write `helm/<chart>/Chart.yaml` `version` and `appVersion`
5. `git commit -m "Release vX.Y.Z"`
6. `git tag -a vX.Y.Z -m "Release vX.Y.Z"`
7. `git push origin HEAD` and `git push origin vX.Y.Z`
8. `helm-push` (if applicable)

The **tag push** triggers `release.yml` — do not rely on `build.yml` alone for version-pinned multi-arch images.

**Local-only variant:** steps 1–6, then `execute_build` instead of push + `release.yml`; optionally SNAPSHOT backend afterward.

## CI: everyday build vs tagged release

| Workflow | Trigger | Images | Architectures | Purpose |
|----------|---------|--------|---------------|---------|
| `build.yml` | push to default branch | `:latest` | arm64 on ARC (native) | Continuous deploy after merge |
| `release.yml` | push tag `v*` | `:vX.Y.Z` and `:latest` | `linux/amd64` + `linux/arm64` | Reproducible release artifacts + GitHub Release |

### Build metadata in CI

Every image build should pass the same three build-args the local script uses:

| Arg | Source in CI | Source in `oglimmer.sh build` |
|-----|----------------|-------------------------------|
| `VERSION` / `VITE_APP_VERSION` | Tag strip `v` or `package.json` | `package.json` `version` |
| `GIT_COMMIT` / `VITE_GIT_COMMIT` | `github.sha` | `git rev-parse --short HEAD` |
| `BUILD_TIME` / `VITE_BUILD_TIME` | `github.event.head_commit.timestamp` or `date -u +%Y-%m-%dT%H:%M:%SZ` | `date -u +%Y-%m-%dT%H:%M:%SZ` |

`build.yml` on ARC: `./oglimmer.sh build -a --platform auto` with `RESTART_TOKEN` — the script injects args.

`release.yml` on hosted runner: `docker/build-push-action` with explicit `build-args` and GHA cache.

Optional: weekly `schedule` + `workflow_dispatch` on `release.yml` to rebuild the latest `v*` tag when base images change — see [github-actions.md](github-actions.md).

## Baking version into binaries

### Go backend

Package `internal/buildinfo` with string vars set via `-ldflags`:

```go
package buildinfo

var Version = "dev"
var Commit = "unknown"
var Time = "unknown"
```

Dockerfile / `go build`:

```dockerfile
ARG VERSION="dev"
ARG GIT_COMMIT="unknown"
ARG BUILD_TIME="unknown"
RUN go build -ldflags "\
  -X myapp/internal/buildinfo.Version=${VERSION} \
  -X myapp/internal/buildinfo.Commit=${GIT_COMMIT} \
  -X myapp/internal/buildinfo.Time=${BUILD_TIME}" \
  -o server ./cmd/server
```

Expose `GET /api/version` (or include in `/healthz` as a separate field) returning JSON:

```json
{
  "name": "myapp-backend",
  "version": "1.4.2",
  "gitCommit": "abc1234",
  "buildTime": "2026-06-28T12:00:00.000Z"
}
```

Unauthenticated, read-only, no secrets. Register the route **before** auth middleware if the SPA footer needs it during login.

### Java Spring backend

Prefer Spring Boot **Actuator** `GET /api/actuator/info` (or `/actuator/info` behind ingress) with `build.version` and `git.commit.id` from `spring-boot-maven-plugin` / `git-commit-id-plugin`, or a thin wrapper matching the Go JSON shape above.

Align ingress so the SPA can reach the info endpoint on the same host (`/api/actuator/info` before SPA catch-all).

### Vue / Vite frontend

**Build-time bake** via `vite.config.ts` `define` — not runtime `import.meta.env` alone (env vars are easy to forget locally):

```ts
const pkg = JSON.parse(readFileSync('./package.json', 'utf-8'))
const appVersion = process.env.VITE_APP_VERSION || pkg.version || 'dev'
// git commit: VITE_GIT_COMMIT || execSync('git rev-parse --short HEAD')
// build time: VITE_BUILD_TIME || new Date().toISOString()

export default defineConfig({
  define: {
    __APP_VERSION__: JSON.stringify(appVersion),
    __GIT_COMMIT__: JSON.stringify(gitCommit),
    __BUILD_TIME__: JSON.stringify(buildTime),
  },
})
```

`frontend/Dockerfile` must declare and pass through:

```dockerfile
ARG VITE_APP_VERSION=""
ARG VITE_GIT_COMMIT=""
ARG VITE_BUILD_TIME=""
ENV VITE_APP_VERSION=${VITE_APP_VERSION}
ENV VITE_GIT_COMMIT=${VITE_GIT_COMMIT}
ENV VITE_BUILD_TIME=${VITE_BUILD_TIME}
RUN npm run build
```

Export a single module (e.g. `src/build-info.ts`) consumed by the UI — do not scatter `import.meta.env.VITE_*` across components.

### `version.json` fallback (optional)

For local dev without Docker build-args, write at image build time:

```json
{ "version": "1.4.2", "git": "abc1234" }
```

to `public/version.json` and `fetch('/version.json')` when `__APP_VERSION__` is `dev`. Production Docker builds should prefer build-args; the file is a dev/fallback path.

## Showing version in the UI

### Minimum

- Users can find **app semver** somewhere unobtrusive (footer, About, debug panel).
- Support staff can read **git commit** without opening devtools.

### Recommended pattern (Vue SPA + API)

1. **`build-info.ts`** — frontend metadata from Vite `define` (zero network cost).
2. **`useBuildInfo()` composable** — lazy-fetch backend once from `/api/version` (cache module-level; tolerate failure).
3. **Footer or popover** — one line per component:

   ```
   v1.4.2 · abc1234 · 28 Jun 2026
   ```

   Label rows `Frontend` / `Backend` when both are shown.

4. **Lazy load** — fetch backend version on first popover open, not on every page load, unless the app is admin-only.

During rolling deploys, frontend `1.4.3` and backend `1.4.2` is normal — do not hide the mismatch.

### Nuxt / static sites

Bake version at `nuxt generate` / `npm run build` the same way (env or `define`). No runtime API — show frontend line only, or fetch a static `version.json` deployed beside the site.

### What not to show

- Internal image digests (unless an operator debug view)
- SNAPSHOT strings on production-facing marketing pages (OK in admin/debug)
- Registry credentials or build hostnames

## Registry tags & Kubernetes

| Tag | When pushed | Consumed by |
|-----|-------------|-------------|
| `:latest` | Every merge to default branch (`build.yml`) | Helm `values.yaml` default; `imagePullPolicy: Always` + rollout restart |
| `:vX.Y.Z` | Tag push (`release.yml`) | Pin in values for reproducible rollbacks; GitHub Release notes |
| `:prod` | Some legacy flows | Avoid for new repos — prefer semver |

**Helm values:** `image.tag: latest` for continuous deploy; override to `v1.4.2` for pin. Document in chart README.

**Cleanup:** `cleanup-images.yml` prunes untagged digests left after `:latest` moves — see [github-actions.md](github-actions.md). Tagged `v*` versions are kept.

## GitHub Release

On tag push (`release.yml` tail):

- `softprops/action-gh-release` with `generate_release_notes: true`
- Attach nothing if images are the artifact (registry is source of truth)
- Release title: `vX.Y.Z` matching the tag

Changelog content comes from merged PR titles since the last tag — encourage conventional commit / PR titles for readable notes.

## New-repo checklist

1. Choose canonical semver file(s) per stack table above.
2. Implement `oglimmer.sh show` and `release` (or document manual process).
3. Pass `VERSION` / `VITE_*` build-args in Dockerfiles and `oglimmer.sh build`.
4. Add backend version endpoint (Go `/api/version` or Spring Actuator info).
5. Add frontend `build-info.ts` + footer/popover.
6. Sync `Chart.yaml` on release if Helm chart exists.
7. Wire `release.yml` on `v*` if using ghcr.io + multi-arch.
8. Wire `build.yml` for `:latest` on merge.
9. Add `cleanup-images.yml` for ghcr.io repos.
10. Document which release variant the repo uses in its README.

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Bump version only in `Chart.yaml` | Single source → propagate in `release` |
| Use `v1.2.3` inside `package.json` | `1.2.3` in files; `v1.2.3` on git tag only |
| Rely on `:latest` as semver | Tag `v*` for support and rollback |
| Hardcode version in Vue components | `build-info.ts` + `define` |
| Assume FE and BE versions always match | Show both; explain rolling deploy |
| Tag SNAPSHOT versions | SNAPSHOT commits on main only, after release |
| Skip build-args in local Docker builds | Defaults `dev` / `unknown` are fine — but production CI must set them |
| Run `npm version` without committing lockfile | Always `git add package-lock.json` with `package.json` |
| Release from CI without a git tag | Tag is the audit trail; images follow the tag |

## Valid deviations

- **Floating `:latest` only** on private single-node clusters — acceptable when semver tags are not yet wired; track as assessment gap.
- **`VITE_GIT_VERSION` naming** vs `VITE_GIT_COMMIT` — older repos use shorter names; new repos use `GIT_COMMIT` / `BUILD_TIME` for clarity.
- **Backend-only apps** (no SPA) — Actuator or `/version` only; no frontend section.
- **Multi-image repos** (api + worker same JAR) — one semver; both deployments restart from the same tag.