# HAProxy Ingress Controller

HAProxy runs on `case` and is the single entry point for all HTTP/HTTPS
traffic into the cluster. MetalLB assigns it a dedicated LAN IP so every
app is reachable via a clean hostname on port 80/443.

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

