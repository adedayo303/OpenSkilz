# Module 02 — Kubernetes Networking
## Calico BGP, Pod CIDRs, MetalLB, and NGINX Ingress — Explained for a Network Engineer

---

## 1. The Problem

Kubernetes defines a networking model but does not ship a complete implementation of it.
The model requires that every pod gets its own IP address and that pods can reach each
other directly, without NAT, regardless of which node they run on. Implementing that
across nodes is the job of a CNI plugin. Kubernetes does not include one; the cluster
operator chooses one. Kubernetes does provide kube-proxy, which implements Service
networking (stable virtual IPs that load-balance across a set of pods) using iptables
rules on every node. The official Kubernetes documentation lists kube-proxy as an
optional component, since some CNI plugins replace its function entirely, but a standard
kubeadm cluster, including this lab, installs it by default. A bare cluster still has no
way for traffic from outside it to reach a service. That needs a LoadBalancer
implementation, and for routing by hostname, an Ingress controller.

| Problem | What it means | Provided by | Solved in this lab by |
|---|---|---|---|
| Pod-to-pod networking | Same node, or different nodes, without NAT | A CNI plugin (not included; the operator chooses one) | Calico, with BGP |
| Service networking | Stable virtual IPs that load-balance across pods | kube-proxy (part of core Kubernetes, officially optional, on by default via kubeadm) | kube-proxy, using iptables rules |
| External access | Traffic from outside the cluster reaching a service | A LoadBalancer implementation and an Ingress controller (not included) | MetalLB assigns the external IP; NGINX Ingress routes it by hostname and path |

---

## 2. Pod Networking — Calico BGP

### 2.1 The pod CIDR

When we initialised the cluster with `--pod-network-cidr 10.244.0.0/16`, we told
Kubernetes: "all pod IPs will come from this range." Kubernetes does not assign IPs
itself — it hands slices of this range to each node, and the CNI plugin (Calico) manages
the actual assignment.

Each node gets a `/26` (or `/24` depending on Calico configuration) out of `10.244.0.0/16`:

```
Node        Pod CIDR          Example pod IPs
cp-01       10.244.84.0/26    10.244.84.1 - 10.244.84.62
cp-02       10.244.65.0/26    10.244.65.1 - 10.244.65.62
cp-03       10.244.194.0/26   10.244.194.1 - 10.244.194.62
w-01        10.244.204.128/26 10.244.204.129 - 10.244.204.190
w-02        10.244.39.64/26   10.244.39.65 - 10.244.39.126
w-03        10.244.54.0/26    10.244.54.1 - 10.244.54.62
```

These are not routable on your physical network (192.168.1.0/24). Only the cluster
nodes know how to reach them — via routes that Calico installs.

### 2.2 How Calico BGP works

Calico runs a BGP daemon called **BIRD** on every node. Each BIRD instance:
1. Announces its node's pod CIDR to all other nodes ("used for this build 10.244.39.64/26, send pod
   traffic to me at 192.168.14.12")
2. Receives pod CIDRs from all other nodes and installs them as kernel routes

This is a **BGP full mesh**: every node peers with every other node. With 6 nodes, that
is 15 BGP sessions.

**The networking parallel (because this is literally your job):**
This is iBGP in a small AS. Each node is a BGP router. The pod CIDR is the route being
announced. The next hop is the node's physical IP. When Calico says "use IP-in-IP
encapsulation" (tunl0 interface), it is exactly like GRE tunnels — the pod traffic (inner
packet, 10.244.x.x → 10.244.y.y) is encapsulated inside a physical packet
(192.168.1.x → 192.168.1.y) to traverse the underlay network.

```
Pod on w-01 (10.244.204.130) → Pod on w-02 (10.244.39.70)

Outer header:  192.168.14.12 → 192.168.15.12  (physical underlay)
Inner header:  10.244.204.130 → 10.244.39.70  (pod overlay)
Protocol:      IP-in-IP (protocol 4, tunl0 interface)
```

Encapsulation is easier to follow as a sequence than as a single header diagram. The pod
packet is never modified; it travels the whole way wrapped inside a second, physical
packet, like a letter mailed inside another envelope:

1. A pod on w-01 sends a packet to a pod on w-02, a different node.
2. w-01's kernel sees that 10.244.39.70 is not local. A route that BIRD installed from
   the BGP session says: reach `10.244.39.64/26` via `192.168.15.12`.
