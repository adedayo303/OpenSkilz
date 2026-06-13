# Day 0 — Active Directory, DNS, and PKI

> **What you will have at the end of this module:**
> - A fully functioning Windows Server domain (`skilz.io`) with `dc` as the DC
> - Internal DNS resolving all lab hostnames
> - Two-tier PKI: Root CA on `rootca` (offline after this day) and Subordinate CA on `dc`
> - All Linux nodes trusting the Root CA
> - NFS storage exported from `dc`'s second disk
> - An automation service account (`<service-account>`) ready for Module 02 cert automation

---

## PKI design

```
rootca  (192.168.13.199, standalone, not domain joined)
  └── skilz-Root-CA
      └── signs the SubCA certificate once, then powered off permanently

dc  (192.168.15.10, domain joined, stays online)
  └── skilz-Sub-CA
      └── issues all leaf certificates for K8s nodes and services
      └── cert-manager calls this automatically via adcs-issuer (Module 02)
```

The Root CA private key lives on an offline VM. If the SubCA is ever compromised,
power on `rootca`, revoke it, and issue a new SubCA. Clients trust the Root CA cert
and automatically trust anything the SubCA signs.

---

## Node reference

| Hostname | IP | OS | Role |
|---|---|---|---|
| rootca | 192.168.13.199 | Windows Server 2022 | Offline Root CA — powered off after Day 1 |
| dc | 192.168.15.10 | Windows Server 2022 | Domain controller · DNS · SubCA · NFS |
| k8s-cp-01 | 192.168.14.11 | Ubuntu 24.04 LTS | K8s control plane |
| k8s-cp-02 | 192.168.15.11 | Ubuntu 24.04 LTS | K8s control plane |
| k8s-cp-03 | 192.168.16.11 | Ubuntu 24.04 LTS | K8s control plane |
| k8s-w-01 | 192.168.14.12 | Ubuntu 24.04 LTS | K8s worker |
| k8s-w-02 | 192.168.15.12 | Ubuntu 24.04 LTS | K8s worker |
| k8s-w-03 | 192.168.16.12 | Ubuntu 24.04 LTS | K8s worker  |
| k8s-jb   | 192.168.13.245 | Ubuntu 24.04 LTS | Jumpbox — admin access |
| gitlab-srv | 192.168.14.13 | Ubuntu 24.04 LTS | GitLab (added later on) |

---

## Section 1 — rootca: The Offline Root CA

> This section runs on **rootca (192.168.13.199)** only.
> rootca does not join the domain. Log in as the local Administrator.

### 1.1 Install AD CS — Root CA

1. **Server Manager → Add Roles and Features**
2. Tick **Active Directory Certificate Services**
3. On Role Services, tick only **Certification Authority** (nothing else)
4. Click **Install**
5. Click the yellow flag → **Configure Active Directory Certificate Services**

In the configuration wizard:

- **Credentials:** local Administrator
- **Role Services:** Certification Authority
- **Setup type:** Standalone CA (not Enterprise, as rootca is not domain joined)
- **CA type:** Root CA
- **Private key:** Create a new private key
- **Cryptography:** Key length: **4096**, Hash: SHA256
- **CA Name:** `skilz-Root-CA`
- **Validity period:** 10 years
- **Database:** accept defaults
- Click **Configure**

### 1.2 Export the Root CA certificate

This certificate is distributed to all Linux nodes so they trust anything signed by the PKI chain.

On rootca (PowerShell as Administrator):

```powershell
New-Item -ItemType Directory -Path C:\Temp -Force | Out-Null

$ca = Get-ChildItem -Path cert:\LocalMachine\Root |
      Where-Object { $_.Subject -like "*skilz-Root-CA*" }

$pem = "-----BEGIN CERTIFICATE-----`r`n" +
       [Convert]::ToBase64String($ca.RawData, 'InsertLineBreaks') +
       "`r`n-----END CERTIFICATE-----"

$pem | Out-File -FilePath C:\Temp\skilz-root-ca.crt -Encoding ascii
Write-Host "Root CA cert exported to C:\Temp\skilz-root-ca.crt"
```

