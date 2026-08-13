# Module 04 — Vault Secrets Management

> **What you will have at the end of this module:** HashiCorp Vault running in HA mode
> across 3 pods using integrated Raft storage, fully unsealed, accessible via HTTPS at
> `https://vault.skilz.io`, and ready to issue dynamic secrets to applications
> running in the cluster.

---

## The Problem With Plain Kubernetes Secrets

Until Vault is running, credentials live in Kubernetes Secrets. Let's be honest about
what a Kubernetes Secret is:

```bash
kubectl get secret adcs-issuer-credentials -n cert-manager \
    -o jsonpath='{.data.password}' | base64 -d
```

That command returns the AD service account password in plaintext to anyone with
`kubectl` access and the `get secret` verb. By default, every service account in the
cluster can potentially read secrets in its own namespace. In a real production
environment, this is a compliance violation.

Vault solves this properly:

- **Nothing is stored in YAML or Kubernetes Secrets** — applications request credentials
  at runtime, credentials are not written to disk
- **Dynamic secrets** — Vault generates a unique username/password pair per application
  instance, with a TTL. When the pod dies, the credentials expire. There is no static
  password to rotate manually.
- **Audit log** — every secret access is logged: who requested it, when, what they got
- **Lease and revocation** — if a pod is compromised, you revoke its lease. The credentials
  become invalid immediately, with no manual rotation needed.

---

## The Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Vault HA Cluster                      │
│                                                         │
│  vault-0 (standby)   vault-1 (active)   vault-2 (standby) │
│       │                   │                   │         │
│       └───────────────────┼───────────────────┘         │
│                           │ Raft consensus              │
│                    /vault/data (Longhorn PVC)            │
└───────────────────────────┼─────────────────────────────┘
                            │
                    vault.skilz.io
                    (NGINX Ingress, adcs-issuer TLS)
```

**Raft storage vs. Consul:** Earlier guides often used Consul as Vault's storage backend,
requiring a full Consul cluster to be running alongside Vault. Vault's integrated Raft
storage (available since Vault 1.4) removes that dependency entirely — Vault nodes form
their own consensus cluster and persist data to local PVCs. For a lab and small
production deployments, this is the correct approach.

**3 key shares, threshold 2:** On initialization, Vault generates a root key and splits
it using Shamir's Secret Sharing into N shares, requiring any M to reconstruct it. This
is a physical security mechanism: in production, each key share goes to a different
trusted person. No single person can unseal Vault alone.

---

## Installing Vault

### Install (Helm)

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

cat > /tmp/vault-raft-values.yaml << 'EOF'
injector:
  enabled: true

server:
  dataStorage:
    enabled: true
    size: 2Gi
    storageClass: longhorn

  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      config: |
        ui = true

        listener "tcp" {
          tls_disable = 1
          address = "[::]:8200"
          cluster_address = "[::]:8201"
          telemetry {
            unauthenticated_metrics_access = "true"
          }
        }

        storage "raft" {
          path = "/vault/data"

          retry_join {
            leader_api_addr = "http://vault-0.vault-internal:8200"
          }
          retry_join {
            leader_api_addr = "http://vault-1.vault-internal:8200"
          }
          retry_join {
            leader_api_addr = "http://vault-2.vault-internal:8200"
          }
        }

        service_registration "kubernetes" {}
EOF

helm install vault hashicorp/vault -n vault --create-namespace -f /tmp/vault-raft-values.yaml
```

> **Note:** `tls_disable = 1` — Vault itself listens on HTTP internally. TLS termination
> is handled by NGINX Ingress. This is the correct pattern: TLS terminates at the ingress
> layer; internal pod-to-pod communication uses HTTP within the cluster network.

### Ingress (TLS via adcs-issuer)

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vault-ingress
  namespace: vault
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    cert-manager.io/issuer: "skilz-adcs-issuer"
    cert-manager.io/issuer-kind: "ClusterAdcsIssuer"
    cert-manager.io/issuer-group: "adcs.certmanager.csf.nokia.com"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - vault.skilz.io
    secretName: vault-tls     # cert-manager creates this automatically
  rules:
  - host: vault.skilz.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vault
            port:
              number: 8200
