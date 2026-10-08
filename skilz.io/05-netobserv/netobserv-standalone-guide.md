# Module 05: SkilzNetObserv Standalone Guide

This guide covers the standalone Docker image. The Kubernetes deployment of the same application is
documented in [netobserv-kubernetes-guide.md](netobserv-kubernetes-guide.md). The
[README](README.md) in this folder explains which of the two guides to follow.

---


A browser-based packet analyser that decodes live traffic on real network device interfaces,
using ERSPAN mirror sessions or SSH-driven captures depending on the device. It reads its device
inventory from Nautobot and shows a live topology view, per-device interface state, and a
Wireshark-style capture window, all from one container. No Kubernetes required.

This guide covers setting it up from nothing: pulling the image, first boot, connecting Nautobot
and (optionally) a directory, and the day-to-day settings worth knowing about.

## Scope of this version

| Platform | Capture method | Per-interface capture | Traffic Trace | Status in this version |
|---|---|---|---|---|
| Cisco IOS and IOS-XE | ERSPAN (GRE) to the collector | Yes | Yes | Tested on lab devices |
| Cisco FTD (managed by FMC) | LINA `capture` command over SSH | Yes | Yes | Tested against an Active/Standby HA pair |
| Cisco NX-OS | ERSPAN (GRE) to the collector | Not offered yet | Yes | Trace is unverified. The device accepts the session, but delivery of mirrored traffic has not been confirmed end to end |

Cisco IOS-XE, Cisco NX-OS and Cisco FTD are the platforms in scope for this version. Devices from other
vendors are outside it. A device with no mapped vendor is skipped when the inventory loads, with a
line in the log, and no capture is offered for it. Support for another platform means adding a new
vendor helper to the source (see the customising section of the standalone guide).

## Before first use

The code has been reviewed and tested, and it follows common security practice: credentials are
stored encrypted or in Vault, access is role-based, and the device account can be limited to the
commands it needs (see Device account and command authorisation). Try it first in an isolated
environment, such as a lab, and confirm that it behaves the way you want before using it elsewhere.

## Design rationale

The image follows the same division of information as the Kubernetes deployment. Nautobot supplies
the device inventory, so the topology always matches the source of truth. Device credentials are
held in one place, the local credential store or Vault. Everything the application writes sits in
one folder, `/app/data`, so a backup or a move to another host is a copy of that folder. Accounts
are local by default so the tool runs with no directory, and a directory can be connected later from
the Settings page. The container needs no configuration file to start, which keeps the first run to
a single `docker run` command.

## 1. Prerequisites

- A Docker host that can reach the network devices you want to observe, and reach your Nautobot
  instance's API.
- A Nautobot instance. You don't need an API token ready before you start the container. You can
  connect Nautobot from the Settings page of the app after first boot.
- Devices tagged with two Nautobot custom fields so the app knows how to talk to each one:
  - `netobserv_vendor`: one of `cisco-xe` (IOS and IOS-XE) or `cisco-nxos`
  - `netobserv_capture_mode`: `mirror` (default if not set), or `ftd-ssh` for Cisco Secure
    Firewall Threat Defense (FTD). An FTD device still needs a vendor value so it passes inventory
    mapping, and `cisco-xe` is the one to use; `ftd-ssh` is what selects the FTD behaviour.
    A device with any other mode (for example `none`) still appears in the topology but has no
    capture.
  - Section 7.1 spells out exactly what each combination does on the device.
- Nautobot interface, IP address and Cable records for every device you want to see. The topology
  view is built from Nautobot alone: each device's interfaces and IPs come from its Nautobot
  interface and IP records, and the links drawn between devices come from Nautobot Cables, with
  /30-/31 subnet inference only filling in pairs that have no Cable. Nothing in the topology
  view SSHes to a device. Use the full interface names the device itself uses
  (`GigabitEthernet1/0/1`, `Ethernet1/6`), because the Capture button sends that name to the
  device unchanged.
- Optionally, an LDAP-compatible directory (Active Directory or similar) if you want directory
  login. It is optional. Local accounts are enough to run the app, and a directory can be added
  later without losing anything.

## 2. Quick start

```bash
docker pull openskilz/skilznetobserv:latest

docker run -d \
  --name skilznetobserv \
  -p 3000:3000 \
  -e SESSION_SECRET=$(openssl rand -hex 32) \
  -e COOKIE_SECURE=false \
  -v $(pwd)/skilznetobserv-data:/app/data \
  openskilz/skilznetobserv:latest
```

Open `http://<your-docker-host>:3000`. That's the whole quick start, everything else (Nautobot,
directory, device credentials) is configured from inside the app on first boot. Continue with
Section 3 below.

