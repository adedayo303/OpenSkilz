# Module 03 — Longhorn Persistent Storage

> **What you will have at the end of this module:** A distributed block storage system
> running across all three worker nodes using dedicated `sdb` disks, providing replicated
> persistent volumes to every application in the cluster. An NFS StorageClass for
> ReadWriteMany volumes (multi-pod access). A backup target pointed at the NFS share.
> The Longhorn UI accessible at `https://longhorn.skilz.io`.

---

## Why This Module Exists

Kubernetes pods are ephemeral by design. When a pod restarts, its local filesystem
is wiped. For any application that needs to remember something between restarts — a
database, a metrics store, an application with user data — you need storage that
lives outside the pod.

Kubernetes has a built-in concept for this: a **PersistentVolume (PV)**. But someone
has to provide that volume. Options:

| Option | What it is | Suitable for this lab? |
|--------|-----------|----------------------|
| `hostPath` | Directory on the node's local disk | No — tied to one node, lost if node changes |
| Cloud provider (EBS, GCS, Azure Disk) | Cloud block storage, auto-provisioned | No — requires a cloud account |
| NFS | Network filesystem from an external server | ReadWriteMany only — no replication |
| **Longhorn** | Distributed block storage, runs inside K8s | **Yes — purpose built for this** |

**Longhorn** is a CNCF project that turns the local disks of your worker nodes into a
distributed, replicated storage pool. When you request a 5GB volume, Longhorn carves it
out across multiple nodes and keeps 3 copies in sync. If one node reboots, the other
copies serve data without interruption.

**Two storage layers in this cluster:**

| Layer | Technology | Access | Used for |
|-------|-----------|--------|---------|
| Block | Longhorn (`sdb` disks) | ReadWriteOnce | Single-pod stateful apps |
| Shared | NFS (`/k8s-nfs/shared-volumes`) | ReadWriteMany | Multi-pod shared storage |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      Longhorn Storage Pool                    │
│                                                              │
│  k8s-w-01          k8s-w-02          k8s-w-03               │
│  sdb: 200 GB       sdb: 150 GB       sdb: 400 GB            │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐           │
│  │ replica  │◄────►│ replica  │◄────►│ replica  │           │
│  │ (copy 1) │      │ (copy 2) │      │ (copy 3) │           │
│  └──────────┘      └──────────┘      └──────────┘           │
│                          │                                   │
│                    Volume Engine                             │
│                 (mounted to the pod)                         │
└──────────────────────────────────────────────────────────────┘

NFS (ReadWriteMany)
dc:/k8s-nfs/shared-volumes  ──  nfs-subdir-provisioner  ──  K8s StorageClass: nfs-shared
dc:/k8s-nfs/longhorn-backup ──  Longhorn backup target
```

**Key concepts:**

- **Volume Engine** — the active process that serves a volume to a pod. Runs on the same node as the pod.
- **Replicas** — copies of the data spread across nodes. Default 3 in this cluster.
- **CSI** — standard Kubernetes storage API. Longhorn implements CSI so K8s treats it like any storage provider.
- **StorageClass** — a K8s object defining how to provision storage. Once installed, `storageClassName: longhorn` is all an app needs.

---

## Part A — Prepare the Dedicated Disks

Each worker has a second disk (`sdb`) that was left unformatted in Day 0. Longhorn
uses a **directory path** on each node as its storage root — by mounting `sdb` to
`/var/lib/longhorn` before installing, every byte Longhorn writes goes to the
dedicated disk instead of the OS disk.

> **Why not just let Longhorn use the OS disk?** The OS disk (`sda`) is shared with
> system processes, container images, and logs. Longhorn doing heavy I/O on `sda`
> competes with the OS. Separate disks means storage I/O never impacts system
> stability, and the full capacity of `sdb` is available to storage.

### A.1 Format and mount sdb on each worker

Run these commands on each worker node. Use the VMware console or SSH from the jumpbox.

**On k8s-w-01 (192.168.14.12):**

```bash
# Confirm sdb is the correct disk and is unpartitioned
lsblk
# Expected: sdb with no partitions listed underneath it

# Format with ext4
sudo mkfs.ext4 /dev/sdb

# Get the UUID (stable across reboots, unlike /dev/sdb)
sudo blkid /dev/sdb
# Example output: /dev/sdb: UUID="a1b2c3d4-..." TYPE="ext4"

# Create the mount point (Longhorn's default storage path)
sudo mkdir -p /var/lib/longhorn

# Add to fstab for persistent mount — replace the UUID with your actual value
echo "UUID=<your-uuid>  /var/lib/longhorn  ext4  defaults  0  2" | sudo tee -a /etc/fstab

# Mount it now
sudo mount /var/lib/longhorn

# Verify
df -h /var/lib/longhorn
# Expected: shows sdb with ~185 GB available (200 GB formatted)
```

Repeat on **k8s-w-02** (sdb: 150 GB) and **k8s-w-03** (sdb: 400 GB) — same commands,
different UUID. The mount point `/var/lib/longhorn` is the same on all nodes.

**Verify all three workers from the jumpbox:**

```bash
for node in 192.168.14.12 192.168.15.12 192.168.16.12; do
  echo -n "$node: "
  ssh <admin-user>@${node} "df -h /var/lib/longhorn | tail -1"
