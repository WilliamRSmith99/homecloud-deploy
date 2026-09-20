## Exposing a service on Tailscale

How to put a cluster service on the tailnet as `<name>.tail1a1d19.ts.net`, and how to
diagnose one that doesn't come up. Part 1 is the happy path. Part 2 is keyed by symptom —
start at the triage step and it'll point you at the right section.

### Prerequisites

- `kubectl` access to the cluster (Tailscale SSH to `oci-cp`, or a local kubeconfig)
- The `tailscale-operator` app synced and its pod Running in the `tailscale` namespace
- Admin access to the Tailscale console for ACL and device changes — those live in the
  tailnet's policy file, **not** in this repo

---

### How it works

The operator watches for Ingresses with `ingressClassName: tailscale`. For each one it
resolves the backend Service, then creates a StatefulSet `ts-<ingress-name>-<random>` in the
`tailscale` namespace. That pod joins the tailnet as its own device, registers a MagicDNS
name, terminates TLS with a `ts.net` cert, and proxies to your Service over the cluster
network.

Three things follow from that, and they're where most of the failures come from:

- **The proxy is a separate device.** MagicDNS resolves it, not your Service. Nothing shows
  up in DNS until the StatefulSet exists and its pod is Running.
- **Argo CD can't see any of this.** It applies the Ingress object, which always succeeds.
  Every failure below happens downstream, in the operator, while Argo reports Synced and
  Healthy.
- **The operator only reconciles on change or resync** — and the resync is ~10 hours. Fix
  something and it can sit broken for most of a day unless you nudge it.

---

## Part 1 — Expose a new service

### 1. Confirm the Service name and port

The Ingress backend must name a Service that actually exists, with a port that actually
exists on it. This is the single most common mistake — see section 2.

```bash
kubectl get svc -n <namespace>
```

Use the `NAME` and the numeric port from `PORT(S)`. Don't copy them from another app's
Ingress; chart naming varies (Argo CD gives you `argocd-server:443`, the Grafana chart gives
you `grafana:80`).

### 2. Write the Ingress

One file per service, alongside the rest of that component's manifests:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    tailscale.com/hostname: myapp   # pins the MagicDNS name
spec:
  ingressClassName: tailscale
  tls:
    - hosts:
        - myapp                     # operator expands to myapp.tail1a1d19.ts.net
  rules:
    - host: myapp
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp         # must match `kubectl get svc`
                port:
                  number: 8080      # must be a port on that Service
```

`host` and the `tls` host are short names, not FQDNs — the operator appends the tailnet
domain. Without `tailscale.com/hostname` the device is named after the Ingress, which is
usually the same thing, but pin it so a rename doesn't move your URL.

### 3. Add it to the kustomization

```yaml
resources:
  - ingress.yaml
```

Then commit and push. Argo CD picks it up on its next poll, or force it:

```bash
argocd app sync <app>
```

### 4. Verify

Work down this list. Each step gates the next.

```bash
# 1. the Ingress exists and its backend resolves (no <error:> in the Backends column)
kubectl describe ingress myapp -n myapp

# 2. the operator built a proxy for it
kubectl get sts -n tailscale -l tailscale.com/parent-resource=myapp

# 3. the proxy pod is Running
kubectl get pods -n tailscale

# 4. the Ingress has an ADDRESS — this is the operator reporting success
kubectl get ingress myapp -n myapp

# 5. it resolves and serves
tailscale status | grep myapp
curl -I https://myapp.tail1a1d19.ts.net
```

Step 4 is the real pass/fail. An empty `ADDRESS` means the operator never finished, and
section 2 explains why.

---

## Part 2 — Troubleshooting

### 1. Triage — get the actual error

The operator log is the only place failures are reported. Start here.

```bash
# every tailscale Ingress and whether it got an address
kubectl get ingress -A --field-selector spec.ingressClassName=tailscale

# what the operator thinks of yours
kubectl logs -n tailscale deploy/operator --tail=200 | grep -i <name>