To confirm the container is serving before opening a browser, call the web page and the setup
status endpoint. Neither needs a login:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://<your-docker-host>:3000/
curl -s http://<your-docker-host>:3000/api/setup/status
```

```
HTTP 200
{"needsSetup":true}
```

`HTTP 200` shows the app is up. `{"needsSetup":true}` shows it started cleanly, and with no accounts
and no directory configured yet it offers the first-run setup wizard (Section 3). After the wizard is
complete the second command returns `{"needsSetup":false}`.

The container log (`docker logs skilznetobserv`) ends with `SkilzNetObserv listening on port 3000`.
A line reading `Collector start failed: spawn EPERM` above it is expected with this quick start
command. It only means the packet collector needs the extra capture flags in Section 7, and the
rest of the app works without them.

A couple of things worth understanding about that command before you run it against anything real:

- `-v $(pwd)/skilznetobserv-data:/app/data` is what makes the account you're about to create, and
  everything you configure afterward, survive a container restart. Skip it and you'll be back at
  the setup wizard every time the container restarts. Point it at a volume or host path you trust,
  since device credentials live there too (see Section 6).
- `COOKIE_SECURE=false` is only for testing over plain HTTP, like the example above. If you're
  putting this behind a reverse proxy or load balancer that terminates TLS, drop that line
  entirely (it defaults to `true`). If you leave it at the default over plain HTTP, your browser
  will refuse to send the session cookie back and login will look broken with no error
  message anywhere, so this is the first thing to check if login seems to do nothing.
- `SESSION_SECRET` should be set explicitly (as above) so existing logins survive a restart. If
  you don't set it, one is generated at every container start and every session is invalidated.

If you want to actually run captures (any device, any mode), a couple more flags are needed. See
Section 7 (Networking and capture modes) before you rely on that.

## 3. First boot: the setup wizard

A fresh container, with nothing configured yet, shows a one-time setup screen in place of the login
form the first time you open it.

1. Open `http://<your-docker-host>:3000`. You'll see "First-time setup" where the login form would normally be.
2. Create a local administrator account: a username and a password (at least 8 characters).
3. Submitting the form signs you in immediately, as that admin, and drops you straight onto
   Settings → Integrations. This is the one moment the app lets you in without any credentials at
   all, and only because nothing else exists yet to authenticate against. As soon as this account
   is created, the setup screen is gone for good (a second attempt to create it returns an error).
4. From Integrations, connect Nautobot: its URL and an API token (Section 4).
5. Optionally, connect a directory in the same page (Section 5). This step is optional.
   The admin account you just created is enough to run the app.
6. Optionally, apply device credentials so interface data and captures work (Section 6).

Nothing here needs a container restart. To skip the wizard and configure
everything as environment variables up front (the usual approach in a
pipeline), see Section 10: setting `LDAP_URL`/`LDAP_BASE` at container start skips the wizard
completely and takes you straight to a normal login screen, matching how a config-as-code
deployment of this app behaves.

Settings → General also has a Dark/Light theme switch, personal to your browser, and a timezone
setting, which is server-side and shared across everyone using this instance.

## 4. Connecting Nautobot

From Settings → Integrations, under Nautobot:

1. Enter your Nautobot instance's URL, e.g. `https://nautobot.example.com`.
2. Enter an API token with read access to devices, interfaces, IP addresses, and cables (and
   write access to custom fields, if you haven't already tagged your devices with
   `netobserv_vendor`/`netobserv_capture_mode`).
3. Save. The device list refreshes immediately, no restart needed.

**Be patient on the first topology load.** The topology is assembled from several Nautobot API
calls (devices, interfaces, IP addresses, IP-to-interface assignments and cables), and Nautobot
answers large interface listings slowly. How long it takes depends on how many interfaces you
have and how fast your Nautobot is. A small lab Nautobot on its built-in development server
took about 13 seconds for 11 devices and 500 interfaces, and a production Nautobot with
thousands of interfaces can take a few minutes. Leave the page open until the topology appears.
Only the first load waits this long: the app refreshes in the background every 30 seconds, and
a refresh that is still running is not started a second time.

If this instance was started with `NAUTOBOT_URL`/`NAUTOBOT_TOKEN` set as environment variables,
this page shows what is already configured, the fields are read-only, and Save is disabled. The
environment variable is applied on every restart whatever is saved here, so the value to change
is the environment variable. Section 10 describes the priority order.

## 5. Connecting a directory (AD/LDAP)

Also from Settings → Integrations, underneath Nautobot:

1. Enter your directory's LDAP URL, e.g. `ldap://dc.example.com`.
2. Enter the search base (e.g. `dc=example,dc=com`), a bind account DN, and its password.
3. Enter the comma-separated AD group names allowed to log in, and, separately, the groups
   allowed to administer the app (manage device credentials, users, and these same settings).
4. Save.

Once saved, the login screen shows a "Directory" tab alongside "Local", and directory accounts in
an allowed group can sign in from there. Local and directory accounts work side by side; connecting a directory doesn't disable or replace the local admin account you
created in the wizard.

## 6. Device credentials

The app needs SSH credentials for each device only to start captures and traces on it. The
topology view and the interface lists do not use them; those come from Nautobot. There are two
ways to set credentials, and they combine:

- **Generic**: from Settings → Integrations, under "Generic credentials", enter one
  username/password used by any device with no credentials set for that device. Set this once for
  environments where every device shares the same account.
- **Per device**: right-click a device in the topology view and choose "Set Credentials…". This
  overrides the generic credential for just that device. The FTD/ASA creds modal and the trace
  creds modal both leave their fields blank by default, meaning "use whatever is saved". Fill
  them in only to use a different account for that one capture or trace.

Renaming a device in Nautobot doesn't orphan either setting: the local file store keys each
device entry by its stable Nautobot ID, so credentials saved before a rename
keep resolving after it.

