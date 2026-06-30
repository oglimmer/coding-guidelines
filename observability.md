# Observability

How we monitor apps on the k3s cluster: **metrics** with Prometheus, dashboards in **Grafana**. Apps expose a token-gated `/metrics`; a plain Prometheus server scrapes them by pod/service annotation. **Logs** today are just container stdout (`kubectl logs`) — Loki exists but only ingests Traefik access logs, not app logs. This doc describes what is actually deployed and the conventions that are real, then flags the gaps.

Grounded in the live cluster: the monitoring stack lives in the **`monitoring`** namespace of the `oglimmer.de` k3s cluster, deployed from the **`global-install-k8s`** repo (not part of this guidelines repo). App-side conventions come from [go-backend.md](go-backend.md) and [java-spring-backend.md](java-spring-backend.md); chart wiring from [helm.md](helm.md). Per-repo adoption: [assessments/](assessments/).

> **Not Operator-based.** There is **no Prometheus Operator** on the cluster — no `ServiceMonitor`, `PodMonitor`, or installed `PrometheusRule` CRDs. Discovery is annotation-based; alert rules (when added) go in the Prometheus server's `serverFiles`, not a CRD. Any guide that tells you to add a ServiceMonitor does not apply here.

## Philosophy

- **App exposes, Prometheus pulls.** Apps serve `/metrics`; the `prometheus-community/prometheus` server scrapes them via its `kubernetes-service-endpoints` job, keyed on `prometheus.io/scrape` annotations. Apps never push metrics.
- **Metrics are RED + a few domain counters.** One app-prefixed request-duration histogram covers rate, errors (via a status label), and latency. Add a handful of business counters; resist one-counter-per-thing.
- **Labels are bounded.** Every label value is a time series. Never label with user ID, request ID, or raw path — templated route only. Identifiers belong in logs.
- **`/metrics` is gated, or cluster-internal.** A `METRICS_TOKEN` bearer guards it; either keep it off the public ingress, or require the token when it's exposed ([helm.md](helm.md)).
- **Logs go to stdout, single line.** Go via stdlib `log/slog`; Java via the Spring default console. One event per line so `kubectl logs` stays readable. (App-log aggregation into Loki is a known gap — see below.)
- **The stack is one external repo, apps add annotations.** Apps don't ship Prometheus/Grafana/Loki; they add scrape annotations + a values toggle. The stack is owned by `global-install-k8s`.

## What's actually deployed

Cluster monitoring stack (`monitoring` namespace, `global-install-k8s`, k3s `HelmChart` CRs + ArgoCD):

| Component | Chart / image | Reality |
|-----------|---------------|---------|
| Prometheus | `prometheus-community/prometheus` (plain server, **no Operator**) | Annotation SD + many static jobs; `scrape_interval: 2m`; ~7d/8GB retention; no `remote_write`; **Alertmanager not wired** |
| Grafana | `grafana/grafana` | Ingress `grafana.oglimmer.de`; provisions **only a Prometheus datasource**; dashboards hand-built in the UI (not as code) |
| Loki | `grafana/loki` SingleBinary on bundled MinIO (S3, schema v13/tsdb) | 24h retention, ingress `loki.oglimmer.de`; **not a provisioned Grafana datasource**; receives **only Traefik access logs** |
| Log shipper | `grafana/promtail` **sidecar on Traefik** | Tails Traefik JSON access logs → Loki. **No cluster-wide DaemonSet; app pod stdout is not shipped anywhere.** |
| Blackbox | `prom/blackbox-exporter` (raw Deployment, ArgoCD) | External HTTP/ICMP probes (`oglimmer.com`, `linky1.com`, …) via a static `blackbox-http` job |
| Cluster metrics | node-exporter, kube-state-metrics, longhorn, mariadb, fritzbox, ping-exporter | Static scrape jobs |

Ingress is **Traefik + cert-manager** (`oglimmer-com-dns` issuer), not nginx-ingress.

> A second, unrelated stack (`grafana-ansible`) runs the `modular-design` / Ärztekammern apps on a Hetzner VM via **Docker Compose + Ansible** (Prometheus static configs, Loki on MinIO, Promtail for Traefik logs, blackbox). It is a separate environment; these guidelines target the k3s app cluster above. `status-tacos` is the one app deployed into both.

## What an app exposes