# backend resolution, inline
kubectl describe ingress <name> -n <namespace>
```

Match what you see:

| Symptom | Section |
|---|---|
| `ADDRESS` empty, no `ts-*` StatefulSet | 2 |
| `<error: services "..." not found>` in `describe` | 2 |
| StatefulSet exists, pod not Running | 3 |
| Device is up, name won't resolve | 4 |
| Resolves, connection refused or 502 | 5 |
| TLS/cert error in the browser | 6 |
| Device came up with a `-2` suffix | 7 |
| Resolves but times out from your laptop | 8 |

---

### 2. No proxy created (`Ingress contains no valid backends`)

```
{"level":"warn","logger":"ingress-reconciler","msg":"Ingress contains no valid backends",
 "ingress-ns":"monitoring","ingress-name":"grafana"}
```

The operator can't resolve the backend Service, so it builds nothing. The Ingress object is
perfectly valid Kubernetes — Argo CD reports it Healthy — but no device is ever created and
the name never resolves.

**This is the failure mode to expect after copying another app's Ingress.** It bit Grafana
for three days (HC-76): the file was copied from `bootstrap/argocd/ingress.yaml` and still
pointed at `grafana-server:443`, a name and port that only exist for Argo CD. The Grafana
chart serves `grafana:80`.

Find the mismatch:

```bash
kubectl describe ingress <name> -n <namespace> | grep -A3 Backends
kubectl get svc -n <namespace>
```

`describe` prints the resolution error inline:

```
Host      Path  Backends
----      ----  --------
grafana
          /   grafana-server:443 (<error: services "grafana-server" not found>)
