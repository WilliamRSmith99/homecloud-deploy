# right-sizing resources for a new app

Every workload gets `requests` and `limits`. No exceptions (R-13) — an unbounded
pod on a 3-node cluster with 6-24GB nodes is how one bad process starves
everything else on its node.

## Starting point, by workload type

| Type | Example | Starting request | Starting limit |
|---|---|---|---|
| Lightweight controller (watches CRDs, no data plane) | cnpg-operator | `100m` / `100Mi` | `100m` / `200Mi` |
| App server (handles real traffic) | overlap-bot | `100m` / `128Mi` | `500m` / `256Mi` |
| Database / stateful | homelab-pg | size to the workload's real footprint, not a guess | 1.5-2x request |

Memory: request = what it needs at idle, limit = headroom for a spike.
CPU: request = what it needs to not get starved when the node's busy; limit
can be looser than memory since CPU throttles instead of getting killed.

## You don't need to know the perfect number up front

Deploy with a reasonable guess. Then, after it's run a while under real load:

```bash
kubectl top pod -n <namespace>
```

Set the request near what it's actually using. Adjust the limit if you see
`OOMKilled` in `kubectl describe pod` or throttling you don't want.

## Gotchas

- No `limits.memory` → pod can eat the whole node before OOM-killer notices.
- `requests` too high across many pods → nothing schedules, node reports
  "Insufficient memory" even with RAM free.
- Don't copy another app's numbers without checking what it actually does —
  a controller and a database have nothing in common here.
