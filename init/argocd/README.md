# ArgoCD Bootstrap

Apply once to bring up ArgoCD on the `case` node.

```bash
kubectl apply -k init/argocd/
```

After pods are running, get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Then port-forward to log in:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080  user: admin
```

Once logged in, point ArgoCD at this repo by applying the app-of-apps:

```bash
kubectl apply -f apps/cluster/
```
