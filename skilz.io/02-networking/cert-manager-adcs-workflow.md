# Module 02 — TLS Certificate Automation: cert-manager + Windows ADCS

> **What you will have at the end of this guide:**
> - certsrv running over HTTPS on dc with a valid TLS cert from the SubCA
> - cert-manager installed in the cluster
> - adcs-issuer installed and configured, bridging cert-manager to Windows ADCS
> - A `ClusterAdcsIssuer` that issues certs on demand from dc's `skilz-Sub-CA`
> - Every Ingress resource able to get a signed cert automatically by adding a single annotation

---

## Design rationale

The straightforward cert-manager setup stores an intermediate CA private key inside a Kubernetes Secret. That means anyone with access to the cluster, or to a backup of it, has access to a key that can sign arbitrary certificates trusted by every node.

What that key can do, and what this lab holds instead, is easier to see side by side than to describe in a sentence:

```
THE RISKY VERSION (not used here)

  Kubernetes Secret
  ┌─────────────────────────────────┐
  │ Intermediate CA private key      │  ← can mint a trusted certificate
  │                                   │     for ANY hostname, not just one
  └─────────────────────────────────┘
  Anyone who can read this Secret, or restore a backup of it, holds that key.

THIS LAB'S VERSION

  rootca-k8s (offline Root CA, powered off after Day 1)
        │  signed the Sub CA once, years ago
        ▼
  dc — skilz-Sub-CA private key
        │  never leaves dc; signs each CSR on request, one at a time
        ▼
  Kubernetes Secret
  ┌─────────────────────────────────┐
  │ One leaf certificate             │  ← valid only for the single
  │                                   │     hostname it was issued for
  └─────────────────────────────────┘
  No CA private key ever exists inside the cluster. Reading every Secret in the
  cluster, or restoring every backup of it, still yields nothing that could mint a
  certificate for anything other than what it already is.
```

This lab routes certificate requests through Windows ADCS instead. cert-manager creates the CSR and asks adcs-issuer to sign it. adcs-issuer submits the CSR to dc over HTTPS. dc signs it using the `skilz-Sub-CA` private key, which never leaves dc. The signed cert comes back to cert-manager and gets stored as a normal Kubernetes Secret.

```
cert-manager (K8s)
  └── creates CertificateRequest (CSR)
      └── adcs-issuer (K8s pod in cert-manager namespace)
          └── POST to https://dc.skilz.io/certsrv/certfnsh.asp
              └── dc SubCA signs CSR, returns cert
                  └── cert stored in K8s Secret
                      └── NGINX serves it via TLS
```

The PKI private key stays on dc. Kubernetes only holds leaf certs.

---

## Critical prerequisite: NTP time synchronisation

> **This is the single most common cause of certificate failures in this setup. Fix
> this first. Skipping it will result in hours of debugging.**

Certificate issuance involves at least three systems — the K8s nodes, dc, and the cert itself — and all three need to agree on the time. Here is what breaks when they do not:

| Misalignment | Symptom |
|---|---|
| K8s node clock wrong | cert-manager webhook tokens expire immediately; `failed to call webhook` errors |
| dc clock ahead of K8s | Issued cert's `NotBefore` is in the future from K8s nodes' perspective; TLS handshakes fail with "certificate not yet valid" |
| dc clock behind K8s | Cert appears expired; same TLS failure |
| Any skew > 5 minutes | cert-manager internal mTLS between its own pods breaks |

### Verify dc time

On dc (run as Administrator in PowerShell or Command Prompt):

```powershell
w32tm /query /status
```

Check:
- **Source** — should reference a real NTP server (e.g. `time.windows.com`) not `Local CMOS Clock`
- **Last Successful Sync Time** — should be recent (within the last hour)
- **Stratum** — value above 1 is fine for a domain controller

If dc is not syncing from an external source, configure it:

```powershell
# Point dc at Windows time server
w32tm /config /manualpeerlist:"time.windows.com,0x1" /syncfromflags:manual /reliable:yes /update
Restart-Service w32tm
w32tm /resync /force
w32tm /query /status
```

### Verify K8s node time

Run this on any Linux node (including the jumpbox):

```bash
timedatectl status
```

