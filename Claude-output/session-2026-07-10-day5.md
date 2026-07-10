# Session Notes — 2026-07-10 (Day 5, in progress)
**Phase:** Day 5 of 25 — ArgoCD RBAC + rollback + sync waves/hooks + interview scenarios
**Status:** In progress. RBAC done (Rahul-driven). Rollback demo interrupted by a real, unrelated cluster bug — decision pending before finishing it. Sync waves/hooks and interview Q&A not started.
**Environment:** AKS (`aks-devopswithai`, westus2) — migrated off minikube this session/previous session, see below.

---

## What changed this session

**Migrated off minikube onto real AKS:**
- Minikube was causing recurring issues; Rahul owns the Azure subscription, so we moved to a real cluster.
- Reused the existing Terraform module from the DevOpsWithAI project (`Infrastructure/`) rather than writing new IaC.
- Hit a subscription quota wall: `eastus` only had 1 vCPU of regional quota left (everything else defaults to 10) — moved the **AKS cluster** to `westus2`, but kept the **ACR pinned to `eastus`** (ACR location is immutable once created, and it already held real images). Added a new `acr_location` Terraform variable so the two regions can diverge without forcing ACR replacement.
- ArgoCD had to be installed with `kubectl apply --server-side --force-conflicts` — plain `apply` fails on the `applicationsets.argoproj.io` CRD ("annotations too long").
- Cost control: cluster is stopped/started between sessions (`az aks stop` / `az aks start`) rather than left running or destroyed — near-zero compute cost when idle, few-minute restart.
- All 4 Applications (`devopswithai`, `devopswithai-dev/qa/prod`) came back Synced/Healthy after a stop/start cycle — confirms the lifecycle approach works cleanly.

**Process correction (important, applies to all future days):**
- Claude ran the entire RBAC and rollback demos hands-off the first time through — Rahul caught this: *"I have not written a single line."*
- Going forward: **Rahul runs every hands-on command himself.** Claude explains the concept, hands over exact commands/edits, and verifies the result afterward — same pattern already used for the Python learning track. Claude only does things Rahul genuinely can't do more efficiently himself (e.g., the cloud provisioning above).
- RBAC and rollback were reset to a clean baseline and redone with Rahul driving from his own terminal.

---

## Mental Model for Revision

### 1. ArgoCD RBAC — Casbin policy, separate from Kubernetes RBAC

`argocd-rbac-cm` holds a `policy.csv` (Casbin format: `p, subject, resource, action, object, effect` and `g, subject, role` for group binding). Empty by default → everyone falls back to built-in `role:readonly`/`role:admin`.

```yaml
data:
  policy.csv: |
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    p, role:developer, applications, action/*, */*, allow
    g, dev-user, role:developer
  policy.default: role:readonly   # safety-net fallback for unmapped subjects
```

Apply via `kubectl patch configmap argocd-rbac-cm -n argocd --patch-file <file>`, then **must restart argocd-server** — RBAC/account config only reloads on restart, not live.

Local accounts (`kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"accounts.dev-user":"apiKey, login"}}'`) are for demos/break-glass only. In production, policy rows bind to **OIDC group claims** from Dex/SSO instead: `g, oidc-group:platform-team, role:developer`.

**Verified live:** `dev-user` (bound to `role:developer`) could `argocd app sync` (allowed) but got `PermissionDenied` on `argocd app delete` (not granted) — rejected by ArgoCD's own policy engine, with zero involvement of the underlying Kubernetes ServiceAccount's RBAC. **This is the core interview point:** ArgoCD authorization is a separate layer from Kubernetes RBAC entirely.

**Also matters for real deployments:** scope policy rows by `AppProject`, not `*/*` (which we used only for demo simplicity) — e.g. `p, role:team-a-dev, applications, sync, team-a-project/*, allow`, so RBAC (who) and AppProjects (on what) compose independently per team.

### 2. Rollback — GitOps-correct vs imperative, and why self-heal fights you

Flow: bump `k8s/base/auth-deployment.yml` replicas `2→3`, commit, push to `main` → auto-sync applies it → `argocd app history devopswithai` shows both revisions.

```
argocd app rollback devopswithai <old-revision-id>
# → FailedPrecondition: "rollback cannot be initiated when auto-sync is enabled"
```

ArgoCD **refuses** the rollback outright when automated sync is on — it knows self-heal would immediately re-apply Git's state and undo the rollback, so it blocks the footgun instead of letting you fight yourself.

To force it anyway: `argocd app set <app> --sync-policy none`, then rollback succeeds — but the app immediately flips to `OutOfSync`, because Git's HEAD still has the "bad" state. You've traded a bad deploy for **drift**. In an actively-shipping repo (CI landing commits in the background), that drift window compounds fast — during the first run-through, a routine CI image-tag commit landed mid-demo and pushed *every* deployment out of sync, not just the one we touched.

**Correct fix:** `git revert <bad-commit>`, push, then re-enable `--sync-policy automated --auto-prune --self-heal`. Git becomes the source of truth again; ArgoCD converges everything with zero manual state-fighting, and the revert is a permanent audit trail — `argocd app rollback` isn't.

**One-line interview answer:** *"With GitOps + self-heal, you don't roll back the cluster — you roll back Git, and let reconciliation do the rest. `argocd app rollback` is a break-glass tool for when you must move faster than a revert+CI cycle allows, and ArgoCD requires you to deliberately pause automation before it'll even let you try."*