Copy `C:\Temp\skilz-root-ca.crt` to a USB or network share accessible from k8s-jb (`192.168.13.245`).
You will need this file in Section 4.

---

## Section 2 — dc: Active Directory and DNS

> This section runs on **dc (192.168.15.10)**. Log in as the local Administrator.
> The machine must already be named `dc` (done in Day 0).

### 2.1 Install Active Directory Domain Services

1. **Server Manager → Add Roles and Features**
2. Tick **Active Directory Domain Services** → Add Features when prompted
3. Click **Install**
4. Click the yellow flag → **Promote this server to a domain controller**

In the AD DS Configuration Wizard:

- **Deployment Configuration:** Add a new forest
- **Root domain name:** `skilz.io`
- **Domain Controller Options:**
  - Forest functional level: Windows Server 2016
  - Domain functional level: Windows Server 2016
  - Tick **DNS Server**
  - Set a DSRM password (store it safely)
- **DNS Options:** ignore the delegation warning, which is expected for a new forest
- **NetBIOS name:** `SKILZ`
- **Paths:** leave defaults
- Click **Install**. The server restarts automatically.

After restart, log in as `SKILZ\Administrator`.

### 2.2 Add DNS records

Open **DNS Manager** → expand `dc → Forward Lookup Zones → skilz.io` →
right-click → **New Host (A or AAAA)**.

Add every record in this table:

| Name | IP | Notes |
|---|---|---|
| rootca | 192.168.13.199 | |
| k8s-cp-01 | 192.168.14.11 | |
| k8s-cp-02 | 192.168.15.11 | |
| k8s-cp-03 | 192.168.16.11 | |
| k8s-w-01 | 192.168.14.12 | |
| k8s-w-02 | 192.168.15.12 | |
| k8s-jb   | 192.168.13.245 | Jumpbox — admin access |
| k8s-api | 192.168.14.30 | kube-vip VIP — no VM |
| grafana | 192.168.14.31 | MetalLB ingress VIP |
| vault | 192.168.14.31 | MetalLB ingress VIP |
| gitlab | 192.168.14.31 | MetalLB ingress VIP |
| nautobot | 192.168.14.31 | MetalLB ingress VIP |
| prometheus | 192.168.14.31 | MetalLB ingress VIP |
| netobserv | 192.168.14.31 | MetalLB ingress VIP |

> All application entries (`grafana`, `vault`, etc.) point to the same IP (`192.168.14.31`).
> That is the MetalLB ingress VIP. NGINX routes by hostname, which is covered in Module 02.

`dc.skilz.io` is auto-registered by the DC itself when it is promoted. Do not
manually add a `dc` A record.

Verify DNS from k8s-jb after this section:
```bash
nslookup k8s-cp-01 192.168.15.10
# Expected: 192.168.14.11
nslookup grafana.skilz.io 192.168.15.10
# Expected: 192.168.14.31
```

---

## Section 3 — dc: Subordinate CA

### 3.1 Install AD CS Subordinate CA

On dc (PowerShell as Administrator or Server Manager):

1. **Server Manager → Add Roles and Features**
2. Tick **Active Directory Certificate Services**
3. Role Services: tick **Certification Authority** and **Certification Authority Web Enrollment**
4. Click **Install**
5. Click the yellow flag → **Configure Active Directory Certificate Services**

In the wizard:

- **Credentials:** `SKILZ\Administrator`
- **Role Services:** Certification Authority + Certification Authority Web Enrollment
- **Setup type:** Enterprise CA
- **CA type:** Subordinate CA
- **Private key:** Create a new private key
- **Cryptography:** Key length: **2048**, Hash: SHA256
- **CA Name:** `skilz-Sub-CA`
- **Parent CA:** Select **Send a certificate request to a parent CA**
  - Parent CA location: `rootca\skilz-Root-CA`
  - *(If rootca is not reachable, choose "Save a certificate request to file" —
    then submit it manually using the steps in Section 3.2)*
- Click **Configure**

### 3.2 Manual SubCA signing (if rootca was not reachable)