Expected output:
```
               Local time: Mon 2026-06-09 10:54:22 UTC
           Universal time: Mon 2026-06-09 10:54:22 UTC
                 RTC time: Mon 2026-06-09 10:54:22
                Time zone: UTC (UTC, +0000)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

`System clock synchronized: yes` is the key line. If it says `no`:

```bash
# Check current NTP config
cat /etc/systemd/timesyncd.conf
```

It should contain:

```ini
[Time]
NTP=192.168.15.10
FallbackNTP=pool.ntp.org
RootDistanceMaxSec=30
```

**`RootDistanceMaxSec=30` is mandatory.** Windows domain controllers are not stratum-1
servers. Their root distance (a measure of how many hops removed from a primary reference
clock) is typically 20–30 seconds. Ubuntu's `systemd-timesyncd` rejects any NTP source
with root distance above 5 seconds by default. Without this setting, every K8s node
silently refuses dc and falls back to `pool.ntp.org`, which may be hours different from
dc's time — and cert issuance will fail in ways that look completely unrelated to NTP.

If the file is missing or wrong, fix it on **all six K8s nodes**:

```bash
sudo tee /etc/systemd/timesyncd.conf > /dev/null << 'EOF'
[Time]
NTP=192.168.15.10
FallbackNTP=pool.ntp.org
RootDistanceMaxSec=30
EOF

sudo systemctl restart systemd-timesyncd
timedatectl timesync-status
```

Check the sync status detail:

```bash
timedatectl timesync-status
# Look for: Server: 192.168.15.10
#           Stratum: <number>
#           Offset: <small number, ideally < 1s>
```

Do this on all nodes before proceeding. Once all clocks agree, continue.

---

## Part A — dc prerequisites

These steps run on **dc (192.168.15.10)** and are done before anything touches Kubernetes.

### A.1 Create an automation service account

adcs-issuer authenticates to certsrv using a domain account. Create a dedicated account
rather than using Administrator.

On dc (PowerShell as `SKILZ\Administrator` or any Domain Admin):

```powershell
# Create the automation user
New-ADUser `
    -Name "<service-account>" `
    -SamAccountName "<service-account>" `
    -UserPrincipalName "<service-account>@skilz.io" `
    -AccountPassword (Read-Host -AsSecureString "Enter password for <service-account>") `
    -Enabled $true `
    -PasswordNeverExpires $true `
    -CannotChangePassword $true `
    -Description "Automation service account — K8s cert issuance via adcs-issuer"

# Add to Domain Admins (gives certsrv enrollment rights)
Add-ADGroupMember -Identity "Domain Admins" -Members "<service-account>"
```

Verify:

```powershell
Get-ADUser <service-account> | Select-Object SamAccountName, Enabled, DistinguishedName
Get-ADGroupMember "Domain Admins" | Where-Object { $_.SamAccountName -eq "<service-account>" }
```

> **Production note:** Domain Admins is used here for the lab. In production, create a
> dedicated group, remove from Domain Admins, and grant only the "Request Certificates"
> right on the specific ADCS template. This follows least-privilege.

### A.2 Issue a TLS cert for IIS on dc

certsrv runs on IIS. By default IIS serves it over HTTP, meaning credentials submitted
by adcs-issuer travel in cleartext. Bind HTTPS to IIS using a cert from the SubCA.

This creates a bootstrap problem: you want to use certsrv to issue certs, but you need a
cert to secure certsrv first. The solution is to request the IIS cert directly from the
local CertSvc RPC service using `certreq`, which does not go through IIS or HTTP at all.

On dc (PowerShell as Administrator):

```powershell
# Create working directory
New-Item -ItemType Directory -Path C:\Temp -Force | Out-Null

# Create the certificate request configuration
# Replace dc.skilz.io with your DC's actual FQDN
$inf = @"
[Version]
Signature="`$Windows NT`$"
[NewRequest]
Subject = "CN=dc.skilz.io"
KeyLength = 2048
KeyAlgorithm = RSA
MachineKeySet = TRUE
RequestType = PKCS10
KeyUsage = 0xa0
HashAlgorithm = SHA256
Exportable = FALSE
[Extensions]
2.5.29.17 = "{text}"
_continue_ = "dns=dc.skilz.io&"
_continue_ = "dns=dc&"
_continue_ = "ipaddress=192.168.15.10&"
[RequestAttributes]
CertificateTemplate = WebServer
"@
Set-Content 'C:\Temp\iis-cert.inf' $inf -Force

