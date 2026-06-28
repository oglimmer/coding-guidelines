# Helm

How we package full-stack apps for Kubernetes: application charts, external secrets, ingress routing, and GitOps via ArgoCD.

Assumes a **Go API + Vue nginx + optional bundled Postgres** chart. Multi-app and Java-backed repos need chart variants — see [java-spring-backend.md](java-spring-backend.md), [nuxt-frontend.md](nuxt-frontend.md). Per-repo adoption: [assessments/helm.md](assessments/helm.md). Path mapping: [assessments/repo-map.md](assessments/repo-map.md).

## Philosophy

- **Chart ships no secrets.** Sensitive values live in a user-supplied `Secret` or `SealedSecret` applied out-of-band.
- **Stateless backend, stateful Postgres.** Backend scales freely; bundled Postgres gets a PVC and stays single-replica.
- **Ingress path order matters.** Specific API paths before the SPA catch-all `/` — the frontend must never shadow the API.
- **Pin images by digest in values.** ArgoCD image-updater writes `@sha256:…` tags; Renovate bumps base image digests.
- **Document every values key.** `values.yaml` is the operator manual — comments explain what each field does and which secret keys it needs.
- **Centralize naming in `_helpers.tpl`.** One place for release names, labels, component suffixes, and secret resolution.

## Layout

```
helm/
  <chart-name>/
    Chart.yaml
    values.yaml
    README.md
    templates/
      _helpers.tpl
      deployment-backend.yaml
      deployment-frontend.yaml
      service-backend.yaml
      service-frontend.yaml
      ingress.yaml
      postgres.yaml          # StatefulSet + PVC when bundled DB
      configmap-frontend-nginx.yaml
      serviceaccount.yaml
      NOTES.txt
    .helmignore
  argocd/
    <app>-app.yaml           # ArgoCD Application
    <app>-secret-app.yaml    # ArgoCD app for SealedSecret
    <app>-sealed-secret.yaml # SealedSecret template (encrypt before commit)
```

## `Chart.yaml`

```yaml
apiVersion: v2
name: my-app
description: ID5 IRL Attendance App (Go backend + Vue frontend + Postgres)
type: application
version: 0.1.0          # chart version — bumped by oglimmer.sh release
appVersion: "0.1.0"     # app version — keep in sync with package.json
maintainers:
  - name: Oli
```

`version` and `appVersion` are bumped together by the release script — never hand-edit in multiple places. See [versioning-release.md](versioning-release.md).

## `values.yaml` structure

Organize by concern with block comments listing required secret keys:

```yaml
existingSecret: ""       # override or defaults to <fullname>-secret

publicBaseURL: "https://app.example.com"

auth:
  mode: oidc             # oidc | password (password = dev stub only)

backend:
  replicaCount: 1
  image:
    repository: ghcr.io/oglimmer/<repo>-backend
    pullPolicy: Always
    tag: "@sha256:…"     # digest pin for ArgoCD

frontend:
  replicaCount: 1
  image:
    repository: ghcr.io/oglimmer/<repo>-frontend
    tag: "@sha256:…"

postgres:
  enabled: true
  image:
    tag: "16-alpine@sha256:…"   # major pinned — see below

ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: "oglimmer-com-dns"
  hosts:
    - host: app.example.com
      paths: […]

podSecurityContext:
  fsGroup: 65532

securityContext:           # backend — distroless nonroot
  runAsNonRoot: true
  runAsUser: 65532
  capabilities:
    drop: [ALL]
  readOnlyRootFilesystem: true

frontendSecurityContext: {}  # nginx needs CHOWN/SETUID for cache dirs
```

### Secrets contract

The chart does **not** create a Secret. Document required keys in `values.yaml` and `README.md`:

| Key | When required |
|-----|---------------|
| `JWT_SECRET` | Always (≥32 chars) |
| `POSTGRES_PASSWORD` | `postgres.enabled=true` |
| `DATABASE_URL` | `postgres.enabled=false` |
| `OIDC_CLIENT_SECRET` | `auth.mode=oidc` |
| `SMTP_PASSWORD` | Authenticating SMTP relay |
| `METRICS_TOKEN` | `backend.metrics.exposeOnIngress=true` |

Resolve the secret name in `_helpers.tpl`:

```yaml
{{- define "<chart>.secretName" -}}
{{- if .Values.existingSecret -}}
{{- .Values.existingSecret -}}
{{- else -}}
{{- printf "%s-secret" (include "<chart>.fullname" .) | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end }}
```

Production uses a `SealedSecret` in `helm/argocd/` — encrypt with the cluster's sealed-secrets controller before committing.

## `_helpers.tpl`

Define helpers for every named resource:

