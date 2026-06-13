# homelab-k8s

GitOps repo for a two-node MicroK8s cluster.

| Node | Role | Taint |
|------|------|-------|
| case | Control-plane + core apps (ArgoCD, cert-manager) | `function=centralcommand:PreferNoSchedule` |
| tars | Worker — application workloads | none |

---

## Bootstrap order

### 1. ArgoCD (on case)

```bash
kubectl apply -k init/argocd/
```

Wait for all pods to be Running, then get the admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

### 2. cert-manager (on case)

```bash
kubectl apply -k init/cert-manager/
```

### 3. Register this repo in ArgoCD

Either via the UI or:

```bash
argocd repo add https://github.com/YOUR_USERNAME/homelab-k8s.git
```

### 4. HAProxy Ingress Controller (on case)

```bash
kubectl apply -k init/haproxy/
```

### 5. Reflector (on case)

```bash
kubectl apply --server-side -k init/reflector/
```

### 6. Deploy apps

```bash
kubectl apply -f apps/cluster/
```

ArgoCD will pick up everything in `apps/cluster/` and sync it.

---

## Directory layout

```
homelab-k8s/
├── init/                        # Manually bootstrapped once (kubectl apply -k)
│   ├── argocd/                  # ArgoCD install, pinned to 'case' node
│   └── cert-manager/            # cert-manager install, pinned to 'case' node
└── apps/
    ├── base/                    # Plain Kubernetes manifests + kustomization.yaml
    │   └── qbittorrent/
    │       ├── namespace.yaml   # Namespace
    │       ├── pvc.yaml         # PersistentVolumeClaims (config + downloads)
    │       ├── deployment.yaml  # Deployment (runs on 'tars')
    │       ├── service.yaml     # ClusterIP (web UI) + NodePort (BitTorrent)
    │       └── kustomization.yaml
    └── cluster/                 # ArgoCD Application CRDs — one per app
        └── qbittorrent.yaml
```

All manifests are heavily commented to explain every field — good for learning
what each resource does and why.

### Adding a new app

1. Create `apps/base/<appname>/` with plain Kubernetes YAML files + a `kustomization.yaml`
   that lists them under `resources:`.
2. Create `apps/cluster/<appname>.yaml` as an ArgoCD `Application` pointing at
   `apps/base/<appname>`.
3. Commit and push — ArgoCD auto-syncs within ~3 minutes (default poll interval).