| Signal | Go API | Java API | Frontend (nginx) |
|--------|--------|----------|------------------|
| Metrics | `/metrics` via `prometheus/client_golang` | `/actuator/prometheus` (Micrometer, mgmt port) | none |
| Health | `/healthz` (liveness) | `/actuator/health` | nginx up = healthy |
| Readiness | `/readyz` (503 until warm) | `/actuator/health/readiness` | n/a |
| Logs | single-line stdout (`slog`) | single-line stdout (Spring default) | nginx stdout |

Probes drive Kubernetes restarts/traffic; Prometheus scrapes `/metrics` separately ([go-backend.md](go-backend.md), [java-spring-backend.md](java-spring-backend.md)).

## Go metrics — the house convention

`internal/metrics` owns a **private registry**, the HTTP middleware, and the gated handler. The mature pattern (`plugin-skill-hosting`, `irl-planner-pro`, `trivia`):

```go
var registry = prometheus.NewRegistry()  // private, not the global default

// RED — app-prefixed histogram; status label carries the error signal.
httpRequestDuration = prometheus.NewHistogramVec(prometheus.HistogramOpts{
    Name:    "trivia_http_request_duration_seconds",   // <app>_ prefix
    Buckets: prometheus.DefBuckets,
}, []string{"method", "route", "status"})              // route = template, never raw path

// Plus: <app>_http_requests_total, <app>_http_requests_in_flight,
// a few domain counters (logins, mutations, MCP tool calls),
// and Go/Process collectors.
registry.MustRegister(collectors.NewGoCollector(), collectors.NewProcessCollector(...))
```

Rules:

- **`route` is the chi route pattern** (`/users/{id}`), never the resolved path — raw paths explode cardinality.
- **Ship a `build_info` gauge.** `trivia` registers `trivia_build_info{version,commit}` via `NewGaugeFunc` so a single PromQL series shows the deployed version — adopt this everywhere (irl/plugin currently lack it).
- **Gate the handler:** `r.Method("GET", "/metrics", metrics.Handler(cfg.MetricsToken))`. Empty token = open (relies on the ingress not routing `/metrics`); set `METRICS_TOKEN` to require `Authorization: Bearer …`.
- Prefix every metric with the app name (`psh_`, `irl_`, `trivia_`) so they don't collide in a shared Prometheus.

## Java metrics — Micrometer

`picz2/server` is the reference (the only fully-wired Java app):

```yaml
management:
  server.port: 8081                       # metrics on a separate mgmt port
  endpoints.web.exposure.include: health,info,metrics,prometheus
  endpoint.health.probes.enabled: true
```

Dependency: `io.micrometer:micrometer-registry-prometheus`. Micrometer auto-instruments `http_server_requests_seconds` (templated `uri` tag) — same RED data, no manual middleware. Expose `/actuator/prometheus` on the **management port**, scraped via a dedicated metrics Service (below).

## Prometheus scraping — two real topologies

No Operator, so discovery is annotations consumed by the server's `kubernetes-service-endpoints` job. Two patterns are in use; prefer **A** (cluster-internal).

**A. Annotation scrape, cluster-internal** (`irl-planner-pro`, `plugin-skill-hosting` on the backend Deployment; `picz2` on dedicated `-metrics` Services). Gate behind a values toggle (`backend.metrics.enabled` / `monitoring.scrape.enabled`):

```yaml
prometheus.io/scrape: "true"
prometheus.io/path: "/metrics"          # or /actuator/prometheus for Java
prometheus.io/port: "8080"              # or 8081 mgmt port for Java
```

`/metrics` stays in-cluster — no `METRICS_TOKEN` needed for the scrape path, no ingress exposure.

**B. Ingress + bearer token** (`trivia`). No scrape annotation; `/metrics` is published as an ingress path and gated by `METRICS_TOKEN` from a sealed secret, so Prometheus scrapes it over the public URL with the token. Use only when the scraper can't reach the pod network.

The `METRICS_TOKEN` key travels in the chart's SealedSecret ([helm.md](helm.md)). Don't world-expose `/metrics` without it.

## Logging — the current reality