done
```

Expected: each node showing its sdb size mounted at `/var/lib/longhorn`.

### A.2 Install open-iscsi on all workers

Longhorn uses iSCSI to attach volumes to pods.

```bash
for node in 192.168.14.12 192.168.15.12 192.168.16.12; do
  ssh <admin-user>@${node} "
    sudo apt-get install -y open-iscsi &&
    sudo systemctl enable --now iscsid &&
    echo \$(hostname): \$(iscsiadm --version)
  "
done
```

Expected per node: `iscsiadm version 2.1.x`. If this step is skipped, Longhorn installs
without errors but volumes never attach — pods stay in `ContainerCreating` indefinitely.

---

## Part B — Install Longhorn

### B.1 Create the Helm values file

On the jumpbox, create `/tmp/longhorn-values.yaml`:

```bash
cat > /tmp/longhorn-values.yaml << 'EOF'
defaultSettings:
  defaultReplicaCount: 3
  defaultDataPath: /var/lib/longhorn   # the mounted sdb path
  backupTarget: nfs://192.168.15.10:/k8s-nfs/longhorn-backup
  backupTargetCredentialSecret: ""     # NFS needs no credentials
  storageOverProvisioningPercentage: 100
  storageMinimalAvailablePercentage: 15

longhornUI:
  replicas: 1

ingress:
  enabled: false   # we create the ingress manually with adcs-issuer TLS
EOF
```

> **`storageOverProvisioningPercentage: 100`** — Longhorn allows provisioning up to
> 2× the physical disk space by default (200%). Setting 100% means you can only
> provision what you actually have. Prevents over-committing a lab cluster.

> **`storageMinimalAvailablePercentage: 15`** — Longhorn stops scheduling new replicas
> on a node when less than 15% of its disk is free. Prevents filling the disk to 100%
> which causes crashes.

### B.2 Install via Helm

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  -f /tmp/longhorn-values.yaml
```

### B.3 Verify installation

```bash
# Watch pods come up (takes 2-3 minutes)
kubectl get pods -n longhorn-system --watch
```

Wait until all pods are `Running` or `Completed`. Expected pods:

| Pod type | Count | What it does |
|----------|-------|-------------|
| `longhorn-manager-*` | 3 (one per worker) | Core storage daemon on each node |
| `longhorn-driver-deployer-*` | 1 | Installs CSI driver onto each node |
| `longhorn-ui-*` | 1 | Web dashboard |
| `csi-attacher-*` | 3 | Handles volume attach/detach |
| `csi-provisioner-*` | 3 | Handles PVC provisioning requests |
| `csi-resizer-*` | 3 | Handles online volume resize |
| `csi-snapshotter-*` | 3 | Handles volume snapshots |
| `longhorn-csi-plugin-*` | 3 (DaemonSet, one per worker) | Mounts volumes into pods |

```bash
# Confirm the StorageClass exists and is default
kubectl get storageclass
# NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE
# longhorn (default)   driver.longhorn.io   Delete          Immediate
```

### B.4 Expose the UI

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    cert-manager.io/issuer: "skilz-adcs-issuer"
    cert-manager.io/issuer-kind: "ClusterAdcsIssuer"
    cert-manager.io/issuer-group: "adcs.certmanager.csf.nokia.com"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - longhorn.skilz.io
    secretName: longhorn-tls
  rules:
  - host: longhorn.skilz.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
EOF
```

Add DNS A record on dc:

```powershell
Add-DnsServerResourceRecordA -ZoneName "skilz.io" -Name "longhorn" -IPv4Address "192.168.14.31"
```

Browse to `https://longhorn.skilz.io`. The dashboard shows total capacity, node
disk usage, and volume health.

### B.5 Verify the disks are correct

In the Longhorn UI: **Node** tab → confirm each worker shows its sdb capacity:

| Node | Expected disk capacity |
|------|----------------------|
| k8s-w-01 | ~186 GB |
| k8s-w-02 | ~140 GB |
| k8s-w-03 | ~372 GB |

If any node shows a small disk (~50-60 GB), sdb is not mounted and Longhorn is using
sda. Fix: mount sdb (Part A.1), then in the Longhorn UI Node settings, remove the
wrong disk entry and add the correct path.

From kubectl:

```bash
# Show per-node storage available to Longhorn
kubectl get nodes.longhorn.io -n longhorn-system
```

---

## Part C — NFS StorageClass (ReadWriteMany)

Longhorn is block storage — one pod at a time. Some workloads (GitLab shared files,
configuration shared across replicas) need multiple pods to read and write the same
volume simultaneously. That requires ReadWriteMany access mode, which NFS provides.

The NFS server is already configured on dc at `/k8s-nfs/shared-volumes`.
`nfs-subdir-external-provisioner` acts as a StorageClass that creates a subdirectory
per PVC in that share, provisioned on demand.

