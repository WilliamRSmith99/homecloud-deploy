# homecloud-deploy

Argo CD GitOps for the k3s cluster. Everything in the cluster is deployed from this repo.

## Start here — the knowledge base

**Before you touch anything, read `kb/`.** Start with `kb/README.md`, then the pages
covering the area you're changing. Form opinions after reading, not before — a
confident answer built on a skim is the failure mode this exists to prevent.

Check `kb/divergences.md` early. It lists places these docs are known to be wrong.

## Rules for every ticket

1. **Look up context first.** Read the KB pages surrounding the code you're about to
   change, plus the live cluster (`kubectl get applications -n argocd`) — the cluster is the source of truth, the KB is the explanation.

2. **Changes to architecture, data flow, interfaces, schema, or context require a KB
   update in the same commit.** Not a follow-up ticket, not "later." If the change makes
   a KB page wrong, fix the page as part of the work.

3. **Found a divergence between the KB and reality? Document it immediately.** Add a
   dated entry to `kb/divergences.md` before you continue the task. Recording is not the
   same as fixing — record even when you can't fix, and record even when you can (then
   close it out in the same commit). Never silently correct and move on; the gap itself
   is information about how the docs drift.

4. **Cite your work.** Reference the Jira key in the commit message (`Refs HC-xx`).

## What does not belong in the KB

Roadmap, backlog and checkpoint history live in
`homecloud/Project_Planning/k3s-setup/`. The KB is current-state truth: what is, how it
flows, why it's that way. Not what's planned.

## Hard rules

- **Never `kubectl apply` to create state.** Argo self-heal will revert it, and
  undocumented drift is what broke the v1 cluster. `kubectl` is for reading and
  debugging.
- **Write manifests to versioned files.** Never pipe inline YAML to `kubectl`.
- **Resource requests and limits on every container.** No exceptions.
- **Sealed secrets only.** Never commit a plaintext `Secret`, never apply one by hand.
- **New app?** Follow `docs/right_sizing_a_new_app.md` and check the headroom in
  `kb/architecture.md` first.

## Layout

```
bootstrap/applications/   one child Application per component (sets the sync wave)
infrastructure/           platform components
data/                     stateful workloads
apps/                     user-facing workloads
sync-waves.md             ordering contract
docs/                     task runbooks
kb/                       current-state knowledge base — read first
```