```

Both halves have to match. A Service that exists but doesn't expose the port you named fails
the same way:

```bash
kubectl get svc <svc> -n <namespace> -o jsonpath='{.spec.ports[*].port}'
```

Fix the backend, commit, push, then **restart the operator** — don't wait out the ~10h
resync:

```bash
kubectl rollout restart deploy/operator -n tailscale
```

---

### 3. Proxy StatefulSet exists but the pod won't run

```bash
kubectl get pods -n tailscale
kubectl describe pod ts-<name>-xxxxx-0 -n tailscale
kubectl logs ts-<name>-xxxxx-0 -n tailscale
```

- **`CreateContainerConfigError` / auth failures** — the operator's OAuth client is bad or
  expired. It comes from the `operator-oauth` SealedSecret in
  `infrastructure/tailscale-operator/`. Re-seal per `docs/deploying_a_new_sealed_secret.md`
  and confirm the client has the `devices` write scope in the Tailscale console.
- **`Pending` on `oci-cp`** — the control-plane node is 1 OCPU / 6 GB and runs tight. Check
  `kubectl describe node oci-cp` for pressure, and see `docs/right_sizing_a_new_app.md`.
- **Logs show the node needs approval** — see section 8.

---

### 4. Device is up but the name doesn't resolve

```bash
tailscale status | grep <name>
tailscale dns status
```

- **Not in `tailscale status`** — the device never joined. Read the proxy pod's logs
  (section 3).
- **In `tailscale status` but DNS fails** — MagicDNS is off for your client, or the name is
  cached. Confirm MagicDNS is enabled in the tailnet DNS settings, then on macOS:

  ```bash
  sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder
  ```

- **Resolves for you, not for someone else** — that's ACLs, section 8.

Always test the FQDN. Bare `grafana` depends on your search domain being set.

---

### 5. Resolves but connection refused, or a 502

The proxy is up and forwarding to the wrong place.

```bash
# does the Service answer from inside the cluster at all?
kubectl run -n <namespace> curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -sv http://<svc>.<namespace>:<port>
```

- **Port exists on the Service but nothing listens behind it** — the Service's selector
  matches no Pods, or matches the wrong ones. `kubectl get endpoints <svc> -n <namespace>`;
  empty means selector mismatch.
- **The backend speaks HTTPS and you pointed at its plaintext port, or the reverse.** Argo
  CD's `argocd-server` serves TLS on 443, so its Ingress targets 443. Grafana serves plain
  HTTP on 80. Match the protocol the Service actually speaks.
- **App rejects the Host header** — some apps need to know their external URL. Grafana's is
  `grafana.ini`'s `server.root_url`; set it in `values.yaml` if redirects come back wrong.

---

### 6. Certificate errors

The `ts.net` cert is issued to the proxy, and it only works if the tailnet has HTTPS
certificates turned on (DNS → HTTPS Certificates in the Tailscale console).

- **`NET::ERR_CERT_AUTHORITY_INVALID` or a self-signed cert** — HTTPS certs are off for the
  tailnet, so the proxy fell back. Enable it, then restart the proxy pod.
- **Cert issued for the wrong name** — `spec.tls[].hosts[0]` doesn't match the actual device
  name. This happens after a hostname collision (section 7) or when the `tls` block is left
  over from a copied file. Both must agree with `tailscale.com/hostname`.

```bash
kubectl logs ts-<name>-xxxxx-0 -n tailscale | grep -i cert
openssl s_client -connect <name>.tail1a1d19.ts.net:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject
```

---

### 7. Hostname collided (device came up as `<name>-2`)

Tailscale doesn't reject a duplicate hostname — it appends a suffix. So the service works,
just at a URL nobody expects, and the cert won't match the name in your `tls` block.

Usually it's a stale device from a previous proxy that was never cleaned up.

```bash
tailscale status | grep <name>
```

Delete the orphan in the Tailscale console (Machines → the old device → Remove), then
restart the proxy pod so it re-registers with the name it wanted:

```bash
kubectl delete pod ts-<name>-xxxxx-0 -n tailscale
```

---

### 8. Resolves but times out — ACLs

The proxies this operator creates are tagged `tag:k8s`
(`proxyConfig.defaultTags` in `infrastructure/tailscale-operator/values.yaml`). Reaching
them requires an ACL grant in the tailnet policy file, which is **not** in this repo.

Two things have to be true in the policy file:

- `tagOwners` lists `tag:k8s` and permits the operator's OAuth client to apply it. Without
  this the device can't even register with the tag.
- An ACL rule lets your user reach `tag:k8s` on the port you're using.

```jsonc
"tagOwners": {
  "tag:k8s": ["autogroup:admin"],
},
"acls": [
  { "action": "accept", "src": ["autogroup:member"], "dst": ["tag:k8s:443"] },
],
```

Use the console's ACL preview to check a specific source and destination before assuming the
rule is right.

---

### 9. Removing a service

Delete the Ingress from git and let Argo prune it. The operator tears down the StatefulSet
and the device deregisters.

```bash
kubectl get sts -n tailscale     # confirm the ts-<name>-* StatefulSet is gone
tailscale status | grep <name>   # confirm the device is gone
```

If the device lingers in the console, remove it by hand — otherwise it holds the hostname
and the next deploy hits section 7.

Don't delete the StatefulSet directly while the Ingress still exists; the operator just
rebuilds it with a new random suffix and a new device.

---

### Reference — annotations worth knowing

| Annotation | Effect |
|---|---|
| `tailscale.com/hostname` | Pins the MagicDNS name. Always set it. |
| `tailscale.com/tags` | Overrides `tag:k8s` for this one proxy. Needs matching `tagOwners`. |
| `tailscale.com/funnel: "true"` | Exposes the service to the **public internet**, not just the tailnet. Not used here — public traffic goes through Cloudflare (CP-08). |

---

### Quick command reference

```bash
kubectl get ingress -A --field-selector spec.ingressClassName=tailscale  # all of them
kubectl describe ingress <name> -n <ns>                    # backend resolution errors
kubectl logs -n tailscale deploy/operator --tail=200       # the only real error source
kubectl get sts,pods -n tailscale                          # proxies
kubectl logs ts-<name>-xxxxx-0 -n tailscale                # one proxy's tailscaled
kubectl rollout restart deploy/operator -n tailscale       # force a reconcile
kubectl get endpoints <svc> -n <ns>                        # is anything behind the Service
tailscale status                                           # tailnet devices
tailscale dns status                                       # MagicDNS config
```
