## Troubleshooting Argo CD sync failures

How to diagnose an `OutOfSync`, `SyncFailed`, or stuck-`Progressing` Application in this
cluster. Work top to bottom — the triage step usually points straight at the section you need.

### Prerequisites

- `kubectl` access to the cluster (over Tailscale SSH to `oci-cp`, or a local kubeconfig)
- `argocd` CLI, logged in — or use the UI at `https://argocd.<tailnet>.ts.net`
- Repo layout context: `bootstrap/applications/` holds the child `Application` manifests
  that the `cluster-bootstrap` app-of-apps renders with `recurse: true`. Each component's
  real manifests live under `infrastructure/`, `platform/`, or `bootstrap/argocd/`.

---

### 1. Triage — get the actual error

```bash
# every app and its state
kubectl -n argocd get applications

# the failing app, with the real message
argocd app get <app>                     # Conditions + per-resource status
kubectl -n argocd get application <app> -o yaml | yq '.status'

# controller's view of why the sync failed
kubectl -n argocd logs statefulset/argocd-application-controller --tail=100 | grep -i <app>

# manifest generation errors (bad kustomize/helm render) show up here
kubectl -n argocd logs deploy/argocd-repo-server --tail=100 | grep -i error
```

Read `.status.operationState.message` and `.status.conditions`. Match the message to a
section below.

---

### 2. Manifest generation fails (`ComparisonError`, `rpc error: ... exit status 1`)

The repo-server can't render the source. Nothing gets applied.

**`must specify --enable-helm`** — a `kustomization.yaml` uses `helmCharts:` but the build
flag is missing. This is set globally in `bootstrap/argocd/values.yaml`:

```yaml
configs:
  cm:
    kustomize.buildOptions: --enable-helm
```

Confirm it landed:

```bash
kubectl -n argocd get cm argocd-cm -o jsonpath='{.data.kustomize\.buildOptions}'
```

If it's empty, the self-managed `argocd` app hasn't synced the value yet. Sync it (section 8)
or patch `argocd-cm` by hand as a stopgap, then restart the repo-server:

```bash
kubectl -n argocd rollout restart deploy/argocd-repo-server
```

**`Helm chart not found` / repo 404** — the chart `repo:` URL or `version:` in the
component's `kustomization.yaml` is wrong. Verify the version exists:

```bash
helm show chart <chart> --repo <repo-url> --version <version>
```

**Reproduce any render locally** before blaming Argo:

```bash
kustomize build --enable-helm <path>
```

---

### 3. Repo access fails (`authentication required`, `permision denied`, `could not resolve host`)

Every source uses `repoURL: git@github.com:WilliamRSmith99/homecloud-deploy` over SSH.

```bash
# is the SSH repo credential registered?
argocd repo list
kubectl -n argocd get secret -l argocd.argoproj.io/secret-type=repository
```

If missing or the deploy key was rotated, re-add it:

```bash
argocd repo add git@github.com:WilliamRSmith99/homecloud-deploy \
  --ssh-private-key-path <path-to-deploy-key>
```

Then confirm the key is a **read-only deploy key on the GitHub repo** and that
`bootstrap/argocd/priv-key.yaml` (if that's where it's sourced) is actually referenced by
`bootstrap/argocd/kustomization.yaml` — right now that file lists only `namespaces.yaml`
under `resources:`, so anything else in that directory is not applied.

---

### 4. CRD too large (`metadata.annotations: Too long: may not be more than 262144 bytes`)

Hits `applicationsets.argoproj.io` and `applications.argoproj.io`. Client-side apply writes
the whole object into the `last-applied-configuration` annotation and overflows the 256 KB
limit.

**Fix:** the Application must sync with server-side apply. The self-managed `argocd` app
already sets it:

```yaml
syncPolicy:
  syncOptions:
    - ServerSideApply=true
```

Add the same option to any app whose chart ships large CRDs. For a one-off manual apply:

```bash
kustomize build --enable-helm bootstrap/argocd \
  | kubectl apply --server-side --force-conflicts -f -
```

If a CRD is already half-created from an earlier client-side apply:

```bash
kubectl annotate crd applicationsets.argoproj.io applications.argoproj.io \
  kubectl.kubernetes.io/last-applied-configuration- 2>/dev/null
```

then re-sync.

---

### 5. Stuck waiting on an earlier sync wave

Argo syncs child apps in ascending `argocd.argoproj.io/sync-wave` order and won't start a
wave until every resource in the previous one is Healthy. One degraded app blocks
everything after it. Order is documented in `sync-waves.md`:

