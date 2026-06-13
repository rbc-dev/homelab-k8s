# Shared Storage

Manually applied once. Two PVs, two PVCs — one for config, one for media.
All apps share them using `subPath` in their `volumeMount`.

## Directory structure on tars

```
/data
├── config/                  ← pv-config  (PVC: config)
│   ├── qbittorrent/
│   └── <appname>/
└── media/                   ← pv-media   (PVC: media)
    ├── images/
    ├── music/
    └── videos/
        ├── movies/
        └── tv/
```

## File permissions — shared group approach

Each container app runs as its own UID (set via PUID env var) but they all
share a common GID (set via PGID env var). The /data tree is owned by that
group with write permission, so every app can read and write regardless of
its individual UID.

### Why this works

Linux file permissions have three levels: owner, group, others.
If a file is `rwxrwxr-x` and owned by group `1002`, any process running with
GID 1002 can read and write it — even if its UID is different.

### The setgid bit

`chmod g+s <dir>` sets the setgid bit on a directory. This means:
- Any file created inside inherits the directory's group automatically
- Any subdirectory created inside also gets the setgid bit
- Future files are always group-accessible without manual intervention

Without setgid, new files inherit the creating process's primary GID, which
may differ per app — breaking cross-app access over time.

### Setup on tars

SSH into tars and run this once before applying the Kubernetes manifests:

```bash
# Create a shared group for all media app processes.
# GID 1002 is used here — it's typically the default first user's group
# on most Linux distros. Adjust if 1002 is already taken on your system.
# Check with: getent group 1002
sudo groupadd -g 1002 mediacenter

# Create the directory tree
sudo mkdir -p /data/config/qbittorrent
sudo mkdir -p /data/media/images
sudo mkdir -p /data/media/music
sudo mkdir -p /data/media/videos/movies
sudo mkdir -p /data/media/videos/tv

# Set ownership: root owns the files, group mediacenter has write access.
# The UID (first 1002) doesn't matter much since containers use their own
# UIDs — the group is what enables shared access.
sudo chown -R 1000:1002 /data

# Set permissions:
#   755 → owner rwx, group r-x, others r-x  (default, too restrictive)
#   775 → owner rwx, group rwx, others r-x  (group can write — what we want)
sudo chmod -R 775 /data

# Set the setgid bit on every directory so new files/dirs inherit group 1002.
# find -type d targets directories only (files don't need setgid here).
sudo find /data -type d -exec chmod g+s {} +
```

Verify it looks right:
```bash
ls -la /data/media
# drwxrwsr-x  root  mediacenter   <-- 's' in group position = setgid set
```

The `s` instead of `x` in the group execute position confirms setgid is active.

### Per-app container config

Every app deployment must set `PGID=1002` to join the shared group.
The PUID can be anything — it's the PGID that enables shared access.

```yaml
env:
  - name: PUID
    value: "1002"   # app's own UID — can differ per app
  - name: PGID
    value: "1002"   # shared group — MUST be the same for all media apps
```

### Adding a new app

```bash
# On tars: create the config subdirectory with correct ownership not compulsory unless issues show up
sudo mkdir -p /data/config/<appname>
sudo chown 1002:1002 /data/config/<appname>
sudo chmod 775 /data/config/<appname>
sudo chmod g+s /data/config/<appname>
```

In the app's `deployment.yaml`, set `PGID=1002` and mount with subPath:

```yaml
env:
  - name: PGID
    value: "1002"
volumeMounts:
  - name: config
    mountPath: /config
    subPath: <appname>
  - name: media
    mountPath: /tv
    subPath: videos/tv    # adjust to what the app needs
volumes:
  - name: config
    persistentVolumeClaim:
      claimName: config
  - name: media
    persistentVolumeClaim:
      claimName: media
```

No new PVs or PVCs needed — just a new directory on tars and a subPath in
the deployment.

---

## Applying the Kubernetes manifests

After the host setup above is complete:

```bash
kubectl apply -k init/storage/

# Verify PVs are Available and PVCs are Bound
kubectl get pv
kubectl get pvc -n media
```

## How apps reference storage

```yaml
volumes:
  - name: config
    persistentVolumeClaim:
      claimName: config
  - name: media
    persistentVolumeClaim:
      claimName: media

volumeMounts:
  - name: config
    mountPath: /config
    subPath: qbittorrent       # → /data/config/qbittorrent

  - name: media
    mountPath: /data/media     # → /data/media (full tree)

  # or scoped to a subtree:
  - name: media
    mountPath: /tv
    subPath: videos/tv         # → /data/media/videos/tv
```