# Generate the CSR (creates C:\Temp\iis-cert.csr and a pending key in the machine store)
certreq.exe -new 'C:\Temp\iis-cert.inf' 'C:\Temp\iis-cert.csr'
```

Submit the CSR to the SubCA. Use `-config localhost\<CA_NAME>` — this contacts the
CertSvc RPC service on the local machine without requiring network authentication:

```powershell
# Get the active CA name from the registry
$caName = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration").Active
Write-Host "CA name: $caName"

# Submit the CSR. The signed cert is saved to C:\Temp\iis-cert.cer
certreq.exe -submit `
    -attrib "CertificateTemplate:WebServer" `
    -config "localhost\$caName" `
    'C:\Temp\iis-cert.csr' `
    'C:\Temp\iis-cert.cer'
```

Accept the issued certificate into the machine certificate store:

```powershell
certreq.exe -accept 'C:\Temp\iis-cert.cer'
```

Verify the cert is in the machine store:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
    Where-Object { $_.Subject -like "*dc*" -and $_.HasPrivateKey } |
    Select-Object Subject, Thumbprint, NotBefore, NotAfter |
    Format-Table -AutoSize
```

Note the thumbprint — you need it for the IIS binding step.

> **If `certreq -submit` hangs indefinitely:** This is the WinRM NTLM double-hop
> problem. It only occurs when running commands over WinRM from a remote machine.
> If you are directly on the DC console or in an RDP session, certreq will not hang.
> If you must run it remotely, use a scheduled task with `RunLevel Highest` and
> `Principal: SYSTEM` — SYSTEM can reach local CertSvc without network auth delegation.

### A.3 Bind the HTTPS cert to IIS

On dc (PowerShell as Administrator). Replace `<THUMBPRINT>` with the value from A.2:

```powershell
Import-Module WebAdministration

$thumbprint = "<THUMBPRINT>"

# Remove any existing HTTPS binding on port 443
Get-WebBinding -Name "Default Web Site" -Protocol https -ErrorAction SilentlyContinue |
    Remove-WebBinding

# Add the HTTPS binding
New-WebBinding -Name "Default Web Site" `
    -Protocol https `
    -Port 443 `
    -HostHeader "" `
    -SslFlags 0

# Bind the certificate to the HTTPS listener
$binding = Get-WebBinding -Name "Default Web Site" -Protocol https
$binding.AddSslCertificate($thumbprint, "My")

# Apply changes
iisreset /restart
```

Verify from a machine that trusts the Root CA:

```bash
# From the jumpbox (Root CA must be in /etc/ssl/certs for this to pass)
curl -sv https://dc.skilz.io/certsrv/ 2>&1 | grep -E "SSL|subject|issuer|HTTP"
# Expected: HTTP/1.1 401 Unauthorized
# (401 is correct — certsrv requires authentication, which means HTTPS is working)
```

An HTTP 401 response over HTTPS is the correct result. It means:
- IIS is listening on port 443
- The TLS cert is valid and trusted
- certsrv is asking for credentials (as it should)

---

## Part B — Kubernetes components

These steps run from the **jumpbox** (`kubectl` configured, cluster running and healthy).

### B.1 Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml
```

Wait for all three pods to be Ready:

```bash
kubectl wait --for=condition=Ready pod \
    -l app.kubernetes.io/instance=cert-manager \
    -n cert-manager \
    --timeout=120s

kubectl get pods -n cert-manager
```

Expected:
```
NAME                                       READY   STATUS    RESTARTS
cert-manager-xxxxxxxxxx-xxxxx             1/1     Running   0
cert-manager-cainjector-xxxxxxxxxx-xxxxx  1/1     Running   0
cert-manager-webhook-xxxxxxxxxx-xxxxx     1/1     Running   0
```

> **If cert-manager pods crash-loop immediately after a NTP clock change,** the
> existing webhook TLS tokens are invalidated. Reinstall cert-manager completely:
> ```bash
> kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml
> # Wait 60 seconds for all resources to clear
> kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml
> ```

### B.2 Install adcs-issuer

adcs-issuer is the cert-manager external issuer plugin that submits CSRs to Windows ADCS:

```bash
kubectl apply -f https://raw.githubusercontent.com/djkormo/adcs-issuer/master/config/deploy/recommended.yaml