### C.1 Install nfs-subdir-external-provisioner

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace kube-system \
  --set nfs.server=192.168.15.10 \
  --set nfs.path=/k8s-nfs/shared-volumes \
  --set storageClass.name=nfs-shared \
  --set storageClass.reclaimPolicy=Retain \
  --set storageClass.archiveOnDelete=false
```

> **`reclaimPolicy: Retain`** — when a PVC is deleted, the directory on the NFS share
> is kept. This prevents accidental data loss. Clean up manually when you're sure.

> **`archiveOnDelete: false`** — when `Retain` is set, this flag has no effect. With
> `Delete` policy, `archiveOnDelete: true` would rename instead of delete the directory.

### C.2 Verify

```bash
kubectl get storageclass
# NAME                 PROVISIONER                                     RECLAIMPOLICY
# longhorn (default)   driver.longhorn.io                              Delete
# nfs-shared           cluster.local/nfs-provisioner-...               Retain

# Test: create a PVC and check it provisions
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-test
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-shared
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc nfs-test
# Expected: STATUS = Bound within a few seconds

# Clean up
kubectl delete pvc nfs-test
```

---

## How Applications Use These StorageClasses

Applications declare what kind of storage they need via a PVC. They never reference
Longhorn or NFS directly.

**Block storage (ReadWriteOnce) — one pod, replicated:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vault-data
  namespace: vault
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

**Shared storage (ReadWriteMany) — multiple pods, shared:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-config
  namespace: myapp
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-shared
  resources:
    requests:
      storage: 1Gi
```

---

## What Broke and Why

### Problem: CSI plugin CrashLoopBackOff after node restart

**Symptom:**
```
longhorn-csi-plugin-xxxxx   2/3   CrashLoopBackOff   18 restarts
```

Logs:
```
Failed to establish connection to CSI driver: context deadline exceeded
Still connecting to unix:///csi/csi.sock
```

**Root cause:** The Longhorn CSI plugin's health check sidecar starts before the
`longhorn-manager` pod on the same node finishes initializing. The socket doesn't
exist yet, the health check fails, and the container crashes. This is a race that
happens consistently after a node reboot because all pods restart simultaneously.

**Fix:** Once `longhorn-manager` reaches `1/1 Ready`, delete the CSI plugin pod:

```bash
kubectl get pods -n longhorn-system -o wide | grep csi-plugin
kubectl delete pod -n longhorn-system <longhorn-csi-plugin-xxxxx>
# The DaemonSet recreates it immediately — this time the socket exists
```

Wait 15 seconds, verify `3/3 Running`.

### Problem: Pod stuck in ContainerCreating — CSI socket missing

**Symptom:**
```
MountVolume.MountDevice failed: dial unix /var/lib/kubelet/plugins/driver.longhorn.io/csi.sock:
  connect: no such file or directory
```

**Root cause:** The kubelet tried to mount the volume before the CSI socket was ready
(same race as above, seen from the pod's side). Even after the CSI plugin recovers,
the kubelet's backoff timer may not retry for several minutes.

**Fix:** Delete the pod — it reschedules and tries the mount fresh:

```bash
kubectl delete pod -n <namespace> <pod-name>
```

### Problem: Volume stuck Degraded — replica on one node unavailable

**Symptom:** In the Longhorn UI, a volume shows `Degraded` with 2 of 3 replicas healthy.

**Root cause:** The node hosting the third replica is either down or the disk is full.

**Fix:**
```bash
# Check which node is unhealthy
kubectl get nodes
# If a node is NotReady, restart it. Longhorn will rebuild the replica automatically.

# If the disk is full on one node, check usage:
ssh <admin-user>@<node-ip> "df -h /var/lib/longhorn"
```

Longhorn rebuilds replicas automatically once the node is back. No manual intervention
needed unless the volume has been degraded for >24 hours (at which point Longhorn may
need the replica recreated manually via the UI).

---

## Verifying Storage Health

From the Longhorn UI (`https://longhorn.skilz.io`):

- **Dashboard** — total capacity, used space, replica health summary
- **Volume** tab — each PVC, replica count, state, last backup
- **Node** tab — per-node disk usage, scheduler status

From kubectl:

```bash
# All volumes
kubectl get volumes.longhorn.io -n longhorn-system

# All PVCs across all namespaces
kubectl get pvc -A

# Check a specific PVC
kubectl describe pvc <pvc-name> -n <namespace>
```

A healthy Longhorn volume shows `State: attached`, `Robustness: healthy`, and all
replicas in `RW` mode.

---

## What Runs on Longhorn in This Cluster

| Application | PVC | Size |
|-------------|-----|------|
| Vault (HA Raft, 3 pods) | `data-vault-0/1/2` | 2 Gi each |
| Prometheus | `prometheus-data` | 10 Gi |
| GitLab | `gitlab-data`, `gitlab-config`, `gitlab-logs` | 15 Gi, 1 Gi, 2 Gi |

---

**Next: [Module 04 — Vault Secrets Management](../04-secrets/vault-secrets-management.md)**