| Helper | Resolves to |
|--------|-------------|
| `<chart>.fullname` | Release-prefixed name (≤63 chars) |
| `<chart>.labels` | Standard `helm.sh/chart`, `app.kubernetes.io/*` labels |
| `<chart>.backend.fullname` | `<fullname>-backend` |
| `<chart>.frontend.fullname` | `<fullname>-frontend` |
| `<chart>.postgres.fullname` | `<fullname>-postgres` |
| `<chart>.secretName` | External secret reference |

Add `app.kubernetes.io/component: backend|frontend|postgres` on component labels for selector clarity.

## Ingress — path ordering

**Backend paths before SPA catch-all.** Example from `values.yaml`:

```yaml
paths:
  - path: /api
    pathType: Prefix
    backend: api
  - path: /healthz
    pathType: Exact
    backend: api
  - path: /readyz
    pathType: Exact
    backend: api
  - path: /
    pathType: Prefix
    backend: web
```

Template rules:

- When optional paths (`/metrics`, `/mcp`, `/oauth`, `/.well-known/*`) are feature-gated, list them in the ingress template **before** the values-driven path loop.
- Use `pathType: Exact` for probe and metrics paths; `Prefix` for API and SPA.
- Match `nginx.ingress.kubernetes.io/proxy-body-size` to the backend's upload cap.
- SSE endpoints need long proxy timeouts and buffering off — document in values comments.

## Deployments

### Backend

- Stateless — no PVC.
- Env from `values.yaml` + `secretKeyRef` for sensitive keys.
- When `postgres.enabled=true`, construct `DATABASE_URL` from individual env vars (user, password from secret, host from helper).
- Liveness: `/healthz`. Readiness: `/readyz` (return 503 until warm if using rematerialize-on-startup).
- Prometheus annotations when `backend.metrics.enabled=true`:
  ```yaml
  prometheus.io/scrape: "true"
  prometheus.io/path: "/metrics"
  prometheus.io/port: "8080"
  ```
- Only expose `/metrics` on ingress when `exposeOnIngress=true` **and** `METRICS_TOKEN` is documented as required.

### Frontend

- nginx serving static `dist/` from the image.
- ConfigMap for nginx config when templates need Helm values (security headers, CSP).
- Separate `frontendSecurityContext` — stock nginx:alpine needs capabilities the backend does not.

### Postgres (bundled)

- `StatefulSet` + `ReadWriteOnce` PVC.
- **Pin Postgres major version.** The PVC data directory is tied to the major version. A major image bump makes Postgres refuse to start. Document in `values.yaml` and disable major bumps in [renovate.md](renovate.md).
- `replicaCount: 1` — do not scale bundled Postgres without shared storage.
- Set `postgres.enabled=false` and supply `DATABASE_URL` in the secret for external DB.

## Security contexts

| Component | Context |
|-----------|---------|
| Backend (distroless) | `runAsNonRoot`, uid 65532, `readOnlyRootFilesystem`, drop ALL caps |
| Frontend (nginx) | Empty default — alpine entrypoint needs CHOWN/SETUID for cache |
| Pod | `fsGroup: 65532` when volumes need group write |

## GitOps (ArgoCD)

```
helm/argocd/
  <app>-app.yaml              # Application → helm/<chart>
  <app>-secret-app.yaml       # Application → SealedSecret manifest
  <app>-sealed-secret.yaml    # Template — kubeseal before apply
```

- Main app syncs the Helm chart from the repo or OCI registry (`oglimmer.sh helm-push`).
- Image-updater watches ghcr.io and writes digest pins into values or a separate image-updater config.
- SealedSecret is a separate ArgoCD Application so secret rotation does not require chart changes.

## Validation

```bash
helm lint helm/<chart-name>
```

Wire into pre-commit ([pre-commit.md](pre-commit.md)). Exclude `templates/` from yamllint — they are Go templates, not valid YAML.

## New-repo checklist

1. Scaffold chart from an existing sibling chart in the org, or from this doc's layout.
2. Replace chart name prefix in `_helpers.tpl` and all templates.
3. Document secret keys in `values.yaml` and `README.md`.
4. Configure ingress paths (API before SPA).
5. Pin initial image digests in `values.yaml`.
6. Add SealedSecret template under `helm/argocd/`.
7. Add ArgoCD Application manifests.
8. Add `helm lint` to pre-commit.
9. Add Postgres major-freeze rule to `renovate.json` if bundled DB is enabled.
10. Smoke-test: `helm template <release> helm/<chart> --debug`.

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Commit plaintext Secrets | SealedSecret or external secret manager |
| Put secrets in `values.yaml` | `secretKeyRef` + `existingSecret` |
| SPA catch-all `/` before `/api` | Specific backend paths first |
| Scale bundled Postgres past 1 | External DB or operator-managed cluster |
| Bump Postgres major via Renovate | Pin major; deliberate pg_upgrade |
| World-readable `/metrics` on ingress | Require `METRICS_TOKEN` |
| Duplicate naming logic in templates | `_helpers.tpl` helpers |
| Skip `helm lint` | Pre-commit + CI if chart changes are frequent |