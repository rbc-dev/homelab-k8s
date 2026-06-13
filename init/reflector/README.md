# Reflector

Reflector copies Secrets and ConfigMaps across namespaces automatically.
We use it to distribute the wildcard TLS certificate to every namespace
that has an Ingress resource.

## Why this is needed

Kubernetes Secrets are namespace-scoped. HAProxy can only read a TLS Secret
from the same namespace as the Ingress. Without reflector:

  cert-manager namespace  →  wildcard-yourdomain-tls Secret
  media namespace         →  ❌ can't see it
  argocd namespace        →  ❌ can't see it

With reflector:

  cert-manager namespace  →  wildcard-yourdomain-tls (source, managed by cert-manager)
  media namespace         →  wildcard-yourdomain-tls (copy, kept in sync by reflector)
  argocd namespace        →  wildcard-yourdomain-tls (copy, kept in sync by reflector)

Copies are updated automatically whenever cert-manager renews the cert.

## Apply

```bash
kubectl apply --server-side -k init/reflector/
```

## Adding a new namespace

Edit `init/cert-manager/wildcard-certificate.yaml` and append the namespace
to both annotation values:

```yaml
reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "media,argocd,my-new-namespace"
reflector.v1.k8s.emberstack.com/reflection-auto-namespaces: "media,argocd,my-new-namespace"
```

Apply the change and reflector copies the Secret within seconds:

```bash
kubectl apply -f init/cert-manager/wildcard-certificate.yaml

# Verify the copy appeared
kubectl get secret wildcard-yourdomain-tls -n my-new-namespace
```

## Bootstrap order

```
init/reflector/  must come before  init/cert-manager/ wildcard-certificate.yaml
```

Reflector needs to be running before the Certificate is created so it picks
up the annotations on the Secret immediately.