If the wizard saved a `.req` file instead of submitting automatically:

On rootca (copy the `.req` file there first):
```powershell
# Submit the SubCA CSR to the Root CA
certreq -submit `
    -config "rootca\skilz-Root-CA" `
    C:\Temp\dc-subca.req `
    C:\Temp\dc-subca.cer
```

Back on dc:
```powershell
# Install the issued SubCA certificate
certutil -installCert C:\Temp\dc-subca.cer
Restart-Service certsvc
```

### 3.3 Power off rootca

After the SubCA is signed and running, power off rootca. Its job is done.
It only needs to be powered on again if the SubCA itself needs to be reissued.

Verify the SubCA is running on dc:
```powershell
Get-Service certsvc
# Status: Running

certutil -ping
# CertUtil: -ping command completed successfully
```

---

## Section 4 — Distribute Root CA trust to all Linux nodes

Linux nodes trust TLS certificates that chain to CAs in their system trust store.
Add the Root CA cert so every node automatically trusts certs issued by `skilz-Sub-CA`.

On k8s-jb, copy `skilz-root-ca.crt` from rootca (or from wherever you stored
it in Section 1.2):

```bash
# Verify the file is on k8s-jb
ls -la /tmp/skilz-root-ca.crt

# Distribute to all 6 nodes
for node in 192.168.14.11 192.168.15.11 192.168.16.11 \
            192.168.14.12 192.168.15.12; do
    echo "=== $node ==="
    scp /tmp/skilz-root-ca.crt <admin-user>@${node}:/tmp/
    ssh <admin-user>@${node} "
        sudo cp /tmp/skilz-root-ca.crt /usr/local/share/ca-certificates/skilz-root-ca.crt &&
        sudo update-ca-certificates
    "
done
# Also install on k8s-jb itself
sudo cp /tmp/skilz-root-ca.crt /usr/local/share/ca-certificates/skilz-root-ca.crt
sudo update-ca-certificates
```

Verify on any node:
```bash
openssl verify \
    -CAfile /usr/local/share/ca-certificates/skilz-root-ca.crt \
    /usr/local/share/ca-certificates/skilz-root-ca.crt
# Expected: OK (self-signed root verifies against itself)
```

---

## Section 5 — NFS storage on dc

### Why NFS alongside Longhorn?

We introduced NFS to demonstrate a traditional RWX storage solution. While Longhorn excels at replicated block storage, NFS provides an easy-to-understand example of shared storage that multiple pods can access simultaneously.(`ReadWriteMany`)

dc's 500 GB data disk provides the NFS shares.

### 5.1 Initialise the second disk

The 500 GB disk was attached in Day 0. Initialise it in Windows:

1. Open **Disk Management** (`diskmgmt.msc`)
2. The 500 GB disk appears as **Unallocated**. If prompted to initialise, click OK.
3. Right-click the unallocated space → **New Simple Volume**
4. Use all available space
5. Drive letter: **E:**
6. Format as NTFS, label: `NFS-Data`
7. Click **Finish**

### 5.2 Install NFS Server and create exports

On dc (PowerShell as Administrator):

```powershell
# Install NFS Server role
Install-WindowsFeature NFS-Server -IncludeManagementTools

# Create the directory structure on E:
New-Item -ItemType Directory -Path E:\k8s-nfs -Force
New-Item -ItemType Directory -Path E:\k8s-nfs\longhorn-backup -Force
New-Item -ItemType Directory -Path E:\k8s-nfs\shared-volumes -Force

# Create the NFS share (single share, subdirectory layout inside)
New-NfsShare -Name "k8s-nfs" -Path "E:\k8s-nfs" `
    -EnableUnmappedAccess $true `
    -Authentication sys

# Grant read/write with root access to the K8s subnets
# Adjust CIDRs to match your environment
foreach ($subnet in @("192.168.14.0/24", "192.168.15.0/24", "192.168.16.0/24")) {
    Grant-NfsSharePermission -Name "k8s-nfs" `
        -ClientName $subnet `
        -ClientType "host" `
        -Permission "ReadWrite" `
        -AllowRootAccess $true
}

