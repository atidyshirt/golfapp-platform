# golfapp-platform

Deploy-side gitops repo for [golfapp](https://github.com/atidyshirt/golf-ai-experiment) (a private Nx monorepo with 3 independently-deployed apps: `api`, `web`, `dotgolf-service`). Structure is modeled on
[homelab-app-template](https://github.com/atidyshirt/homelab-app-template)/[homelab-deploy-template](https://github.com/atidyshirt/homelab-deploy-template) — kept as a separate repo from golfapp itself so the pipeline's own image-tag-bump commits never collide with app commits, and so cluster/platform concerns stay decoupled from golfapp's own code and issue tracking.

Synced by the real home-server homelab cluster's ArgoCD, via `projects/golfapp-appset.yaml` in [home-server](https://github.com/atidyshirt/home-server) — a git directory generator that turns each `deploy/*` folder here into its own ArgoCD `Application`. `bootstrap/argocd/projects.yaml`'s `golf-application` `AppProject` scopes all of them to the `golf` namespace.

## Structure

```
deploy/
  api/               golfapp-api: Rollout (canary+preview), Service, canary Service, HTTPRoute, canary HTTPRoute
  web/               golfapp-web: same shape as api/
  dotgolf-service/   golfapp-dotgolf-service: Rollout, Service, canary Service only -- no
                     HTTPRoute, deliberately: cluster-internal only, called by
                     apps/api's DotGolfMembershipProvider, never exposed publicly
```

Each app's `canaryService`/`stableService` pair lets Argo Rollouts manage two distinct Services (no service mesh needed) so the in-flight canary is reachable on its own hostname
(`golfapp-api-canary.homelab.arpa`, `golfapp-web-canary.homelab.arpa`) before promoting:

```bash
kubectl argo rollouts get rollout golfapp-api -n golf --watch
# ...inspect https://golfapp-api-canary.homelab.arpa...
kubectl argo rollouts promote golfapp-api -n golf
```

Domain templating (`addons_domain: placeholder-domain` + each `kustomization.yaml`'s `replacements` block) mirrors the gitops-bridge pattern used by home-server's own platform addons (`applications/dex/base`, `applications/traefik/base`) — the `golfapp-appset.yaml` ApplicationSet passes the real cluster domain down via `kustomize.commonAnnotations`, overwriting the placeholder at sync time. `kubectl kustomize deploy/<app>/` still works standalone without that wrapper.

## What changed here

This repo used to bootstrap its own local `kind` cluster + standalone ArgoCD (app-of-appsets, a git-directory generator scanning golfapp's own `apps/*/deploy`). That's retired now that golfapp is wired into the real home-server pipeline end to end (build -> GHCR -> here -> ArgoCD -> canary). One consequence: golfapp's `Tiltfile` (fast local inner-loop dev) now reads its manifests from `../deploy/{api,web}/` in this repo (sibling checkout) instead of its own `apps/*/deploy` — see golfapp's `Tiltfile` for the up-to-date local dev flow. Local dev still just needs a bare `kind create cluster` + the Argo Rollouts controller installed (Tilt applies everything else directly, bypassing ArgoCD).

## Onboarding a new app in this monorepo

1. Add `deploy/<app>/` here, modeled on `deploy/api/` (or `deploy/dotgolf-service/` if it's cluster-internal only) — `golfapp-appset.yaml`'s git directory generator picks it up automatically, no changes needed in home-server.
2. Add a `trigger-<app>.yml` in golfapp's `.github/workflows/`, matching the existing `trigger-api.yml` pattern.