EOF
```

### Initialize Vault

Run once on vault-0 — this generates the unseal keys and root token. **These cannot be
recovered if lost.**

```bash
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=3 \
  -key-threshold=2
```

Output:
```
Unseal Key 1: <key-1>
Unseal Key 2: <key-2>
Unseal Key 3: <key-3>

Initial Root Token: hvs.<token>
```

**Store these securely.** In production: a password manager, an HSM, or a printed copy
in a physical safe. In this lab: a password manager at minimum.

> **Why vault-0?** Init on vault-0 makes it the initial Raft leader. vault-1 and vault-2
> join automatically via `retry_join`. If you init on a non-0 pod, vault-0 won't unseal
> cleanly until the Raft cluster forms.

### Unseal All Pods

Vault starts sealed after every restart. Each pod must be individually unsealed with
any 2 of the 3 keys.

```bash
KEY1="<unseal-key-1>"
KEY2="<unseal-key-2>"

# Run both unseal commands in a single exec — if the container restarts between
# two separate execs, the Shamir progress resets to 0/2
for pod in vault-0 vault-1 vault-2; do
  kubectl exec -n vault $pod -- sh -c "
    vault operator unseal $KEY1 && vault operator unseal $KEY2
  "
done
```

> **vault-1 / vault-2 unseal behaviour:** After init on vault-0, vault-1 and vault-2 join
> the Raft cluster and replicate vault-0's sealed state. Their unseal progress may reset
> if the container restarts mid-command — that is why both keys go in a single `sh -c` call.
> In practice the cluster forms within seconds and both standbys unseal cleanly.

Check status:
```bash
for pod in vault-0 vault-1 vault-2; do
  echo -n "$pod: "
  kubectl exec -n vault $pod -- vault status 2>&1 | grep -E "Sealed|HA Mode"
done
```

Expected output:
```
vault-0: Sealed false  HA Mode active
vault-1: Sealed false  HA Mode standby
vault-2: Sealed false  HA Mode standby
```

---

## Troubleshooting

### Vault pods running but 0/1 Ready, pointing at Consul

**Symptom:**
```
vault-0   0/1   Running   0   4d
vault-1   0/1   Running   0   4d
```

Logs showing:
```
storage migration check error: Get "http://192.168.14.12:8500/v1/kv/vault/core/migration": 
  dial tcp 192.168.14.12:8500: connect: connection refused
```

**Root cause:** The Vault Helm chart, when `ha.enabled: true` is set without an explicit
storage config, defaults to Consul storage. It looks for a Consul agent at the host IP on
port 8500. No Consul was deployed at all. the error is Vault repeatedly failing to reach
a backend that does not exist.

**Fix:** Uninstall Vault, delete the PVCs (no data was written, since Vault had not
initialized), and reinstall with explicit Raft storage config.

```bash
helm uninstall vault -n vault
kubectl delete pvc -n vault --all
helm install vault hashicorp/vault -n vault -f /tmp/vault-raft-values.yaml
```

> **Why not just `helm upgrade`?** Helm cannot modify a StatefulSet's storage config in
> place — Kubernetes forbids changes to most StatefulSet spec fields after creation. This
> is a safety mechanism: changing volume claims on a running StatefulSet would lose data.
> Since Vault had not been initialized yet (no data), a clean reinstall was safe.

### Unseal progress resets mid-operation

**Symptom:**
```
vault operator unseal <key-1>   → Progress: 1/2
vault operator unseal <key-2>   → Progress: 0/2  (reset!)
```

**Root cause:** When `kubectl exec` runs sequentially (two separate commands), if the
vault container restarts between the two commands — even briefly — the Shamir unseal
progress is discarded. The new container instance starts with 0/2 progress.

**Fix:** Run both unseal commands in a single exec:
```bash
kubectl exec -n vault vault-0 -- sh -c "
  vault operator unseal <key-1> && vault operator unseal <key-2>
