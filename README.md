# argocd-apps

GitOps configuration repository for a home Kubernetes cluster (Talos). All workloads are
declared as YAML manifests and managed by [ArgoCD](https://argo-cd.readthedocs.io/) using
the **App of Apps** pattern.

There is no application code in this repo — only Kubernetes manifests, Helm values, and
Kustomize overlays.

---

## Repository structure

```
argocd-apps/
├── argocd/              # Helm umbrella chart — ArgoCD manages itself
│   ├── Chart.yaml
│   ├── Chart.lock
│   └── values.yaml
│
├── apps/                # One directory per application
│   └── <app-name>/
│       └── overlays/
│           └── <cluster>/
│               ├── application.yaml   # ArgoCD Application CR
│               └── kustomization.yaml
│
└── clusters/            # Per-cluster bootstrap entry points
    ├── home/
    │   ├── app-of-apps.yaml   # Bootstrap — apply once manually
    │   └── kustomization.yaml # Lists all apps for this cluster
    └── prod/            # Not yet configured
```

---

## How it works

1. `clusters/home/app-of-apps.yaml` is applied **once manually** to bootstrap the cluster.
2. ArgoCD then watches `clusters/home/` via Kustomize, which references every app under
   `apps/*/overlays/home/`.
3. Each app directory contains an ArgoCD `Application` CR that tells ArgoCD where to find
   the app's source (Helm chart, Kustomize overlay, plain manifests, etc.).
4. From that point on, every `git push` to `main` is automatically reconciled by ArgoCD —
   no manual `kubectl apply` needed.

---

## Bootstrap a cluster (one-time)

```sh
# 1. Make sure kubectl points at the target cluster
kubectl config current-context

# 2. Create the argocd namespace if it does not exist yet
kubectl create namespace argocd

# 3. Install ArgoCD itself (only needed before the first sync)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 4. Apply the App of Apps to hand control over to GitOps
kubectl apply -f clusters/home/app-of-apps.yaml
```

After step 4, ArgoCD takes over and manages everything in this repo, including its own
Helm deployment (`apps/argocd/`).

---

## Deploying a new application

Follow these three steps every time you add an app to the cluster.

### 1 — Create the ArgoCD Application manifest

```
apps/<app-name>/overlays/home/application.yaml
```

```yaml
---
# <App name> — <one-line description of what this app is and why it exists>
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:SimonJaricot/argocd-apps.git
    targetRevision: HEAD
    path: apps/<app-name>/overlays/home   # or a Helm chart path, etc.
  destination:
    server: https://kubernetes.default.svc
    namespace: <target-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

> **Note:** Namespaces must be created outside of GitOps — do not set `CreateNamespace=true`.

### 2 — Create the Kustomize entry point

```
apps/<app-name>/overlays/home/kustomization.yaml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - application.yaml
```

### 3 — Register the app in the cluster's kustomization

Add a line to `clusters/home/kustomization.yaml`:

```yaml
resources:
  - ../../apps/argocd/overlays/home
  - ../../apps/<app-name>/overlays/home   # add this line
```

### Done

Push to `main`. ArgoCD detects the new `Application` CR on the next sync (or immediately
if a webhook is configured) and begins deploying the app.

---

## Local validation

```sh
# Lint the ArgoCD Helm chart
helm lint argocd/

# Preview rendered Kustomize output
kubectl kustomize clusters/home/
kubectl kustomize apps/<app-name>/overlays/home/

# Preview rendered Helm output
helm template argocd argocd/ -f argocd/values.yaml

# Update Helm dependencies after changing Chart.yaml
helm dependency update argocd/
```
