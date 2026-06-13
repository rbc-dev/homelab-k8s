# HAProxy Ingress Controller

HAProxy runs on `case` and is the single entry point for all HTTP/HTTPS
traffic into the cluster. MetalLB assigns it a dedicated LAN IP so every
app is reachable via a clean hostname on port 80/443.

```
Internet / LAN
      │
      ▼
  <metallb-ip>:80 / :443   (dedicated LAN IP, no NodePort)
      │
      ▼
  HAProxy pod  (pinned to 'case')
      │  routes by Host header
      ├──▶  qbittorrent-webui.media:80
      ├──▶  sonarr.media:80
      └──▶  ...
```

## Before applying

1. Find a free IP in your MetalLB pool:
   ```bash
   kubectl get ipaddresspools -A
   ```
   Your pool is `192.168.0.105-192.168.0.111`. Edit `kustomization.yaml`
   and update the `metallb.universe.tf/loadBalancerIPs` annotation to your
   chosen IP if `192.168.0.105` is already taken.

3. Apply:
   ```bash
   kubectl apply -k init/haproxy/
   ```

4. Verify the Service got its IP:
   ```bash
   kubectl get svc -n haproxy-controller
   # EXTERNAL-IP should show your MetalLB IP, not <pending>
   ```

5. Verify the controller pod is running on `case`:
   ```bash
   kubectl get pods -n haproxy-controller -o wide
   ```

## DNS setup

Point a wildcard DNS entry at the MetalLB IP. In Pi-hole or AdGuard Home:

```
*.home.lab  →  192.168.1.200
```

Every Ingress host you create (e.g. `qbittorrent.home.lab`) now resolves
to HAProxy automatically — no individual DNS entries needed per app.

## Adding an Ingress for an app

Add an `ingress.yaml` to the app's `apps/base/<app>/` directory:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: qbittorrent
  namespace: media
  annotations:
    # Uncomment to force HTTPS redirect after cert-manager is set up:
    # haproxy.org/ssl-redirect: "true"
spec:
  ingressClassName: haproxy
  rules:
    - host: qbittorrent.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: qbittorrent-webui
                port:
                  name: webui
  # tls:
  #   - hosts:
  #       - qbittorrent.home.lab
  #     secretName: qbittorrent-tls   # cert-manager creates this Secret
```

## Bootstrap order

```
init/storage/  →  init/argocd/  →  init/cert-manager/  →  init/haproxy/
```

cert-manager before haproxy if you want TLS from day one — cert-manager
needs to be running so it can respond to ACME challenges and create the
certificate Secrets that HAProxy will serve.
