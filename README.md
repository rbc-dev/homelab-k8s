# homelab-k8s

GitOps repo for a two-node MicroK8s cluster running at `rbcdev.site`.

## Cluster nodes

| Node | Role | Taint |
|------|------|-------|
| case | Control-plane — ArgoCD, cert-manager, HAProxy, Reflector | `function=centralcommand:PreferNoSchedule` |
| tars | Worker — all application workloads | none |

---

## How it works

```
Git push → dev branch
      │
      ▼
ArgoCD ApplicationSet  (apps/base/applicationset.yaml)
      │  scans apps/base/*/kustomization.yaml
      │  generates one Application per directory
      ▼
Kustomize build + apply to cluster
      │
      ▼
Pods scheduled on tars
      │
      ▼
HAProxy Ingress  ←  MetalLB IP 192.168.0.108
      │
      ├── qbittorrent.rbcdev.site
      └── emby.rbcdev.site
```

TLS uses a single wildcard cert (`*.rbcdev.site`) issued by Let's Encrypt via Cloudflare DNS-01. Reflector copies the cert Secret into every namespace that needs it.

---

## Repository layout

```
homelab-k8s/
├── init/                            # Applied once manually — never touched by ArgoCD
│   ├── storage/                     # StorageClass, PVs, PVCs, media namespace
│   ├── argocd/                      # ArgoCD install + config patches
│   ├── cert-manager/                # cert-manager + ClusterIssuer + wildcard cert
│   ├── haproxy/                     # HAProxy Ingress Controller + MetalLB service
│   └── reflector/                   # Copies TLS Secret across namespaces
└── apps/
    └── base/                        # One subdirectory per app
        ├── applicationset.yaml      # ArgoCD ApplicationSet — bootstrapped once manually
        ├── qbittorrent/
        │   ├── kustomization.yaml
        │   ├── namespace.yaml
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── ingress.yaml
        └── emby/
            ├── kustomization.yaml
            ├── deployment.yaml
            ├── service.yaml
            └── ingress.yaml
```

---

## init/ — what each piece does

| Directory | Contents |
|-----------|----------|
| `init/storage/` | `local-storage` StorageClass · `pv-config` → `/data/config` on tars · `pv-media` → `/data/media` on tars · `config` and `media` PVCs in the `media` namespace |
| `init/argocd/` | ArgoCD upstream manifest · insecure mode (TLS at HAProxy) · Ingress health check fix · Ingress for `argocd.rbcdev.site` |
| `init/cert-manager/` | cert-manager · `letsencrypt-prod` ClusterIssuer (Cloudflare DNS-01) · wildcard cert `*.rbcdev.site` with Reflector annotations |
| `init/haproxy/` | HAProxy Ingress Controller pinned to `case` · Service patched to LoadBalancer · MetalLB IP `192.168.0.105` · IngressClass `haproxy` (default) |
| `init/reflector/` | Emberstack Reflector pinned to `case` · syncs `wildcard-yourdomain-tls` into `media`, `argocd`, `haproxy-controller` |

---

## Storage layout on tars

Two PVs, two PVCs. All apps mount the same PVCs and use `subPath` to scope access to their own directory.

```
/data
├── config/              ← pv-config  /  PVC: config  (in media namespace)
│   ├── qbittorrent/         subPath: qbittorrent
│   └── emby/                subPath: emby
└── media/               ← pv-media   /  PVC: media   (in media namespace)
    ├── images/
    ├── music/
    └── videos/
        ├── movies/          subPath: videos/movies
        └── tv/              subPath: videos/tv
```

All containers run with `PGID=1002` (group `mediacenter`). The setgid bit on every directory ensures files created by any app are readable and writable by all other apps.

---

## Bootstrap — run once on a new cluster

```bash
# 0. Prepare tars filesystem
ssh tars
sudo groupadd -g 1002 mediacenter
sudo mkdir -p /data/config \
              /data/media/images \
              /data/media/music \
              /data/media/videos/movies \
              /data/media/videos/tv
sudo chown -R {user}:1002 /data
sudo chmod -R 775 /data
sudo find /data -type d -exec chmod g+s {} +
exit

# 1. Storage
kubectl apply -k init/storage/

# 2. Reflector (must exist before cert-manager writes the wildcard Secret)
kubectl apply --server-side -k init/reflector/

# 3. ArgoCD
kubectl apply --server-side --force-conflicts -k init/argocd/

# 4. Cloudflare token (before cert-manager requests the cert)
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=<YOUR_CLOUDFLARE_TOKEN> \
  --namespace cert-manager

# 5. cert-manager
kubectl apply --server-side -k init/cert-manager/
kubectl rollout status deployment/cert-manager -n cert-manager

# 6. HAProxy
kubectl apply --server-side -k init/haproxy/

# 7. Bootstrap ArgoCD ApplicationSet — the only manual app apply you ever do
kubectl apply -f apps/base/applicationset.yaml
```

After step 7, ArgoCD owns everything in `apps/base/`. Merge to `dev` = deploy.

---

## Adding a new app

```bash
# 1. If the app needs its own namespace, add it to the reflector annotations
#    in init/cert-manager/wildcard-certificate.yaml then re-apply:
kubectl apply -f init/cert-manager/wildcard-certificate.yaml

# 2. Create the app manifests
mkdir apps/base/<appname>
# deployment.yaml  — nodeSelector: tars, PGID: 1002, subPath for volumes
# service.yaml     — ClusterIP pointing at the app port
# ingress.yaml     — host: <appname>.rbcdev.site, secretName: wildcard-yourdomain-tls
# kustomization.yaml — lists all of the above under resources:

# 4. Commit and push
git add apps/base/<appname>
git commit -m "add <appname>"
git push origin dev
# ArgoCD picks it up within ~3 minutes
```

---

## DNS

```
*.rbcdev.site  →  192.168.0.108
```

MetalLB pool: `192.168.0.105 – 192.168.0.111`

---

## TLS

Single wildcard cert covering all subdomains. Renewed automatically 30 days before the 90-day expiry. Every Ingress references the same Secret:

```yaml
tls:
  - hosts: [<appname>.rbcdev.site]
    secretName: wildcard-yourdomain-tls
```
