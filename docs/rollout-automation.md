# Automated rollout policy: blue/green + analysis-gated promotion + synthetic traffic

## Status

Accepted. Applies to `deploy/base/services/api` and `deploy/base/services/web`. `deploy/base/services/dotgolf-service` is unchanged (it isn't
even enabled in `deploy/overlays/production/kustomization.yaml` today) and can adopt the same pattern later if it's
turned on.

## Problem

Both `golfapp-api` and `golfapp-web` previously used Argo Rollouts' `canary` strategy with a
`setWeight: 25` / `pause: {}` step. `pause: {}` (no duration) pauses indefinitely until a human runs
`kubectl argo rollouts promote` or patches rollout status directly - which is exactly the manual
intervention this change removes. On a single-Pi homelab cluster with no on-call and infrequent
deploys, a stuck rollout just sits paused until someone happens to notice.

## Blue/green strategy

`spec.strategy.blueGreen` replaces `spec.strategy.canary` in both `deploy/base/services/api/rollout.yaml` and
`deploy/base/services/web/rollout.yaml`:

- `activeService` (`golfapp-api` / `golfapp-web`) keeps serving 100% of real traffic from the old
  ReplicaSet until the new one is promoted - never partial-weighted.
- `previewService` (`golfapp-api-preview` / `golfapp-web-preview`, renamed from the old
  `*-canary` Service/HTTPRoute pair - same resources, clearer name for what they now do) routes
  100% of its traffic to the new ReplicaSet as soon as it's created, for manual inspection via
  `golf-api-preview.<domain>` / `golf-preview.<domain>` before promotion. Outside of a rollout it
  selects the same pods as `activeService`.
- `autoPromotionEnabled: true` plus `prePromotionAnalysis` means promotion happens automatically
  the moment analysis passes - no `pause: {}`, no manual promote. If analysis fails, Argo Rollouts
  aborts automatically and `activeService` never moves.

Two full environments (old + new, both fully scaled) is a simpler mental model than partial-weight
canary steps, and was the explicit starting point requested for this change. A tuned canary
(gradual weight shifting, per-step analysis) is a reasonable next step once the analysis signal
below is proven out, but isn't part of this change.

## Automated promotion policy

`deploy/base/services/api/analysistemplate.yaml` and `deploy/base/services/web/analysistemplate.yaml` define `AnalysisTemplate`s
referenced by each Rollout's `prePromotionAnalysis`. Argo Rollouts injects
`rollouts-pod-template-hash` (via `valueFrom: podTemplateHashValue: Latest`) so every query is
scoped to only the new ReplicaSet's pods, matched on the `pod` label's `<rollout>-<hash>-<random>`
naming convention.

Two metrics, both queried against the cluster's existing `kube-prometheus-stack` (`monitoring`
namespace, `kube-prometheus-stack-prometheus` Service, port 9090 - no changes needed there):

1. **`no-pod-restarts`** - `increase(kube_pod_container_status_restarts_total{...}[2m])` must stay
   at `0`. Any crash, OOM kill, or failed liveness probe on the new pods fails the gate.
2. **`pods-ready`** - `kube_pod_status_ready{condition="false", ...}` must stay at `0`. Catches a
   version that starts but never becomes ready (e.g. can't reach postgres or Dex) without waiting
   for a full crash/restart cycle.

Both sample every 30s, 4 times (a ~2 minute observation window), `failureLimit: 1` - any single bad
sample aborts the rollout immediately. This is deliberately conservative and coarse: it catches
"the new version is visibly broken," not subtle regressions. Tightening it (longer windows, more
nuanced thresholds) is future work once there's a track record of it running.

### Known gap: no HTTP-level signal yet

The task that motivated this change specifically asked for "HTTP error rate, p95/p99 latency" as
example signals. golfapp's `api` does not currently expose a Prometheus `/metrics` endpoint (no
`prom-client` or similar - confirmed by reading `apps/api/src/app/*`), and neither does `web`. So
there's no request-level error rate or latency series to query today, only pod/container-level
signals from kube-state-metrics and cAdvisor (already scraped cluster-wide, no `ServiceMonitor`
needed for those).