Where credentials actually live depends on what's configured:

- **Local file (default)**: if Vault isn't configured (see below), credentials are kept in a JSON
  file inside the container, at `/app/data/credentials.json` by default. This has no external
  dependency, which is why it's the default, but it's plaintext on disk, so make sure the volume
  you mounted at `/app/data` is somewhere you trust.
- **HashiCorp Vault**: set `VAULT_ADDR`, `VAULT_ROLE_ID`, and `VAULT_SECRET_ID` as environment
  variables and the app uses Vault AppRole auth, storing each device's credentials at
  `secret/data/devices/<device-name-lowercased>` (the generic credential at
  `secret/data/devices/__generic__`). Use this if Vault is already running and secret
  management (access policies, audit logging, rotation) is wanted for device credentials. The
  Vault path of a device stays name-based, since an administrator may provision it directly at that path; a
  rename there needs the secret moved or re-saved through the app. This one switch is env-var-only,
  not GUI-configurable, since it changes where secrets are trusted to live.

## 7. Networking and capture modes

Every ERSPAN (mirror-mode) capture is decoded by `tshark` inside the container, and `tshark`
needs `CAP_NET_RAW`/`CAP_NET_ADMIN` to run at all, regardless of which mode a given device uses.
Add these to the `docker run` command from Section 2 before relying on capture:

```bash
  --cap-add=NET_RAW \
  --cap-add=NET_ADMIN \
```

Devices tagged `mirror` (the default) need one more thing: the container has to actually receive
ERSPAN/GRE-encapsulated traffic arriving on the Docker host's real network interface, which a
container-private network namespace can't see. Add `--network host` as well for those:

```bash
  --network host \
```

If every device you're using is an FTD (`ftd-ssh`), you can leave `--network host` out. Either way, the
container doesn't need to run as root: it runs as a non-root user by default, and that
user already has the capabilities above through the image setup, so nothing more is
needed.

Cisco FTD devices (mode `ftd-ssh`) are the exception to both of the above: the per-interface
**Capture** button in the device sidebar runs the LINA `capture` command on that interface's
nameif over SSH and shows the parsed packet rows in the capture window, so no `tshark` or ERSPAN
is involved. Details worth knowing:

- The clicked interface has to have a nameif (`show nameif` on the device). A parent interface
  that only carries sub-interfaces, such as `GigabitEthernet0/0`, has none, and the app reports that
  and does not capture.
- The device-side capture matches every IP packet; the filter box is applied in software and
  supports `host`, `src host`, `dst host`, `port`, `src port`, `dst port`, `icmp`, `tcp`, `udp`
  and `ip`, joined with `and`. Any other expression is rejected with a message.
- LINA shows no Ethernet header or payload, so rows carry addresses, protocol and the LINA
  description, and the hex pane stays empty. PCAP export is not meaningful for these rows.
- Packets arrive in batches roughly every 2 seconds, since the capture is read back with
  `show capture`. Stopping the capture, or closing the window, removes it from the device
  (`no capture NETOBSERV_<nameif>`).

Mirror-mode devices also need to be told where to actually send that ERSPAN/GRE traffic: the
reachable IP address of this host. Set it from Settings → Integrations → Capture (no restart
needed), or as `ERSPAN_COLLECTOR_IP` at container start. There's no sensible default across
different deployments, so capture on a mirror-mode device fails with a clear error until this is
set to something.

### 7.1 What a capture does on each device

This section is the full picture of what SkilzNetObserv sends to a device, what the device does
with it, and what is left behind afterwards.
Everything here is also true for the Traffic Trace panel (Section 8), apart from the session
number and the differences noted below.

Every device is reached over SSH with the credentials from Section 6. The account needs to be able
to enter configuration mode (privilege 15 on Cisco IOS-XE and NX-OS, an admin account on FTD).
Nothing is installed on any device, and no agent runs on it.

```
                click Capture on an interface
                              |
        +---------------------+----------------------+
        |                                            |
   mode: mirror                                 mode: ftd-ssh
 (Cisco IOS-XE, NX-OS)                           (Cisco FTD)
        |                                            |
 SSH: configure an                          SSH: start a LINA
 ERSPAN source                              "capture" on the
 session on the                             interface's nameif
 device                                              |
        |                                    poll "show capture"
 device sends GRE/ERSPAN                     every 2 seconds
 to this host                                        |
        |                                            |
 collector decapsulates,                             |
 tshark decodes                                      |
        +---------------------+----------------------+
                              |
                 packet rows in the browser's capture window
                              |
                   click Stop (or close the window)
                              |
        device-side session / capture is removed over SSH
```

**Cisco IOS and IOS-XE (`cisco-xe`, mode `mirror`)**

When you click Capture on an interface, the app opens an SSH session and sends this, in order
(session number 1 by default, `ERSPAN_SESSION_ID`):

```
configure terminal
no monitor session 1
monitor session 1 type erspan-source
description SkilzNetObserv-Auto
source interface <the interface you clicked> both
no shutdown
destination
erspan-id 1
ip address <ERSPAN collector IP, Settings -> Integrations -> Capture>
origin ip address <the device's management IP from Nautobot>
exit
exit
end
write memory
```