- Apps log **single-line to stdout/stderr**: Go uses stdlib `log/slog` (logfmt text, **not JSON**); Java uses the Spring default console pattern. Keep a correlation/request ID in each line.
- **App logs are not aggregated.** The only Promtail is a sidecar on Traefik shipping **access logs**; there is no DaemonSet tailing pod stdout, and Loki is not a Grafana datasource. To read app logs today: `kubectl logs`.
- No log files, no app-embedded shippers. (A `loki4j` logback appender and the docker `loki` log-driver exist as references in `loki-logging-example` and are used in the Ansible stack — not on the k3s app cluster.)

## Grafana & alerting

- **Grafana** provisions only the Prometheus datasource (`http://prometheus-server`). Dashboards are **built in the UI, not committed as code** — today the cluster has `K8S Dashboard`, `Node Exporter Full`, `Pod Restarts`, and per-app dashboards for `plugin-skill-hosting` (overview + backend) and `trivia` (backend).
- **Alerting is not configured as code.** Alertmanager is not wired and no `PrometheusRule` is installed. `picz2/templates/NOTES.txt` ships a *pasteable* example (`PhotoUploadJobsBacklog`, `PhotoUploadWorkerDown`) with instructions to add it to the Prometheus release's `serverFiles.alerting_rules.yml` — that (editing the server chart values in `global-install-k8s`), not a CRD, is how a rule gets added.

## Gaps / roadmap

These are real and worth knowing before you assume a signal exists:

- **Metrics adoption is partial.** Only 3 Go apps (`plugin-skill-hosting`, `irl-planner-pro`, `trivia`) and 1 Java app (`picz2`) are fully wired. `linky`, `yt-infographics`, `video-msg`, `status-tacos`, `cybernight`, `start-renovate`, `boardwalk-billionaire` expose **no** metrics. `coffee-diary` (basic-auth `/metrics`) and `deep-digest-rss` (`/actuator/prometheus`) expose metrics but **nothing scrapes them** — add the annotations.
- **No app-log aggregation.** Ship pod stdout to Loki (a Promtail/Alloy DaemonSet, or per-app push) and add a Loki datasource to Grafana, then you can correlate logs with metrics.
- **No alerting.** Wire Alertmanager and define `alerting_rules.yml` (error rate, latency, probe-down, crashloop) in `global-install-k8s`.
- **Dashboards aren't code.** Export the existing dashboards to JSON and provision them (Grafana sidecar or `dashboardProviders`) so they survive a Grafana reinstall.
- **Standardize `build_info`** across all metric'd apps.

## New-repo checklist

1. Expose `/metrics` — Go `internal/metrics` with a private registry and app-prefixed RED histogram, or Java Micrometer `/actuator/prometheus` on the mgmt port.
2. Add a `<app>_build_info` / Actuator `info` metric so dashboards show the running version.
3. Gate `/metrics` with `METRICS_TOKEN` (Go) / management port + cluster-internal scrape (Java).
4. Add `prometheus.io/scrape|path|port` annotations behind a `backend.metrics.enabled` (or `monitoring.scrape.enabled`) values toggle ([helm.md](helm.md)); carry `METRICS_TOKEN` in the SealedSecret.
5. Confirm the app logs single-line to **stdout** (no files).
6. After deploy, verify in Prometheus that the target is `UP` (the connected Grafana MCP / `grafana_api_request` can read targets and existing dashboards).
7. If the app needs a dashboard, build it in Grafana for now — and ideally export the JSON into the repo until provisioning-as-code lands.

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Add a `ServiceMonitor`/`PrometheusRule` CRD | No Operator here — use `prometheus.io/scrape` annotations; rules go in the server's `serverFiles` |
| Push metrics/logs from the app | Expose `/metrics`; log to stdout — Prometheus pulls, `kubectl logs` reads |
| Label metrics with user/request ID or raw path | Templated route label; identifiers go in logs |
| Use the global default Prometheus registry | Private `prometheus.NewRegistry()` per app |
| Unprefixed metric names in a shared Prometheus | `<app>_` prefix on every metric |
| Public `/metrics` on ingress without a token | `METRICS_TOKEN`, or keep it cluster-internal |
| Write logs to files or add a per-app shipper to the k3s cluster | Single-line stdout; aggregation is a cluster concern |
| Assume logs are searchable in Grafana | Loki only has Traefik access logs today — use `kubectl logs` for apps |
| Expose `/actuator/*` broadly | `include: health,info,metrics,prometheus` only, on the mgmt port |
| Hand-build a dashboard and never export it | Save the JSON to the repo so it survives a Grafana reinstall |
