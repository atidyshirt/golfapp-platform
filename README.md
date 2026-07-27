# golfapp-platform

Local kind cluster + ArgoCD bootstrap for [golfapp](https://github.com/atidyshirt/golf-ai-experiment). Kept as a separate repo from golfapp itself so cluster/platform concerns stay fully decoupled from the app's own code and issue tracking — golfapp's monorepo may get split apart later, and this repo shouldn't care either way.

Modeled on the bootstrap shape already proven out in [atidyshirt/homelab-argocd](https://github.com/atidyshirt/homelab-argocd) (`argocd-bootstrap/` + a root "app of apps" `Application`), adapted for a local kind cluster instead of k3s/1Password, and one level up — the root `Application` here manages `ApplicationSet`s rather than individual `Application`s directly, so each app in golfapp is auto-discovered instead of hand-declared.

## Structure

```
kind/kind-config.yaml               # local cluster config
argocd-bootstrap/                   # one-time, manually-applied - installs ArgoCD itself
  namespace.yaml
  kustomization.yaml                 # pulls the official ArgoCD install manifest
argocd/
  bootstrap.yaml                     # the one other manually-applied resource - root "app of apps"
  applicationsets/
    platform-components.yaml         # cluster-wide deps (Argo Rollouts controller for now)
    golfapp.yaml                     # git directory generator -> one ArgoCD Application per app in golfapp
```

## Bootstrap flow

```sh
# 1. Create the local cluster
kind create cluster --config kind/kind-config.yaml

# 2. Install ArgoCD (one-time, not GitOps-managed - can't deploy itself from nothing)
kubectl apply -k argocd-bootstrap/
kubectl -n argocd wait --for=condition=available --timeout=300s deployment/argocd-server

# 3. Register credentials for golfapp's private repo, so the golfapp
#    ApplicationSet can actually clone it (do this before step 4, or the
#    ApplicationSet will just fail to sync until you do)
kubectl -n argocd exec -it deploy/argocd-repo-server -- argocd repo add \
  git@github.com:atidyshirt/golf-ai-experiment.git \
  --ssh-private-key-path /path/to/your/deploy-key
# (or use `argocd repo add` from the argocd CLI against a port-forwarded
# argocd-server, or a Secret of type `Opaque` with the
# `argocd.argoproj.io/secret-type: repository` label - whatever's easiest
# for your machine)

# 4. Apply the root Application - from here on, everything is pure GitOps
#    driven by commits to this repo
kubectl apply -f argocd/bootstrap.yaml
```

After step 4, ArgoCD syncs `argocd/applicationsets/*.yaml`, which in turn produce:
- `platform-argo-rollouts` (an `Application` deploying the Argo Rollouts controller via its Helm chart)
- `golfapp-api` / `golfapp-web` (one `Application` per `apps/*/deploy` folder found in golfapp, via the git directory generator)

## Explicitly out of scope for now

Istio, Gateway API, user-segment canary/blue-green routing, org-level tenant isolation, NATS/messaging traffic-management. These become a future epic here once this base is proven out — see golfapp's AWP-11 for the fuller rationale.

## Local dev loop

This repo only gets you a cluster with ArgoCD running. For fast local iteration on golfapp itself once the cluster exists, see golfapp's own `Tiltfile` — it applies straight into this cluster, bypassing ArgoCD entirely so the two never fight over reconciling the same objects.