### 3. Bug discovered mid-exercise: single-replica Deployment + ReadWriteOnce PVC + RollingUpdate = deadlock

While redoing the rollback demo, `db-deployment` got stuck in `Progressing` forever. Root cause (`kubectl describe pod` on the new pod):

```
Warning  FailedAttachVolume  Multi-Attach error for volume "pvc-...":
         Volume is already used by pod(s) db-deployment-bc68b97d-cqzz9
```

- `db-deployment` uses the default `RollingUpdate` strategy: bring the new pod up *before* killing the old one.
- The PVC is an Azure Disk — `ReadWriteOnce`, attachable to only one node at a time.
- New pod can never start (disk still held by old pod); old pod never gets killed (Deployment controller waits for new pod to be `Ready` first under RollingUpdate). **Permanent deadlock**, not a transient blip.
- This was a latent bug the whole time — it just hadn't been triggered because nothing had recently forced a rollout of `db-deployment` specifically.

**Fix (not yet applied — decision pending):** add `strategy: type: Recreate` under `spec:` in `db-deployment.yml`. Tells Kubernetes to kill the old pod first, so the disk is free when the new one needs it. Correct long-term answer for a real stateful service is a `StatefulSet`, but `Recreate` on a `Deployment` is the standard pragmatic fix for a single-replica database.

**Immediate unblock options (either doesn't fix root cause except the Recreate edit):**
```bash
# Quick, temporary:
kubectl delete pod db-deployment-bc68b97d-cqzz9 -n default

# Durable fix — edit k8s/base/db-deployment.yml:
spec:
  strategy:
    type: Recreate
  replicas: 1
  ...
```

---

## Key CLI Commands

```bash
# RBAC
kubectl get configmap argocd-rbac-cm -n argocd -o yaml
kubectl patch configmap argocd-rbac-cm -n argocd --patch-file /tmp/rbac-patch.yaml
kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"accounts.dev-user":"apiKey, login"}}'
kubectl rollout restart deployment argocd-server -n argocd
argocd account update-password --account dev-user --new-password '<pw>' --current-password '<admin-pw>'
argocd app sync devopswithai-qa            # allowed for role:developer
argocd app delete devopswithai-qa --yes    # denied for role:developer

# Rollback
argocd app history devopswithai
argocd app rollback devopswithai <id>                       # blocked if auto-sync on
argocd app set devopswithai --sync-policy none               # required before manual rollback
argocd app rollback devopswithai <id>                         # now succeeds, causes drift
git revert --no-edit <bad-commit-sha>
argocd app set devopswithai --sync-policy automated --auto-prune --self-heal

# AKS lifecycle (cost control between sessions)
az aks stop  -g rg-devopswithai -n aks-devopswithai
az aks start -g rg-devopswithai -n aks-devopswithai
az aks get-credentials -g rg-devopswithai -n aks-devopswithai --overwrite-existing
```

---

## Interview Questions This Session Covers

**"How would you set up RBAC for a team of 20 engineers on ArgoCD?"**
→ Define custom roles in `argocd-rbac-cm` policy.csv (e.g. `role:developer` = sync but not delete), bind them to OIDC group claims from SSO (not local accounts — those are for break-glass only), scope permissions per `AppProject` so teams can only act on their own apps, and set `policy.default: role:readonly` so anyone unmapped defaults to safe read-only rather than silently getting nothing or everything.

**"How do you handle a bad deploy at 2am with GitOps?"**
→ Don't fight the cluster — revert Git and let auto-sync + self-heal reconcile. `argocd app rollback` exists for emergencies but ArgoCD blocks it while auto-sync is on (because self-heal would instantly undo it), and even forcing it just creates drift until Git matches the cluster again. The durable, audited fix is always `git revert` + push.

**"What's a common failure mode for stateful workloads on Kubernetes?"**
→ Single-replica Deployment backed by a `ReadWriteOnce` PVC, using the default `RollingUpdate` strategy — new pod can't attach the disk while the old pod holds it, old pod won't terminate until the new one is Ready → permanent deadlock. Fix: `strategy: type: Recreate` (kill-then-start) for a pragmatic Deployment-based fix, or move to a `StatefulSet` for the correct long-term architecture.

---

## Pending — continue Day 5 next session

1. **Decide + apply the `db-deployment` fix** (quick pod-delete unblock vs. durable `Recreate` strategy — leaning durable) before finishing the rollback exercise (the `git revert` + re-enable-auto-sync steps haven't run yet on this redo).
2. **Sync waves demo:** annotate `configmap.yml`, `secret.yml`, `db-pvc.yml`, `db-deployment.yml`, `db-service.yml` with `argocd.argoproj.io/sync-wave: "-1"` so the DB tier applies before the app tier (wave `0`, default).
3. **Sync hooks demo:** add a `PreSync` hook `Job` (`k8s/base/db-migration-hook.yml`) using `postgres:16` to simulate a schema migration against `db-service` before the main sync — annotations `argocd.argoproj.io/hook: PreSync` + `hook-delete-policy: HookSucceeded`.
4. **Interview scenario Q&A:** "GitOps for a team of 20 engineers," "ArgoCD vs Flux."

All of the above: Rahul drives the commands/edits, Claude explains + reviews.