kubectl wait --for=condition=Ready pod \
    -l app=adcs-issuer \
    -n cert-manager \
    --timeout=60s

kubectl get pods -n cert-manager -l app=adcs-issuer
```

### B.3 Fix leader election RBAC (Kubernetes 1.32+)

Kubernetes 1.32 moved leader election from ConfigMaps to Leases. adcs-issuer's default
RBAC only grants ConfigMaps access. Without this patch, the adcs-issuer pod crashes on
startup with an `Unauthorized` error on the `coordination.k8s.io/leases` resource.

```bash
kubectl patch role adcs-issuer-leader-election-role \
    -n cert-manager \
    --type=json \
    -p='[{
      "op": "add",
      "path": "/rules/-",
      "value": {
        "apiGroups": ["coordination.k8s.io"],
        "resources": ["leases"],
        "verbs": ["get","list","watch","create","update","patch","delete"]
      }
    }]'
```

Verify:

```bash
kubectl get role adcs-issuer-leader-election-role -n cert-manager \
    -o jsonpath='{.rules[*].resources}' | tr ' ' '\n' | grep leases
# Expected: leases
```

### B.4 Fix approval RBAC

cert-manager uses SubjectAccessReview (SAR) to check whether a requester is authorised
to get a cert from a given issuer. It checks multiple resource name formats depending on
the issuer kind and API group version. The approval ClusterRole must list all the formats
cert-manager might use, or `CertificateRequest` objects will stay in `Pending` indefinitely
with no error message explaining why.

```bash
kubectl patch clusterrole \
    adcs-issuer-cert-manager-controller-approve:adcs-certmanager-csf-nokia-com \
    --type=json \
    -p='[{
      "op": "replace",
      "path": "/rules/0/resourceNames",
      "value": [
        "clusteradcsissuers.adcs.certmanager.csf.nokia.com/*",
        "clusteradcsissuers.adcs.certmanager.csf.nokia.com/skilz-adcs-issuer",
        "adcs.certmanager.csf.nokia.com/ClusterAdcsIssuer/*",
        "adcs.certmanager.csf.nokia.com/ClusterAdcsIssuer/skilz-adcs-issuer",
        "adcsissuers.adcs.certmanager.csf.nokia.com/*"
      ]
    }]'
```

Verify:

```bash
kubectl get clusterrole \
    adcs-issuer-cert-manager-controller-approve:adcs-certmanager-csf-nokia-com \
    -o jsonpath='{.rules[0].resourceNames}' | tr ',' '\n'
# Should list all five entries above
```

### B.5 Create the issuer credentials Secret

adcs-issuer needs a domain account to authenticate to certsrv. Create the Secret directly
— do not write credentials to a file on disk:

```bash
kubectl create secret generic adcs-issuer-credentials \
    --namespace cert-manager \
    --from-literal=username='SKILZ\<service-account>' \
    --from-literal=password='<<service-account>-password>'
```

> Vault (Module 04) will replace this manual secret creation. Until then, avoid writing
> credentials to any file. The `--from-literal` flag passes the value directly from the
> shell without touching disk.

### B.6 Get the Root CA certificate for the issuer

adcs-issuer uses the Root CA cert to verify dc's TLS certificate when connecting to
certsrv. Get it from dc and encode it as base64.

On dc (PowerShell):

```powershell
# Export the Root CA cert
$ca = Get-ChildItem -Path cert:\LocalMachine\Root |
      Where-Object { $_.Subject -like "*skilz-Root-CA*" }

$pem = "-----BEGIN CERTIFICATE-----`r`n" +
       [Convert]::ToBase64String($ca.RawData, 'InsertLineBreaks') +
       "`r`n-----END CERTIFICATE-----"

$pem | Out-File -FilePath C:\Temp\skilz-Root-CA.crt -Encoding ascii
Write-Host "Exported to C:\Temp\skilz-Root-CA.crt"
```

On the jumpbox (copy the file and encode it):

```bash
# Copy from dc (use whatever method is available — SCP, SMB share, etc.)
# Example via Python WinRM or direct SCP if SSH is enabled on dc:
scp Administrator@192.168.15.10:'C:\Temp\skilz-Root-CA.crt' /tmp/