An SVI such as `Vlan10` uses `source vlan 10 both`, since IOS-XE does not accept an SVI as a
source interface. The leading `no monitor session 1` removes any earlier session first, so
one interface is mirrored per device at a time, and clicking Capture on a second
interface replaces the first one's session.

From that moment the device copies every frame in both directions on that interface, wraps each in
GRE with an ERSPAN header, and sends it to the collector IP, which is this host. The app's
collector listens for GRE (IP protocol 47) with `tshark`, strips the encapsulation, works out which
device the frames came from using the outer source IP (the `origin ip address` above), and hands
the inner Ethernet frames to the decoder. The mirrored traffic is additional load on the device's
CPU and on the path to the collector, so pick the interface you actually want to look at.

When you click Stop, or close the capture window, the app opens SSH again and sends:

```
configure terminal
no monitor session 1
end
write memory
```

`write memory` is part of both the start and the stop, so the device's saved configuration is
written each time. If the container is killed before it can send the stop, the session stays on
the device, and `no monitor session 1` removes it by hand.

**Cisco NX-OS (`cisco-nxos`, mode `mirror`)**

NX-OS is available in Traffic Trace (Section 8), with the caveat on its verification status in that
section. The per-interface Capture button is not offered for NX-OS yet. A trace sends, using
session 2 by default (`ERSPAN_TRACE_SESSION_ID`):

```
configure terminal
monitor erspan origin ip-address <device management IP> global
no monitor session 2
monitor session 2 type erspan-source
source interface <each interface in the trace> both
destination ip <ERSPAN collector IP>
erspan-id 2
vrf default
no shut
exit
end
```

and on stop, `no monitor session 2`. The global `monitor erspan origin ip-address` line stays
on the device after a stop. It is harmless and is reused by the next trace.

**Cisco FTD (`ftd-ssh`)**

FTD is managed by FMC, and a policy deploy from FMC would overwrite an ERSPAN session. A
persistent session is therefore not used. The app logs in to the LINA CLI (the admin login opens
there directly) and uses the LINA `capture` command, a diagnostic command that sits outside
the FMC-pushed configuration.

When you click Capture on an interface:

```
show nameif                                   find the nameif for the interface you clicked
no capture NETOBSERV_<nameif>                 remove a leftover from an earlier run, if any
capture NETOBSERV_<nameif> interface <nameif> match ip any any
```

While the capture runs, every 2 seconds the app sends `show capture NETOBSERV_<nameif>`, parses
the packet lines, and then sends `clear capture NETOBSERV_<nameif>` so the buffer on the device
does not fill up. Nothing is mirrored to this host, so the collector and `tshark` are not involved,
and traffic does not leave the firewall. The capture is kept in device memory only.

When you click Stop or close the window, the app sends `no capture NETOBSERV_<nameif>` and closes
the SSH session. Run `show capture` on the device afterwards to confirm nothing is left; a
leftover `NETOBSERV_*` entry (for example after a container crash) can be removed with
`no capture NETOBSERV_<nameif>`.

Things specific to FTD:

- The interface has to have a nameif. A parent interface that only carries sub-interfaces
  (`GigabitEthernet0/0` on a trunk) has none, and the app reports that and does not capture. Click one of
  its sub-interfaces.
- The device-side capture is `match ip any any`. The filter box is applied in this app (Section 7
  lists the supported words), so a filter does not reduce what the firewall captures.
- On an HA pair, capture on the unit you want to look at. Traffic Trace starts one on each unit,
  and only the unit that is forwarding traffic produces hits.

**What the device account and network need**

- SSH reachability from the container to each device's management IP. By default an unknown host
  key is accepted and a warning with its fingerprint is written to the container log. Set
  `NETOBSERV_STRICT_HOST_KEYS=true` and list fingerprints in a `known_hosts.json` file
  (`{"10.1.1.1": "SHA256:..."}`, one entry per management IP) mounted into the container with
  `-v $PWD/known_hosts.json:/app/known_hosts.json:ro` to refuse any host that is not listed. The
  image ships that file empty.
- For ERSPAN devices, the container's host must receive GRE (IP protocol 47) from the devices,
  which is why `--network host` and the capture capabilities in the section above are needed.
- Credentials from Section 6 with permission to run the commands listed above.

### 7.2 Device account and command authorisation

SkilzNetObserv logs in to each device it captures from. Create a dedicated account for the
application and limit it to the commands listed in this section. They cover the ERSPAN session on
IOS-XE and NX-OS, the LINA capture on FTD, and one `show` command on NX-OS. The limit is applied by
the device administration service, for example a TACACS+ server, a RADIUS server such as Cisco ISE,
or local privilege levels where no central service exists. The application holds the credential
(Section 6) and the administration service decides what that credential may do.

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
interfaces and cables come from Nautobot. Live gateway state (Section 8.1) is read with SNMPv3 and
uses a separate read-only SNMP user, which is also kept out of the SSH account.

**Checking the policy**

Before relying on a restricted account, run one capture and one trace on each platform in a lab and
confirm that every command in the tables is accepted.

## 8. Traffic Trace

The collapsible "Traffic Trace" panel below the topology view lights up, in real time, the links that matching
traffic crosses. Full packet detail on one device at a
time comes from the per-device capture window described in Section 7.