3. `tunl0` wraps the original pod-to-pod packet inside a new IP packet addressed
   between the two nodes' real IPs. The original packet is not altered; it becomes the
   payload of the new one.
4. The outer packet crosses the physical network like any ordinary packet between two
   hosts. Nothing on the physical switches or routers needs to know pod IPs exist.
5. w-02's `tunl0` receives the outer packet, strips the outer header back off, and
   hands the untouched inner packet to the pod.

```
      w-01 (sender, wraps)                       w-02 (receiver, unwraps)
   ┌─────────────────────────────┐             ┌─────────────────────────────┐
   │ outer: 192.168.14.12        │             │ outer: 192.168.14.12        │
   │     →  192.168.15.12        │   ───────▶  │     →  192.168.15.12        │
   │  ┌───────────────────────┐  │  physical   │  ┌───────────────────────┐  │
   │  │ inner: 10.244.204.130 │  │  underlay,  │  │ inner: 10.244.204.130 │  │
   │  │     →  10.244.39.70   │  │  protocol 4 │  │     →  10.244.39.70   │  │
   │  └───────────────────────┘  │             │  └───────────────────────┘  │
   └─────────────────────────────┘             └─────────────────────────────┘
        packet built here                        packet unwrapped here,
                                                   delivered to the pod
```

### 2.3 Verifying BGP sessions

```bash
# On any node — exec into the calico-node pod
NODE=k8s-w-01
POD=$(kubectl get pods -n kube-system -l k8s-app=calico-node \
  --field-selector spec.nodeName=$NODE -o name | head -1)

kubectl exec -n kube-system $POD -- birdcl show protocols
```

Expected output (5 BGP peers for each node, all "Established"):
```
Name       Proto      Table      State  Since         Info
Node_192_168_1_11  BGP     ---        up     2026-06-01  Established
Node_192_168_1_12  BGP     ---        up     2026-06-01  Established
...
```

If any session is "Connect" or "Active", that node cannot route to that peer's pods.

### 2.4 Verifying routes are installed

```bash
# On any node — check the kernel routing table
ip route | grep tunl0
```

Expected: one route per node, pointing to that node's pod CIDR via tunl0:
```
10.244.39.64/26 via 192.168.15.12 dev tunl0 proto bird
10.244.54.0/26  via 192.168.16.12 dev tunl0 proto bird
10.244.84.0/26  via 192.168.14.11 dev tunl0 proto bird
...
```

If a route is missing, pods on this node cannot reach pods on the corresponding node.

---

## 3. What Broke — BGP Debugging

### Bug 1: Stale ARP cache blocked BGP sessions

**Symptom:** BGP session from w-01 to cp-01 stuck in "Connect" state.
`ping 192.168.14.11` from w-01 → 100% packet loss.

**Root cause:** w-01 had been rebuilt (reinstalled from scratch). Its MAC address changed.
But cp-01 still had the **old MAC address** in its ARP cache — the two share the same
`192.168.14.0/24` L2 segment (see the subnet table in "What Broke — Bug 3" below), which
is why a stale ARP entry could break their session; nodes on different segments never
have direct ARP entries for each other in the first place:

```bash
# On cp-01:
arp -n | grep 192.168.14.12
# Shows: 192.168.14.12  84:14:4d:9d:e7:4d   ← OLD MAC (pre-rebuild)
# w-01's actual MAC after rebuild: 00:0c:29:e8:f7:87
```

cp-01 was sending packets to w-01's old MAC. The physical switch dropped them (wrong
MAC, wrong port). TCP SYN for BGP session never arrived. BGP sat in Connect.

**This is pure L2 networking knowledge.** ARP cache poisoning / stale ARP entries are a
classic network troubleshooting scenario. The Kubernetes layer was fine — the problem
was below it.

**Fix:**
```bash
# Flush stale ARP entry on cp-01
sudo ip neigh del 192.168.14.12 dev ens160
# Next packet to .12 triggers a fresh ARP request → correct MAC learned
```

After the flush:
- ARP resolved the new MAC
- ICMP worked
- BGP session went to Established within seconds
- Pod routes populated on all nodes
- DNS (CoreDNS, which lives on CP nodes) became reachable from w-01 pods

**Lesson:** Kubernetes networking problems are often not Kubernetes problems. Calico's
BGP is built on top of your physical network. Any L2/L3 issue in the underlay will
appear as a "Kubernetes networking problem." Always check the physical layer first:
ping, arp, traceroute, before diving into Calico configs.