# Encode for the ClusterAdcsIssuer caBundle field
base64 -w0 /tmp/skilz-Root-CA.crt
# Copy the output — you need it for the next step
```

### B.7 Create the ClusterAdcsIssuer

Replace `<CA_BUNDLE_BASE64>` with the output from the `base64` command above:

```bash
kubectl apply -f - <<'EOF'
apiVersion: adcs.certmanager.csf.nokia.com/v1
kind: ClusterAdcsIssuer
metadata:
  name: skilz-adcs-issuer
spec:
  credentialsRef:
    name: adcs-issuer-credentials
  caBundle: <CA_BUNDLE_BASE64>
  url: https://dc.skilz.io/certsrv
  templateName: WebServer
  retryInterval: 5m
  statusCheckInterval: 1h
EOF
```

Check the issuer status:

```bash
kubectl get clusteradcsissuer skilz-adcs-issuer
# Expected: READY=True

kubectl describe clusteradcsissuer skilz-adcs-issuer
# Look for: Status.Conditions: Type=Ready, Status=True
```

---

## Part C — Verification

Test the full pipeline before relying on it for application certs.

### C.1 Issue a test certificate

```bash
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-adcs-cert
  namespace: default
spec:
  secretName: test-adcs-tls
  duration: 8760h
  renewBefore: 720h
  commonName: test.skilz.io
  dnsNames:
    - test.skilz.io
  issuerRef:
    name: skilz-adcs-issuer
    kind: ClusterAdcsIssuer
    group: adcs.certmanager.csf.nokia.com
EOF
```

Watch the status — `READY` should change to `True` within 30 seconds:

```bash
kubectl get certificate test-adcs-cert -w -n default
```

### C.2 Inspect the issued certificate

```bash
# Verify the Secret was created
kubectl get secret test-adcs-tls -n default

# Decode and inspect the certificate
kubectl get secret test-adcs-tls -n default \
    -o jsonpath='{.data.tls\.crt}' | \
    base64 -d | \
    openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

Expected output:
```
subject=CN = test.skilz.io
issuer=CN = skilz-Sub-CA, DC = skilz, DC = io
notBefore=Mon Jun  9 10:41:00 2026
notAfter=Mon Jun  8 10:41:00 2027
X509v3 Subject Alternative Name:
    DNS:test.skilz.io
```

Verify the chain trusts the Root CA:

```bash
# This works if the Root CA is installed on the jumpbox
openssl verify \
    -CAfile /usr/local/share/ca-certificates/skilz-Root-CA.crt \
    <(kubectl get secret test-adcs-tls -n default \
        -o jsonpath='{.data.tls\.crt}' | base64 -d)
# Expected: stdin: OK
```

### C.3 Clean up

```bash
kubectl delete certificate test-adcs-cert -n default
kubectl delete secret test-adcs-tls -n default
```

---

## Part D — Usage in Ingress resources

Any Ingress resource can trigger automatic certificate issuance by adding three
annotations. cert-manager detects them, creates a `CertificateRequest` with the correct
external issuer reference, adcs-issuer signs it via dc, and the cert appears in
the named Secret. NGINX Ingress picks it up without any restart.

> **Why three annotations?** `cert-manager.io/cluster-issuer` only works with cert-manager's
> built-in `ClusterIssuer` type. `ClusterAdcsIssuer` is an external type — cert-manager
> needs the `issuer-kind` and `issuer-group` annotations to route the request to the
> adcs-issuer controller instead of its own internal handlers.

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

Either way, the cert issuance happens automatically with no further action needed.
Check with:

```bash
kubectl get certificate -n myapp
kubectl get secret myapp-tls -n myapp
```

---

## Troubleshooting

### Certificate stays READY=False

Work through the chain from the bottom up:

```bash
# 1. Check the CertificateRequest
kubectl get certificaterequest -n <namespace>
kubectl describe certificaterequest -n <namespace> | grep -A10 "Status:"

# 2. Check the AdcsRequest
kubectl get adcsrequest -n <namespace>
kubectl describe adcsrequest -n <namespace>

# 3. Check adcs-issuer logs
kubectl logs -n cert-manager -l app=adcs-issuer --tail=80
```

