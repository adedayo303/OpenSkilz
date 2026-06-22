# Module 01 — HA Kubernetes Cluster
## Building the Control Plane: Why Three Masters, What etcd Actually Does, and How kube-vip Works

---

## 1. The Problem This Solves

Before building anything, you need to understand what problem a highly available (HA)
Kubernetes cluster is solving — because if you don't know the problem, you won't know
when you're solving it correctly.

**The single control plane problem:**
In a basic Kubernetes setup, there is one control plane node. That node runs the API
server, the scheduler, the controller manager, and etcd (the database). If that node
goes down — reboots, loses power, fails — your entire cluster stops working. Existing
pods continue running, but:
- No new pods can be scheduled
- No configuration changes take effect
- No health checks can repair crashed pods
- kubectl stops working entirely

For a lab, that's fine. For production, it is not acceptable.

**The solution:** Three control plane nodes. Requests go to whichever one is up. As long
as any two of the three are alive (quorum), the cluster keeps working. This is what
we built.

---

## 2. Concept: etcd — The Brain

`etcd` is a distributed key-value store. Every piece of cluster state lives here:
- Every node that has joined
- Every pod, service, deployment, secret
- Every config map, RBAC policy, network policy
- The current desired state of everything

**The networking parallel:** Think of etcd like a distributed routing table. In BGP, your
RIB (Routing Information Base) stores all the routes. In Kubernetes, etcd stores all the
cluster state. Just as losing your RIB would break routing, losing etcd would lose your
cluster. Calico actually uses etcd internally for the same reason — it needs a consistent
store for its BGP state when running in etcd mode.

**Quorum:** etcd uses the Raft consensus algorithm. For a cluster of `n` nodes, it needs
`(n/2 + 1)` nodes alive to accept writes. With 3 nodes, you need 2 alive. With 5 nodes,
you need 3. **You should always run an odd number of etcd nodes** — never 2 or 4.

| etcd nodes | Quorum needed | Can tolerate loss of |
|---|---|---|
| 1 | 1 | 0 nodes |
| 3 | 2 | 1 node |
| 5 | 3 | 2 nodes |

In this lab: 3 etcd nodes (one on each control plane), tolerates loss of 1 CP node.

**What writes to etcd:** Only the API server (`kube-apiserver`) reads and writes etcd
directly. Everything else — kubelet, kubectl, controllers — talks to the API server.
etcd is never exposed directly outside the cluster.

---

## 3. Concept: The Control Plane Components

Each control plane node runs four processes:

```
┌───────────────────────────────────────────────────┐
│              Control Plane Node                    │
│                                                   │
│  kube-apiserver        ← The front door. All      │
│  (port 6443)             kubectl commands land    │
│                          here. Validates, auth,   │
│                          writes to etcd.          │
│                                                   │
│  etcd                  ← The database. Stores     │
│  (port 2379/2380)        all cluster state.       │
│                                                   │
│  kube-controller-manager ← Watches for drift.    │
│                            "3 replicas desired,   │
│                             2 running — create 1" │
│                                                   │
│  kube-scheduler        ← "Which node should       │
│                           this pod run on?"       │
│                           (CPU, memory, affinity) │
└───────────────────────────────────────────────────┘
```

**The networking parallel for kube-scheduler:**
A scheduler picking a node for a pod is exactly like OSPF or IS-IS picking a path for a
packet. It looks at available resources (metrics), constraints (node selectors, taints),
and picks the best fit. In both cases, the algorithm is making an optimal placement
decision based on current state.

---

## 4. Concept: kube-vip — The Virtual IP

With 3 control plane nodes at 192.168.14.11, 192.168.15.11, and 192.168.16.11, kubectl
(and all cluster components) need a single stable endpoint. You cannot tell workers "talk
to .11 or .15.11 or .16.11, whichever is up." That is not how TCP works.

**kube-vip** is a lightweight process that runs on each control plane node and elects a
leader. The leader owns a virtual IP (VIP): `192.168.14.30`. When the leader fails, another
node wins the election and the VIP moves to that node via a gratuitous ARP announcement.