### Bug 3: MetalLB VIP unreachable from external hosts — multi-subnet L2 mismatch

**Symptom:** `https://longhorn.skilz.io` returns `HTTP 000` (connection timeout) from
any host outside the cluster. curl from within the cluster works. ping to `192.168.14.31`
always fails (expected — see note below).

**Root cause:** This cluster spans three separate L2 broadcast domains:

| Network | Nodes |
|---------|-------|
| 192.168.14.0/24 | k8s-cp-01, k8s-w-01 |
| 192.168.15.0/24 | k8s-cp-02, k8s-w-02 |
| 192.168.16.0/24 | k8s-cp-03, k8s-w-03 |

The MetalLB IP pool is `192.168.14.31-192.168.14.50` — entirely in the `14.x` subnet.

The NGINX Ingress pod landed on `k8s-w-03` (`192.168.16.12`). MetalLB with
`externalTrafficPolicy: Local` is only allowed to elect nodes that run the service pod
as the L2 speaker. So MetalLB elected `k8s-w-03` as the L2 speaker.

`k8s-w-03` only has a `192.168.16.x` interface. Its ARP reply for `192.168.14.31` was
sent on the **16.x broadcast domain**. The upstream gateway routing `14.x` traffic never
saw this ARP reply and dropped all packets to `192.168.14.31`.

An ARP reply is a broadcast on the local segment only. It never reaches a gateway
sitting on a different segment, no matter how correct the reply is:

```
BROKEN — L2 speaker elected on the wrong segment

  14.x segment (the gateway lives here)        16.x segment
  ┌─────────────────────────┐                  ┌──────────────────────────┐
  │ gateway asks:            │        ✗         │ k8s-w-03 replies:        │
  │ "who has 192.168.14.31?" │◀── never heard ─ │ "I have 192.168.14.31"   │
  │                          │   (different     │ (elected L2 speaker)     │
  │                          │    broadcast     │                          │
  │                          │    domain)       │                          │
  └─────────────────────────┘                  └──────────────────────────┘
  Gateway has no MAC to send 192.168.14.31 traffic to. Packets are dropped.

FIXED — L2 speaker restricted to the 14.x segment by nodeSelector

  14.x segment (the gateway lives here)
  ┌─────────────────────────┐
  │ gateway asks:            │        ✓
  │ "who has 192.168.14.31?" │◀── heard directly
  │                          │   (same broadcast domain)
  │ k8s-w-01 replies:        │
  │ "I have 192.168.14.31"   │
  └─────────────────────────┘
  Gateway learns the MAC. kube-proxy on k8s-w-01 then forwards the traffic on to
  wherever the ingress pod actually runs, on any of the three segments, over the
  Calico overlay.
```

**Confirming the root cause** (revert test):

```
State                                     | L2 Speaker  | Result
------------------------------------------|-------------|--------
Local + no nodeSelectors (original)       | k8s-w-03   | HTTP 000  ← broken
nodeSelectors only, keep Local            | nobody      | HTTP 000  ← broken (no eligible node)
Cluster only, no nodeSelectors            | k8s-w-03   | HTTP 200  ← works but fragile *
Both: Cluster + nodeSelectors (final)     | k8s-w-01   | HTTP 200  ← correct
```

\* `Cluster` alone happened to work because the upstream router accepted the ARP reply
from `k8s-w-03` on the `16.x` interface (proxy-ARP or cross-subnet ARP learning). This
is router-dependent and cannot be relied on.

**Fix:**

```bash
# 1. Restrict L2 speaker election to 14.x nodes
kubectl apply -f - << 'EOF'
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: skilz-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - skilz-pool
  nodeSelectors:
  - matchExpressions:
    - key: kubernetes.io/hostname
      operator: In
      values:
      - k8s-cp-01
      - k8s-w-01
EOF

# 2. Allow MetalLB to elect 14.x nodes regardless of where the ingress pod runs
kubectl patch svc ingress-nginx-controller -n ingress-nginx \
  --type=merge -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
```

After the fix, MetalLB elected `k8s-w-01` (`192.168.14.12`) as the L2 speaker. ARP for
`192.168.14.31` is now answered on the `14.x` broadcast domain. kube-proxy on `k8s-w-01`
DNAT-forwards to the ingress pod via the Calico overlay.