---

### "certificate has expired or is not yet valid"

**Root cause: clock skew.** This is the most common cause of cert failures in this setup.

Check the clock on dc and on the K8s nodes:
```powershell
# dc
w32tm /query /status
```
```bash
# Any K8s node
timedatectl status
```

If they differ by more than a few minutes:
1. Fix dc NTP (see Prerequisites section)
2. Fix Linux node NTP (see Prerequisites section)
3. Once clocks agree, reissue the IIS cert on dc if it was originally issued with the wrong time — a cert's `NotBefore`/`NotAfter` is baked in at issuance and cannot be changed without reissuing

---

### adcs-issuer pod crashes on startup — "Unauthorized" on leases

The K8s 1.32 leader election RBAC patch (step B.3) was not applied. Apply it and the
pod will come up cleanly:

```bash
kubectl logs -n cert-manager -l app=adcs-issuer --tail=20
# Look for: leases.coordination.k8s.io is forbidden
```

Apply the patch from B.3, then delete and let the pod restart:

```bash
kubectl delete pod -n cert-manager -l app=adcs-issuer
```

---

### CertificateRequest stuck in Pending state (no error)

The approval RBAC patch (step B.4) is missing or incomplete.

Check what format cert-manager used in its SAR by inspecting the controller log:

```bash
kubectl logs -n cert-manager \
    -l app.kubernetes.io/component=controller \
    --tail=100 | grep -i "approve\|sar\|adcs"
```

If you see `SubjectAccessReview denied for resource: adcs.certmanager.csf.nokia.com/ClusterAdcsIssuer/skilz-adcs-issuer`,
that exact string is the resourceName that needs to be in the ClusterRole. Re-apply the
patch from B.4.

---

### TLS handshake error connecting to certsrv

```bash
# Test TLS to certsrv from the jumpbox
curl -v --cacert /usr/local/share/ca-certificates/skilz-Root-CA.crt \
    https://dc.skilz.io/certsrv/ 2>&1 | head -30
```

| curl output | Likely cause |
|---|---|
| `Connection refused` | IIS not running or not bound to 443 — run `iisreset /status` on dc |
| `certificate verify failed` | Wrong CA bundle in the ClusterAdcsIssuer, or caBundle is the SubCA cert instead of Root CA |
| `certificate is not yet valid` | dc clock was ahead when the IIS cert was issued — reissue the cert after fixing NTP |
| `HTTP/1.1 401` | Working correctly — certsrv requires auth |

---

### cert-manager webhook "failed to call webhook"

```bash
kubectl logs -n cert-manager -l app.kubernetes.io/component=webhook --tail=30
```

If you see TLS errors or `server has asked for client to provide credentials`, a
significant clock change happened and existing webhook mTLS certs are invalid. Reinstall
cert-manager (see B.1).

---

## End of guide checklist

- [ ] dc time synchronised to external NTP source (`w32tm /query /status` shows non-local source)
- [ ] All 6 K8s nodes show `System clock synchronized: yes` (`timedatectl status`)
- [ ] `RootDistanceMaxSec=30` in `/etc/systemd/timesyncd.conf` on all nodes
- [ ] `<service-account>` created in AD, member of Domain Admins
- [ ] TLS cert issued for `dc.skilz.io` from SubCA
- [ ] TLS cert bound to IIS Default Web Site port 443
- [ ] `curl https://dc.skilz.io/certsrv/` returns HTTP 401 (not a connection error)
- [ ] cert-manager installed, all 3 pods Running
- [ ] adcs-issuer installed, pod Running
- [ ] Leader election RBAC patch applied (coordination.k8s.io/leases)
- [ ] Approval ClusterRole patch applied (five resourceName entries)
- [ ] `adcs-issuer-credentials` Secret created in cert-manager namespace
- [ ] `ClusterAdcsIssuer` `skilz-adcs-issuer` shows `READY=True`
- [ ] Test certificate issued successfully, `READY=True`, Secret contains tls.crt + tls.key
- [ ] Test cert `issuer` field shows `skilz-Sub-CA` (not self-signed)
- [ ] Test cert cleaned up

**Next: [Module 03 — Persistent Storage with Longhorn](../03-storage/longhorn-persistent-storage.md)**