| Wave | Contents |
|-----:|----------|
| `-3` | argocd (self-managed) |
| `-2` | sealed-secrets, cert-manager, cnpg-operator |
| `-1` | traefik, tailscale-operator, cloudflared |
| `0`  | monitoring, argo-workflows, argo-events, actions-runner-controller |
| `1`  | postgres-cluster |
| `2`  | apps |

```bash
# find the earliest non-Healthy app — that's the blocker
kubectl -n argocd get applications -o custom-columns=\
NAME:.metadata.name,\
WAVE:.metadata.annotations.argocd\\.argoproj\\.io/sync-wave,\
SYNC:.status.sync.status,\
HEALTH:.status.health.status
```

Fix the blocker, not the symptom app.

---

### 6. SealedSecret won't decrypt (`no key could decrypt secret`, secret never appears)

A workload is stuck because its `Secret` never materializes.

- The `sealed-secrets` controller must be running in `kube-system` (wave `-2`).
- The controller's private key must be the one that sealed the manifest. After a cluster
  rebuild you restore it from the password manager **before** anything else syncs (see the
  disaster-recovery runbook, R-04).
- A SealedSecret is namespace- and name-scoped by default. Moving it to a different
  namespace or renaming the target breaks decryption. Re-seal per
  `docs/deploying_a_new_sealed_secret.md`.

```bash
kubectl -n kube-system logs deploy/sealed-secrets-controller --tail=50
kubectl get sealedsecret -A
```

---

### 7. Syncs but immediately drifts back to `OutOfSync`

The live resource is being mutated after apply, so Argo sees permanent diff. With
`selfHeal: true` it re-applies in a loop.

**Known case — `argocd-secret`.** The argo-cd chart re-renders it every build while the
server writes its signing key into the same Secret. The self-managed `argocd` app handles
this with:

```yaml
syncPolicy:
  syncOptions:
    - RespectIgnoreDifferences=true
ignoreDifferences:
  - group: ""
    kind: Secret
    name: argocd-secret
    jsonPointers:
      - /data
```

`RespectIgnoreDifferences=true` is required — without it `selfHeal` still reverts ignored
fields.

**General approach:** `argocd app diff <app>` to see the churning field, then either add an
`ignoreDifferences` entry or stop whatever controller/webhook is writing it.

---

### 8. Self-managed `argocd` app is the one failing

The `argocd` Application (`bootstrap/applications/argocd.yaml`, wave `-3`) manages Argo CD's
own chart. If it breaks, GitOps can't fix GitOps — recover by hand:

```bash
kustomize build --enable-helm bootstrap/argocd \
  | kubectl apply --server-side --force-conflicts -f -

kubectl -n argocd rollout restart deploy/argocd-repo-server deploy/argocd-server
kubectl -n argocd rollout restart statefulset/argocd-application-controller
```

Then re-sync the app so state converges:

```bash
argocd app sync argocd
```

**SSA field-manager conflicts on first adoption** (`conflict with "kubectl"`) are expected
when ownership moves from a manual apply to `argocd-controller`. Argo forces conflicts on
SSA sync, so it clears itself; if not, re-run the manual apply above with
`--force-conflicts`.

Don't `kubectl delete application argocd` — the `resources-finalizer` cascades and tears
down Argo CD.

---

### 9. Prune / selfHeal deleted something it shouldn't have

Most apps run `automated: { prune: true, selfHeal: true }`. A bad commit that drops a
resource, or a render failure, can prune live objects.

- Revert the offending commit. `selfHeal` re-creates the resources on the next sync.
- To pause the loop while you investigate:

  ```bash
  argocd app set <app> --sync-policy none
  # ... fix ...
  argocd app set <app> --sync-policy automated --auto-prune --self-heal
  ```

- CRDs are protected: `crds.keep: true` adds `helm.sh/resource-policy: keep`, which Argo
  treats as `Prune=false`.

---

### 10. Escalation — full manual re-bootstrap

If Argo CD is unrecoverable, follow the disaster-recovery runbook: restore the Sealed
Secrets key, then

```bash
kustomize build --enable-helm bootstrap/argocd \
  | kubectl apply --server-side --force-conflicts -f -

kubectl apply -f bootstrap/cluster-bootstrap.yaml
```

Argo restores every other app from git in wave order.

---

### Quick command reference

```bash
argocd app get <app>                              # status + conditions
argocd app diff <app>                             # live vs desired
argocd app sync <app>                             # force a sync
argocd app sync <app> --prune                     # include prunes
argocd app history <app>                          # revisions
argocd app rollback <app> <id>                    # revert to a revision
argocd app set <app> --sync-policy none           # pause automation
kubectl -n argocd logs deploy/argocd-repo-server           # render errors
kubectl -n argocd logs statefulset/argocd-application-controller   # sync errors
```