# Verify
Get-NfsShare k8s-nfs | Format-List
```

### 5.3 Verify NFS from k8s-jb

```bash
# Test mount
sudo mkdir -p /mnt/nfs-test
sudo mount -t nfs 192.168.15.10:/k8s-nfs /mnt/nfs-test

# Write and read back
echo "NFS working from $(hostname)" | sudo tee /mnt/nfs-test/test.txt
cat /mnt/nfs-test/test.txt

# Check subdirectories
ls /mnt/nfs-test/
# Expected: longhorn-backup  shared-volumes  test.txt

sudo umount /mnt/nfs-test
```

### 5.4 NFS paths used by Modules 03 and later

| Path | Used for |
|---|---|
| `192.168.15.10:/k8s-nfs/longhorn-backup` | Longhorn backup target (Module 03) |
| `192.168.15.10:/k8s-nfs/shared-volumes` | RWX persistent volumes (Module 03+) |

---

## Section 6 — AD service accounts

### 6.1 Automation account for cert issuance

This account is used by `adcs-issuer` in Module 02 to authenticate to certsrv and
request certificates on behalf of Kubernetes workloads.

On dc (PowerShell as Administrator):

```powershell
New-ADUser `
    -Name "<service-account>" `
    -SamAccountName "<service-account>" `
    -UserPrincipalName "<service-account>@skilz.io" `
    -AccountPassword (Read-Host -AsSecureString "Enter password for <service-account>") `
    -Enabled $true `
    -PasswordNeverExpires $true `
    -CannotChangePassword $true `
    -Description "Automation service account for K8s cert issuance via adcs-issuer"

Add-ADGroupMember -Identity "Domain Admins" -Members "<service-account>"

# Verify
Get-ADUser <service-account> | Select-Object SamAccountName, Enabled
```

> **Production note:** Domain Admins is used here for the lab. In production, create a
> dedicated security group with only the "Request Certificates" right on the specific
> ADCS template.

### 6.2 Application service accounts and groups

```powershell
# LDAP bind account (used by Grafana, Nautobot, NetObserv for AD authentication)
New-ADUser -Name "<svc-ldap-bind>" `
    -SamAccountName "<svc-ldap-bind>" `
    -UserPrincipalName "<svc-ldap-bind>@skilz.io" `
    -AccountPassword (Read-Host -AsSecureString "Enter password for <svc-ldap-bind>") `
    -Enabled $true `
    -PasswordNeverExpires $true `
    -CannotChangePassword $true `
    -Description "LDAP bind service account"

# Developer accounts
foreach ($user in @(
    @{Name="git-admin";       Display="Git Admin"},
    @{Name="dev-alice";       Display="Alice Dev"},
    @{Name="dev-bob";         Display="Bob Dev"},
    @{Name="view-charlie";    Display="Charlie Viewer"}
)) {
    New-ADUser -Name $user.Display `
        -SamAccountName $user.Name `
        -UserPrincipalName "$($user.Name)@skilz.io" `
        -AccountPassword (Read-Host -AsSecureString "Enter password for $($user.Name)") `
        -Enabled $true `
        -ChangePasswordAtLogon $false
}

# Groups
New-ADGroup -Name "k8s-admins"     -GroupScope Global -GroupCategory Security
New-ADGroup -Name "k8s-developers" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "k8s-viewers"    -GroupScope Global -GroupCategory Security

Add-ADGroupMember -Identity "k8s-admins"     -Members "git-admin"
Add-ADGroupMember -Identity "k8s-developers" -Members "dev-alice","dev-bob"
Add-ADGroupMember -Identity "k8s-viewers"    -Members "view-charlie"

