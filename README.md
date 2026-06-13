# OpenSkilz

A collection of open projects covering NetDevOps, infrastructure automation,
and platform engineering. Each project lives in its own directory with its own
documentation, build guides, and context.

The goal is to share practical, working implementations rather than theoretical
examples — things that were actually built, tested on real or virtual hardware,
and documented in enough detail that someone else can follow the same path.

---

## Projects

### [skilz.io](./skilz.io)

The pilot project. A production-pattern Kubernetes platform built across six
nodes and three subnets, hosting SkilzNetObserv ,a browser-based multi-vendor
network traffic analyser that uses ERSPAN hardware mirroring for passive,
real-time packet capture.

The platform covers the full stack: two-tier PKI, Active Directory, HashiCorp
Vault, Calico BGP networking, Longhorn storage, NGINX ingress with automatic
TLS issuance, and a GitLab CI/CD pipeline. The application built on top of it
captures and decodes live traffic from Cisco, Arista, and Juniper devices in a
browser, without holding open sessions on production equipment.

Full documentation, build guides, and screenshots are inside the directory.

---

## What gets added here

Projects added to this repository will generally fall into one of these areas:

- Network visibility and traffic analysis
- Infrastructure automation and NetDevOps tooling
- Kubernetes platform patterns for on-premises and hybrid environments
- Network source of truth and inventory management
- CI/CD pipelines for network and infrastructure change

Each project is self-contained. Its README, build documentation, and any
relevant configuration files live inside its own directory.

---

## Contributions

Issues and pull requests are welcome. If something is unclear, incorrect, or
could be improved, please raise an issue or open a pull request against the
relevant project directory. Suggestions for new projects or directions are
welcome in Discussions.