**Why ping to `192.168.14.31` always fails even when working:**
MetalLB LoadBalancer IPs are not real host IPs. The kernel on the L2 speaker node does
not have `192.168.14.31` configured on any interface. kube-proxy creates DNAT rules only
for TCP and UDP service ports (80 and 443). ICMP (ping) has no DNAT rule and is dropped.
This is expected — test with `curl`, not `ping`.

**Why adding a secondary NIC doesn't work cleanly:** Adding a 14.x secondary NIC to the
16.x nodes causes asymmetric routing. MetalLB's memberlist binds to the node's primary
IP (`192.168.16.12`). With a 14.x connected route now present, the kernel routes TCP
replies from `192.168.16.12` out the 14.x interface — the upstream gateway receives a
16.x source address on the 14.x segment and drops it. MetalLB's gossip protocol breaks,
speakers can't elect a leader, VIP goes unannounced. The workaround (policy routing
per source IP) adds fragile complexity that defeats the purpose.

**The correct long-term fix** is MetalLB BGP mode — peering with the upstream router
eliminates the L2 broadcast domain constraint entirely. Requires a BGP-capable router
that you control.

### Bug 2: kube-proxy iptables broken on w-02

**Symptom:** Calico CNI init container on w-02 kept failing. Error: couldn't reach the
Kubernetes API at ClusterIP `10.96.0.1:443`.

**Root cause:** `10.96.0.1` is the ClusterIP of the `kubernetes` service (the API server).
ClusterIPs are virtual — they only exist as iptables DNAT rules installed by kube-proxy.
On w-02, kube-proxy had installed rules that pointed to the wrong destination or were
corrupted after a node reboot.

```bash
# Verify ClusterIP is reachable
curl -k https://10.96.0.1:443/healthz  # should return "ok"

# If not reachable, check iptables DNAT rules
sudo iptables -t nat -L KUBE-SERVICES | grep 10.96.0.1
```

**Fix:** Delete and restart the kube-proxy pod. It re-syncs all iptables rules on startup:
```bash
kubectl delete pod -n kube-system -l k8s-app=kube-proxy --field-selector spec.nodeName=k8s-w-02
```

**What kube-proxy does:**
kube-proxy watches the API server for Service and Endpoints objects. When you create a
Service, kube-proxy installs iptables rules on every node that DNAT ClusterIP traffic
to one of the actual pod IPs. This is Kubernetes' built-in load balancing for east-west
traffic (pod → service → pod).

**The networking parallel:** ClusterIP is a virtual IP with DNAT, exactly like a NAT
pool on a firewall. kube-proxy is the control plane that pushes NAT rules to the data
plane (iptables). When kube-proxy's rules get corrupted, the "NAT pool" breaks.

---

## 4. Service Networking — ClusterIP, NodePort, LoadBalancer

Kubernetes has three types of Service, each solving a different exposure problem:

### ClusterIP (internal only)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

Creates a virtual IP (e.g. `10.96.45.100`) that is stable even as pods restart. Any pod
in the cluster can reach `netobserv-service.netobserv.svc.cluster.local:3000`. DNS
(CoreDNS) resolves this to the ClusterIP. kube-proxy DNAT forwards to the actual pod IP.

**The networking parallel:** ClusterIP is like an anycast address or a VIP behind a
load balancer that only exists within the cluster's routing domain.

### NodePort (external access via node IP)

Exposes the service on a static port (30000–32767) on every node's physical IP.
```
http://192.168.14.12:30080  → service → pod
http://192.168.15.12:30080  → same service → pod (round-robin)
```

Used for quick testing. Not used in production because you cannot use port 443 and you
have no hostname-based routing.

### LoadBalancer (external access via a dedicated IP)

Requests an external IP from the cloud provider — or in our case, MetalLB.

---

## 5. MetalLB — External IP for Bare Metal

Cloud Kubernetes (EKS, GKE, AKS) gives you a LoadBalancer IP automatically — the cloud
provider allocates an IP from its infrastructure. On bare metal, there is no cloud
provider. MetalLB fills that gap.

**Configuration:**
```yaml
# IPAddressPool — the range MetalLB can allocate from
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: skilz-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.14.31-192.168.14.50

---
# L2Advertisement — MetalLB announces IPs via ARP, restricted to 14.x nodes only
# See "What Broke" below for why nodeSelectors are required in a multi-subnet cluster
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: skilz-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - skilz-pool
  nodeSelectors:
  - matchExpressions:
    - key: kubernetes.io/hostname
      operator: In
      values:
      - k8s-cp-01   # 192.168.14.11
      - k8s-w-01   # 192.168.14.12
```

