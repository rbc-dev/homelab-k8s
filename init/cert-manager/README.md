# cert-manager Bootstrap

Apply once after ArgoCD is running.

```bash
kubectl apply -k init/cert-manager/
```

Verify all pods are running on `case`:

```bash
kubectl get pods -n cert-manager -o wide
```

After this you can create ClusterIssuers via ArgoCD (add them under apps/base/).
