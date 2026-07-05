# Session Notes — 2026-07-05 (Day 4)
**Phase:** Day 4 of 25 — ArgoCD ApplicationSet + multi-environment promotion
**Status:** Complete. ApplicationSet working with 4 apps (default/dev/qa/prod). Pending: fix degraded env apps + image-tag promotion demo.
**Environment:** minikube, ArgoCD v3.4.4

---

## What we fixed this session

**ApplicationSet CRD missing:**
- `argocd-applicationset-controller` was in CrashLoopBackOff since Day 1 — CRD `applicationsets.argoproj.io` was never registered
- Fix: `curl -sLo /tmp/argocd-install.yaml https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.4/manifests/install.yaml && kubectl apply --server-side -f /tmp/argocd-install.yaml`
- `--server-side` avoids annotation size limit failures common with large CRDs

**GitHub Actions workflow path fix:**
- After restructuring `k8s/` to base+overlays, `update-manifests` job was still referencing `k8s/auth-deployment.yml`
- Fixed: all `sed` paths and `git add` updated to `k8s/base/*.yml`

---

## Mental Model for Revision

### 1. Kustomize base + overlays

**Problem with duplicating full manifests per environment:** changing a port = editing 3 files, missing one causes permanent environment drift.

**Kustomize pattern:**
```
k8s/
├── base/                        ← shared manifests, no namespace set
│   ├── auth-deployment.yml
│   ├── kustomization.yaml       ← lists all resources
└── overlays/
    ├── dev/
    │   └── kustomization.yaml   ← resources: ../../base + namespace: dev + replica patches
    ├── qa/
    │   └── kustomization.yaml   ← resources: ../../base + namespace: qa + replica patches
    └── prod/
        └── kustomization.yaml   ← resources: ../../base + namespace: prod + replica patches
```

ArgoCD auto-detects `kustomization.yaml` and runs `kustomize build` before applying. No extra config needed.

**Verify before deploying:**
```bash
kustomize build k8s/overlays/dev | grep -E "replicas|namespace"
```

**Important:** Do NOT set `namespace:` in base — overlays should own namespace assignment.

### 2. ApplicationSet — List generator

One ApplicationSet YAML generates one ArgoCD Application per list element:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: devopswithai-envs
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
          - env: qa
          - env: prod
  template:
    metadata:
      name: "devopswithai-{{env}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/rmonde/DevOpsWithAI.git
        targetRevision: main
        path: "k8s/overlays/{{env}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{env}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

**Adding a new environment** = add one entry to `elements:` + create `k8s/overlays/<env>/` in Git. No manual ArgoCD commands.

### 3. Complete hierarchy after Day 4

```
parent-app  (k8s/argocd/)
├── namespace: argocd
├── Application: devopswithai         → k8s/base/        → default namespace
└── ApplicationSet: devopswithai-envs
        ├── Application: devopswithai-dev   → k8s/overlays/dev  → dev namespace (1 replica)
        ├── Application: devopswithai-qa    → k8s/overlays/qa   → qa namespace (2 replicas)
        └── Application: devopswithai-prod  → k8s/overlays/prod → prod namespace (3 replicas)
```

### 4. Multi-environment promotion patterns

| Pattern | Gate | Trade-off |
|---|---|---|
| Manual sync on prod | Human runs `argocd app sync devopswithai-prod` | Simple but relies on discipline |
| Branch-based | PR merge from develop → main | Clean audit trail; risk of branch divergence |
| Image tag per overlay | CI updates dev overlay SHA; promotion PR updates prod SHA | Most precise; no branch management |

**Branch-based flow:**
```
Feature PR → develop → auto-sync dev + qa
    → Lead validates → PR develop → main
    → auto-sync prod
```

**Image tag per overlay flow (TODO — demo pending):**
```
CI builds image → updates k8s/overlays/dev/kustomization.yaml images: newTag
    → ArgoCD auto-deploys to dev
    → validated in dev
    → promotion PR copies same SHA to k8s/overlays/prod/kustomization.yaml
    → PR merge → ArgoCD auto-deploys to prod
```

Uses Kustomize `images:` transformer in overlay:
```yaml
images:
  - name: devopswithai.azurecr.io/auth-service
    newTag: <sha>
```

### 5. Degraded apps — root cause + fix

**Symptom:** dev/qa/prod apps Synced + Degraded. `db-deployment` → `ImagePullBackOff`. auth/books/user → `Init:0/1`.

**Root cause chain:**
1. `acr-secret` (imagePullSecret for ACR) only exists in `default` namespace — not copied to `dev`/`qa`/`prod`
2. DB pod can't pull image → `ImagePullBackOff`
3. DB never starts → `wait-for-db` initContainer loops forever → app pods stuck in `Init:0/1`

**Fix:**
```bash
for ns in dev qa prod; do
  kubectl get secret acr-secret -n default -o yaml | \
    sed "s/namespace: default/namespace: $ns/" | \
    kubectl apply -f -
done
```

**Production solution:** External Secrets Operator (Day 14) — pulls from Azure Key Vault into every namespace automatically. Or Kubernetes Reflector operator to mirror secrets across namespaces.

---

## Key CLI Commands

```bash
# Verify Kustomize overlay output before deploying
kustomize build k8s/overlays/dev | grep -E "replicas|namespace|image"

# Apply ApplicationSet
kubectl apply -f k8s/argocd/appset-devopswithai.yaml

# Check ApplicationSet status
kubectl get applicationset devopswithai-envs -n argocd -o yaml | grep -A10 "elements"

# Fix acr-secret across namespaces
for ns in dev qa prod; do
  kubectl get secret acr-secret -n default -o yaml | \
    sed "s/namespace: default/namespace: $ns/" | \
    kubectl apply -f -
done
```

---

## Interview Questions This Session Covers

**"How do you manage multiple environments with ArgoCD?"**
→ Kustomize base + overlays: shared manifests in base, environment-specific patches (replicas, resource limits) in overlays. ApplicationSet with a List generator creates one ArgoCD Application per environment from a single template. Adding an environment = one overlay directory + one list entry in Git.

**"How do you promote a deployment from dev to prod?"**
→ Three patterns: manual sync (prod has manual sync policy, human approves), branch-based (dev watches develop branch, prod watches main — PR merge is the gate), or image tag per overlay (CI writes new SHA to dev overlay, validation passes, promotion PR writes same SHA to prod overlay). Image tag per overlay is most precise at scale — Git history shows exactly what SHA is in each environment.

**"What's the difference between Synced+Degraded vs OutOfSync+Healthy?"**
→ Synced+Degraded: manifest applied correctly from Git, but the running resource is unhealthy (pods crashing, image pull failing). OutOfSync+Healthy: resource is running fine but its config in the cluster differs from Git — someone edited it directly.

---

## Pending Before Day 5

1. **Fix degraded apps:** run `acr-secret` copy command across dev/qa/prod namespaces
2. **Image tag per overlay promotion demo:** add `images:` transformer to overlays, update CI to write SHA to dev overlay only, demo promotion PR to prod

---

## Day 5 Plan
- ArgoCD RBAC
- Rollback via `argocd app history` + `argocd app rollback`
- Sync waves and sync hooks (resource ordering)
- Interview scenarios: "bad deploy at 2am", "GitOps for 20 engineers", "ArgoCD vs Flux"