When NGINX Ingress Controller creates a Service of type LoadBalancer, MetalLB:
1. Picks an IP from the pool (`192.168.14.31`)
2. Elects one of the `14.x` nodes as the L2 leader (deterministic hash)
3. That node answers ARP for `192.168.14.31` on the `14.x` broadcast domain
4. The upstream gateway learns the MAC and forwards traffic to that node
5. kube-proxy DNAT on the receiving node forwards to NGINX (wherever the pod runs)

**The networking parallel:** MetalLB L2 mode is GARP-based VIP announcement — same
mechanism as VRRP gratuitous ARP when a master takes over. MetalLB also supports a
BGP mode (the MetalLB speaker peers with your physical router and announces the VIP
as a BGP route) — which for a network engineer is the more elegant option for
multi-subnet production deployments.

### NGINX Ingress — externalTrafficPolicy

The NGINX Ingress LoadBalancer service must use `externalTrafficPolicy: Cluster`:

```yaml
# Applied to the ingress-nginx-controller Service
spec:
  externalTrafficPolicy: Cluster
```

With `Cluster`, kube-proxy creates DNAT rules for the LoadBalancer VIP on **all** nodes.
Traffic arriving at `k8s-w-01` (the L2 speaker) is forwarded via kube-proxy to the
ingress pod regardless of which node it runs on. See "What Broke" below for why `Local`
fails in a multi-subnet cluster.

**Result:** `192.168.14.31` → NGINX Ingress Controller. DNS A records per service
(`longhorn.skilz.io`, `grafana.skilz.io`, etc.) all resolve to this single IP.

---

## 6. NGINX Ingress — L7 Routing

MetalLB gives us a single IP. But we have multiple applications (netobserv, grafana,
vault, gitlab). How does traffic for `netobserv.skilz.io` go to the NetObserv pods
and traffic for `grafana.skilz.io` go to the Grafana pods — both on port 443?

This is **L7 (HTTP/HTTPS) routing by hostname**. NGINX Ingress Controller does this.

```
Client → 192.168.14.31:443
  │
  └─ NGINX reads SNI / Host header
       ├─ netobserv.skilz.io → Service: netobserv → Pods
       ├─ grafana.skilz.io   → Service: grafana → Pods
       └─ vault.skilz.io     → Service: vault → Pods
```

An `Ingress` object defines the routing rules:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: netobserv-ingress
  namespace: netobserv
  annotations:
    cert-manager.io/issuer: "skilz-adcs-issuer"
    cert-manager.io/issuer-kind: "ClusterAdcsIssuer"
    cert-manager.io/issuer-group: "adcs.certmanager.csf.nokia.com"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"   # WebSocket long-lived connections
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - netobserv.skilz.io
      secretName: netobserv-tls     # cert-manager creates this automatically
  rules:
    - host: netobserv.skilz.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: netobserv
                port:
                  number: 3000
```

**cert-manager** watches for Ingress objects with the `cert-manager.io/cluster-issuer`
annotation and automatically provisions a TLS certificate from the specified issuer.
Our issuer is `skilz-adcs-issuer` — backed by the DC SubCA. The certificate gets
stored in the `netobserv-tls` Secret and NGINX serves it.

**The networking parallel:** NGINX Ingress is a reverse proxy with SNI-based routing,
equivalent to an F5 BIG-IP virtual server or a Cisco ACE with content switching policies.
The Ingress object is the policy. cert-manager is the PKI automation layer that keeps
certificates current without manual renewal.

---

## 7. TLS — How certificate issuance works in this lab

This lab uses **cert-manager + adcs-issuer** to automate per-service certificate
issuance from dc's Subordinate CA. Every service gets its own cert with the correct
SAN. No wildcards, no shared private keys, no manual renewal.

### Architecture

```
cert-manager (K8s)
  └── watches Ingress/Certificate objects
      └── adcs-issuer (K8s pod, cert-manager namespace)
          └── submits CSR to https://dc.skilz.io/certsrv
              └── dc skilz-Sub-CA signs the cert
                  └── cert stored as K8s Secret → NGINX serves it