1. Add one or more filter rules: source IP, destination IP, protocol, and port, any of which can
   be left blank as a wildcard, each with a highlight colour.
2. Optionally expand **Devices** to choose which devices to configure for the trace. The list
   comes straight from Nautobot and includes every device NetObserv can capture on, switches and
   FTDs alike. Every device is selected by default; narrow it down if you already know which devices sit
   in the traffic's path, or to keep the config changes on production devices minimal.
3. Click **Start Trace**. Both credential fields are optional: leave them blank to use each
   the credentials saved for each device (Section 6), or type a
   username/password to use for just this trace, held in server memory only, cleared on
   stop and not written to disk. The FTD field is for Cisco FTD devices, which use a different
   capture mechanism from ERSPAN (see below). The nameif to capture on is auto-detected from the
   IPs in the filter rules; only set the override field if auto-detection picks the wrong one.
4. Matching traffic flashes the link it crosses, in that rule's colour, for a couple of seconds
   per hit.

A few things worth knowing:

- This needs an ERSPAN collector IP set (Section 7) before it can start on any ERSPAN-capable
  device, for the same reason per-device mirror-mode capture does.
- If the exact link a hit crossed can't be identified from the IP addresses in the packet (e.g.
  transit traffic between two end hosts, neither of which is a device-owned IP), every link on
  the reporting device flashes, so there is always a visible result.
- A running trace survives a browser refresh (it re-attaches automatically) and survives a second
  browser tab being open at the same time; only closing the last connected session stops it.
- Vendor support: Cisco IOS-XE and Cisco NX-OS use ERSPAN (different config syntax, handled
  automatically per device). Cisco FTD uses the LINA `capture` command directly on a real
  nameif, because an FMC-managed FTD cannot keep a persistent ERSPAN session (FMC would
  overwrite it on the next policy deploy). `capture` is a diagnostic command outside the
  persistent configuration, so FMC leaves it alone. Every FTD device in a trace gets an independent
  capture whichever HA unit is currently active, so an Active/Standby failover is
  followed with no reconfiguration: the unit that is forwarding
  traffic is the one that produces hits.
- NX-OS status: the ERSPAN configuration above is the documented NX-OS syntax and the device's CLI
  accepts it, but delivery of mirrored traffic has not been verified end to end. On the virtual
  Nexus 9000v used in testing the session is accepted and then reports `state: error` with zero
  packets sent, whether the source is a switchport or a routed interface, so a trace cannot work on
  that platform. After configuring a Nexus, the app runs `show monitor session` and, if the state
  is `error`, removes the session and lists that device as failed in the trace's device list with
  the reason. Physical Nexus hardware has not been tested. Treat NX-OS trace as unverified.

### 8.1 Tracking traffic across a gateway failover

A trace hit's source MAC identifies the previous L2 hop, resolved to a device name from the
interface records in your source of truth. Under HSRP, VRRP, or a Cisco FTD/ASA Active/Standby HA
pair, that resolution fails: HSRP and VRRP forward traffic using a shared virtual MAC that does not
appear in any interface record, and the new active unit in an FTD/ASA HA pair inherits the
physical MAC of the previous active unit, so the static record then points at the wrong device.
Without further information, a trace would flash the wrong link, or none, the moment a failover happens.

With `SNMP_USER`/`SNMP_AUTH_PASS`/`SNMP_PRIV_PASS` set, SkilzNetObserv queries each device's live
HSRP/VRRP/FTD-HA state (SNMPv3 authPriv) on every topology refresh and again every 30 seconds in
the background, building a second MAC-to-device map that is checked before the static one. A
failover is detected and every open browser tab updated automatically within 30 seconds, no manual
refresh needed. When these are not set, this step is skipped and trace uses the static map alone.

Coverage: HSRP v1/v2, VRRP v2/v3, and Cisco FTD/ASA Active/Standby HA for traffic observed from
other, ERSPAN-capturing devices. Trace hits from an FTD/ASA do not need any of this: LINA capture
runs independently on every unit in the pair, so the correct active unit is identified directly
from Section 8's capture mechanism, with no SNMP polling involved. GLBP is not yet covered. The
SNMP user needs read access to `CISCO-FIREWALL-MIB` (for FTD/ASA) and `CISCO-HSRP-MIB` (for HSRP)
on the devices it queries.

## 9. Accounts

Two account types work side by side:

- **Local accounts**: created from Settings → Users (or the setup wizard, for the first one).
  Sign in from the "Local" tab on the login screen. Useful when no directory is in use, or when an account should not depend on one.
- **Directory (AD/LDAP) accounts**: sign in from the "Directory" tab, subject to the allowed and
  admin group settings from Section 5.

Both account types carry the same admin/non-admin distinction. Admins can manage device
credentials, local users, and Integrations settings from Settings. Non-admins can view the
topology and run the captures they are permitted to, and the Settings item is hidden from their
user menu.

## 10. Config-as-code

Everything in Sections 4 and 5 can also be set as environment variables at container start, for
deployments driven by a pipeline, with no GUI steps on a fresh container:

