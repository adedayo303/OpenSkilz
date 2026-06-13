# Day 0 — Infrastructure Setup

> **Goal:** All nodes created, powered on, with static IPs, SSH accessible, and reachable from a jumpbox. Kubernetes is not installed here; that is Module 01. This document is the physical foundation only.

---

## Why This Architecture

The goal was to build SkilzNetObserv, a browser-based network visibility tool, on something resilient, secure, and self-healing. Kubernetes was the natural choice for that. But rather than a minimal single-node deployment, the aim was to get as close as reasonably possible to a production-ready platform: proper identity, a real PKI chain, persistent storage, HA control planes, and network segmentation. Every decision in the topology reflects that goal.

## Not Everyone Has Nine VMs — and That Is Fine

The full platform in this guide spans nine virtual machines across three subnets, with a
dedicated domain controller, an offline Root CA, distributed block storage, and HA control
planes. That topology exists because it reflects how a real on-premises Kubernetes platform
is designed: with fault tolerance, network segmentation, and a proper PKI chain.

Not everyone has the hardware to match that, and you do not need to in order to learn from
this material or get SkilzNetObserv running.

### A minimal setup that works on a single laptop

If resources are limited, the following two or three VMs are enough to follow the concepts
throughout this guide and run every application:

```
VM 1  Kubernetes master and worker (combined)
  RAM: 6–8 GB
  CPU: 2–4 vCPU
  Disk: 60 GB OS + 100 GB second disk (for storage)
  Role: runs kubeadm in single-node mode (control plane tolerates scheduling)
        hosts NetObserv, Nautobot, Vault, and the observability stack

VM 2  Services (optional but recommended)
  RAM: 4 GB
  CPU: 2 vCPU
  Disk: 40 GB
  Role: runs the domain controller, DNS, and acts as the CA
        can combine Root CA and SubCA on one machine at this scale

VM 3  Jumpbox (optional)
  RAM: 2 GB
  CPU: 1 vCPU
  Disk: 20 GB
  Role: kubectl, helm, VS Code Remote SSH
        can be omitted entirely; your laptop fills this role directly
```

A single Ubuntu VM running Kubernetes in single-node mode alongside a Windows Server VM
for Active Directory and certificates is a completely valid starting point. The Kubernetes
concepts (pod scheduling, persistent volumes, ingress, secrets management, certificate
issuance) are all present in exactly the same form. What changes is that there are no
failure domain boundaries to demonstrate, not that the concepts themselves are absent.

### What simplifies at smaller scale

**PKI:** The two-tier PKI (offline Root CA on its own VM, SubCA on the domain controller)
is a security best practice for production. At lab scale, running both on the same Windows
Server VM is perfectly reasonable. The CA chain still works identically. The offline Root CA
discipline of powering off the Root CA after signing the SubCA can be simulated by simply
not using the Root CA role again after initial setup.

**Networking:** The three-subnet layout in this guide reflects availability zone separation.
On a single laptop, all VMs can live on the same bridged subnet with no loss of conceptual
value. The Kubernetes networking concepts (Calico CNI, MetalLB L2 VIPs, NGINX Ingress) work
the same way on a flat network.

**High availability:** A single control plane node is not HA, but it is entirely sufficient
for learning. If the goal is understanding how etcd Raft quorum works, that is covered in the
module documentation. If the goal is running the applications and following the build, one
node does the job.

**Storage:** Longhorn on a single node with one disk works. Replication is set to 1 replica
instead of 3. The PVC lifecycle, StorageClass provisioning, and persistent volume concepts
are unchanged.

### Minimum laptop specification

| Component | Minimum | Comfortable |
|---|---|---|
| Host RAM | 16 GB | 32 GB |
| Host CPU | 4 cores | 8 cores |
| Host disk | 200 GB free | 500 GB free |
| Hypervisor | VMware Workstation, Fusion, VirtualBox, Hyper-V, Parallels | Any of these |

With 16 GB of host RAM, running two VMs (one 8 GB Kubernetes node and one 4 GB Windows
Server) leaves 4 GB for the host operating system, which is workable. With 32 GB the
experience is noticeably more comfortable.

