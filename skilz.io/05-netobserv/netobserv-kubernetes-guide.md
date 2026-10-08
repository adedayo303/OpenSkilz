# Module 05: SkilzNetObserv Kubernetes Guide

SkilzNetObserv is a browser-based packet and traffic observer for network devices. An engineer opens
the topology map, clicks a device and an interface, and sees decoded packets in real time with the
same depth of decoding as Wireshark. This module builds it as a Kubernetes workload in the Skilz
cluster, wired to the cluster's Nautobot (device inventory and cabling), Vault (device credentials)
and Active Directory (login). The same application also exists as a standalone Docker image for
sites with no Kubernetes; that variant is documented separately in `netobserv-standalone-guide.md`
in this folder. Both guides run the same published image, `openskilz/skilznetobserv` from Docker Hub,
and the two deployments share no configuration. The [README](README.md) in this
folder explains which guide to follow.

---

## Purpose and design rationale

Troubleshooting a multi-vendor network normally means logging in to each device and starting a
capture by hand. SkilzNetObserv puts that on one page: a topology map, a click on an interface, and
decoded packets within seconds. The device configuration it needs is created when a capture starts and
removed when it stops.

Access follows the directory. Members of two Active Directory groups sign in with their domain
account, and administrators can also store device credentials. A person who leaves the directory
group loses access with no change to the application.

Each kind of information has a single home. Nautobot holds devices, interfaces, addresses and cables.
Vault holds device credentials. Active Directory holds identity and group membership. The pod volume
holds only application settings. The application keeps no second copy of any of these, so a change in
Nautobot appears on the next topology refresh and a rotated credential in Vault applies on the next
capture.

The application is one Node.js service that drives the devices over SSH, receives mirrored traffic on
a raw socket and decodes it with `tshark`. It runs as a single replica pinned to one worker, because
every device sends mirrored traffic to one fixed address. That keeps device configuration simple, and
the cost is that capture depends on one node (see Failure of the pinned worker under Operations).

---

## Architecture Overview

### Scope of this version

| Platform | Capture method | Per-interface capture | Traffic Trace | Status in this version |
|---|---|---|---|---|
| Cisco IOS and IOS-XE | ERSPAN (GRE) to the collector | Yes | Yes | Tested on lab devices |
| Cisco FTD (managed by FMC) | LINA `capture` command over SSH | Yes | Yes | Tested against an Active/Standby HA pair |
| Cisco NX-OS | ERSPAN (GRE) to the collector | Not offered yet | Yes | Trace is unverified. The device accepts the session, but delivery of mirrored traffic has not been confirmed end to end |

Cisco IOS-XE, Cisco NX-OS and Cisco FTD are the platforms in scope for this version. Devices from other
vendors are outside it. A device with no mapped vendor is skipped when the inventory loads, with a
line in the log, and no capture is offered for it. Support for another platform means adding a new
vendor helper to the source (see the customising section of the standalone guide).

### Before first use

The code has been reviewed and tested, and it follows common security practice: credentials are
stored encrypted or in Vault, access is role-based, and the device account can be limited to the
commands it needs (see Device account and command authorisation). Try it first in an isolated
environment, such as a lab, and confirm that it behaves the way you want before using it elsewhere.

### How a capture works

Mirror mode (IOS-XE, NX-OS): the application configures an ERSPAN source session on the device when a
capture or trace starts. The device encapsulates the mirrored frames in GRE and sends them to the
collector, a raw-socket listener inside the pod. The collector strips the GRE header, writes each
device's frames to a named pipe, and `tshark` decodes the pipe into JSON that streams to the browser
over a WebSocket. Stopping the capture removes the session from the device.

FTD mode: a Firepower Threat Defense is managed by FMC, and a policy deploy from FMC would overwrite
an ERSPAN session. The application therefore logs in to the LINA CLI and runs LINA's `capture`
command, which sits outside the FMC-pushed configuration. The capture is read back with `show
capture` every two seconds and removed with `no capture` when the operator clicks Stop.

### Traffic Trace

Traffic Trace configures a mirror on every eligible device at once, filters the incoming frames
against one or more rules (source IP, destination IP, protocol, port, each optionally a wildcard),
and flashes the link a matching packet crossed in that rule's colour. Links are identified from the
packet addresses against the topology, with a MAC-based fallback that resolves the previous hop. The
virtual gateway addresses used by HSRP, VRRP and FTD HA pairs are resolved through SNMP to whichever
peer is currently forwarding.

