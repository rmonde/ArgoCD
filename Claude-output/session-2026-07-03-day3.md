# Session Notes — 2026-07-03 (Day 3)
**Phase:** Day 3 of 25 — GitHub Actions CI pipeline wired to ArgoCD
**Status:** Complete. Full end-to-end GitOps loop verified working.
**Environment:** minikube, ArgoCD v3.4.4, devopswithai app (auto-sync + self-heal + prune)

---

## Mental Model for Revision

### 1. The CI/CD split — who does what

```
Developer pushes code
    → GitHub Actions (CI):
        build image → push to ACR → update k8s/*.yml with new SHA → commit to main [skip ci]

    → ArgoCD (CD):
        detects Git commit → auto-syncs cluster → kubectl apply
```

**Key rule:** CI never touches the cluster. ArgoCD never builds images. They communicate only through Git.

---

### 2. Why [skip ci] is mandatory on the manifest update commit

The `update-manifests` job commits back to main. Without `[skip ci]`, GitHub sees a push to main and re-triggers the entire workflow — rebuilding the same images with the same SHA. That bot commit triggers another bot commit, creating an **infinite loop**.

`[skip ci]` is a GitHub Actions convention that suppresses workflow triggers on that commit.

---

### 3. Multi-arch image fix (carried over from Day 2 blocker)

**Problem:** Images built for `linux/amd64`, minikube on Apple Silicon needs `linux/arm64`.
**Error:** `no matching manifest for linux/arm64/v8 in the manifest list entries`

**Fix:** Added to both build steps in `.github/workflows/docker-build-push-ci.yml`:
```yaml
platforms: linux/amd64,linux/arm64
```

**How it works:**
- `docker/setup-buildx-action` registers QEMU emulation via `binfmt_misc`
- GitHub's Intel runner emulates ARM64 using QEMU to build the arm64 binary
- Both binaries pushed to ACR under the same tag as a **multi-arch manifest**
- minikube pulls the arm64 variant automatically; Intel servers pull amd64

**Production note:** QEMU is 4–10x slower than native. Production teams use self-hosted ARM64 runners.

---

### 4. ArgoCD CLI session expiry

ArgoCD CLI tokens expire after 24 hours. Fix:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
argocd login localhost:8080 --username admin \
  --password $(argocd admin initial-password -n argocd | head -1) --insecure
```

---

### 5. Rollback with ArgoCD

**Fast rollback during incident:**
```bash
argocd app history devopswithai          # list revisions
argocd app rollback devopswithai <id>    # sync cluster to that revision
```

**Problem:** Cluster is now ahead of Git — violates GitOps principle. Git is no longer source of truth.

**GitOps-correct rollback:**
```bash
git revert HEAD
git push origin main
# ArgoCD auto-syncs to reverted commit
```

**Interview answer:** Use `argocd app rollback` for speed during the incident. Immediately follow with `git revert` to restore Git as source of truth and create an audit trail.

---

## Key CLI Commands

```bash
# Watch cluster pods during deploy
kubectl get pods -w

# Watch ArgoCD sync in real time
argocd app get devopswithai --watch

# View deployment history
argocd app history devopswithai

# Rollback to a previous revision
argocd app rollback devopswithai <revision-id>
```

---

## Interview Questions This Session Covers

**"Walk me through your CI/CD pipeline."**
→ GitHub Actions handles CI: build image, push to ACR, update the image SHA in the k8s manifest files, commit back to Git with [skip ci]. ArgoCD handles CD: it watches the Git repo, detects the new commit, and auto-syncs the cluster. No manual kubectl. No manual argocd sync. Git is the only control plane.

**"Why does the manifest update commit include [skip ci]?"**
→ Without it, the bot's commit to main re-triggers the GitHub Actions workflow. That rebuild pushes the same image, makes another manifest commit, triggers the workflow again — infinite loop. [skip ci] tells GitHub Actions to skip workflow triggers on that commit.

**"Your team deployed a bad image. How do you recover with GitOps?"**
→ During the incident: `argocd app rollback devopswithai <id>` — syncs the cluster to the last known good state immediately. Then follow up with `git revert HEAD && git push` to restore Git as source of truth. The rollback then becomes an auditable Git commit, not just a CLI action.

**"Why doesn't CI run kubectl apply directly?"**
→ Direct kubectl from CI loses the audit trail, drift detection, and self-heal guarantees. If CI applies manifests directly, ArgoCD has no record of it — the cluster can drift without detection. In GitOps, the pipeline writes to Git; ArgoCD is the only thing that touches kubectl. Every cluster change is traceable to a Git commit.

---

## Day 4 Plan
- ArgoCD ApplicationSet — generate apps from templates (one per env: dev/staging/prod)
- Multi-environment promotion flow
- Image Updater concept