The concepts, the tooling, the security patterns, and the application itself are identical
regardless of how many VMs are involved. Start with what you have.

---

## Platform Choice

**This platform runs on any hypervisor or bare metal that can host VMs.** The instructions below cover Linux configuration only, not how to click through a specific hypervisor GUI. Use whatever you have.

| Platform | Notes |
|---|---|
| **Proxmox VE** | Best choice for a dedicated lab server. Free, KVM-based, web UI. Use bridged networking on the physical NIC. |
| **KVM / libvirt** | Native Linux hypervisor. Use `virt-manager` or `virsh`. Bridge to physical NIC with `br0`. |
| **VMware ESXi** | Enterprise choice. Add VMs to a port group with uplink to physical switch. |
| **VMware Workstation / Fusion** | Desktop hypervisor on Windows or Mac. Set network adapter to Bridged. |
| **Hyper-V** | Windows Server or Windows 11 host. Create an External virtual switch bound to the physical NIC. |
| **VirtualBox** | Free, cross-platform. Set network adapter to Bridged Adapter. |
| **Bare metal** | Any combination of physical machines on the same LAN works — nodes do not need to be VMs. |
| **Cloud VMs** | Any cloud provider (EC2, Azure, GCP) with a VPC where VMs can reach each other. Adjust the IP plan to match your CIDR. |

**One networking requirement applies to all platforms:** every node must be able to reach every other node by its static IP. They do not need to be on the same subnet, as this platform itself demonstrates with three separate /24 networks. What does not work is outbound NAT, where the hypervisor translates a VM's address to the host IP. With NAT, other nodes send traffic to an address that is not routable back to the source VM. Use bridged, passthrough, or routed networking so each VM has a real IP that other machines can reach directly. If your VMs span multiple subnets, ensure the host or your router has routes between them.

---

## Node Specifications

Nine nodes in total. Two are Windows Server (for AD and Root CA). Seven are Ubuntu Linux.

### Windows nodes

| Hostname | IP | Role | RAM | CPU | Disk |
|---|---|---|---|---|---|
| rootca | 192.168.13.199 | Offline Root CA — **powered off after Day 1** | 2 GB | 1 | 60 GB |
| dc | 192.168.15.10 | Domain Controller, DNS, SubCA, NFS | 4 GB | 2 | 80 GB OS + 500 GB data |

The Root CA VM is powered off permanently after it signs the SubCA certificate on Day 1. It is only powered on again if the SubCA needs to be re-issued (e.g. the DC is rebuilt). Windows Server 2022 Evaluation is available free for 180 days from Microsoft.

The Domain Controller's second disk (500 GB) is formatted as the NFS share used by Kubernetes for ReadWriteMany volumes. Longhorn uses the dedicated sdb disks on worker nodes for ReadWriteOnce.

### Kubernetes nodes

| Hostname | IP | Role | RAM (min) | RAM (rec) | CPU | OS disk | Data disk |
|---|---|---|---|---|---|---|---|
| k8s-cp-01 | 192.168.14.11 | Control plane | 4 GB | 8 GB | 2 | 60 GB | — |
| k8s-cp-02 | 192.168.15.11 | Control plane | 4 GB | 8 GB | 2 | 60 GB | — |
| k8s-cp-03 | 192.168.16.11 | Control plane | 4 GB | 8 GB | 2 | 60 GB | — |
| k8s-w-01 | 192.168.14.12 | Worker | 4 GB | 8 GB | 2 | 60 GB | 200 GB (Longhorn) |
| k8s-w-02 | 192.168.15.12 | Worker | 4 GB | 8 GB | 2 | 60 GB | 150 GB (Longhorn) |
| k8s-w-03 | 192.168.16.12 | Worker, NetObserv ERSPAN collector | 4 GB | 8 GB | 2 | 60 GB | 400 GB (Longhorn) |
| k8s-jb | 192.168.13.245 | Jumpbox — kubectl, helm, admin access | 2 GB | 4 GB | 2 | 40 GB | — |
| gitlab-srv | 192.168.14.13 | GitLab CE + runner | 8 GB | 16 GB | 4 | 100 GB | — |