This is a **blocker that needs a code change inside `golfapp` itself**, out of scope for this repo
per the task constraints. The concrete follow-up: instrument `apps/api` (and optionally `apps/web`'s
nginx) with a Prometheus client library, expose `/metrics`, then add a `ServiceMonitor` here in
`golfapp-platform` (the normal way to opt a workload into the existing Prometheus Operator setup
without touching the `monitoring` stack's own repo) and extend the `AnalysisTemplate`s with
`http_requests_total`/`http_request_duration_seconds`-style queries for real error-rate/latency
gating. Until then, the two metrics above are the real, working starting point.

## Synthetic traffic

`deploy/base/services/synthetic-traffic/deployment.yaml` is a single-replica `Deployment` (not a `CronJob`) that
loops `curl` against `golfapp-api-preview`'s `/api/health` and `/api`, and `golfapp-web-preview`'s
`/`, roughly every 30s per endpoint (~6 requests/minute total). A `Deployment` was chosen over a
`CronJob` because `prePromotionAnalysis` can start at any point in a deploy, and a `CronJob`'s
minute-granularity schedule isn't guaranteed to line up with a given analysis window; a
continuously-running low-rate loop always does.

Why this exists at all: this is a personal homelab app with essentially no real traffic most of the
time, so the two metrics above (restart count, readiness) would otherwise be observing mostly-idle
pods during the analysis window - a bug that only surfaces under actual requests (a crash on a
specific route, a connection pool exhausting under load) wouldn't get exercised in time to block
promotion. Hitting `previewService` specifically (not `activeService`) means this traffic reaches
the new, not-yet-promoted ReplicaSet during the exact window that matters.

`/api/health` and `/api` are the *only* two unauthenticated `GET` routes golfapp's `api` exposes
today (everything else - `/api/courses/*`, `/api/rounds/*`, `/api/matches/*`, `/api/auth/me`, etc.
- requires a JWT via `JwtAuthGuard`). Deeper "core" endpoint coverage would mean either scripting
the `GET /auth/dev-login` bypass (real, and enabled in this cluster's `NODE_ENV=development`
config - see `deploy/base/services/api/rollout.yaml` - but it depends on a demo user being seeded via golfapp's
own `npm run demo:seed`, external state this repo doesn't control; failing that silently would
mean every synthetic request 404s and permanently red the error-rate signal once one exists) or a
golfapp-side change to add a stable, unauthenticated read endpoint for this purpose. Neither is
implemented here - noted as a follow-up rather than built on a fragile assumption.

Every request carries `X-Synthetic-Traffic: true` and `User-Agent: golfapp-synthetic-traffic/1.0`,
so it's trivially excludable from any real-usage dashboard or analytics added later, and so a
future `AnalysisTemplate` query could deliberately include or exclude it.

Resource footprint is intentionally tiny (`5m`/`8Mi` request, `20m`/`32Mi` limit) - this is meant to
produce just enough signal for the rollout analysis above, not load-test the app, on hardware that's
a single Raspberry Pi.

## Manual overrides (still available)

Automation replaces the *need* for manual promotion in the common case, not the ability to
intervene:

```bash
kubectl argo rollouts get rollout golfapp-api -n golf --watch
kubectl argo rollouts abort golfapp-api -n golf      # force-fail an in-progress analysis
kubectl argo rollouts promote golfapp-api -n golf     # force-promote, skipping remaining analysis
kubectl argo rollouts undo golfapp-api -n golf        # roll back a bad promotion
```

## Follow-ups

- Instrument golfapp's `api` (and `web`) with Prometheus metrics and add a real HTTP error-rate/
  latency signal, per "Known gap" above.
- Once that lands, add a `ServiceMonitor` here and extend both `AnalysisTemplate`s.
- Consider a tuned canary strategy (gradual `setWeight` + per-step analysis) once the current
  restart/readiness-based gate has a track record.
- Host-aware OAuth redirects (`AWP-59`, referenced in `deploy/base/services/api/rollout.yaml`) so a
  preview-hosted login doesn't bounce back to the active host - unrelated to this change but noted
  since the preview Service naming makes it more visible.