**The networking parallel:** This is identical to VRRP (Virtual Router Redundancy
Protocol) or HSRP (Cisco's proprietary version). A master router owns the virtual IP
and MAC. If the master fails, a backup takes over with a gratuitous ARP. kube-vip
does exactly this for the Kubernetes API endpoint.

```
Normal operation:                    k8s-cp-01 fails:
k8s-cp-01 owns 192.168.14.30         k8s-cp-02 wins election
  │                                 k8s-cp-02 sends gratuitous ARP
  └─ GARP: "I am 192.168.14.30"      k8s-cp-02 owns 192.168.14.30
                                      │
                                      └─ GARP: "I am now 192.168.14.30"
```

DNS record `k8s-api.skilz.io` → `192.168.14.30`. Workers, kubectl, and all
control plane components point to this single address. They do not need to know which
physical node is currently the leader.

**kube-vip mode used here:** ARP mode (L2). Requires all nodes on the same L2 segment.
For multi-site deployments, you would use BGP mode instead — kube-vip would advertise
the VIP via BGP to your network, exactly like announcing a loopback via OSPF.

---

## 5. Concept: kubelet — The Node Agent

Every node (control plane and worker) runs one process that Kubernetes requires: `kubelet`.

kubelet is the local agent. It:
- Receives pod specifications from the API server
- Talks to the container runtime (containerd) to start/stop containers
- Reports node status back to the API server
- Runs the liveness and readiness probes for every pod on that node

If kubelet stops, the node stops working. The API server marks it `NotReady` after the
node grace period (default 40 seconds), and pods on that node get evicted.

**The networking parallel:** kubelet is like an NMS agent (think SNMP agent or a Cisco
DNA Center plug-in) running on every device. The API server is the NMS controller. kubelet
is the local enforcement point that makes the desired state real.

---

## 6. Concept: The Container Runtime (containerd)

Kubernetes does not run containers itself. It delegates that to a **container runtime**.
In this lab: `containerd` v2.2.4 (the same runtime that Docker uses internally).

The chain is:
```
kubectl apply → API server → kubelet → containerd → runc → container
```

`runc` is the low-level tool that actually calls Linux kernel namespaces and cgroups to
create isolated processes. Everything above it is orchestration.

**Why containerd and not Docker?**
Kubernetes removed Docker as a supported runtime in v1.24 (the "dockershim" deprecation).
containerd is lighter, purpose-built for Kubernetes, and does not carry Docker's extra
layers (build, compose, etc.). If you want to build images, you still use Docker on a
build machine. The cluster runtime does not need to build images.

---

## 7. Concept: cgroups — Resource Isolation

`cgroups` (control groups) is the Linux kernel feature that limits what resources a
process can use — CPU, memory, network bandwidth. It is the foundation of container
resource isolation.

Kubernetes uses cgroups to enforce the `resources.limits` and `resources.requests` you
set on each pod. When you say:

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "50m"
  limits:
    memory: "512Mi"
```

...kubelet translates that into cgroup settings. The container cannot use more than 512Mi
of memory regardless of what the host has available.

**cgroup v1 vs v2:**
- cgroup v1: The older interface. Multiple hierarchies.
- cgroup v2: Unified hierarchy. More efficient. Required by some Kubernetes features.
  Ubuntu 24.04 uses v2 by default.

**What broke in this lab:** k8s-w-03 had swap enabled. kubelet refused to start because
running with swap violates Kubernetes' memory accounting assumptions. If a pod is OOM-killed,
K8s expects the memory to be immediately reclaimed. With swap, the kernel might swap it out
instead, confusing the memory pressure signals.

**Fix applied:** Disabled swap entirely (`swapoff -a`, commented out `/etc/fstab` entry).

---

## 8. What We Built — Deployed State

### Component versions

| Component | Version | Notes |
|---|---|---|
| Kubernetes | v1.32.13 | kubeadm-bootstrapped |
| containerd | v2.2.4 | Container runtime |
| kube-vip | v1.2.0 | ARP mode, VIP 192.168.14.30 |
| Calico | v3.30.1 | CNI, calico-system namespace |
| MetalLB | v0.14.9 | L2 mode, pool 192.168.14.31–192.168.14.50 |
| cert-manager | v1.17.2 | + ADCS issuer for Windows SubCA |
| NGINX Ingress | v1.12.2 | ingress-nginx namespace |
| Longhorn | v1.12.0 | longhorn-system namespace, sdb disks |
| Vault | v2.0.2 | 3-pod Raft HA, vault namespace |

### Node reference

| Hostname | IP | Role | Disk |
|---|---|---|---|
| k8s-cp-01 | 192.168.14.11 | Control plane + etcd | sda (OS only) |
| k8s-cp-02 | 192.168.15.11 | Control plane + etcd | sda (OS only) |
| k8s-cp-03 | 192.168.16.11 | Control plane + etcd | sda (OS only) |
| k8s-w-01 | 192.168.14.12 | Worker | sda (OS) + sdb 200 GB (Longhorn) |
| k8s-w-02 | 192.168.15.12 | Worker | sda (OS) + sdb 150 GB (Longhorn) |
| k8s-w-03 | 192.168.16.12 | Worker | sda (OS) + sdb 400 GB (Longhorn) |

**Management box** (`mgmt-host`, 192.168.13.245): where `kubectl`, Docker, and all
automation tooling runs. Not a K8s node. All `kubectl` commands in this doc run from here.

All nodes: Ubuntu 24.04.4 LTS, kernel 6.17.0-35-generic, joined to `SKILZ.IO` domain.

---

## 9. Step by Step — What Was Run

### 9.1 Pre-flight on every node

Every node needed the same base setup before kubeadm could run:

```bash
# Disable swap — Kubernetes requires this
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

# Load kernel modules needed for pod networking
modprobe overlay
modprobe br_netfilter

cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# sysctl: allow iptables to see bridged traffic
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system
```

**Why `net.bridge.bridge-nf-call-iptables = 1`?**
By default, the Linux kernel does not pass bridged traffic through netfilter (iptables).
But Kubernetes uses iptables rules to implement Services (ClusterIP, NodePort) and network
policies. Without this setting, traffic between pods on the same node would bypass those
rules. For a network engineer: this is equivalent to enabling `ip inspect` on a bridge
interface — you're telling the kernel to apply L3 policy to L2 traffic.

### 9.2 Install containerd

```bash
apt-get install -y containerd
containerd config default > /etc/containerd/config.toml

# Enable systemd cgroup driver (required for Kubernetes)
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
```

**Why SystemdCgroup = true?** kubelet and containerd must agree on which cgroup driver
to use. If containerd uses cgroupfs directly but kubelet expects systemd, they create
conflicting cgroup paths. Setting both to systemd ensures consistency.

### 9.3 Install kubeadm, kubelet, kubectl

```bash
# Add Kubernetes apt repo (v1.32)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" \
  > /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

| Tool | What it does |
|---|---|
| `kubeadm` | One-time bootstrap tool. Initialises the cluster, generates certs, creates the initial configuration. Not needed after setup. |
| `kubelet` | The node agent. Runs permanently as a systemd service on every node. |
| `kubectl` | The CLI. Run from the management box (mgmt-host). |

### 9.4 Install kube-vip on the first control plane

Before `kubeadm init`, kube-vip needs to be in place as a static pod. Static pods are
defined in `/etc/kubernetes/manifests/` and kubelet starts them directly, without the
API server. This is how the API server itself starts — it is a static pod.

```bash
export VIP=192.168.14.30
export INTERFACE=ens192   # adjust per node

ctr image pull ghcr.io/kube-vip/kube-vip:v1.2.0

ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:v1.2.0 vip \
  /kube-vip manifest pod \
  --interface $INTERFACE \
  --address $VIP \
  --controlplane \
  --arp \
  --leaderElection \
  | tee /etc/kubernetes/manifests/kube-vip.yaml
```

### 9.5 Initialise the cluster (k8s-cp-01 only)

```bash
kubeadm init \
  --control-plane-endpoint "k8s-api.skilz.io:6443" \
  --pod-network-cidr 10.244.0.0/16 \
  --upload-certs
```

**Flag breakdown:**

`--control-plane-endpoint k8s-api.skilz.io:6443`
: All cluster components use this address to talk to the API server. It points to the
  kube-vip VIP (192.168.14.30). If you used a node's direct IP, adding new control planes
  would require reconfiguring all workers. The VIP abstracts that.

`--pod-network-cidr 10.244.0.0/16`
: The IP range Kubernetes will use for pod IPs. Each node gets a /24 slice. This is the
  range Calico manages.

`--upload-certs`
: Generates a certificate key and uploads cluster certificates to a Kubernetes Secret.
  Required for joining additional control plane nodes securely.

Output includes:
- A `kubeadm join` command for control plane nodes (includes `--certificate-key`)
- A `kubeadm join` command for worker nodes

```bash
# Copy kubeconfig to management box
mkdir -p $HOME/.kube
scp root@192.168.14.11:/etc/kubernetes/admin.conf $HOME/.kube/config
```

### 9.6 Install Calico CNI

Without a CNI plugin, pods cannot communicate. Calico v3.30.1 is installed using the
Tigera operator method (which places resources in `calico-system` namespace):

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.1/manifests/tigera-operator.yaml

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.1/manifests/custom-resources.yaml
```

The full Calico deep-dive is in Module 02.

### 9.7 Join remaining control plane nodes

Run on k8s-cp-02 and k8s-cp-03 (using the join command from kubeadm init output):

```bash
# Copy kube-vip manifest to each CP node first
scp /etc/kubernetes/manifests/kube-vip.yaml <admin-user>@192.168.15.11:/tmp/
ssh <admin-user>@192.168.15.11 'sudo mv /tmp/kube-vip.yaml /etc/kubernetes/manifests/'

# Then join
kubeadm join k8s-api.skilz.io:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key>
```

Repeat for k8s-cp-03 (192.168.16.11).

### 9.8 Join worker nodes

```bash
kubeadm join k8s-api.skilz.io:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

Workers run:
- kubelet
- kube-proxy
- Application pods (via containerd)

They do not run etcd or any control plane components.

---

## 10. Verification

```bash
kubectl get nodes -o wide
```

Expected output:
```
NAME        STATUS   ROLES           AGE   VERSION    INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-cp-01   Ready    control-plane   Xd    v1.32.13   192.168.14.11   <none>        Ubuntu 24.04.4 LTS   6.17.0-35-generic   containerd://2.2.4
k8s-cp-02   Ready    control-plane   Xd    v1.32.13   192.168.15.11   <none>        Ubuntu 24.04.4 LTS   6.17.0-35-generic   containerd://2.2.4
k8s-cp-03   Ready    control-plane   Xd    v1.32.13   192.168.16.11   <none>        Ubuntu 24.04.4 LTS   6.17.0-35-generic   containerd://2.2.4
k8s-w-01    Ready    <none>          Xd    v1.32.13   192.168.14.12   <none>        Ubuntu 24.04.4 LTS   6.17.0-35-generic   containerd://2.2.4
k8s-w-02    Ready    <none>          Xd    v1.32.13   192.168.15.12   <none>        Ubuntu 24.04.4 LTS   6.17.0-35-generic   containerd://2.2.4
k8s-w-03    Ready    <none>          Xd    v1.32.13   192.168.16.12   <none>        Ubuntu 24.04.4 LTS   6.17.0-35-generic   containerd://2.2.4
```

```bash
# Verify etcd cluster health
kubectl -n kube-system exec etcd-k8s-cp-01 -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health --cluster
```

Expected:
```
https://192.168.14.11:2379 is healthy: successfully committed proposal: took = Xms
https://192.168.15.11:2379 is healthy: successfully committed proposal: took = Xms
https://192.168.16.11:2379 is healthy: successfully committed proposal: took = Xms
```

```bash
# Verify kube-vip is running on each CP node
kubectl -n kube-system get pods | grep kube-vip
# Expected: kube-vip-k8s-cp-01   Running   192.168.14.11   k8s-cp-01
#           kube-vip-k8s-cp-02   Running   192.168.15.11   k8s-cp-02
#           kube-vip-k8s-cp-03   Running   192.168.16.11   k8s-cp-03

# Verify VIP is reachable
ping 192.168.14.30
```

---

## 11. What Broke — and What It Teaches

### Bug 1: k8s-w-03 failed to join — kubelet healthz timeout

**Symptom:**
```
[kubelet-check] The HTTP call equal to 'curl -sSL http://127.0.0.1:10248/healthz'
failed with error: Get "http://127.0.0.1:10248/healthz": dial tcp: connection refused
```

**Root cause:** Swap was enabled on k8s-w-03. kubelet checks for swap on startup and
refuses to run by default.

**Why Kubernetes hates swap:**
Kubernetes memory limits assume that when a container hits its limit, the OOM killer
terminates it and that memory is immediately available. With swap, a process hitting its
memory limit might just get swapped to disk instead — appearing to run but actually
degraded, and consuming disk I/O. Kubernetes cannot meaningfully schedule or limit memory
if swap is in play.

**Fix:**
```bash
sudo swapoff -a
sudo sed -i '/swap/s/^/#/' /etc/fstab
sudo systemctl restart kubelet
```

**Lesson for platform engineers:**
When designing systems, understand the assumptions a tool makes about its environment.
Before adding any node to a production cluster, run a pre-flight check (`kubeadm preflight`).

### Bug 2: Multiple failed join attempts left stale state

**Symptom:** After each failed join attempt, the next attempt also failed even after
fixing the root cause.

**Root cause:** `kubeadm join` writes files before it discovers the error:
- `/etc/kubernetes/kubelet.conf` — references certificates that were never created
- `/var/lib/kubelet/pki/` — partial PKI state

**Fix — full cleanup before retry:**
```bash
sudo systemctl stop kubelet
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd /etc/cni/net.d
sudo systemctl start kubelet
# Then re-run kubeadm join with a fresh token
```

**Lesson:** kubeadm init and join assume a clean slate. If they fail partway, always clean
up completely before retrying. This is analogous to a failed OSPF neighbour that left
partial state in the LSDB — you would clear the OSPF process, not just retry adjacency.

### Bug 3: unattended-upgrades consuming memory during setup

**Symptom:** calico-node on k8s-w-02 kept getting OOM-killed (exit 137).

**Root cause:** Ubuntu's `unattended-upgrades` service was running during debugging,
consuming ~300 MB of RAM and leaving insufficient memory for calico-node startup.

**Fix:**
```bash
# Disable on all nodes — run from management box
for NODE in 192.168.14.11 192.168.15.11 192.168.16.11 192.168.14.12 192.168.15.12 192.168.16.12; do
  sshpass -p '<vault-managed>' ssh <admin-user>@$NODE \
    "echo '<vault-managed>' | sudo -S systemctl disable --now unattended-upgrades"
done
```

### Bug 4: containerd not picking up insecure registry config

When the NetObserv temporary local registry was configured later (Module 06), containerd
refused to pull from `<your-registry>` even after creating
`/etc/containerd/certs.d/<your-registry>/hosts.toml`.

**Root cause:** `/etc/containerd/config.toml` had `config_path = ''` — containerd was
not reading the `certs.d` directory at all.

**Fix on each worker node:**
```bash
sudo sed -i "s|config_path = ''|config_path = '/etc/containerd/certs.d'|g" \
  /etc/containerd/config.toml
sudo systemctl restart containerd
```

**Lesson:** containerd's `certs.d` host override mechanism (the modern registry
configuration approach) requires `config_path` to be explicitly set. It is empty by
default in the generated config.