### Complete architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  Browser                                                              │
│  Topology map · Device sidebar · Capture table · Decode panel ·       │
│  Traffic Trace panel                                                  │
└──────────────────────────┬────────────────────────────────────────────┘
                           │  HTTPS + WebSocket (wss://)
                           │  TLS from the platform Sub-CA
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  netobserv, Node.js, namespace netobserv, pinned to k8s-w-03         │
│                                                                       │
│  server.js     Express + WebSocket, AD login, all routes              │
│  capture.js    Dispatch to the mirror backend or the FTD backend      │
│  collector.js  GRE raw socket, one named pipe per device              │
│  decoder.js    tshark subprocess, pcap in, packet JSON out            │
│  tracer.js     Whole-topology trace, MAC fallback, FTD capture        │
│  topology.js   On-demand SSH show commands + Nautobot cables          │
│  devices.js    Nautobot device registry + Vault credentials           │
│  mirror/       cisco-erspan.js (IOS-XE and NX-OS)                     │
└──────────┬──────────────────────────┬──────────────────┬─────────────┘
           │ SSH                      │ HTTPS            │ HTTPS
           ▼                          ▼                  ▼
   Cisco IOS-XE / NX-OS / FTD      Nautobot             Vault
                                   devices, interfaces, AppRole login,
                                   IPs, cables           secret/devices/*

           ▲
           │  GRE (ERSPAN) to 10.10.16.12, sent by the device
           │  once a capture or trace has configured the session
  Cisco IOS-XE and NX-OS devices
```

### Layout inside the image

The code sits in the image under `/app`. No source files are needed on any host.

```
/app/
├── package.json
├── server.js            Express + WebSocket, AD auth, all routes
├── capture.js           Dispatch to the mirror pipe or the FTD backend
├── decoder.js           tshark subprocess
├── collector.js         GRE raw socket listener, per-device named pipes
├── tracer.js            Whole-topology trace
├── topology.js          On-demand poll + Nautobot cable links
├── cdp.js / lldp.js     SNMP neighbour discovery used by topology.js
├── devices.js           Nautobot-backed device registry, Vault credentials
├── lib/                 audit.js, validate.js, ssh.js
├── mirror/              cisco-erspan.js
└── public/              index.html, app.js, style.css
```

---

## Prerequisites

- Modules 01 to 04 complete: the Kubernetes cluster, networking, Longhorn storage and Vault.
- A Nautobot instance reachable from the cluster, at `https://nautobot.example.com` here, holding the devices, interfaces, addresses and cables.
- `kubectl` configured on a management host with access to the cluster.
- Outbound HTTPS from `k8s-w-03` to Docker Hub, so the node can pull the image (see Step 4).
- Cluster layout used below: control planes `k8s-cp-01`, `k8s-cp-02`, `k8s-cp-03`; workers
  `k8s-w-01` (10.10.14.12), `k8s-w-02` (10.10.15.12), `k8s-w-03` (10.10.16.12);
  ingress address 10.10.14.31.
- DNS: `skilznetobserv.example.com` resolves to 10.10.14.31.

---

## Step 1: Create the Active Directory groups

Only members of two security groups can log in. `netobserv-admins` and `netobserv-users` are created
once in the `example.com` domain from a management host that reaches the domain controller over
WinRM.

```python
import winrm
s = winrm.Session('http://10.10.15.10:5985/wsman',
    auth=('<service-account>@example.com', '<password>'),
    transport='ntlm', server_cert_validation='ignore')

ps = """
foreach ($g in @('netobserv-admins', 'netobserv-users')) {
    New-ADGroup -Name $g -GroupScope Global -GroupCategory Security `
        -Description "SkilzNetObserv access - $g"
}
Add-ADGroupMember -Identity 'netobserv-admins' -Members '<service-account>'
"""
s.run_ps(ps)
```

Add further people with `Add-ADGroupMember -Identity 'netobserv-users' -Members '<username>'`.
After a successful LDAP bind, `server.js` reads the `memberOf` attribute and returns HTTP 403 to any
account that is in neither group. Members of `Domain Admins` are treated as administrators and can
store device credentials from the GUI.

---

## Step 2: Prepare Vault

Vault runs as three raft pods (`vault-0`, `vault-1`, `vault-2`) in the `vault` namespace, reached at
`https://vault.example.com`. Device SSH credentials live under `secret/devices/<device-name-lowercased>`.

### 2.1 Log in as an administrator

```bash
export VAULT_ADDR=https://vault.example.com
VAULT_TOKEN=$(curl -sk -X POST "$VAULT_ADDR/v1/auth/ldap/login/<service-account>" \
  -H "Content-Type: application/json" \
  -d '{"password":"<password>"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['auth']['client_token'])")
```

`<service-account>` must be a member of the `vault-admins` AD group.

### 2.2 Mount KV version 2 at `secret/`

```bash
curl -sk -X POST "$VAULT_ADDR/v1/sys/mounts/secret" \
  -H "X-Vault-Token: $VAULT_TOKEN" -H "Content-Type: application/json" \
  -d '{"type":"kv","options":{"version":"2"}}'
```

### 2.3 Create the `netobserv-reader` policy

The policy reads device credentials, and adds create and update on the data path so an administrator
can set or rotate a device credential from the SkilzNetObserv GUI.

```bash
curl -sk -X PUT "$VAULT_ADDR/v1/sys/policies/acl/netobserv-reader" \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -d '{"policy": "path \"secret/data/devices/*\" { capabilities = [\"read\",\"create\",\"update\"] }\npath \"secret/metadata/devices/*\" { capabilities = [\"read\",\"list\"] }"}'
```

### 2.4 Create the `netobserv` AppRole

```bash
curl -sk -X POST "$VAULT_ADDR/v1/auth/approle/role/netobserv" \
  -H "X-Vault-Token: $VAULT_TOKEN" -H "Content-Type: application/json" \
  -d '{"policies":["netobserv-reader"],"token_ttl":"1h","token_max_ttl":"4h",
       "secret_id_ttl":"0","token_num_uses":0,"secret_id_num_uses":0}'

ROLE_ID=$(curl -sk "$VAULT_ADDR/v1/auth/approle/role/netobserv/role-id" \
  -H "X-Vault-Token: $VAULT_TOKEN" | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['role_id'])")
SECRET_ID=$(curl -sk -X POST "$VAULT_ADDR/v1/auth/approle/role/netobserv/secret-id" \
  -H "X-Vault-Token: $VAULT_TOKEN" | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['secret_id'])")
```

Keep both values in shell variables and pass them straight into the Kubernetes secret in Step 5.
Writing them to a file on disk is not recommended.

### 2.5 Store device credentials

```bash
vault kv put secret/devices/<device-name-lowercased> ssh_user=<user> ssh_password='<password>'
```

A device whose credential is missing shows `Credentials: Not set` in the sidebar. An administrator
can also right-click the device node and choose Set Credentials, which writes the same path.

### 2.6 Unseal after any restart

Vault uses Shamir keys with a threshold of two and no auto-unseal, so a restarted pod starts sealed.
Unseal each restarted pod with two of the three keys:

```bash
for p in vault-0 vault-1 vault-2; do
  kubectl -n vault exec $p -- vault operator unseal <key-1>
  kubectl -n vault exec $p -- vault operator unseal <key-2>
done
kubectl -n vault exec vault-0 -- vault status | grep -E "Sealed|HA Mode"
```

The `vault-ingress` backend points at the `vault-active` service, which selects only the active pod,
so requests do not reach a sealed standby.

---

## Step 3: Prepare Nautobot

### 3.1 Service account and permissions

SkilzNetObserv reads Nautobot through a service account, `netobserv-svc`, whose API token is
read-only. Nautobot grants access per model, so one `ObjectPermission` is created for each model the
application reads. Run this from the Nautobot shell:

```bash
kubectl -n nautobot exec deploy/nautobot-web -- \
  nautobot-server -c /opt/nautobot/nautobot_config.py shell
```

```python
from django.contrib.contenttypes.models import ContentType
from nautobot.users.models import User, ObjectPermission
from nautobot.dcim.models import Cable, Device, Interface
from nautobot.ipam.models import IPAddress, IPAddressToInterface

u = User.objects.get(username='netobserv-svc')
for name, model in (('device', Device), ('interface', Interface), ('ipaddress', IPAddress),
                    ('cable', Cable), ('ipaddresstointerface', IPAddressToInterface)):
    p, _ = ObjectPermission.objects.get_or_create(
        name=f'netobserv-svc-read-{name}', defaults={'enabled': True, 'actions': ['view']})
    p.object_types.set([ContentType.objects.get_for_model(model)])
    p.users.set([u])
    p.save()
```

Topology refresh sends one GraphQL query for interfaces, addresses and cables, which needs only view
access on `dcim.interface`, `ipam.ipaddress` and `dcim.device`. When that query fails, the application
falls back to paged REST requests, which read all five models above. A missing permission shows up as a
403 from the matching endpoint. `ipam.ipaddresstointerface` is the model that links an address to an
interface on the REST path.

For a large inventory (several hundred devices and thousands of interfaces) the GraphQL query returns in
a few seconds, and the paged REST requests can time out against the default `nautobot-web` resource limits.

### 3.2 Custom fields on devices

| Custom field | Type | Values |
|---|---|---|
| `netobserv_vendor` | Text | `cisco-xe` or `cisco-nxos`. Optional when the device platform has a NAPALM driver of `iosxe`, `ios`, `nxos` or `nxos_ssh` |
| `netobserv_capture_mode` | Select | `mirror` (default), `ftd-ssh`, `none` |

```python
from django.contrib.contenttypes.models import ContentType
from nautobot.dcim.models import Device
from nautobot.extras.models import CustomField, CustomFieldChoice

ct = ContentType.objects.get_for_model(Device)
cf, _ = CustomField.objects.get_or_create(key='netobserv_capture_mode',
                                          defaults={'label': 'netobserv_capture_mode', 'type': 'select'})
cf.content_types.add(ct)
for v in ('mirror', 'ftd-ssh', 'none'):
    CustomFieldChoice.objects.get_or_create(custom_field=cf, value=v)

vf, _ = CustomField.objects.get_or_create(key='netobserv_vendor',
                                          defaults={'label': 'netobserv_vendor', 'type': 'text'})
vf.content_types.add(ct)
```

### 3.3 Device onboarding rules

A device appears in SkilzNetObserv when it has a primary IPv4 address and a vendor mapping. Devices
with neither a mapped platform nor `netobserv_vendor` are skipped with a log line, which is expected
for devices with no capture support.

| Device type | Platform | `netobserv_vendor` | `netobserv_capture_mode` |
|---|---|---|---|
| Cisco IOS-XE | any platform with NAPALM driver `iosxe` or `ios` | optional | `mirror` (default) |
| Cisco NX-OS | platform with NAPALM driver `nxos` or `nxos_ssh` | optional | `mirror` (default) |
| Cisco FTD | any | `cisco-xe` | `ftd-ssh` |

FTD devices carry `cisco-xe` as the vendor purely so they pass inventory mapping. The capture mode
`ftd-ssh` is the value that selects the FTD behaviour.

### 3.4 Cables

Nautobot cables are the authoritative source of links on the map. CDP and LLDP adjacencies and subnet
inference only fill in device pairs that have no cable record.

### 3.5 API token

Create a token for `netobserv-svc` in Nautobot and keep it for Step 5. The application sends it on
every request to `https://nautobot.example.com`.

---

## Step 4: Pull the image

The Deployment in Step 5 runs the published image `openskilz/skilznetobserv:latest` from Docker Hub.
No source code, build or private registry is involved. The pod is pinned to `k8s-w-03`, so that is the
only node that pulls it. Confirm the node can reach Docker Hub before deploying:

```bash
ssh <admin-user>@10.10.16.12 'sudo crictl pull docker.io/openskilz/skilznetobserv:latest'
```

The command ends with the image ID. A timeout means the node has no outbound access to Docker Hub.
In that case pull the image on a host that has access, then copy it to the node:

```bash
docker pull openskilz/skilznetobserv:latest
docker save openskilz/skilznetobserv:latest | ssh <admin-user>@10.10.16.12 'sudo ctr -n k8s.io images import -'
```

`latest` always follows the newest build. A version tag such as `1.0.0` pins a release, and the
`image:` line in the manifest takes whichever tag is preferred.

---

## Step 5: Deploy to Kubernetes

### 5.1 CA bundle

Node.js uses a separate trust store, so HTTPS calls to `nautobot.example.com` and `vault.example.com` need the
`skilz` Root and Sub-CA certificates added through `NODE_EXTRA_CA_CERTS`. Build a PEM bundle and store
it as a ConfigMap:

```bash
cat /usr/local/share/ca-certificates/skilz-Root-CA.crt skilz-sub-ca.pem > skilz-ca-bundle.pem
kubectl create namespace netobserv --dry-run=client -o yaml | kubectl apply -f -
kubectl -n netobserv create configmap skilz-ca-bundle --from-file=ca-bundle.pem=skilz-ca-bundle.pem
```

### 5.2 Secret

```bash
BIND_DN='CN=<service-account>,CN=Users,DC=example,DC=com'
kubectl -n netobserv create secret generic netobserv-config \
  --from-literal=VAULT_ADDR=https://vault.example.com \
  --from-literal=VAULT_ROLE_ID="$ROLE_ID" \
  --from-literal=VAULT_SECRET_ID="$SECRET_ID" \
  --from-literal=LDAP_URL=ldap://10.10.15.10 \
  --from-literal=LDAP_BIND_DN="$BIND_DN" \
  --from-literal=LDAP_BIND_PASS='<password>' \
  --from-literal=LDAP_BASE=dc=example,dc=com \
  --from-literal=LDAP_ALLOWED_GROUPS=netobserv-admins,netobserv-users \
  --from-literal=NAUTOBOT_URL=https://nautobot.example.com \
  --from-literal=NAUTOBOT_TOKEN='<nautobot-token>' \
  --from-literal=SESSION_SECRET="$(python3 -c 'import secrets; print(secrets.token_hex(32))')" \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 5.3 Apply the manifest

Save the manifest as `netobserv.yaml` and apply it:

```bash
cat > netobserv.yaml << 'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: netobserv
  labels:
    app.kubernetes.io/name: netobserv

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: netobserv-data
  namespace: netobserv
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netobserv
  namespace: netobserv
  labels:
    app: netobserv
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: netobserv
  template:
    metadata:
      labels:
        app: netobserv
    spec:
      # The GRE/ERSPAN collector opens a raw IP socket (protocol 47). That socket
      # has to sit in the node's network namespace to receive the mirrored frames.
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      # Pinned to k8s-w-03 so the ERSPAN destination is always 10.10.16.12.
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/hostname
                    operator: In
                    values:
                      - k8s-w-03
      serviceAccountName: default
      securityContext:
        fsGroup: 0
      containers:
        - name: netobserv
          image: openskilz/skilznetobserv:latest
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          securityContext:
            # Root, so the file capabilities on tshark reach the effective set when
            # Node.js starts it for an ERSPAN capture.
            runAsUser: 0
            allowPrivilegeEscalation: true
            capabilities:
              add:
                - NET_RAW
                - NET_ADMIN
              drop:
                - ALL
          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "3000"
            - name: NODE_EXTRA_CA_CERTS
              value: "/etc/ssl/skilz/ca-bundle.pem"
            - name: VAULT_ADDR
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: VAULT_ADDR
            - name: VAULT_ROLE_ID
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: VAULT_ROLE_ID
            - name: VAULT_SECRET_ID
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: VAULT_SECRET_ID
            - name: LDAP_URL
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: LDAP_URL
            - name: LDAP_BIND_DN
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: LDAP_BIND_DN
            - name: LDAP_BIND_PASS
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: LDAP_BIND_PASS
            - name: LDAP_BASE
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: LDAP_BASE
            - name: LDAP_ALLOWED_GROUPS
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: LDAP_ALLOWED_GROUPS
            - name: NAUTOBOT_URL
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: NAUTOBOT_URL
            - name: NAUTOBOT_TOKEN
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: NAUTOBOT_TOKEN
            - name: SESSION_SECRET
              valueFrom:
                secretKeyRef:
                  name: netobserv-config
                  key: SESSION_SECRET
            - name: ERSPAN_COLLECTOR_IP
              value: "10.10.16.12"
            # The ingress terminates TLS and the pod receives plain HTTP from it.
            # With TRUST_PROXY on, Express reads X-Forwarded-Proto and treats the
            # request as secure, which lets express-session set its secure cookie.
            - name: TRUST_PROXY
              value: "true"
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          volumeMounts:
            - name: pipes
              mountPath: /tmp/netobserv-pipes
            - name: ca-bundle
              mountPath: /etc/ssl/skilz
              readOnly: true
            - name: data
              mountPath: /app/data
          livenessProbe:
            httpGet:
              path: /metrics
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 30
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /metrics
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
      volumes:
        - name: pipes
          emptyDir: {}
        - name: ca-bundle
          configMap:
            name: skilz-ca-bundle
        - name: data
          persistentVolumeClaim:
            claimName: netobserv-data
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: netobserv
  namespace: netobserv
  labels:
    app: netobserv
spec:
  type: ClusterIP
  selector:
    app: netobserv
  ports:
    - name: http
      port: 3000
      targetPort: 3000
      protocol: TCP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: netobserv
  namespace: netobserv
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    cert-manager.io/issuer: "skilz-adcs-issuer"
    cert-manager.io/issuer-kind: "ClusterAdcsIssuer"
    cert-manager.io/issuer-group: "adcs.certmanager.csf.nokia.com"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - skilznetobserv.example.com
      secretName: skilznetobserv-tls
  rules:
    - host: skilznetobserv.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: netobserv
                port:
                  number: 3000
EOF

kubectl apply -f netobserv.yaml
```

The manifest creates the namespace, a 256Mi Longhorn volume `netobserv-data` mounted at `/app/data`
for local users and GUI-saved settings, a single-replica Deployment, a ClusterIP Service on port
3000, and an Ingress for `skilznetobserv.example.com` with a cert-manager certificate from the Sub-CA.

| Setting | Value | Reason |
|---|---|---|
| `hostNetwork: true` | on | The collector opens a raw GRE socket and must see traffic on the node's real interface |
| `strategy` | `Recreate` | Two host-network pods cannot hold port 3000 on the same node |
| `nodeAffinity` | `k8s-w-03` | The ERSPAN destination on every device is 10.10.16.12, the address of that node |
| `securityContext` | root, `NET_RAW` and `NET_ADMIN` added, all others dropped | tshark file capabilities need a root process to reach the effective set |
| `imagePullPolicy` | `Always` | A rollout restart pulls the newest image |
| `ERSPAN_COLLECTOR_IP` | `10.10.16.12` | Address devices send mirrored traffic to |
| `TRUST_PROXY` | `true` | The ingress terminates TLS, so Express has to trust `X-Forwarded-Proto` |

The `configuration-snippet` ingress annotation is disabled on this cluster by the admission webhook.
WebSocket upgrades work without it once `proxy-read-timeout` is set to 3600.

### 5.4 Verify the rollout

```bash
kubectl -n netobserv get pods -o wide        # one pod Running on k8s-w-03
kubectl -n netobserv get ingress             # ADDRESS 10.10.14.31
kubectl -n netobserv get certificate         # netobserv-tls Ready True
kubectl -n netobserv logs deploy/netobserv | grep -E "Loaded|AUDIT"
```

The log reports `Loaded N devices from Nautobot` once the registry has synced.

---

## Step 6: Configure the devices

### 6.1 Cisco IOS-XE

The application builds the session itself when a capture starts and removes it when the capture
stops. The manual equivalent is:

```
monitor session 1 type erspan-source
 source interface GigabitEthernet1/0/1 both
 destination
  erspan-id 1
  ip address 10.10.16.12
  origin ip address <device-management-ip>
 no shutdown
```

A VLAN interface is mirrored with `source vlan <id> both`, because IOS-XE rejects an SVI under the
interface keyword. The device reaches 10.10.16.12 through the routed path, and `origin ip address`
has to equal the management IP held in Nautobot, since the collector identifies a device by the
source address of its GRE packets.

### 6.2 Cisco NX-OS

NX-OS uses a flat syntax and a global origin address:

```
monitor erspan origin ip-address <device-management-ip> global
monitor session 2 type erspan-source
 source interface Ethernet1/1 both
 destination ip 10.10.16.12
 erspan-id 2
 vrf default
 no shut
```

After configuring a trace session, the application reads `show monitor session 2`. If the state is
`error`, it removes the session and reports that the platform does not mirror traffic.

### 6.3 Cisco FTD

No configuration is needed on the device beyond SSH reachability and an account with access to the LINA
CLI. The capture is named `NETOBSERV_<nameif>` and removed on stop. After an unexpected pod restart, a
leftover entry is cleared with `no capture NETOBSERV_<nameif>`.

### 6.4 SSH reachability

Topology polling and session set-up connect over SSH from 10.10.16.12. If a device restricts its
VTY lines with an `access-class`, 10.10.16.0/24 has to be permitted. Populate `known_hosts.json`
with device fingerprints (`ssh-keyscan -t rsa,ecdsa,ed25519 <ip> | ssh-keygen -l -f -`) and set
`NETOBSERV_STRICT_HOST_KEYS=true` for production.

### 6.5 Device account and command authorisation

SkilzNetObserv logs in to each device it captures from. Create a dedicated account for the
application and limit it to the commands listed in this section. They cover the ERSPAN session on
IOS-XE and NX-OS, the LINA capture on FTD, and one `show` command on NX-OS. The limit is applied by
the device administration service, for example a TACACS+ server, a RADIUS server such as Cisco ISE,
or local privilege levels where no central service exists. The application holds the credential
(Step 2) and the administration service decides what that credential may do.

| Aspect | Approach |
|---|---|
| Purpose | Allow the application to start and stop a capture and nothing else on the device |
| Account | One dedicated service account for SkilzNetObserv, separate from engineer accounts |
| Control | Command authorisation on the AAA server, with every command denied unless it is listed below |
| Credential | Held in the application's credential store or in Vault, one place for all devices |
| Accountability | AAA accounting records each command. The session description `SkilzNetObserv-Auto` or `SkilzNetObserv-Trace` and the `NETOBSERV_` capture names identify the application's changes |
| Lifetime | A mirror session or capture exists only while a capture or trace is running |

Two device settings apply to every platform in this table. Enable command authorisation for the
exec level and for configuration mode (on IOS-XE, `aaa authorization commands 15` and `aaa
authorization config-commands`), because a device that does not send configuration commands to the
AAA server cannot enforce the list. Keep a local break-glass account for the administrators.

**Cisco IOS and IOS-XE**

| Command | Sent when |
|---|---|
| `terminal length 0` | Start of every SSH session |
| `configure terminal` | Capture start and stop, trace start and stop |
| `no monitor session <1 or 2>` | Before a new session is built, and on stop. Session 1 is a capture and session 2 is a trace (defaults) |
| `monitor session <1 or 2> type erspan-source` | Capture start, trace start |
| `description SkilzNetObserv-Auto` or `description SkilzNetObserv-Trace` | Capture start, trace start |
| `source interface <interface> both` or `source vlan <id> both` | Capture start, trace start |
| `no shutdown` | Capture start, trace start |
| `destination` | Capture start, trace start |
| `erspan-id <1 or 2>` | Capture start, trace start |
| `ip address <collector IP>` | Capture start, trace start |
| `origin ip address <device management IP>` | Capture start, trace start |
| `exit` and `end` | End of each block of configuration |
| `write memory` | Capture start and stop only. A trace does not send it. Leave it out of the list if the application should not save the configuration |

**Cisco NX-OS**

| Command | Sent when |
|---|---|
| `terminal length 0` | Start of every SSH session |
| `configure terminal` | Trace start and stop |
| `monitor erspan origin ip-address <device management IP> global` | Trace start |
| `no monitor session 2` | Trace start and stop |
| `monitor session 2 type erspan-source` | Trace start |
| `source interface <interface> both` or `source vlan <id> both` | Trace start |
| `destination ip <collector IP>` | Trace start |
| `erspan-id 2` | Trace start |
| `vrf default` | Trace start |
| `no shut` | Trace start |
| `exit` and `end` | End of each block of configuration |
| `show monitor session 2` | Trace start, to confirm the session is not in an error state |

**Cisco FTD (LINA CLI)**

| Command | Sent when |
|---|---|
| `show nameif` | Capture or trace start, to find the interface name |
| `show ip address` | Trace start, to map interface addresses |
| `capture NETOBSERV_<nameif> interface <nameif> match ip any any` | Capture or trace start |
| `no capture NETOBSERV_<nameif>` | Before a capture starts, and on stop |
| `show capture NETOBSERV_<nameif>` | Every 2 seconds while a capture or trace runs |
| `clear capture NETOBSERV_<nameif>` | After each read, to keep the device buffer empty |

LINA command limits depend on the FTD version and on how the account is authenticated. Where the
platform allows external authentication for CLI access, the same AAA server can manage the account.
Where it does not, give the account the lowest role that still permits the commands above.

**Commands the application does not send**

The application does not read the running configuration, does not change interface settings, and
does not use `copy`, `reload`, `debug` or any other configuration command. Device inventory,
interfaces and cables come from Nautobot. Live gateway state (see the Architecture Overview) is read with SNMPv3 and
uses a separate read-only SNMP user, which is also kept out of the SSH account.

**Checking the policy**

Before relying on a restricted account, run one capture and one trace on each platform in a lab and
confirm that every command in the tables is accepted.

---

## Step 7: Verify

1. Open `https://skilznetobserv.example.com` and log in as `<service-account>`. A user outside both groups
   receives a 403 and an `event="login_failure"` audit line.
2. Click Refresh topology. Devices appear as nodes and Nautobot cables appear as links. An FTD node
   appears with its interfaces.
3. Open the Traffic Trace panel. The device picker lists every IOS-XE, NX-OS and FTD device with a
   credential in Vault.
4. Click a device, then Capture on an interface. Decoded packets stream into the table.
5. Check the audit trail with `kubectl -n netobserv logs deploy/netobserv | grep AUDIT`.

Rate limiting allows 10 failed logins per 15 minutes per client address.

---

## Environment Variables

All values come from the `netobserv-config` secret unless the Source column says otherwise.

| Variable | Example | Source |
|---|---|---|
| `VAULT_ADDR` | `https://vault.example.com` | secret |
| `VAULT_ROLE_ID`, `VAULT_SECRET_ID` | AppRole identifiers | secret |
| `LDAP_URL` | `ldap://10.10.15.10` | secret |
| `LDAP_BIND_DN`, `LDAP_BIND_PASS` | service bind account | secret |
| `LDAP_BASE` | `dc=example,dc=com` | secret |
| `LDAP_ALLOWED_GROUPS` | `netobserv-admins,netobserv-users` | secret |
| `NAUTOBOT_URL`, `NAUTOBOT_TOKEN` | `https://nautobot.example.com`, read-only token | secret |
| `SESSION_SECRET` | 32-byte hex | secret |
| `NODE_EXTRA_CA_CERTS` | `/etc/ssl/skilz/ca-bundle.pem` | manifest |
| `ERSPAN_COLLECTOR_IP` | `10.10.16.12` | manifest |
| `TRUST_PROXY` | `true` | manifest |
| `NODE_ENV`, `PORT` | `production`, `3000` | manifest |
| `NETOBSERV_STRICT_HOST_KEYS` | `true` | optional |

---

## Operations

Pod status and logs:

```bash
kubectl -n netobserv get pods -o wide
kubectl -n netobserv logs -f deploy/netobserv
```

Update to the newest published image (the Deployment pulls on every start because `imagePullPolicy` is `Always`):

```bash
kubectl -n netobserv rollout restart deployment/netobserv
kubectl -n netobserv rollout status deployment/netobserv --timeout=120s
```

Rotate a device credential: set it from the GUI (right-click the node, Set Credentials) or with
`vault kv put secret/devices/<name> ssh_user=... ssh_password=...`. The next capture uses it.

Rotate the session secret: replace `SESSION_SECRET` in the secret and restart the deployment. Every
browser session is logged out.

Check that GRE is arriving, on `k8s-w-03`: `tcpdump -i any -n proto gre -c 10`.

Remove a stuck mirror session on an IOS-XE device after an interrupted capture:

```
conf t
no monitor session 1
end
write mem
```

Prometheus metrics at `/metrics`: `netobserv_ws_connections_active`, `netobserv_capture_packets_total`,
`netobserv_capture_errors_total`, `netobserv_login_attempts_total`.

### Failure of the pinned worker (k8s-w-03)

The pod is pinned to `k8s-w-03` because every device sends its ERSPAN stream to 10.10.16.12. If the
node is down the pod stays Pending and capture is unavailable. Nothing else is affected, since mirroring
is passive and forwarding on the devices continues. The pod reschedules by itself when the node returns,
and sessions are rebuilt on the next capture. An FTD capture or a device-local capture
(`monitor capture` on IOS-XE) remains available in the meantime.

---

## Troubleshooting

### Pod in ImagePullBackOff

Run `kubectl -n netobserv describe pod -l app=netobserv | grep -A5 Events`. The usual causes are no
outbound HTTPS from `k8s-w-03` to Docker Hub, a Docker Hub pull rate limit, or a mistyped image name.
`sudo crictl pull docker.io/openskilz/skilznetobserv:latest` on the node reproduces the error. Step 4
describes how to import the image by hand when the node has no internet access.

### Login returns 401

Check the bind account in the secret and test the bind from the management host:

```bash
ldapsearch -H ldap://10.10.15.10 -D "CN=<service-account>,CN=Users,DC=example,DC=com" -w '<password>' \
  -b "dc=example,dc=com" "(sAMAccountName=<user>)" memberOf
```

A stale `LDAP_BIND_PASS` after a password reset produces the same symptom.

### Login returns 200 but every later request returns 401

The ingress terminates TLS, so Express needs `TRUST_PROXY=true` to treat the request as secure. Without
it `express-session` does not send the `Set-Cookie` header for a `secure` cookie. Confirm the variable is
in the deployment.

### Login returns 403

The account authenticated but is in neither AD group. Add it with `Add-ADGroupMember`.

### Device list is empty or shorter than expected

Check the log for `no vendor mapping` lines. The device needs a platform with a NAPALM driver, or the
`netobserv_vendor` custom field, and a primary IPv4 address. An FTD also needs `netobserv_capture_mode`
set to `ftd-ssh`.

### Nautobot returns 403 on a topology refresh

The service account is missing a view permission on one of the models in Step 3.1. The response names
the endpoint; create the matching `ObjectPermission`. When the log shows `GraphQL fetch failed, using
REST`, check that `NAUTOBOT_URL` serves `/api/graphql/` and that the token can read it.

### Custom field rejects `ftd-ssh`

The select field has no such choice. Add `ftd-ssh` and `none` as `CustomFieldChoice` values (Step 3.2).

### Vault reports sealed or a fetch fails

Run `vault status` on each pod and unseal as in Step 2.6. If a pod crashes with
`failed to open bolt file ... input/output error`, delete that pod and let it restart, then unseal it.
Test the AppRole directly:

```bash
curl -sk -X POST https://vault.example.com/v1/auth/approle/login \
  -d '{"role_id":"<role-id>","secret_id":"<secret-id>"}' | python3 -m json.tool
```

### TLS error: unable to get local issuer certificate

The CA ConfigMap is missing or `NODE_EXTRA_CA_CERTS` is not set. `VAULT_CACERT` applies to the Vault CLI
only, and `NODE_EXTRA_CA_CERTS` applies to Node.js.

### Collector receives no frames

`raw-socket` has no `Protocol.GRE` constant, so the collector opens a raw socket with protocol number 47.
Confirm GRE reaches the node with `tcpdump -i any proto gre`, that the pod is on `k8s-w-03` with
`hostNetwork`, and that the device `origin ip address` matches the management IP in Nautobot.

### Pod stuck Pending during a rollout

Two host-network pods cannot share port 3000. The deployment uses `Recreate` for this reason, so the old
pod terminates before the new one starts.

### Capture on a VLAN interface shows no traffic

IOS-XE rejects an SVI under `source interface`. The application emits `source vlan <id> both` for
`Vlan<id>` names.

### Settings lost after a restart

`/app/data` has to be backed by the `netobserv-data` volume. Check `kubectl -n netobserv get pvc`.

### Wrong interface traffic on an IOS-XE capture

The mirror session is rebuilt for the requested interface on every capture start. Confirm with
`show monitor session 1 detail`.

### FTD interface refuses to capture

The interface has no nameif. Run `show nameif` on the FTD. A parent interface carrying only
sub-interfaces has none, so capture on a sub-interface.

---

## Known Limitations

- NX-OS ERSPAN trace is unverified. A virtual Nexus 9000v (NX-OS 9.2(4)) accepts the session
  configuration and then reports `state: error (no such pss key)` for both switchport and routed sources.
  The application detects this and reports it. Physical Nexus hardware has not been tested.
- FTD rows carry addresses, protocol and LINA's description only. The hex pane is empty and PCAP export
  does not apply.
- One capture per WebSocket connection. A second capture stops the first. Use another browser tab for
  simultaneous captures.
- Topology refreshes on demand only, with no background polling.
- The collector identifies a device by the source address of its GRE packets.
- The ERSPAN session ID defaults to 1 and is set globally with `ERSPAN_SESSION_ID`.
- The image is pulled from Docker Hub. A private registry hosted on the platform is planned for a future module.