"
```

### Kubernetes API VIP unreachable when cp-01 is offline

**Symptom:** With cp-01 offline, the Kubernetes API VIP (192.168.14.30) becomes
unreachable even though cp-02 and cp-03 are healthy, and `kubectl` stops working entirely.
This isn't specific to Vault, but it blocks every `kubectl exec` command in this module,
so it's worth knowing about here.

**Root cause:** The kube-vip static pod manifest (`/etc/kubernetes/manifests/kube-vip.yaml`)
was only placed on cp-01 during initial cluster setup. kube-vip uses leader election via
a Kubernetes Lease object — but it can only participate in leader election if it is
running. With kube-vip only on cp-01, when cp-01 went down, no other node could claim
the VIP.

**Fix:** Copy the kube-vip manifest to all control plane nodes:

```bash
# Get the kube-vip config from the existing pod spec
kubectl get pod kube-vip-k8s-cp-01 -n kube-system \
  -o jsonpath='{.spec.containers[0].env}' | python3 -m json.tool

# Create /etc/kubernetes/manifests/kube-vip.yaml on cp-02 and cp-03
# (see Day 0 foundations for the full manifest)
for node in 192.168.15.11 192.168.16.11; do
  scp /tmp/kube-vip.yaml ae@${node}:/tmp/
  ssh ae@${node} "sudo cp /tmp/kube-vip.yaml /etc/kubernetes/manifests/"
done
```

kubelet picks up static pod manifests automatically — kube-vip starts within seconds and
participates in VIP leader election. Now if any single control plane goes down, another
claims the VIP within the `vip_leaseduration` (5 seconds in this config).

> **Analogy for network engineers:** This is identical to VRRP with all three routers
> configured as participants, rather than one primary and two that are not configured.
> The VIP floats between participants based on priority and availability.

---

## Accessing Vault

**UI:** `https://vault.skilz.io` — sign in with your AD account (`vault-admin` or any
member of `vault-admins`).

**CLI from the jumpbox:**
```bash
export VAULT_ADDR=https://vault.skilz.io
vault login -method=ldap username=vault-admin
vault status
vault secrets list
```

> The root token is revoked. Never use `VAULT_TOKEN=<root-token>` for day-to-day access.
> See the break glass section below for how to recover if AD is unavailable.

---

## Unseal Keys vs Root Token — What Each One Does

These are two completely different things. Confusing them is a common mistake.

| | Unseal Keys | Root Token |
|---|---|---|
| **Purpose** | Reconstruct the master key so Vault can decrypt its storage | Authenticate as a superuser to make API calls |
| **When needed** | After every pod/node restart — Vault starts sealed | Only when no other auth method works (break glass) |
| **How many** | 2 of 3 keys required (Shamir threshold) | 1 token |
| **Current state** | Stored securely (password manager, separate entries) | **Revoked** — intentionally destroyed after bootstrap |
| **Can be regenerated?** | No — these are the originals, protect them | Yes — using 2 of 3 unseal keys at any time |

**Unsealing after a pod restart does not use the root token.** You use the unseal keys:

```bash
KEY1="<unseal-key-1>"
KEY2="<unseal-key-2>"

for pod in vault-0 vault-1 vault-2; do
  kubectl exec -n vault $pod -- sh -c "
    vault operator unseal $KEY1 && vault operator unseal $KEY2
  "
done

# Confirm
for pod in vault-0 vault-1 vault-2; do
  echo -n "$pod: "
  kubectl exec -n vault $pod -- vault status 2>&1 | grep -E "Sealed|HA Mode"
done
```

The root token is not needed and plays no role in unsealing.

---

## Break Glass — AD Is Dead or Unreachable

When AD (dc) is down, LDAP auth fails for all human accounts. The root token is
revoked. This is the procedure to regain admin access without either.

**What you need:** any 2 of the 3 unseal keys. Nothing else.

### Step 1 — Confirm Vault is unsealed

```bash
kubectl exec -n vault vault-0 -- vault status 2>&1 | grep Sealed
# Sealed  false  → proceed
# Sealed  true   → unseal first (see above), then continue
```

### Step 2 — Generate a new root token

The `generate-root` command reconstructs a root token from unseal key shares without
needing the original root token.

```bash
# Initialise — Vault returns a nonce and a one-time pad (OTP)
kubectl exec -n vault vault-0 -- vault operator generate-root -init
```