**4 GB works but is tight.** With all platform components scheduled, a control plane or worker at 4 GB runs at 80–90% memory utilisation, enough to operate, but there is no headroom for traffic spikes or a misbehaving pod. 8 GB is the comfortable production-like target. See the [Resource Calculator](#resource-calculator) below for a full breakdown.

`gitlab-srv` is created after all K8s applications are running and stable, not during Day 0.

Worker nodes each need a **second disk** (`sdb`) separate from the OS disk. Leave it unformatted. Longhorn (Module 03) claims and formats it. Do not add it to any partition or filesystem manually.

---

## Resource Calculator

This section shows every workload scheduled on the cluster, its typical memory footprint, and which node type hosts it. Use this to size your nodes and understand where memory pressure comes from.

All figures are measured steady-state RSS (resident set size), not limits. Kubernetes itself does not enforce limits on most of these unless you set them manually.

### Control Plane — Per Node

Three control planes run on k8s-cp-01, k8s-cp-02, k8s-cp-03. Each runs the same set of processes.

| Process | Typical Memory | Notes |
|---|---|---|
| kube-apiserver | 500 MB – 1 GB | Largest consumer. Scales with number of objects and connected clients. |
| etcd | 300 – 500 MB | Memory-mapped files. Grows slowly as cluster state accumulates. |
| kube-controller-manager | 150 MB | Steady. Only the leader is active; the others idle. |
| kube-scheduler | 100 MB | Minimal. Mostly idle between scheduling decisions. |
| kube-vip | 50 MB | One leader active, two standby at ~10 MB each. |
| kubelet | 200 MB | Runs on every node. |
| Calico node daemon | 200 MB | CNI — routes and enforces network policy. |
| CoreDNS (2 pods, spread) | 100 MB | May land on control planes depending on scheduling. |
| OS + system services | 500 MB | Ubuntu base, sshd, journald, etc. |
| **Total per control plane** | **~2.1 – 2.8 GB** | |

**At 4 GB:** ~50–70% utilisation , functional but spikes (certificate renewals, kubectl bulk operations, etcd compaction) push it higher temporarily.

**At 8 GB:** ~25–35% utilisation, with plenty of headroom for etcd growth and API bursts.

---

### Worker Nodes — Base Overhead

Every worker runs these regardless of what applications are scheduled on it:

| Process | Typical Memory | Notes |
|---|---|---|
| kubelet | 200 MB | |
| Calico node daemon | 200 MB | |
| MetalLB speaker | 50 MB | Daemonset — runs on all workers. |
| Longhorn manager | 200 MB | Daemonset — manages storage on the node. |
| Longhorn engine + replica | 150 – 300 MB | Per active volume. Varies with the number of PVCs on the node. |
| OS + system services | 500 MB | |
| **Base overhead** | **~1.3 – 1.5 GB** | Before any application pods |

---

### Application Workloads

These are the pods scheduled across the three worker nodes. Kubernetes decides placement unless a pod has an explicit `nodeAffinity` rule (NetObserv is pinned to k8s-w-03).

| Application | Namespace | Pods | Typical Memory | Notes |
|---|---|---|---|---|
| Vault | `vault` | 3 | 300 MB each (900 MB total) | Raft HA — one pod per worker. |
| cert-manager | `cert-manager` | 3 | 60 MB each (180 MB total) | Controller + webhook + cainjector. |
| NGINX Ingress | `ingress-nginx` | 1 | 150 – 200 MB | Ingress controller. |
| Nautobot | `nautobot` | 1 | 500 MB – 1 GB | Django application. Memory grows with plugin count. |
| PostgreSQL (Nautobot) | `nautobot` | 1 | 300 MB | Nautobot's primary datastore. |
| Redis (Nautobot) | `nautobot` | 1 | 100 MB | Nautobot task queue and cache. |
| Nautobot worker | `nautobot` | 1 | 400 MB | Celery task worker for background jobs. |
| NetObserv | `netobserv` | 1 | 300 – 400 MB | **Pinned to k8s-w-03** (ERSPAN collector). Node.js + tshark subprocess. |
| Prometheus | `monitoring` | 1 | 500 MB – 1 GB | Scales with scrape targets and retention. |
| Grafana | `monitoring` | 1 | 150 – 200 MB | |
| Loki | `monitoring` | 1 | 200 – 300 MB | Log ingestion. |
| Promtail | `monitoring` | 3 | 50 MB each | Daemonset — log shipper on each worker. |
| Kubernetes Dashboard | `kubernetes-dashboard` | 2 | 100 MB total | |
| **Total applications** | | | **~4.1 – 5.8 GB** | Spread across 3 workers |

---

### Per-Worker Estimated Totals

Kubernetes scheduling is not deterministic, but with default spread constraints the distribution approximates:

```
k8s-w-01  (192.168.14.12)
  Base overhead             ~1.4 GB
  Vault pod                  0.3 GB
  cert-manager (all 3)       0.2 GB
  NGINX Ingress              0.2 GB
  Nautobot + PostgreSQL      1.3 GB
  Redis                      0.1 GB
  Promtail                   0.05 GB
  ─────────────────────────────────
  Estimated total            ~3.6 GB    ← 4 GB tight  |  8 GB comfortable

k8s-w-02  (192.168.15.12)
  Base overhead             ~1.4 GB
  Vault pod                  0.3 GB
  Nautobot worker            0.4 GB
  Prometheus                 0.7 GB
  Grafana                    0.2 GB
  Loki                       0.25 GB
  Kubernetes Dashboard       0.1 GB
  Promtail                   0.05 GB
  ─────────────────────────────────
  Estimated total            ~3.4 GB    ← 4 GB tight  |  8 GB comfortable

k8s-w-03  (192.168.16.12)  worker, NetObserv ERSPAN collector
k8s-jb    (192.168.13.245) jumpbox, admin access
  Base overhead             ~1.4 GB
  Vault pod                  0.3 GB
  NetObserv (pinned)         0.4 GB
  Promtail                   0.05 GB
  Jumpbox tools (kubectl, etc) 0.2 GB
  ─────────────────────────────────
  Estimated total            ~2.4 GB    ← 4 GB comfortable  |  8 GB ideal
```

### Summary

| Node type | At 4 GB | At 8 GB |
|---|---|---|
| Control planes | Works — ~60–70% utilised, spikes to 80%+ | Comfortable — ~30–40% |
| Workers w-01, w-02 | Tight — ~85–90% with all apps running | Comfortable — ~45–50% |
| Worker w-03 | Comfortable — ~60% | Plenty of headroom |
| k8s-jb (jumpbox) | Very light — <20% | Only runs kubectl, helm, SSH |
| gitlab-srv | Functional at 8 GB — GitLab needs 4 GB just for the app | 16 GB ideal for CI runners |

**Bottom line:** If you have 8 GB to give each node, do it, as you will spend less time investigating OOMKilled pods. If you are constrained to 4 GB, the platform runs but give the workers the full 4 GB and keep nothing else running on the host machine during heavy operations like `kubectl apply` on large manifests or Vault unsealing after a restart.

### Virtual IPs (no VM — assigned by software)

| VIP | IP | What manages it |
|---|---|---|
| Kubernetes API HA | 192.168.14.30 | kube-vip |
| Ingress (all services) | 192.168.14.31 | MetalLB |

Reserve all IPs in the tables above outside your DHCP pool, or configure your router to exclude the entire `192.168.13–16.x` range from dynamic assignment.

---

## Network Layout

```
192.168.13.0/24  PKI / management
  .199   rootca        Offline Root CA (powered off after Day 1)
  .245   k8s-jb        Jumpbox, kubectl, helm, SSH to all nodes, VS Code Remote SSH

192.168.14.0/24  K8s zone 1
  .11    k8s-cp-01     Control plane
  .12    k8s-w-01      Worker
  .13    gitlab-srv    GitLab (added later)
  .30    kube-vip      K8s API HA VIP
  .31    MetalLB VIP   Ingress, all *.skilz.io hostnames

192.168.15.0/24  K8s zone 2 and DC
  .10    dc            Domain Controller, DNS, SubCA, NFS
  .11    k8s-cp-02     Control plane
  .12    k8s-w-02      Worker

192.168.16.0/24  K8s zone 3
  .11    k8s-cp-03     Control plane
  .12    k8s-w-03      Worker, NetObserv ERSPAN collector
```

The three-subnet layout mirrors how production clusters span availability zones or physical racks. It also ensures that when a control plane or worker is lost, the failure mode is understood rather than just restarting a process.

---

## Part 1: Create All VMs

Use your chosen hypervisor to create all nodes. Key requirements regardless of platform:

- **Network adapter: bridged to physical NIC** (not NAT, not host-only)
- Ubuntu nodes: Ubuntu Server 24.04 LTS, OpenSSH installed during setup
- Windows nodes: Windows Server 2022 Standard (Desktop Experience)
- Worker nodes: second disk (`sdb`) added and left unformatted
- All VMs reachable by IP from each other (same subnet or routed, not NAT'd)

During Ubuntu install:
- Choose "Ubuntu Server" (not minimised)
- Storage: use entire disk (the OS disk only, do not touch the second disk)
- Tick **Install OpenSSH server**
- Create a local admin user (use a consistent username across all nodes)

---

## Part 2: Configure Static IPs on Ubuntu Nodes

Log into each Ubuntu node (via hypervisor console for first login) and set a static IP.

Ubuntu 24.04 uses **netplan**. Edit the config file:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Template, adjust `addresses`, `via` (your gateway), and the interface name (`ip a` to find it, may be `eth0`, `ens3`, `ens18`, etc.):

```yaml
network:
  version: 2
  ethernets:
    ens3:                         # replace with your interface name
      dhcp4: false
      addresses:
        - 192.168.14.11/24        # replace with this node's IP
      routes:
        - to: default
          via: 192.168.1.1        # replace with your gateway
      nameservers:
        addresses:
          - 192.168.15.10         # dc, becomes DNS after Day 1
          - 8.8.8.8               # fallback until dc is up
```

Apply:
```bash
sudo netplan apply
ip a                    # confirm IP is set
ping 8.8.8.8 -c 3      # confirm gateway and internet reach
```

Repeat for each Ubuntu node using its assigned IP from the table above.

### Static IP on Windows (DC and Root CA)

On each Windows Server node:

1. **Network and Sharing Center → Change adapter settings**
2. Right-click the adapter → **Properties → Internet Protocol Version 4 (TCP/IPv4) → Properties**
3. Set static IP, subnet mask, default gateway
4. Set Primary DNS to `127.0.0.1` on the DC (it is its own DNS once AD is installed), `8.8.8.8` as alternate
5. Rename the computer to match the hostname in the table (**System → Rename this PC**)
6. Restart

---

## Part 3: Set Up the Jumpbox (k8s-jb)

The jumpbox is `k8s-jb` (`192.168.13.245`). All `kubectl`, `helm`, and SSH commands run from here. It lives on the dedicated management subnet (`192.168.13.0/24`) and has no Kubernetes workloads. Connect to it from your local machine via VS Code Remote SSH.

Use whatever machine suits you. The jumpbox can run on Windows, macOS, or Linux. If you already have a Linux VM or a spare machine on the network, that works. The only requirements are: SSH access to all cluster nodes, `kubectl` installed, and a browser for VS Code Remote SSH.

### 3.1 Install Tools

SSH into k8s-jb:

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  curl wget git vim jq \
  net-tools tcpdump nmap \
  nfs-common \
  openssh-server

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -sSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### 3.2 Distribute SSH Keys

Generate a key pair on the jumpbox and copy it to all other nodes so you can SSH without a password:

```bash
ssh-keygen -t ed25519 -C "k8s-jb"

for node in 192.168.14.11 192.168.15.11 192.168.16.11 \
            192.168.14.12 192.168.15.12; do
  ssh-copy-id <admin-user>@${node}
done
```

Test:
```bash
ssh <admin-user>@192.168.14.11 hostname   # should return: k8s-cp-01
```

### 3.3 SSH Config (Optional Convenience)

```bash
cat >> ~/.ssh/config << 'EOF'

Host cp01
  HostName 192.168.14.11
  User <admin-user>
Host cp02
  HostName 192.168.15.11
  User <admin-user>
Host cp03
  HostName 192.168.16.11
  User <admin-user>
Host w01
  HostName 192.168.14.12
  User <admin-user>
Host w02
  HostName 192.168.15.12
  User <admin-user>
EOF
```

### 3.4 Connect VS Code

1. Install the **Remote - SSH** extension in VS Code
2. `Ctrl+Shift+P` → **Remote-SSH: Connect to Host**
3. Enter `<admin-user>@192.168.16.12`

All further work happens from this VS Code window connected to k8s-jb (`192.168.13.245`).

---

## Part 4: Configure All Nodes

Run these steps on **every Ubuntu node** (k8s-cp-01/02/03, k8s-w-01/02/03).

### NTP

Certificate validation fails if clocks are out of sync. All nodes must point at the same NTP source, which is the domain controller (`192.168.15.10`).

```bash
sudo tee /etc/systemd/timesyncd.conf > /dev/null << 'EOF'
[Time]
NTP=192.168.15.10
FallbackNTP=pool.ntp.org
RootDistanceMaxSec=30
EOF

sudo systemctl restart systemd-timesyncd
timedatectl status
# Look for: System clock synchronized: yes
```

> `RootDistanceMaxSec=30` is required. Windows DCs are not stratum-1 time servers and Ubuntu rejects them by default without this setting.

Run across all nodes from k8s-jb:

```bash
for node in 192.168.14.11 192.168.15.11 192.168.16.11 \
            192.168.14.12 192.168.15.12; do
  ssh <admin-user>@${node} "
    sudo tee /etc/systemd/timesyncd.conf > /dev/null << 'EOF'
[Time]
NTP=192.168.15.10
FallbackNTP=pool.ntp.org
RootDistanceMaxSec=30
EOF
    sudo systemctl restart systemd-timesyncd
    echo \$(hostname): \$(timedatectl | grep synchronized)"
done
```

### nfs-common

Required on all nodes for the NFS StorageClass (Module 03):

```bash
for node in 192.168.14.11 192.168.15.11 192.168.16.11 \
            192.168.14.12 192.168.15.12; do
  ssh <admin-user>@${node} "sudo apt install -y nfs-common && echo \$(hostname): done"
done
```

### Disable automatic updates

Unattended upgrades can restart nodes mid-operation:

```bash
for node in 192.168.14.11 192.168.15.11 192.168.16.11 \
            192.168.14.12 192.168.15.12; do
  ssh <admin-user>@${node} "sudo systemctl disable --now unattended-upgrades"
done
```

---

## Part 5: Verify Connectivity

From k8s-jb, confirm all nodes are reachable:

```bash
for ip in 192.168.15.10 \
          192.168.14.11 192.168.15.11 192.168.16.11 \
          192.168.14.12 192.168.15.12; do
  echo -n "$ip ... "
  ping -c 1 -W 1 $ip &>/dev/null && echo "OK" || echo "UNREACHABLE"
done
```

All nodes should respond OK. If any are unreachable:
- Confirm the VM is powered on
- Confirm the network adapter is set to bridged (not NAT)
- Run `ip a` on the unreachable node via hypervisor console and verify the IP

---

## End of Day 0 Checklist

- [ ] All 9 VMs created (rootca, dc, cp-01/02/03, w-01/02/03, gitlab-srv deferred)
- [ ] All VMs on bridged networking, no NAT
- [ ] Windows VMs: static IPs set, computers renamed
- [ ] Ubuntu VMs: static IPs set via netplan, SSH enabled
- [ ] Worker nodes: second disk (`sdb`) added, unformatted
- [ ] DC: second disk (500 GB) added, will become NFS share on Day 1
- [ ] SSH keys distributed, `ssh cp01 hostname` works without password from k8s-jb
- [ ] VS Code connected to k8s-jb (192.168.13.245) via Remote SSH
- [ ] All nodes reachable by ping from k8s-jb
- [ ] NTP configured on all nodes (`System clock synchronized: yes`)
- [ ] `nfs-common` installed on all nodes
- [ ] `kubectl` installed on k8s-jb
- [ ] Automatic updates disabled on all nodes

**Next: [Day 1 Active Directory, DNS, and Certificate Authority](day1-active-directory-and-pki.md)**
