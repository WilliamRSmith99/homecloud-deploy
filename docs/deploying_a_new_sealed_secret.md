## Deploying a new sealed secret object
### Prerequisites
- helm
- brew
- kubernetes with kubectl access

### creating the yaml

```
kubectl create secret generic <NAME> \
  --namespace <NAMESPACE> \
  --from-literal=client_id='<oauth-client-id>' \
  --from-literal=client_secret='<oauth-client-secret>' \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
    --controller-namespace kube-system \
    --controller-name sealed-secrets-controller \
> PATH/TO/<NAME>.yaml
```