Note both the **Nonce** and the **OTP** from the output. The OTP is used at the end to
decode the result; it is not stored anywhere.

```bash
# Provide unseal key 1
kubectl exec -n vault vault-0 -- vault operator generate-root \
  -nonce=<nonce> \
  <unseal-key-1>

# Output: Progress: 1/2

# Provide unseal key 2 — this completes the process
kubectl exec -n vault vault-0 -- vault operator generate-root \
  -nonce=<nonce> \
  <unseal-key-2>

# Output: Encoded Token: <encoded-token>
```

### Step 3 — Decode the token

```bash
kubectl exec -n vault vault-0 -- vault operator generate-root \
  -decode=<encoded-token> \
  -otp=<otp>

# Output: hvs.<new-root-token>
```

### Step 4 — Use it

```bash
export VAULT_ADDR=https://vault.skilz.io
export VAULT_TOKEN=hvs.<new-root-token>
vault status
vault secrets list
```

You now have full `*` access. Do what you need — fix the AD connection, create a
temporary token for an operator, rotate credentials, etc.

### Step 5 — Revoke the root token when done

**Do not leave the root token active.** Once the emergency is resolved, revoke it:

```bash
vault token revoke $VAULT_TOKEN
unset VAULT_TOKEN
```

If AD is back, verify LDAP login works before revoking:
```bash
vault login -method=ldap username=vault-admin
# Confirm: token_policies = [default vault-admin]
# Then revoke the root token
```

### Summary

```
AD down → LDAP auth fails
            ↓
        vault operator generate-root -init
        (needs 2 unseal keys, not the root token)
            ↓
        New root token → fix the problem
            ↓
        Revoke root token → back to normal
```

> **Key insight:** The unseal keys are the true master credential. The root token is
> ephemeral — it can always be regenerated from the keys. Protect the unseal keys above
> everything else. If you lose 2 of 3 unseal keys, Vault's data is **permanently
> unrecoverable** — there is no vendor support path, no backdoor.

---

## Active Directory Integration — Who Can Access Vault

### Why AD for Vault access?

Without AD integration, only the root token controls Vault. The root token is a single
point of failure and should only exist long enough to bootstrap. After integrating AD:

- The root token is revoked
- `vault-admins` AD group members get full Vault access
- `vault-ops` AD group members can read/write secrets
- GitLab uses an AppRole (machine identity, not human)
- Every action is logged with the AD username

### AD Structure for Vault (under `OU=Vault,DC=skilz,DC=io`)

| Object | Type | Vault Access |
|--------|------|-------------|
| `vault-admins` | Security Group | Full Vault access (all paths) |
| `vault-ops` | Security Group | Read/write `secret/*` and `gitlab/*` |
| `vault-bind` | Service Account | LDAP bind account for Vault's search |
| `vault-admin` | User | Human admin (in `vault-admins`) |
| `vault-ops-user` | User | Operator (in `vault-ops`) |

> `git-admin` (the GitLab admin) is also added to `vault-admins` — a GitLab admin who
> deploys infrastructure needs Vault access too.

**Create via PowerShell on dc:**

```powershell
$domain  = "DC=skilz,DC=io"
$vaultOU = "OU=Vault,$domain"
$pass    = ConvertTo-SecureString "REPLACE_LDAP_BIND_PASSWORD" -AsPlainText -Force

New-ADOrganizationalUnit -Name "Vault" -Path $domain
New-ADGroup -Name "vault-admins" -GroupScope Global -GroupCategory Security -Path $vaultOU
New-ADGroup -Name "vault-ops"    -GroupScope Global -GroupCategory Security -Path $vaultOU

New-ADUser -SamAccountName "vault-bind" -Name "Vault Bind" -Path $vaultOU `
    -AccountPassword $pass -Enabled $true -PasswordNeverExpires $true -ChangePasswordAtLogon $false

New-ADUser -SamAccountName "vault-admin" -Name "Vault Admin" -Path $vaultOU `
    -AccountPassword $pass -Enabled $true -PasswordNeverExpires $true -ChangePasswordAtLogon $false
Add-ADGroupMember -Identity "vault-admins" -Members "vault-admin"

New-ADUser -SamAccountName "vault-ops-user" -Name "Vault Operator" -Path $vaultOU `
    -AccountPassword $pass -Enabled $true -PasswordNeverExpires $true -ChangePasswordAtLogon $false