```bash
docker run -d \
  --name skilznetobserv \
  -p 3000:3000 \
  -e NAUTOBOT_URL=https://nautobot.example.com \
  -e NAUTOBOT_TOKEN=your-nautobot-api-token \
  -e LDAP_URL=ldap://your-dc.example.com \
  -e LDAP_BASE=dc=example,dc=com \
  -e LDAP_BIND_DN='CN=service-account,CN=Users,DC=example,DC=com' \
  -e LDAP_BIND_PASS=your-bind-password \
  -e LDAP_ALLOWED_GROUPS=netobserv-admins,netobserv-users \
  -e SESSION_SECRET=$(openssl rand -hex 32) \
  -e COOKIE_SECURE=false \
  -v $(pwd)/skilznetobserv-data:/app/data \
  openskilz/skilznetobserv:latest
```

An environment variable takes priority over anything saved through the GUI, and the GUI fills in only what
an environment variable did not already provide. A container started this way skips the setup wizard,
since an LDAP config from the environment satisfies it, and the corresponding
Settings fields become read-only (Nautobot, Directory, and Capture each lock independently, based
on which of them has an environment variable set). A GUI save would be overwritten on the next
restart, so the fields stay locked. Config saved through the GUI is safe to
leave in place if a deployment later moves to environment variables: the environment variable
wins and nothing needs to be cleaned up first.

## 11. Updating to a new version

```bash
docker pull openskilz/skilznetobserv:latest
docker stop skilznetobserv && docker rm skilznetobserv
docker run -d \
  --name skilznetobserv \
  -p 3000:3000 \
  -e SESSION_SECRET=$(openssl rand -hex 32) \
  -e COOKIE_SECURE=false \
  -v $(pwd)/skilznetobserv-data:/app/data \
  openskilz/skilznetobserv:latest
```

As long as you reuse the same volume, local accounts, device credentials, and any GUI-saved
Nautobot/LDAP settings survive the update.

## 12. Using a different source of truth

Out of the box, this image only knows how to read device inventory from Nautobot. That is the
only integration built into the published image, and there is no GUI switch for it. Pointing it
at a CMDB, a flat inventory file or another source of truth is a source change: it means
building a custom image on top of this one.

The integration point is small and self-contained on purpose, so it is a realistic first customisation. `devices.js` only needs three things out of whatever you plug in:

1. A device list: name, primary management IP, and a vendor identifier the app can map to one of
   its supported vendors (`cisco-xe`, `cisco-nxos`).
2. Somewhere to read/write two per-device settings, a vendor override and a capture mode, or an
   equivalent lookup table kept elsewhere.
3. Optional periodic refresh, since device inventories change over time.

Write a module with the same exported functions this file already has (`getDevice`,
`getAllDevices`, `getCredentials`, `setCredentials`, `getCredentialStatusMap`, `startRefresh`,
`refreshDevices`) that maps your system's device model onto the internal shape the rest of the
app already expects: `{name, vendor, mode, mgmt_ip, vault_path}`, plus five optional display-only
fields the device sidebar shows when present (`manufacturer`, `model`, `serial`, `site`, `role`);
leave any of those `null` if your source doesn't have it, the sidebar just skips that row. The
credential functions (`getCredentials`/`setCredentials`/`getCredentialStatusMap`) don't need to
change at all when only the device inventory source is being replaced. Leave
them as they are and rewrite the inventory-fetching half.

A minimal skeleton for a flat JSON file as the source of truth, as a starting point to adapt:

```js
// my-source.js: replaces devices.js's Nautobot calls with a local file
const fs = require('fs');

let _devices = [];

function _load() {
  // Expects a JSON array of { name, vendor, mode, mgmt_ip } objects, plus any
  // of the optional manufacturer/model/serial/site/role fields you have
  const raw = JSON.parse(fs.readFileSync(process.env.DEVICE_INVENTORY_FILE, 'utf8'));
  _devices = raw.map((d) => ({ ...d, vault_path: `devices/${d.name.toLowerCase()}` }));
}

function getDevice(name) { return _devices.find((d) => d.name === name) || null; }
function getAllDevices() { return _devices; }
function startRefresh() { _load(); setInterval(_load, 60000).unref(); }
async function refreshDevices() { _load(); }

// getCredentials / setCredentials / getCredentialStatusMap: copy verbatim from devices.js,
// they don't depend on where the device list came from.

module.exports = { getDevice, getAllDevices, startRefresh, refreshDevices, /* ...credential fns */ };
```

Then swap the `require('./devices')` calls in `server.js` and `topology.js` for your new module,
and rebuild the image. This is an advanced, source-level path for people comfortable maintaining
a custom build of the image. The published image and its GUI do not expose it, and nothing in
Sections 3 through 8 depends on it. A change of general use is a reasonable candidate for a pull
request.

## 13. Bring-your-own code

The application code is part of the image. A host running the container has no source files for it,
only the data folder mounted at `/app/data`, which holds accounts, settings and device credentials.
The code itself sits inside the image at `/app`:

