# Session Notes — 2026-07-01 (Day 2)
**Phase:** Day 2 of 25 — Sync policies + App of Apps
**Status:** Complete. All three sync behaviours demonstrated live. App of Apps working.
**Environment:** minikube, ArgoCD v3.4.4, devopswithai app

---

## What we fixed before Day 2

**Problem:** Images in ACR were built for `linux/amd64`. minikube on Apple Silicon (M-series Mac) needs `linux/arm64`. Error: `no matching manifest for linux/arm64/v8`.

**Fix:** Added `platforms: linux/amd64,linux/arm64` to both build steps in `.github/workflows/docker-build-push-ci.yml`.

**How it works:** `docker/setup-buildx-action` registers QEMU emulation handlers via `binfmt_misc`. The GitHub runner (Intel) emulates ARM64 to build the arm64 binary. Both binaries are pushed to ACR under the same tag as a **multi-arch manifest**. minikube pulls the arm64 variant automatically.

**Production note:** QEMU emulation is 4–10x slower than native. Production teams use self-hosted ARM64 runners (AWS Graviton, Mac runners) to avoid this penalty.

---

## Mental Model for Revision

### 1. Three sync behaviours — how they differ

| Flag | Trigger | Without it |
|---|---|---|
| `--sync-policy automated` | New Git commit detected | Must run `argocd app sync` manually |
| `--self-heal` | Cluster-side drift (kubectl edit/scale/delete) | ArgoCD shows OutOfSync but does NOT correct it |
| `--auto-prune` | Manifest deleted from Git | Orphaned resource keeps running in cluster |

**Key distinction:** automated sync fires on Git changes. self-heal fires on cluster changes. They handle different triggering conditions.

**Why ArgoCD detects drift so fast:** Uses the Kubernetes **watch API** (same as `kubectl get -w`). application-controller maintains a live watch on all managed resources. When `kubectl scale` fires, the API server sends a watch event immediately — no polling needed.

### 2. Self-heal demo (what happened)

```bash
kubectl scale deployment auth-deployment --replicas=3
# replicas jump to 3
# ArgoCD receives watch event → compares 3 vs Git's 2 → applies correction
# replicas back to 2 within seconds
```

**Interview answer:** "Git is the only durable way to change cluster state. Direct kubectl access is effectively neutralised by self-heal."

### 3. Auto-prune demo (what happened)

```bash
git rm k8s/books-service.yml
git commit -m "demo: remove books-service to test auto-prune"
git push origin main
# ArgoCD detects Git change → books-service deleted from cluster

git revert HEAD
git push origin main
# ArgoCD detects Git change → books-service recreated in cluster
```

**Without auto-prune:** Resource becomes an orphan — running in cluster, not in Git, not deleted. To force-delete: `argocd app sync devopswithai --prune`. Auto-prune is intentionally opt-in — accidental deletion of a production DB from a manifest removal would be catastrophic.

### 4. App of Apps pattern

**Problem it solves:** ArgoCD Application objects themselves aren't in Git. Created manually via `argocd app create`. No audit trail. Cluster rebuild = manual recreation.

**Solution:** Store Application YAMLs in Git. One parent Application watches the folder and applies them.

```
parent-app  (watches k8s/argocd/)
├── namespace: argocd          ← namespace.yml
└── Application: devopswithai  ← app-application.yml
        └── (watches k8s/)
            ├── auth-deployment, books-deployment, ...
```

**Full cluster recovery from scratch:**
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.4/manifests/install.yaml
argocd app create parent-app --repo https://github.com/rmonde/DevOpsWithAI.git \
  --path k8s/argocd --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd --sync-policy automated --self-heal
# Everything else rebuilt from Git automatically
```

**Footgun to know:** `namespace.yml` defines the argocd namespace. If deleted from Git with auto-prune enabled on parent-app, ArgoCD deletes its own namespace. In production, exclude the argocd namespace from prune.

---

## Key CLI Commands

```bash
# Enable auto-sync + self-heal + prune
argocd app set devopswithai \
  --sync-policy automated \
  --self-heal \
  --auto-prune

# Verify sync policy in raw spec
argocd app get devopswithai -o yaml | grep -A5 syncPolicy

# Create parent app (App of Apps)
argocd app create parent-app \
  --repo https://github.com/rmonde/DevOpsWithAI.git \
  --path k8s/argocd \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --sync-policy automated \
  --self-heal

# List all apps
argocd app list
```

---

## Interview Questions This Session Covers

**"What is the difference between auto-sync and self-heal in ArgoCD?"**
→ Auto-sync fires when ArgoCD detects a new Git commit — it syncs the cluster to the new desired state automatically. Self-heal fires when the cluster drifts from Git due to direct kubectl changes — it corrects the cluster back to Git. Two different triggering conditions: Git changes vs cluster changes.

**"How does ArgoCD detect drift so quickly?"**
→ ArgoCD uses the Kubernetes watch API. The application-controller holds a live watch on all managed resources. When kubectl changes a resource, the API server fires a watch event immediately. No polling interval — ArgoCD reacts in real time.

**"What happens if you delete a manifest from Git without auto-prune?"**
→ The resource becomes an orphan — still running in the cluster but no longer tracked by Git or ArgoCD. The app shows OutOfSync but ArgoCD will not delete the resource. You'd need to run `argocd app sync --prune` explicitly to remove it.

**"What is the App of Apps pattern and why do you need it?"**
→ Without it, ArgoCD Application objects are created manually — no Git history, no audit trail, hard to recover. App of Apps stores Application YAMLs in Git and uses one parent Application to manage them. Adding a service = adding a YAML file to Git. Full cluster recovery requires only two commands.

---

## Day 3 Plan
- GitHub Actions CI pipeline
- Wire CI output (image SHA) to Git manifest update
- ArgoCD detects manifest change and deploys new image