Add-ADGroupMember -Identity "vault-ops" -Members "vault-ops-user"

# Give GitLab admin Vault access too
Add-ADGroupMember -Identity "vault-admins" -Members "git-admin"
```

### Configure Vault LDAP auth

```bash
export VAULT_TOKEN=<root-token>

# Enable LDAP auth
vault auth enable ldap

# Configure against AD
vault write auth/ldap/config \
    url="ldap://192.168.15.10" \
    binddn="CN=Vault Bind,OU=Vault,DC=skilz,DC=io" \
    bindpass="REPLACE_LDAP_BIND_PASSWORD" \
    userdn="DC=skilz,DC=io" \
    userattr="sAMAccountName" \
    discoverdn=true \
    groupdn="DC=skilz,DC=io" \
    groupattr="cn" \
    groupfilter="(&(objectClass=group)(member={{.UserDN}}))" \
    insecure_tls=true \
    starttls=false
```

> **`discoverdn=true` is required against Active Directory.** Without it, Vault skips the
> search step and constructs the user's bind DN directly as `{userattr}={username},{userdn}`
> — e.g. `sAMAccountName=vault-admin,DC=skilz,DC=io`. That DN doesn't exist in AD, since AD's
> real RDN is always `CN=`, not `sAMAccountName=`. The login then fails with a generic "invalid
> credentials" error that looks identical to a wrong password, even when the password is
> correct. the failure happens before the password gets checked. `discoverdn=true` makes
> Vault bind as `binddn`/`bindpass` first, search for the user's real DN using `userfilter`,
> and bind as that DN with the supplied password — the two-step flow AD actually needs.

> **`groupfilter`** is the LDAP query Vault uses to find which groups a user belongs to.
> `{{.UserDN}}` is replaced with the authenticated user's Distinguished Name at query time.
> This is how Vault discovers that `vault-admin` is a member of `vault-admins`.

### Create Vault policies

```bash
# Full admin access
vault policy write vault-admin - << 'EOF'
path "*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF

# Operations: read/write secrets, no sys management
vault policy write vault-ops - << 'EOF'
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "gitlab/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOF

# GitLab AppRole: read only from gitlab/* paths
vault policy write gitlab-reader - << 'EOF'
path "gitlab/data/*" {
  capabilities = ["read"]
}
path "gitlab/metadata/*" {
  capabilities = ["list", "read"]
}
EOF
```

### Map AD groups to policies

```bash
vault write auth/ldap/groups/vault-admins  policies=vault-admin
vault write auth/ldap/groups/vault-ops     policies=vault-ops
vault write auth/ldap/groups/gitlab-admins policies=vault-ops
```

**Testing AD login:**
```bash
vault login -method=ldap username=vault-admin
# Password: REPLACE_LDAP_BIND_PASSWORD
# Should receive a token with vault-admin policy attached
```

### GitLab AppRole

```bash
vault auth enable approle

vault write auth/approle/role/gitlab \
    token_policies="gitlab-reader" \
    token_ttl=1h \
    token_max_ttl=4h

# Get credentials for the .env file on w-01
vault read -field=role_id auth/approle/role/gitlab/role-id
vault write -field=secret_id -f auth/approle/role/gitlab/secret-id
```

See [GitLab CI/CD](../appendix-cicd/gitlab-pipeline.md) (outside the main module sequence,
GitLab runs on a shared host, not inside the cluster) for how these credentials are used by
the inject-secrets.sh startup script.

---

## Secrets Stored in Vault

| Path | Keys | Used By |
|------|------|---------|
| `gitlab/ldap` | `host`, `bind_dn`, `bind_password`, `port`, `base` | GitLab LDAP auth |
| `gitlab/admin` | `root_password`, `root_pat`, `url` | Administrative access |

```bash
# Store GitLab LDAP secrets
vault secrets enable -path=gitlab kv-v2
vault kv put gitlab/ldap \
    bind_dn="CN=automateuser,CN=Users,DC=skilz,DC=io" \
    bind_password="REPLACE_LDAP_BIND_PASSWORD" \
    host="192.168.15.10" \
    port="389" \
    base="DC=skilz,DC=io"