| Path in the image | Contents |
|---|---|
| `/app/server.js` | Web server, login, API routes and WebSocket handling |
| `/app/topology.js` | Topology build from Nautobot interfaces, IPs and Cables |
| `/app/devices.js` | Device inventory and credential store (the Nautobot integration point, see Section 12) |
| `/app/tracer.js`, `/app/capture.js`, `/app/collector.js`, `/app/decoder.js` | Traffic Trace and packet capture pipeline |
| `/app/lib/`, `/app/mirror/` | Vendor helpers and ERSPAN mirror session handling |
| `/app/public/` | Browser front end (HTML, CSS and JavaScript) |
| `/app/Dockerfile`, `/app/package.json` | Build definition and dependencies, copied into the image alongside the code |

There are three ways to customise it. Option A is the recommended one.

### Option A: build your own image

Get a copy of the source by copying the files out of a running container:

```bash
mkdir mynetobserv && cd mynetobserv
for f in server.js topology.js devices.js tracer.js capture.js collector.js decoder.js \
         config.js local-users.js api-tokens.js known_hosts.json package.json \
         package-lock.json Dockerfile README.md; do
  docker cp skilznetobserv:/app/$f ./$f
done
docker cp skilznetobserv:/app/lib ./lib
docker cp skilznetobserv:/app/mirror ./mirror
docker cp skilznetobserv:/app/public ./public
```

Copy individual paths as above and leave out `/app/node_modules`. Its native modules are compiled for
the Alpine base image and the build recompiles them.

Edit the files, then build and run the new image, reusing the same data folder so existing accounts
and settings carry over:

```bash
docker build -t mynetobserv:dev .

docker stop skilznetobserv && docker rm skilznetobserv
docker run -d \
  --name skilznetobserv \
  -p 3000:3000 \
  -e SESSION_SECRET=$(openssl rand -hex 32) \
  -e COOKIE_SECURE=false \
  -v $(pwd)/skilznetobserv-data:/app/data \
  mynetobserv:dev
```

Keep the `Dockerfile` as it is unless there is a reason to change it. It installs `tshark` and applies
the file capabilities that let the non-root `node` user run captures, and removing those steps stops
capture working. Section 7 lists the extra `docker run` flags captures need, and they apply to a
custom image in the same way.

### Option B: edit files in place with a bind mount

For a quick edit loop with no rebuild, mount an edited file over the one in the image:

```bash
docker run -d \
  --name skilznetobserv \
  -p 3000:3000 \
  -e SESSION_SECRET=$(openssl rand -hex 32) \
  -e COOKIE_SECURE=false \
  -v $(pwd)/skilznetobserv-data:/app/data \
  -v $(pwd)/public/style.css:/app/public/style.css:ro \
  openskilz/skilznetobserv:latest
```

Front-end files under `/app/public/` are served as static files, so a browser refresh shows the
change. Server-side files such as `topology.js` are read once at start, so restart the container
(`docker restart skilznetobserv`) after each edit. The mounted file needs to be readable by uid 1000,
the `node` user inside the container (`chmod a+r` on the file is enough).

### Keeping a customised image up to date

A custom image does not receive changes from the published one. To pick up a new release, copy the new
source over a clean checkout, re-apply the customisations (a version control diff of the files changed
makes this quick), rebuild and restart with the same data folder. Section 11 covers updating the
published image.

## 14. Troubleshooting

- **Login form looks broken, nothing happens on submit, no error shown**: almost always
  `COOKIE_SECURE` set to `true` (the default) on a deployment reached over plain HTTP. Set it to
  `false` for plain-HTTP testing, or put a TLS-terminating proxy in front of the app and
  leave it unset.
- **Page loads with no styling, blank white/unstyled page**: same root cause as above. The
  browser tries to upgrade the CSS/JS requests to HTTPS and there's nothing there to answer.
- **Topology is blank or still spinning after Nautobot is connected**: wait for it. The first
  load pulls every interface, IP address and cable from Nautobot, which takes seconds on a small
  lab and a few minutes on a large inventory (see Section 4).