---

## 12. Key Commands for Daily Operations

```bash
# Overall cluster health
kubectl get nodes -o wide
kubectl get pods -A | grep -v Running

# Control plane component status (static pods in kube-system)
kubectl get pods -n kube-system | grep -E 'etcd|apiserver|controller|scheduler'

# Check which CP node owns the kube-vip VIP
kubectl -n kube-system get pods | grep kube-vip

# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods -A

# Events (useful for debugging scheduling failures)
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# Drain a node for maintenance (moves pods off safely)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon after maintenance
kubectl uncordon <node-name>
```

---

## 13. Concepts Summary

| Concept | K8s term | Networking analogy |
|---|---|---|
| Cluster database | etcd | RIB / routing table |
| Single stable API endpoint | kube-vip VIP | VRRP / HSRP virtual IP |
| Node agent | kubelet | NMS device agent |
| Container runtime | containerd / runc | The thing that actually does the work |
| Resource isolation | cgroups | QoS / policing / shaping |
| Cluster bootstrap | kubeadm | Initial device provisioning / ZTP |
| CNI plugin | Calico | The routing protocol stack (see Module 02) |

---

*Next: [Module 02 — Kubernetes Networking: Calico BGP, MetalLB, NGINX Ingress](../02-networking/calico-bgp-metallb-nginx.md)*