```

The SubCA private key stays on dc. Kubernetes only holds leaf certificates.

### How to trigger cert issuance

Add three annotations to any Ingress object: `cert-manager.io/issuer`,
`cert-manager.io/issuer-kind`, and `cert-manager.io/issuer-group` (a single
`cert-manager.io/cluster-issuer` annotation is not enough here; see the design
rationale in the [cert-manager + Windows ADCS Workflow](./cert-manager-adcs-workflow.md)
guide, Part D, for why three are needed).

Run this from the jumpbox. The manifest can be applied directly from standard input,
with nothing saved to disk first, so any working directory is fine:

```bash
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    cert-manager.io/issuer: "skilz-adcs-issuer"
    cert-manager.io/issuer-kind: "ClusterAdcsIssuer"
    cert-manager.io/issuer-group: "adcs.certmanager.csf.nokia.com"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.skilz.io
      secretName: myapp-tls
  rules:
    - host: myapp.skilz.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
EOF
```

Replace `myapp`, `myapp-ingress`, `myapp.skilz.io`, `myapp-tls`, and `myapp-service`
with the real namespace, Ingress name, hostname, Secret name, and Service name.

To keep the manifest as a file instead of applying it straight from stdin, useful if
this Ingress will be edited again or checked into git, create a working directory and
the file first:

```bash
mkdir -p ~/manifests
cd ~/manifests

cat > myapp-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    cert-manager.io/issuer: "skilz-adcs-issuer"
    cert-manager.io/issuer-kind: "ClusterAdcsIssuer"
    cert-manager.io/issuer-group: "adcs.certmanager.csf.nokia.com"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.skilz.io
      secretName: myapp-tls
  rules:
    - host: myapp.skilz.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
EOF

kubectl apply -f myapp-ingress.yaml
```

The `cat > myapp-ingress.yaml << 'EOF' ... EOF` block writes the file directly from
the terminal; nothing needs typing into an interactive editor. To use an editor
instead, create the same file with `nano myapp-ingress.yaml` (or `vim`), paste in the
same content, save, and exit, then run the same `kubectl apply -f myapp-ingress.yaml`
command.

Either way, cert-manager sees the annotations, creates a `CertificateRequest`,
adcs-issuer signs it via dc, and the cert appears in the `myapp-tls` Secret within
about 30 seconds.

### Check cert status

```bash
# All certificates across all namespaces
kubectl get certificates -A

# Certificate requests (in-progress CSRs)
kubectl get certificaterequests -A

# Inspect a cert
kubectl get secret myapp-tls -n myapp \
    -o jsonpath='{.data.tls\.crt}' | base64 -d | \
    openssl x509 -noout -subject -issuer -dates

# Check issuer status
kubectl get clusteradcsissuer skilz-adcs-issuer
```

### Full setup guide

Installing and configuring cert-manager + adcs-issuer involves several steps including
NTP synchronisation (the most common cause of cert failures), dc HTTPS setup, RBAC
patches for Kubernetes 1.32, and end-to-end verification.

See: **[cert-manager + Windows ADCS Workflow](./cert-manager-adcs-workflow.md)**

---

## 8. DNS

All application hostnames resolve to 192.168.14.31 (the MetalLB/NGINX IP):

```
*.skilz.io → 192.168.14.31
```

This wildcard record is configured on dc (the AD DNS server). When a client
browses to `https://netobserv.skilz.io`, their DNS query goes to dc, returns
192.168.14.31, traffic hits MetalLB/NGINX, NGINX routes it to the right pod.

**Inside the cluster**, pods use CoreDNS (not dc). CoreDNS runs on the control
plane nodes. Kubernetes service discovery works via DNS:
- `netobserv` → `netobserv.netobserv.svc.cluster.local` → ClusterIP
- `vault` → `vault.vault.svc.cluster.local` → ClusterIP

---

## 9. Concepts Summary

| Concept | What it does | Networking analogy |
|---|---|---|
| Calico BIRD BGP | Distributes pod CIDRs between all nodes | iBGP full mesh |
| IP-in-IP (tunl0) | Encapsulates pod traffic for cross-node delivery | GRE tunnel / IPIP |
| kube-proxy iptables | DNAT from ClusterIP to real pod IP | NAT pool / firewall DNAT |
| MetalLB L2 | Announces service IPs via ARP on the physical network | VRRP/GARP |
| NGINX Ingress | Routes HTTPS traffic by hostname to the right service | F5 virtual server / reverse proxy |
| cert-manager | Automates TLS certificate provisioning and renewal | PKI automation / SCEP |
| CoreDNS | DNS for pod-to-service resolution inside the cluster | Internal DNS resolver |

---

*Next: [Module 03 — Persistent Storage with Longhorn](../03-storage/longhorn-persistent-storage.md)*