```

`automateuser` is reused here rather than a dedicated GitLab service account, a lab-scale
decision covered in the GitLab module. The base is the whole domain (`DC=skilz,DC=io`)
rather than a dedicated OU, since no separate GitLab OU was created.

---

## Working With Secrets, Start to Finish

Everything above got Vault installed, reachable, and connected to Active Directory. This
section is about the part you'll actually do most days once that's in place: putting a
secret in, reading it back, changing it, and eventually getting rid of it. None of it
assumes you've used Vault before.

### A quick vocabulary check

A few terms come up constantly, so it's worth pinning them down before diving in.

- **Secrets engine**: a plugin that handles one kind of secret storage. The one used here,
  KV v2 (Key-Value version 2), is a simple place to put arbitrary secret data, like a
  password or an API key, at a path you choose.
- **Mount**: where a secrets engine lives in Vault's path namespace. `secret/` and `gitlab/`
  are both KV v2 mounts already in use in this environment, but they're separate, unrelated
  stores that happen to use the same engine type.
- **Path**: the location of a specific secret inside a mount, similar to a file path. For
  example, `secret/devices/cisco-cat9k-01` is a path inside the `secret/` mount.
- **Version**: KV v2 keeps a history of every write to a path. Writing to a path that
  already has data doesn't overwrite it, it adds a new version and keeps the old ones
  around (up to a configurable limit).
- **Lease**: applies to dynamic secrets, not KV. A lease is a rented, time-limited grant on
  something Vault generated for you, like a database username and password. When the lease
  ends, Vault takes the credential back.

### Why KV v2's paths look different depending on what you're doing

People writing a Vault policy for the first time often miss this, so it is worth calling out
early. When you run `vault kv put secret/devices/cisco-cat9k-01 ...`, the CLI is friendly
and hides something from you: under the hood, that command talks to
`secret/data/devices/cisco-cat9k-01`. Listing secrets talks to `secret/metadata/...`.
Deleting a specific version talks to `secret/delete/...`. Permanently destroying a version
talks to `secret/destroy/...`.

The reason this matters is policies. The `gitlab-reader` policy created earlier in this
module only grants `read` on `gitlab/data/*` and `list`/`read` on `gitlab/metadata/*`, with
no `delete` or `destroy` path in it at all, so an application using that AppRole can read
secrets but can't touch their history or wipe them out. Separating data operations from
version-management operations at the path level is what makes that kind of narrow, safe
permission possible.

### Writing a secret for the first time

```bash
vault kv put secret/devices/switch-lab-01 \
    ssh_user="netadmin" \
    ssh_password="ExamplePassword123"
```

Output:
```
== Secret Path ==
secret/data/devices/switch-lab-01

======= Metadata =======
Key                Value
---                -----
created_time        2026-08-13T10:02:11.123456Z
custom_metadata     <nil>
deletion_time       n/a
destroyed           false
version             1
```

That confirms the write succeeded and tells you this is version 1. Nothing else needs to
happen for this secret to be usable straight away.

### Reading it back

```bash
vault kv get secret/devices/switch-lab-01
```

That prints both the metadata and the key/value data. If a script needs just one value
without any of the surrounding formatting:

```bash
vault kv get -field=ssh_password secret/devices/switch-lab-01
```

### Updating a secret: `put` versus `patch`

This is one of the more common mistakes to make with KV v2, so it's worth walking through
carefully.

`vault kv put` replaces everything at that path. If the secret has three keys and you
`put` with only one of them, the other two are gone in the new version, not preserved.

```bash
# This creates version 2 with ONLY ssh_password set, ssh_user is dropped
vault kv put secret/devices/switch-lab-01 ssh_password="NewerPassword456"
```

`vault kv patch`, on the other hand, merges what you give it into the existing version,
leaving everything else untouched:

```bash
# Only bind_password changes, host, port, base, and bind_dn are all preserved
vault kv patch gitlab/ldap bind_password="a-new-password"
```

If `put` had been used there instead, the command would have needed to repeat every other
field (`host`, `port`, `base`, `bind_dn`) or those values would have quietly disappeared
from the new version. `patch` avoids that entirely.

A rough rule of thumb: reach for `patch` when you're changing one or two fields on an
existing secret, and `put` when you're deliberately replacing the whole thing.

### Looking at version history

```bash
vault kv metadata get secret/devices/switch-lab-01
```

This lists every version, when it was created, and whether it's been deleted or destroyed,
without returning the actual secret data. To read the data from a specific older version:

```bash
vault kv get -version=1 secret/devices/switch-lab-01
```

### Rolling back to an earlier version

If a bad update needs to be undone, there's a dedicated command for it rather than having
to manually copy old values back in:

```bash
vault kv rollback -version=1 secret/devices/switch-lab-01
```

This doesn't delete version 2 or rewrite history. It reads version 1's data and writes it
as a brand new version (in this example, version 3), so the full history stays intact and
auditable, and the current data now matches what version 1 had.

### Deleting a secret (the reversible way)

```bash
vault kv delete secret/devices/switch-lab-01
```

This is a soft delete of the latest version. `vault kv get` now returns "no data found" as
if the secret were gone, but the version and its data are still sitting there, just marked
deleted. This is deliberately reversible.

To delete specific versions rather than just the latest:

```bash
vault kv delete -versions=1,2 secret/devices/switch-lab-01
```

### Bringing a deleted secret back

```bash
vault kv undelete -versions=1 secret/devices/switch-lab-01
```

That clears the deletion marker on version 1, and `vault kv get -version=1` works again.
Undelete only works on soft-deleted versions, not destroyed ones (see next).

### Destroying a secret (not reversible)

Delete hides a version. Destroy actually erases the underlying data for that version, and
there's no undelete for it afterward.

```bash
vault kv destroy -versions=1 secret/devices/switch-lab-01
```

The version number still shows up in `vault kv metadata get` (so the history isn't
rewritten), but its data is gone, and the metadata will show `destroyed: true` for that
version. This is the appropriate step when a secret value is known to have leaked and needs
to stop existing anywhere, not just be hidden from normal reads.

### Removing a secret path entirely

Everything above operates on individual versions. To remove the whole path, every version
and all its metadata, in one step:

```bash
vault kv metadata delete secret/devices/switch-lab-01
```

After this, the path behaves as if nothing was written to it before. There's no CAS protection or
confirmation prompt on this one, so it's worth double-checking the path before running it,
the same way you'd double-check before an `rm` on a file you can't easily recreate.

### Seeing what secrets exist under a path

```bash
vault kv list secret/devices
```

This lists the child paths under `secret/devices/`, without showing any secret values,
useful for getting a sense of what's stored somewhere without needing read access to
each individual secret.

### Guarding against accidental overwrites with check-and-set

By default, nothing stops two people (or two scripts) from writing to the same path at
the same time, with the second write silently winning. If that's a concern for a
particular path, KV v2 supports requiring a check-and-set value on every write:

```bash
# Turn on CAS enforcement for this path
vault kv metadata put -cas-required=true secret/devices/switch-lab-01

# Now writes must say which version they're building on top of
vault kv put -cas=2 secret/devices/switch-lab-01 ssh_password="AnotherPassword789"
```

If someone else has written version 3 in the meantime and you try to write with
`-cas=2`, Vault rejects the write instead of silently clobbering their change. This
isn't turned on for the paths in this environment today, since it adds friction that
isn't needed for a small team, but it's there for a path that needs that protection later.

### A full rotation, from start to finish

Putting the pieces above together, here's a worked example based on a real password
rotation performed for this environment's GitLab LDAP bind account:

```bash
# 1. Confirm the current state before touching anything
vault kv get gitlab/ldap

# 2. Update just the changed field, leaving the rest of the secret alone
vault kv patch gitlab/ldap bind_password="a-new-password"

# 3. Confirm the new version is there
vault kv get -field=bind_password gitlab/ldap

# 4. Tell the consumer (GitLab, in this case) to pick up the new value
#    inject-secrets.sh fetches from Vault and rewrites gitlab.rb
sudo /opt/gitlab/scripts/inject-secrets.sh
sudo docker exec gitlab gitlab-ctl reconfigure

# 5. Verify the whole chain actually works with the new value
sudo docker exec gitlab gitlab-rake gitlab:ldap:check
```

Step 5 matters as much as step 2. Vault confirming the write succeeded only tells you the
secret changed inside Vault, not that whatever depends on it is actually using the new
value correctly.

### Dynamic secrets work differently, on purpose

Everything above is about secrets you write and manage yourself in KV v2. The Database
secrets engine set up in this environment (see "Next Steps with Vault" below) works
differently, and it's worth knowing the difference so you don't go looking for a "delete"
command that isn't relevant there.

```bash
# Ask Vault to generate a fresh, unique, time-limited Postgres login
vault read database/creds/nautobot-readonly
```

Vault creates a brand new database user at that moment, hands you the credentials along
with a lease ID and a TTL, and the user genuinely doesn't exist in Postgres until this
command runs. There's nothing sitting in KV to update or delete. Instead:

```bash
# Extend how long a lease has left, if you're still using it
vault lease renew <lease_id>

# End it early, on purpose, e.g. if a pod using it looks compromised
vault lease revoke <lease_id>
```

Revoking a lease early is the dynamic-secrets equivalent of `destroy` above: once it's
gone, the underlying Postgres user Vault created is also gone, immediately.

### Enabling, moving, and retiring a whole secrets engine

Occasionally the unit of change isn't a single secret but the entire mount. These commands
already appeared earlier in this module for setting `gitlab/` up in the first place, and
they're included here for completeness alongside the rest of the lifecycle:

```bash
# Enable a new KV v2 mount at a custom path
vault secrets enable -path=myapp kv-v2

# Move an existing mount to a new path (secrets and history come with it)
vault secrets move gitlab gitlab-old

# Retire a mount entirely, this removes every secret and all history under it
vault secrets disable gitlab-old
```

`disable` on a mount is the broadest of everything covered in this section. It's the
mount-level equivalent of `vault kv metadata delete`, just applied to every path in that
mount at once rather than one path at a time.

### Quick reference

| What you want to do | Command |
|---|---|
| Write a new secret, replacing anything already there | `vault kv put <path> key=value` |
| Update just some fields, leaving the rest alone | `vault kv patch <path> key=value` |
| Read the current version | `vault kv get <path>` |
| Read a specific field only | `vault kv get -field=<key> <path>` |
| Read an older version | `vault kv get -version=<n> <path>` |
| See version history without reading data | `vault kv metadata get <path>` |
| Copy an old version forward as the new current one | `vault kv rollback -version=<n> <path>` |
| Soft-delete (recoverable) | `vault kv delete <path>` |
| Undo a soft delete | `vault kv undelete -versions=<n> <path>` |
| Permanently erase specific version data | `vault kv destroy -versions=<n> <path>` |
| Remove a path entirely, all versions and metadata | `vault kv metadata delete <path>` |
| List secrets under a path | `vault kv list <path>` |
| Require version-aware writes on a path | `vault kv metadata put -cas-required=true <path>` |
| Issue a fresh dynamic credential | `vault read database/creds/<role>` |
| Extend a dynamic credential's lease | `vault lease renew <lease_id>` |
| End a dynamic credential early | `vault lease revoke <lease_id>` |
| Add a new secrets engine mount | `vault secrets enable -path=<name> <engine>` |
| Relocate a mount | `vault secrets move <old-path> <new-path>` |
| Remove an entire mount and everything in it | `vault secrets disable <path>` |

---

## Regenerating GitLab's AppRole Credentials

Store `role_id` and `secret_id` in a password manager, not written to disk outside of
`/opt/gitlab/vault/.env` on the GitLab host itself (see the GitLab module). A new
`secret_id` can be generated at any time, without needing to touch `role_id` or the
policy it's tied to:

```bash
vault write -field=secret_id -f auth/approle/role/gitlab/secret-id
```

Update `/opt/gitlab/vault/.env` with the new value afterward, so `inject-secrets.sh`
authenticates with it on the next run.

---

**Next: [Module 05 — SkilzNetObserv](../05-netobserv/NETOBSERV-SOLUTION-OVERVIEW.md)**
