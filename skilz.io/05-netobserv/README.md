# Module 05: SkilzNetObserv

SkilzNetObserv is a browser-based packet and traffic observer for network devices. An engineer opens
a topology map, clicks a device and an interface, and sees decoded packets in real time with the
same depth of decoding as Wireshark. A second view, Traffic Trace, follows a flow across the whole
topology and lights up each link it crosses.

The application can be run in two ways, and this folder holds one guide for each. The guides are
kept separate so that each can be followed from the first step to the last without switching
between two sets of instructions.

---

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

---

## The two guides

| | [Kubernetes guide](netobserv-kubernetes-guide.md) | [Standalone guide](netobserv-standalone-guide.md) |
|---|---|---|
| File | `netobserv-kubernetes-guide.md` | `netobserv-standalone-guide.md` |
| Runs on | The Kubernetes cluster built in Modules 01 to 04 | Any Docker host |
| Image | `openskilz/skilznetobserv` from Docker Hub | `openskilz/skilznetobserv` from Docker Hub |
| Sign-in | Active Directory groups (`netobserv-admins`, `netobserv-users`) | Local accounts, with an optional directory connected from the Settings page |
| Device inventory | The Nautobot instance used by the platform | Any Nautobot instance, entered on the Settings page |
| Device credentials | Vault, through an AppRole | A local file by default, or Vault when configured |
| TLS | cert-manager certificate from the platform Sub-CA, terminated at the ingress | Left to the operator, for example a reverse proxy |
| Configuration | Kubernetes secret and manifest | The setup wizard and Settings page, or environment variables |
| Good fit | A platform that already runs the cluster, Vault, Nautobot and Active Directory | A quick start, a lab, or a site with no Kubernetes |

### Kubernetes guide

Builds SkilzNetObserv as a workload in the cluster. It creates the Active Directory groups, prepares
Vault and Nautobot, pulls the published Docker Hub image, deploys the manifest, configures the devices
and verifies the result. The manifest is part of the guide, so no source code, build step or private
registry is needed. It also covers day-to-day operation, troubleshooting and the known limitations of
the Kubernetes deployment. Follow it when Modules 01 to 04 are complete and a Nautobot instance holds
the devices.

### Standalone guide

Runs the published Docker image on a single host. It covers the first-boot setup wizard, connecting
Nautobot and an optional directory, device credentials, capture modes and exactly what a capture
changes on each device, Traffic Trace, configuration as code, updating, customising the source and a
troubleshooting list. It needs nothing from the rest of the platform. The guide suggests trying it in an isolated
environment first.

---

## What the two share

Both deployments run the same application code, support the same platforms (see Scope of this
version) and behave the same way on the devices. Both read devices, interfaces, addresses and cables
from Nautobot, and both create the mirror session on a device only for the length of a capture.

## What the two do not share

The deployments hold separate configuration, separate accounts and separate credential stores. A
setting changed in one has no effect on the other. A device accepts one ERSPAN session at a time on
a given session number, so a capture started from one deployment replaces a session created by the
other. Use one deployment at a time against any given device.

---

## Device access

The application logs in to each device with a dedicated account. That account should be limited to
the ERSPAN session commands (IOS-XE and NX-OS), the LINA `capture` commands (FTD) and the few `show`
commands that the application sends. The limit can be set on any device administration service, for
example TACACS+. Each guide lists the commands in full under "Device account and command
authorisation".

---

## Folder contents

| File | Purpose |
|---|---|
| `README.md` | This overview |
| `netobserv-kubernetes-guide.md` | SkilzNetObserv on the Kubernetes cluster |
| `netobserv-standalone-guide.md` | SkilzNetObserv as a standalone Docker image |
| `pictures/` | Screenshots of the interface |
