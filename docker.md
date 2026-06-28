# Docker

How we containerize full-stack apps: multi-stage builds, pinned base images, build metadata, local compose, and CI deploy via `oglimmer.sh`.

This doc assumes a **Vue + Go** two-image layout (`backend/` + `frontend/`). Other stacks (Java, Nuxt, multi-component, server-rendered Go) need their own Dockerfiles — see [java-spring-backend.md](java-spring-backend.md), [nuxt-frontend.md](nuxt-frontend.md), [oglimmer-sh.md](oglimmer-sh.md). For per-repo adoption, see [assessments/docker.md](assessments/docker.md).

## Philosophy

- **Multi-stage builds.** Compile in a fat image; ship a minimal runtime image.
- **Pin base images to digests.** Tags are mutable; `@sha256:…` is not. Renovate keeps digests current ([renovate.md](renovate.md)).
- **Bake build metadata.** Inject `VERSION`, `GIT_COMMIT`, and `BUILD_TIME` at build time; expose them at runtime for diagnostics.
- **Static Go binaries.** `CGO_ENABLED=0`, strip with `-ldflags "-s -w"`, inject via `-X`.
- **Non-root at runtime.** Distroless or nginx's own user model; drop all capabilities in Kubernetes.
- **Local dev via compose.** Postgres healthcheck-gated; full env surface mirrored in `.env.example`.
- **ARC builds use plain docker.** `--platform auto` runs native `docker build` + `docker push`; avoid the buildx docker-container driver on the cluster ([github-actions.md](github-actions.md)).

## Layout

```
backend/
  Dockerfile
  .dockerignore
frontend/
  Dockerfile
  .dockerignore
  nginx.conf
  nginx-security-headers.conf
compose.yml
oglimmer.sh
.env.example
```

## Backend Dockerfile

Go API → distroless static nonroot:

```dockerfile
FROM golang:1.26-alpine@sha256:… AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
ARG VERSION="dev"
ARG GIT_COMMIT="unknown"
ARG BUILD_TIME="unknown"
RUN CGO_ENABLED=0 go build -trimpath \
    -ldflags "-s -w \
      -X <module>/internal/buildinfo.Version=${VERSION} \
      -X <module>/internal/buildinfo.Commit=${GIT_COMMIT} \
      -X <module>/internal/buildinfo.Time=${BUILD_TIME}" \
    -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot@sha256:…
COPY --from=build /out/server /server
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

Rules:

- Copy `go.mod` + `go.sum` before source for layer caching.
- `-trimpath` for reproducible paths.
- The runtime image has no shell. If you need one for debugging, use `distroless/debug`, and only in dev.
- The `internal/buildinfo` package holds the `Version`, `Commit`, and `Time` vars set by ldflags.

## Frontend Dockerfile

Node build → nginx serve:

```dockerfile
FROM node:24-alpine@sha256:… AS build
WORKDIR /src
COPY package.json package-lock.json* ./
RUN npm install --no-audit --no-fund
COPY . .
ARG VITE_APP_VERSION=""
ARG VITE_GIT_COMMIT=""
ARG VITE_BUILD_TIME=""
ENV VITE_APP_VERSION=${VITE_APP_VERSION}
ENV VITE_GIT_COMMIT=${VITE_GIT_COMMIT}
ENV VITE_BUILD_TIME=${VITE_BUILD_TIME}
RUN npm run build

FROM nginx:1.27-alpine@sha256:…
COPY --from=build /src/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY nginx-security-headers.conf /etc/nginx/snippets/security-headers.conf
EXPOSE 80
```

Rules:

- Set Vite env vars as `ARG` + `ENV` before `npm run build` — they are compile-time, not runtime.
- nginx serves only the SPA. API routes go through Kubernetes Ingress, not nginx `proxy_pass`.
- Security headers live in a separate snippet file that `nginx.conf` includes.

## `.dockerignore`

Keep build contexts small:

**backend:**
```
.git
*_test.go
.env
```

**frontend:**
```
.git
node_modules
dist
*.log
.env
.env.*
```

Never copy `node_modules` into the frontend build context — install fresh inside the image.

## `compose.yml` — local development

```yaml
services:
  db:
    image: postgres:18-alpine@sha256:…
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-app}"]
      interval: 3s
      retries: 20

  backend:
    build:
      context: ./backend
    environment:
      DATABASE_URL: postgres://…
      JWT_SECRET: ${JWT_SECRET:-dev-secret-change-me-please-32-chars-min}
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build:
      context: ./frontend
    ports:
      - "8080:80"
    depends_on:
      - backend
