# golfapp-platform

Deploy-side gitops repo for [golfapp](https://github.com/atidyshirt/golf-ai-experiment) (a private Nx monorepo with 3 independently-deployed apps: `api`, `web`, `dotgolf-service`). Structure is modeled on
[homelab-app-template](https://github.com/atidyshirt/homelab-app-template)/[homelab-deploy-template](https://github.com/atidyshirt/homelab-deploy-template) — kept as a separate repo from golfapp itself so the pipeline's own image-tag-bump commits never collide with app commits, and so cluster/platform concerns stay decoupled from golfapp's own code and issue tracking.

Synced by the real home-server homelab cluster's ArgoCD, via `addons/addon-golfapp-appset.yaml` in [home-server](https://github.com/atidyshirt/home-server) — a `clusters` generator whose single ArgoCD `Application` points at `deploy/overlays/production`, the aggregating root kustomization. `bootstrap/argocd/projects.yaml`'s `golf-application` `AppProject` scopes it to the `golf` namespace.

## Structure

```
deploy/
  base/
    services/
      api/               golfapp-api: Rollout (blue/green+preview), AnalysisTemplate, Service, preview Service, HTTPRoute, preview HTTPRoute
      web/               golfapp-web: same shape as api/
      dotgolf-service/   golfapp-dotgolf-service: Rollout (still canary), Service, canary Service only -- no
                         HTTPRoute, deliberately: cluster-internal only, called by
                         apps/api's DotGolfMembershipProvider, never exposed publicly
      postgres/          golfapp-postgres: prod StatefulSet + bootstrap job
      synthetic-traffic/ golfapp-synthetic-traffic: low-rate curl-loop Deployment, gives the
                         AnalysisTemplates above enough request volume to have a signal
    mocks/
      postgres/          lightweight postgres:16-alpine shim (emptyDir) for preview envs
  overlays/
    production/          aggregates base/services/{api,web,postgres,synthetic-traffic}; this is what
                         home-server's ArgoCD Application points at
    pr-preview/           aggregates base/services/{api,web} + base/mocks/postgres, patched down to
                         single-replica/canary for per-PR preview namespaces
```

`api` and `web` use Argo Rollouts' `blueGreen` strategy: `activeService`/`previewService` let Argo
Rollouts manage two distinct Services (no service mesh needed) so the in-flight rollout is reachable
on its own hostname (`golf-api-preview.homelab.arpa`, `golf-preview.homelab.arpa`) before promoting.
Promotion itself is automatic - each Rollout's `prePromotionAnalysis` gates on an `AnalysisTemplate`
querying the cluster's Prometheus, and Argo Rollouts promotes (or aborts) on its own. See
[`docs/rollout-automation.md`](docs/rollout-automation.md) for the full design, thresholds, and
known gaps.

```bash
kubectl argo rollouts get rollout golfapp-api -n golf --watch
# ...inspect https://golf-api-preview.homelab.arpa...
kubectl argo rollouts abort golfapp-api -n golf   # only needed to force-fail a stuck/bad rollout
```

Domain templating (`addons_domain: placeholder-domain` + each `kustomization.yaml`'s `replacements` block) mirrors the gitops-bridge pattern used by home-server's own platform addons (`applications/dex/base`, `applications/traefik/base`) — the `golfapp-appset.yaml` ApplicationSet passes the real cluster domain down via `kustomize.commonAnnotations`, overwriting the placeholder at sync time. `kubectl kustomize deploy/base/services/<app>/` still works standalone without that wrapper.

## What changed here

This repo used to bootstrap its own local `kind` cluster + standalone ArgoCD (app-of-appsets, a git-directory generator scanning golfapp's own `apps/*/deploy`). That's retired now that golfapp is wired into the real home-server pipeline end to end (build -> GHCR -> here -> ArgoCD -> automated blue/green rollout). One consequence: golfapp's `Tiltfile` (fast local inner-loop dev) now reads its manifests from `../golfapp-platform/deploy/base/services/{api,web}/` in this repo (sibling checkout) instead of its own `apps/*/deploy` — see golfapp's `Tiltfile` for the up-to-date local dev flow. Local dev still just needs a bare `kind create cluster` + the Argo Rollouts controller installed (Tilt applies everything else directly, bypassing ArgoCD).

## Onboarding a new app in this monorepo

1. Add `deploy/base/services/<app>/` here, modeled on `deploy/base/services/api/` (or `deploy/base/services/dotgolf-service/` if it's cluster-internal only), then add it to `deploy/overlays/production/kustomization.yaml`'s `resources:` list — `golfapp-appset.yaml`'s ArgoCD `Application` picks it up automatically on next sync, no changes needed in home-server.
2. Add a `trigger-<app>.yml` in golfapp's `.github/workflows/`, matching the existing `trigger-api.yml` pattern.