Write-Host "AD users and groups created"
```

---

## Section 7 — Prepare certsrv for HTTPS

certsrv (Certificate Authority Web Enrollment) is the HTTP endpoint that `adcs-issuer`
uses in Module 02 to submit CSRs. By default it only serves HTTP, meaning credentials
travel in cleartext.

Binding a TLS certificate to IIS before Module 02 starts is the correct approach.
This requires a cert from the SubCA, issued directly via `certreq` on the CA server
(not via certsrv itself, as that is the bootstrap problem).

The full procedure for IIS cert issuance, binding, and verification is in Module 02
cert workflow guide:

**See: [Module 02 cert-manager and ADCS Workflow](../02-networking/cert-manager-adcs-workflow.md), Part A**

Complete Part A of that guide before moving to Module 01. The steps are:
1. Issue an HTTPS cert for `dc.skilz.io` from the local SubCA using `certreq`
2. Bind it to IIS Default Web Site port 443
3. Verify `https://dc.skilz.io/certsrv/` returns HTTP 401 (not a TLS error)

---

## Section 8 — (Optional) Join Ubuntu nodes to Active Directory

Domain-joining Linux nodes is not required for Kubernetes to function but is recommended
for consistent lab setup. It means SSH access can be governed by AD group membership.

Run on each Ubuntu 24.04 node:

```bash
# Install realmd and SSSD
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss \
    adcli samba-common-bin krb5-user oddjob oddjob-mkhomedir

# Discover the domain (DNS must point at dc)
realm discover skilz.io

# Join (enter SKILZ\Administrator password when prompted)
sudo realm join --user=Administrator skilz.io
```

Configure SSSD:

```bash
sudo tee /etc/sssd/sssd.conf > /dev/null << 'EOF'
[sssd]
domains = skilz.io
config_file_version = 2
services = nss, pam

[domain/skilz.io]
ad_domain = skilz.io
krb5_realm = SKILZ.IO
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
id_provider = ad
fallback_homedir = /home/%u@%d
default_shell = /bin/bash
ldap_id_mapping = True
use_fully_qualified_names = False
access_provider = simple
simple_allow_groups = k8s-admins, k8s-developers
EOF

sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl restart sssd
sudo systemctl enable sssd
sudo pam-auth-update --enable mkhomedir

# Allow k8s-admins to sudo without password
echo '%k8s-admins ALL=(ALL:ALL) NOPASSWD:ALL' | \
    sudo tee /etc/sudoers.d/k8s-admins
sudo chmod 440 /etc/sudoers.d/k8s-admins
```

Verify:
```bash
realm list
# Should show: skilz.io — configured

id git-admin
# Should return uid/gid from AD
```

---

## End of Day 1 checklist

**rootca:**
- [ ] AD CS Root CA (`skilz-Root-CA`) installed as Standalone Root CA
- [ ] Root CA cert exported to `C:\Temp\skilz-root-ca.crt`
- [ ] SubCA certificate signed (either online or via manual `.req` submit)
- [ ] `rootca` powered off

**dc:**
- [ ] Promoted to domain controller for `skilz.io`
- [ ] DNS records added for all nodes and service VIPs
- [ ] AD CS Subordinate CA (`skilz-Sub-CA`) installed and running
- [ ] `certsvc` service running (`Get-Service certsvc` shows Running)
- [ ] Second disk (500 GB) added to dc and formatted as E: (`NFS-Data`)
- [ ] NFS Server role installed
- [ ] NFS share `k8s-nfs` created with `longhorn-backup` and `shared-volumes` subdirectories
- [ ] NFS mount tested from k8s-jb: write and read succeed
- [ ] `<service-account>` created in AD, member of Domain Admins
- [ ] Application service accounts and groups created
- [ ] IIS HTTPS cert issued for `dc.skilz.io` and bound to port 443
- [ ] `https://dc.skilz.io/certsrv/` returns HTTP 401 (not a TLS error)

**All Linux nodes:**
- [ ] Root CA cert in system trust store (`update-ca-certificates` run)
- [ ] `nfs-common` installed
- [ ] NTP synchronised (`System clock synchronized: yes`)
- [ ] DNS resolves `k8s-cp-01` to `192.168.14.11` (`nslookup k8s-cp-01 192.168.15.10`)
- [ ] (Optional) Domain-joined, `realm list` shows `skilz.io`

**Next: [Module 01 HA Kubernetes Cluster](../01-foundations/ha-kubernetes-cluster.md)**