```

Rules:

- Pin Postgres image to a digest.
- Gate backend on `service_healthy`, not just `depends_on`.
- Mirror every env var the backend reads in `.env.example`, with dev-safe defaults.
- Use explicit dev placeholders (`ALLOW_INSECURE_JWT_SECRET`, `AUTH_MODE=password`) only for local boot, and document that production must not set them.

## `oglimmer.sh` — build, push, restart

Every deployable repo has a root `oglimmer.sh` that wraps docker build/push and rollout.

### Subcommands

| Command | Purpose |
|---------|---------|
| `build -f/-b/-a` | Build and optionally push frontend/backend/both |
| `release` | Bump semver, commit, tag, push (triggers `release.yml`) |
| `helm-push` | Package and push OCI Helm chart |
| `start` / `stop` / `status` / `logs` | Local backend dev loop |
| `test` | Run backend tests |

### Flags

| Flag | Purpose |
|------|---------|
| `--platform auto\|arm64\|amd64\|multi` | `auto` = plain docker (required on ARC) |
| `--verbose` | Show docker output |
| `--dry-run` | Print commands without executing |
| `--no-push` | Build locally only |
| `--no-restart` | Skip rollout (keel variant) |
| `--registries` | Override image registries |

### Build args passed to docker

Backend: `VERSION`, `GIT_COMMIT`, `BUILD_TIME`.
Frontend: `VITE_APP_VERSION`, `VITE_GIT_COMMIT`, `VITE_BUILD_TIME`.

`oglimmer.sh` reads the version from the single source of truth (`frontend/package.json`) and passes the current git SHA and timestamp.

### Restart without kubectl

CI runners have no cluster access. `oglimmer.sh` must:

- Accept **kubectl OR `RESTART_TOKEN`** — don't require kubectl unconditionally.
- POST to `${RESTART_HOOK_URL:-https://restart.oglimmer.com/restart}/<ns>/<deployment>` with `Authorization: Bearer $RESTART_TOKEN`.
- Prefer kubectl when present (local dev), and fall back to the hook in CI.
- Never echo the token.

Default image names: `ghcr.io/oglimmer/<repo>-backend` and `ghcr.io/oglimmer/<repo>-frontend`.

## CI integration

| Workflow | What it does |
|----------|--------------|
| `build.yml` | ARC runner: `./oglimmer.sh build -b/-f --platform auto` |
| `release.yml` | Hosted runner: multi-arch buildx push with same build-args |
| `cleanup-images.yml` | Prune untagged ghcr.io versions left by digest-pinned deploys |

See [github-actions.md](github-actions.md).

## Image tagging

| Tag | When | Where |
|-----|------|-------|
| `:latest` | Every push to main | `build.yml` on ARC |
| `:vX.Y.Z` + `:latest` | Release tag push | `release.yml` multi-arch |
| `@sha256:…` | Helm values / ArgoCD | Pin deployment to digest |

ArgoCD image-updater pins by digest, which is why `cleanup-images.yml` exists.

## New-repo checklist

1. Add multi-stage `Dockerfile` per component with digest-pinned bases.
2. Add `.dockerignore` per component.
3. Add `compose.yml` with healthchecked Postgres and `.env.example`.
4. Add `oglimmer.sh` with build/restart hook support (copy from a sibling repo).
5. Wire `build.yml` on `arc-<repo>` with `--platform auto`.
6. Add `release.yml` if the repo ships versioned releases.
7. Run `shellcheck oglimmer.sh`; add to pre-commit ([pre-commit.md](pre-commit.md)).
8. Enable Renovate `docker:pinDigests` ([renovate.md](renovate.md)).

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Single-stage images with compilers in production | Multi-stage: build → minimal runtime |
| `FROM node:latest` without digest | Pin `@sha256:…`; let Renovate bump |
| Runtime env vars for Vite `VITE_*` | Build-args at image build time |
| `proxy_pass` API routes in frontend nginx | Ingress routes `/api` to backend |
| buildx docker-container on ARC | `--platform auto` in `build.yml` |
| Require kubectl in `oglimmer.sh` for CI | Restart hook via `RESTART_TOKEN` |
| Hand-edit image tags in Helm | ArgoCD / image-updater pins digest; Renovate bumps base images |