- **Devices show only a handful of interfaces, far fewer than the real device has**: the app
  shows exactly what Nautobot holds, and devices created by hand usually only have the few
  interfaces someone typed in. Populate Nautobot with the full interface lists (Nautobot's
  Device Onboarding app, or a script that reads each device and writes its interfaces and IPs
  to Nautobot's API). No change to this app is needed.
- **A device shows "No interfaces recorded in Nautobot for this device"**: Nautobot has no interface records for that
  device. Add or import them in Nautobot; the app reads them on its next refresh (every 30
  seconds).
- **No links drawn between devices you know are connected**: there's no Nautobot Cable record
  between them. Create the Cable record in Nautobot.
- **Setup wizard doesn't appear on a fresh container**: it's gated on there being zero local users
  *and* no LDAP config from either the environment or a previous GUI save. If you're reusing a
  data volume from an earlier run, delete it (or mount an empty one) to get a genuinely fresh
  first boot.
- **Capture fails immediately with a "Capture error" toast mentioning `spawn tshark` or
  `dumpcap`**: the container wasn't started with `--cap-add=NET_RAW --cap-add=NET_ADMIN`. See
  Section 7. This applies to FTD devices as well as mirror-mode devices.
- **Traffic Trace starts but nothing flashes**: check that an ERSPAN collector IP is set
  (Section 7); a trace can start successfully and still have nowhere to send mirrored traffic if
  that's missing. If it's set and traffic still isn't flashing, confirm the two devices you
  expect to see a link between actually have a Nautobot Cable record between them (Section 1's
  Cable note). Traffic Trace can only flash a link that
  exists in the topology view in the first place.
- **A device shows "SSH failed" in the trace's device list**: its ERSPAN session, or FTD capture,
  couldn't be configured with the credentials given. Check the credentials, and that the device
  is actually Cisco IOS-XE, NX-OS, or FTD.
- **An FTD device fails with "could not work out which interface to capture on"**: none of the
  trace's filter rules' IPs matched a subnet on any of that device's nameifs, and no nameif
  override was given. Either add a filter rule with a real srcIp/dstIp from the traffic of that
  FTD, or set the override field to the nameif directly (check with `show nameif` on the
  device if unsure which one).
- **Login returns success (200, a real user object) but every request right after it comes back
  401/unauthenticated**: the reverse of the COOKIE_SECURE issue above. The app is running behind a
  TLS-terminating reverse proxy or ingress, but `TRUST_PROXY` is still unset. Without it, the
  app does not trust the `X-Forwarded-Proto: https` header from the proxy, so it does not consider the
  connection secure, and since the session cookie is `secure: true` by default, the cookie is
  not sent to the browser at all (this is the `shouldSetCookie()` behaviour of `express-session`).
  Set `TRUST_PROXY=true`. Typical with a Kubernetes ingress that
  terminates TLS and forwards plain HTTP to the pod.
- **Topology shows the wrong devices, or a Nautobot connection you configured from Settings
  seems to have reverted, after updating the container**: `NAUTOBOT_URL`/`NAUTOBOT_TOKEN` (and
  the LDAP/`ERSPAN_COLLECTOR_IP` equivalents) take priority over a value saved in Settings. The environment
  variable is re-applied on every process start, so if one is set anywhere in a compose file or
  manifest (including an old one that was forgotten), it overrides the value last saved in
  the GUI. Settings refuses to save over a value set by an environment variable (409 error, fields
  greyed out), and the value to change is the environment variable.

## 15. Environment variable reference

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `NAUTOBOT_URL` | no | none | Base URL of your Nautobot instance |
| `NAUTOBOT_TOKEN` | no | none | Nautobot API token |
| `NAUTOBOT_REFRESH_INTERVAL` | no | `120000` | Device list refresh interval, in ms |
| `ERSPAN_COLLECTOR_IP` | no | none | This host's IP, for mirror-mode devices to send ERSPAN/GRE traffic to. No default; mirror-mode capture fails clearly until this is set (env var or Settings → Integrations → Capture) |
| `ERSPAN_SESSION_ID` | no | `1` | ERSPAN session number configured on mirror-mode Cisco devices (per-device capture) |
| `ERSPAN_TRACE_SESSION_ID` | no | `2` | ERSPAN session number used for Traffic Trace (Section 8), kept separate from per-device capture so the two do not collide |
| `LDAP_URL` | no | none | Your directory's LDAP URL |
| `LDAP_BASE` | no | none | Search base, e.g. `dc=example,dc=com` |
| `LDAP_BIND_DN` | no | none | Bind account DN used for the initial LDAP search |
| `LDAP_BIND_PASS` | no | none | Bind account password |
| `LDAP_ALLOWED_GROUPS` | no | `netobserv-admins,netobserv-users` | Comma-separated group CNs allowed to log in |
| `LDAP_ADMIN_GROUPS` | no | `Domain Admins` | Comma-separated group CNs allowed to manage device credentials and settings |
| `LDAP_TLS_CA_FILE` | no | none | CA bundle path, for `ldaps://` |
| `VAULT_ADDR` / `VAULT_ROLE_ID` / `VAULT_SECRET_ID` | no | none | Enables Vault-backed device credentials; omit all three to use the local file store |
| `CREDENTIALS_FILE` | no | `/app/data/credentials.json` | Local device credential store path, only used when Vault isn't configured |
| `LOCAL_USERS_FILE` | no | `/app/data/local-users.json` | Local account store path |
| `INTEGRATIONS_CONFIG_FILE` | no | `/app/data/integrations-config.json` | GUI-saved Nautobot/LDAP/general settings path |
| `NETOBSERV_STRICT_HOST_KEYS` | no | `false` | Set to `true` to refuse SSH connections to any device whose host key fingerprint is not listed in `known_hosts.json`. Left off, unknown hosts are accepted and a warning with the fingerprint is logged |
| `SESSION_SECRET` | recommended | random per-start | Session signing key. Set this explicitly so logins survive a container restart |
| `COOKIE_SECURE` | no | `true` | Set to `false` only when testing over plain HTTP |
| `TRUST_PROXY` | no | `false` | Set to `true` only if a reverse proxy or ingress sits in front of this container and overwrites `X-Forwarded-For` itself. Leaving this off when there is no proxy keeps IP-based rate limiting (login attempts, general API abuse) reliable. Turning it on with no proxy in place lets a client set the address it appears from and bypass those limits |
| `PORT` | no | `3000` | Port the app listens on inside the container |
| `SNMP_USER` / `SNMP_AUTH_PASS` / `SNMP_PRIV_PASS` | no | none | SNMPv3 authPriv credentials for gateway-redundancy detection (Section 8). When unset, redundancy queries are skipped and Traffic Trace uses the static map alone |
