# Open-Source SOC Stack

> A production-grade, 100 % open-source Security Operations Center blueprint designed for home networks, SMBs and startups.

[![Status](https://img.shields.io/badge/status-design%20phase-blueviolet)](#roadmap)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Stack](https://img.shields.io/badge/stack-Linux%20%7C%20Docker%20%7C%20Proxmox-blue)](#prerequisites)
[![Focus](https://img.shields.io/badge/focus-SOC%20%7C%20Blue%20Team-red)](#project-goals)

---

## Table of contents

1. [Project goals](#project-goals)
2. [Why an open-source SOC?](#why-an-open-source-soc)
3. [Target audiences](#target-audiences)
4. [Stack at a glance](#stack-at-a-glance)
5. [Tool roles](#tool-roles)
6. [High-level architecture](#high-level-architecture)
7. [Lab architecture](#lab-architecture)
8. [Integration matrix](#integration-matrix)
9. [Implementation strategy (Linux + Docker)](#implementation-strategy-linux--docker)
10. [Prerequisites & constraints](#prerequisites--constraints)
11. [Repository layout](#repository-layout)
12. [Documentation map](#documentation-map)
13. [Roadmap](#roadmap)
14. [Portfolio context](#portfolio-context)
15. [Disclaimer](#disclaimer)

---

## Project goals

Design and document a **defensible, repeatable and auditable** Security Operations Center stack made entirely of open-source components. The deliverable covers the full SOC lifecycle:

- **Detect** — collect logs, telemetry and endpoint signals at scale.
- **Analyse** — correlate events, hunt for threats, score alerts.
- **Respond** — triage incidents, collect forensic artefacts, contain.
- **Recover** — restore systems, document lessons learned, harden again.
- **Defend the perimeter** — firewalling, WAF, anti-DDoS at the edge.

The project follows a strict engineering methodology:

1. **Analyse** — what does each tool solve? What are its limits?
2. **Design** — how do components interact? What data flows where?
3. **Feasibility** — can this run on commodity hardware and OSS?
4. **Implement** — automate with Docker / Ansible / Terraform.
5. **Validate** — attack simulations (atomic-red-team, Caldera), tabletop.

> **Current phase:** *Analyse, design and feasibility.* No production deployment yet. The next phases will deliver reproducible artefacts (Docker Compose stacks, Ansible playbooks, hardening guides, IR runbooks).

---

## Why an open-source SOC?

| Commercial SIEM/XDR | This stack |
|---------------------|------------|
| Per-GB or per-EPS licence cost | Free, predictable infra cost |
| Vendor lock-in, opaque scoring | Transparent detection rules you can audit |
| Closed telemetry pipeline | Open data formats (Syslog, JSON, ECS-style) |
| Heavy hardware footprint | Tunable from a Raspberry Pi 4 to a 1U server |
| Slow iteration cycles | Patch and contribute upstream |

The trade-off is **more integration work** — which is exactly the engineering value this project showcases.

---

## Target audiences

| Audience | Scale | Typical use case |
|----------|-------|------------------|
| **Home / prosumers** | 5–20 devices | Self-hosted monitoring, parental control, IoT visibility |
| **Small & Medium Businesses (SMBs)** | 20–500 endpoints | Compliance (NIS2, GDPR, ISO 27001), MSSP-ready, insurance requirements |
| **Startups** | 10–100 endpoints | Cost-efficient security from day 1, investor due-diligence friendly |
| **Security students / analysts** | Lab | Learn SIEM tuning, DFIR, threat hunting on a realistic stack |

---

## Stack at a glance

| Layer | Tool | Role |
|-------|------|------|
| **SIEM & XDR** | **Wazuh** | Log management, endpoint detection, file integrity monitoring, vulnerability detection, compliance |
| **DFIR** | **Velociraptor** | Endpoint telemetry at scale, digital forensics, threat hunting, live response |
| **Perimeter firewall** | **pfSense** | Network segmentation, VPN, IDS/IPS (Suricata), routing, captive portal |
| **Web Application Firewall** | **SafeLine (Chaitin Tech)** | L7 HTTP protection, bot mitigation, OWASP Top 10 |
| **Edge / Anti-DDoS** | **Cloudflare** | DNS, CDN, WAF, DDoS mitigation, Zero Trust access |

> **Extended ecosystem (documented, not yet wired in):** Suricata (IDS), MISP (threat intel), TheHive + Cortex (case management), Shuffle (SOAR), Arkime / Zeek (full packet capture), Grafana + Prometheus (observability).

---

## Tool roles

### Wazuh — SIEM & XDR

- Centralised log collection (agents on Linux/Windows/macOS).
- File Integrity Monitoring (FIM), Registry monitoring on Windows.
- Vulnerability scanner (CVE matching against installed packages).
- Compliance mapping: PCI DSS, HIPAA, NIST 800-53, GDPR, TSC.
- Custom rules in XML, MITRE ATT&CK tagging.
- REST API + indexer (OpenSearch) + dashboard (OpenSearch Dashboards).

### Velociraptor — DFIR

- Agent-based, written in Go, cross-platform (Linux/Windows/macOS).
- VQL (Velociraptor Query Language) for live forensics, hunt artefacts.
- Triaging, memory acquisition, file carving, persistence hunting.
- Notebooks for collaborative investigations, timeline reconstruction.
- Complements Wazuh: Wazuh *detects*, Velociraptor *investigates*.

### pfSense — Network security

- FreeBSD-based, open-source router / firewall.
- Netgate appliance or any x86 box (2+ NICs).
- Suricata / Snort inline IDS/IPS, pfBlockerNG (geo-IP, threat feeds).
- WireGuard / OpenVPN, Captive Portal, VLANs, traffic shaping.
- Suricata logs can be forwarded to Wazuh for correlation.

### SafeLine — Web Application Firewall

- Self-hosted WAF, reverse-proxy in front of web apps.
- Protects against SQLi, XSS, SSRF, RCE, path traversal, bot abuse.
- Built-in semantic engine + ML-based detection.
- Dashboard for live traffic, blocked requests, geo distribution.
- Sits between Cloudflare (edge) and the origin application.

### Cloudflare — Edge & Anti-DDoS

- Authoritative DNS, Anycast CDN.
- Layer 3/4/7 DDoS mitigation (always-on).
- Managed WAF rules + rate limiting + Bot Fight Mode.
- Tunnel (Cloudflared) to expose services without public IP.
- Zero Trust Access for admin panels (Wazuh dashboard, pfSense, Velociraptor GUI).

---

## High-level architecture

```
                     ┌─────────────────────────────────────────┐
                     │           INTERNET / USERS              │
                     └────────────────────┬────────────────────┘
                                          │
                              ┌───────────▼───────────┐
                              │      Cloudflare       │  DNS, CDN,
                              │  (Edge / Anti-DDoS)   │  L3-L7 DDoS,
                              │  Zero Trust Access    │  WAF, Tunnel
                              └───────────┬───────────┘
                                          │ (clean HTTPS)
                              ┌───────────▼───────────┐
                              │       pfSense         │  Firewall,
                              │  (Perimeter / VPN)    │  VLANs, IDS/IPS
                              │  Suricata, pfBlocker  │
                              └───────────┬───────────┘
                                          │
              ┌───────────────────────────┼────────────────────────────┐
              │                           │                            │
       ┌──────▼──────┐             ┌──────▼──────┐             ┌────────▼────────┐
       │  Servers    │             │   Users     │             │   DMZ / Web     │
       │  VLAN 10    │             │  VLAN 20    │             │   VLAN 30       │
       │ (Linux/Win) │             │ (PCs/Mobile)│             │ (Apps, APIs)    │
       └──────┬──────┘             └──────┬──────┘             └────────┬────────┘
              │                            │                            │
              │   Wazuh agent              │   Wazuh agent              │
              │   Velociraptor agent       │   Velociraptor agent       │
              │                            │                            │
              └─────────────┬──────────────┴──────────────┬─────────────┘
                            │                             │
                  ┌─────────▼──────────┐        ┌─────────▼──────────┐
                  │   SafeLine WAF     │        │  Reverse proxy     │
                  │   (L7 protection   │◄──────►│   (Nginx / Caddy)  │
                  │   on web traffic)  │        │                    │
                  └─────────┬──────────┘        └─────────┬──────────┘
                            │                             │
                            └──────────────┬──────────────┘
                                           │
                                ┌──────────▼──────────┐
                                │     SOC PLATFORM    │
                                │ ┌────────────────┐  │
                                │ │  Wazuh Manager │  │  SIEM, XDR,
                                │ │  + Indexer     │  │  FIM, vuln,
                                │ │  + Dashboard   │  │  compliance
                                │ └────────────────┘  │
                                │ ┌────────────────┐  │
                                │ │ Velociraptor   │  │  DFIR, hunt,
                                │ │ Server + GUI   │  │  live response
                                │ └────────────────┘  │
                                │ ┌────────────────┐  │
                                │ │ TheHive+Cortex │  │  Case mgmt
                                │ │ (planned)      │  │  (phase 2)
                                │ └────────────────┘  │
                                │ ┌────────────────┐  │
                                │ │ MISP           │  │  Threat intel
                                │ │ (planned)      │  │  (phase 2)
                                │ └────────────────┘  │
                                └─────────────────────┘
                                           │
                                ┌──────────▼──────────┐
                                │  Observability      │
                                │  Grafana + Prom.    │  Metrics, SLOs
                                └─────────────────────┘
```

Full breakdown: see [`docs/01-architecture.md`](docs/01-architecture.md) and the ASCII diagrams in [`diagrams/`](diagrams/).

---

## Lab architecture

A realistic, single-host lab that any engineer can rebuild on commodity hardware.

```
                    ┌────────────────────────────────────────┐
                    │        Host machine (Linux/WSL)        │
                    │     16 GB RAM · 8 vCPU · 200 GB SSD    │
                    │                                        │
                    │  ┌──────────────────────────────────┐  │
                    │  │  Proxmox VE  (or Docker host)    │  │
                    │  │                                  │  │
                    │  │  ┌────────────┐  ┌────────────┐  │  │
                    │  │  │ pfSense VM │  │  Windows   │  │  │
                    │  │  │ (FW/VPN)   │  │  10 victim │  │  │
                    │  │  └────────────┘  └────────────┘  │  │
                    │  │  ┌─────────────────────────────┐ │  │
                    │  │  │  SOC VM (Ubuntu 22.04)      │ │  │
                    │  │  │   - Wazuh Manager/Indexer   │ │  │
                    │  │  │   - Velociraptor Server     │ │  │
                    │  │  │   - SafeLine WAF            │ │  │
                    │  │  │   - TheHive/Cortex (P2)     │ │  │
                    │  │  └─────────────────────────────┘ │  │
                    │  │  ┌─────────────────────────────┐ │  │
                    │  │  │  Linux victim (Ubuntu/Deb)  │ │  │
                    │  │  │   - Wazuh agent             │ │  │
                    │  │  │   - Velociraptor agent      │ │  │
                    │  │  └─────────────────────────────┘ │  │
                    │  │  ┌─────────────────────────────┐ │  │
                    │  │  │  Web target (DVWA/Nginx)    │ │  │
                    │  │  │   - behind SafeLine         │ │  │
                    │  │  └─────────────────────────────┘ │  │
                    │  └──────────────────────────────────┘  │
                    └────────────────────────────────────────┘
                                        │
                              ┌─────────▼─────────┐
                              │     Internet      │
                              │  (Cloudflare      │
                              │   free tier)      │
                              └───────────────────┘
```

See [`docs/02-lab-architecture.md`](docs/02-lab-architecture.md) for the full topology, IP plan and traffic flows.

---

## Integration matrix

| Source → Destination | Mechanism | Purpose |
|----------------------|-----------|---------|
| pfSense → Wazuh | Syslog (UDP 514) or Wazuh agent | Firewall, Suricata, DHCP logs |
| Wazuh agents → Wazuh manager | TCP 1514 (encrypted) | Endpoint telemetry |
| Velociraptor agents → Velociraptor server | TLS 8000/gRPC | Forensics, hunt artefacts |
| Wazuh → Velociraptor | Wazuh custom integration (`integrations/virustotal`, custom script) | Trigger triage hunt on high-severity alert |
| SafeLine → Wazuh | Syslog + webhook | WAF events, blocked requests |
| Cloudflare → Wazuh | Logpush → S3 / R2 → Wazuh indexer | Edge WAF, DDoS events, DNS logs |
| Wazuh → TheHive | TheHive Wazuh integration | Alert → case, observables, observables enrichment (Cortex) |
| All → Grafana | Prometheus exporters, Loki | Unified dashboards |
| All → MISP | MISP feed sync | Threat intel correlation (phase 2) |

Full integration detail (config snippets, ports, encryption) in [`docs/03-integration.md`](docs/03-integration.md).

---

## Implementation strategy (Linux + Docker)

### Containerised SOC services

```text
Docker host (Ubuntu 22.04 / Debian 12)
 ├── compose/soc/
 │    ├── wazuh.manager      # wazuh/wazuh-manager:4.9
 │    ├── wazuh.indexer      # wazuh/wazuh-indexer:4.9  (OpenSearch)
 │    ├── wazuh.dashboard    # wazuh/wazuh-dashboard:4.9 (OpenSearch Dashboards)
 │    ├── velociraptor       # ghcr.io/velociraptor/velociraptor:latest
 │    ├── safeline           # chaitin/safeline:latest
 │    ├── thehive            # strangler/thehive:5
 │    ├── cortex             # thehiveproject/cortex:3
 │    └── grafana            # grafana/grafana:latest
 │
 ├── compose/edge/
 │    ├── cloudflared        # cloudflare/cloudflared  (tunnel)
 │    └── nginx-proxy        # nginx + ModSecurity (alt to SafeLine)
 │
 └── compose/data/
      ├── postgres           # backing store for TheHive/Cortex
      └── redis              # cache for TheHive
```

### pfSense and Velociraptor on bare metal / VM

- **pfSense** stays on a dedicated VM (or physical appliance) — its kernel-level packet processing is incompatible with Docker.
- **Velociraptor** can run containerised for the **server**, while **agents** remain native binaries on endpoints.
- **Wazuh manager + indexer** are best deployed via the official single-node container bundle or directly on a hardened VM (the latter is more production-faithful for the manager).

### Why hybrid (Docker + VM)?

| Component | Best deployment | Reason |
|-----------|-----------------|--------|
| Wazuh Manager | VM / bare metal | Performance, kernel access, syslog ingestion |
| Wazuh Indexer | VM or container | OpenSearch cluster friendliness |
| Wazuh Dashboard | Container | Stateless UI |
| Velociraptor Server | Container or VM | Stateful; backup is critical |
| Velociraptor Agent | Native binary | Endpoint-resident |
| SafeLine | Container | Designed for Docker |
| pfSense | VM / appliance | BSD kernel, packet processing |
| TheHive/Cortex | Container | Easier upgrades |
| Cloudflare Tunnel | Sidecar container | `cloudflared` is a single binary |

See [`docs/04-implementation.md`](docs/04-implementation.md) for the full `docker-compose.yml` skeleton, volume layout, backup and upgrade strategy.

---

## Prerequisites & constraints

### Hardware (lab minimum)

| Component | Spec |
|-----------|------|
| CPU | 4 vCPU (8 recommended for full lab) |
| RAM | 16 GB (Wazuh indexer alone is comfortable at 4 GB) |
| Storage | 200 GB SSD (Wazuh indexes can grow fast) |
| Network | 2 NICs (1 WAN, 1 LAN) for pfSense |

### Software

- Linux x86_64, kernel ≥ 5.15 (Ubuntu 22.04 LTS or Debian 12).
- Docker ≥ 24 + Compose v2.
- Proxmox VE 8.x (recommended) or a plain Docker host.
- 30 GB free for ISO / agent installers / PCAP samples.

### Skills

- Linux system administration (services, systemd, networking).
- Basic Docker / Compose.
- Reading logs (JSON, syslog, pcap).
- Networking fundamentals (VLAN, NAT, DNS, TLS).

### Constraints & known limitations

- **Wazuh indexer (OpenSearch)** is RAM-hungry; tune shard count aggressively.
- **Suricata on pfSense** requires careful NIC tuning to avoid packet drops.
- **SafeLine** requires ≥ 2 vCPU and 4 GB RAM for the semantic engine.
- **Velociraptor** agents are powerful — production deployment requires strict client cert management and ACL policy.
- **Cloudflare free tier** lacks Logpush; a Pro plan ($) or an R2 bucket is needed for full edge log ingestion.
- **No SIEM replaces process**: detection engineering, IR runbooks and regular purple-team exercises remain mandatory.

See [`docs/05-prerequisites.md`](docs/05-prerequisites.md) for the full checklist.

---

## Repository layout

```
soc-stack/
├── README.md # this file (project entry point)
├── LICENSE # MIT
├── .gitignore
├── docs/
│   ├── 01-architecture.md       # global + lab architecture detail
│   ├── 02-tools-overview.md    # per-tool deep-dive
│   ├── 03-integration.md        # integration matrix, configs, ports
│   ├── 04-implementation.md     # Docker / VM / Compose plans
│   ├── 05-prerequisites.md      # HW, SW, skills, constraints
│   └── 06-roadmap.md            # phased roadmap
├── diagrams/
│   ├── global-architecture.txt
│   └── lab-architecture.txt
└── docker/ # to be added in phase 2
    ├── soc/
    ├── edge/
    └── data/
```

---

## Documentation map

| You want to… | Read |
|--------------|------|
| Understand the big picture | `README.md` (you are here) |
| Detail every component | `docs/02-tools-overview.md` |
| See the lab topology & IP plan | `docs/01-architecture.md` |
| Wire components together | `docs/03-integration.md` |
| Deploy with Docker / VMs | `docs/04-implementation.md` |
| Check requirements first | `docs/05-prerequisites.md` |
| Plan the next phases | `docs/06-roadmap.md` |

---

## Roadmap

### Phase 0 — Analysis & feasibility (this phase)
- [x] Repository skeleton
- [x] Global README, architecture, integration matrix
- [x] Tooling overview
- [x] Lab design
- [x] Implementation strategy
- [x] Prerequisites & constraints
- [x] Roadmap

### Phase 1 — Reproducible lab
- [ ] Proxmox / Docker host provisioning scripts (Ansible)
- [ ] pfSense VM template + Suricata tuning
- [ ] Wazuh single-node deployment (manager + indexer + dashboard)
- [ ] Velociraptor server + first hunt artefact
- [ ] SafeLine in front of a vulnerable web app (DVWA / Juice Shop)
- [ ] Cloudflare account + Tunnel pointing at SafeLine

### Phase 2 — Detection engineering
- [ ] Custom Wazuh rules mapped to MITRE ATT&CK
- [ ] Velociraptor notebooks for top 10 hunts (persistence, lateral movement, exfil)
- [ ] MISP feed integration + IOC matching
- [ ] SafeLine tuning playbook (false-positive analysis)
- [ ] Purple-team exercises with Atomic Red Team

### Phase 3 — SOAR & IR
- [ ] TheHive + Cortex wired to Wazuh
- [ ] Shuffle SOAR playbooks (phishing triage, account lockout, host isolation)
- [ ] IR runbooks (ransomware, BEC, web defacement, DDoS)
- [ ] Tabletop scenarios & lessons-learned template

### Phase 4 — Hardening & observability
- [ ] Grafana dashboards (SOC KPIs: MTTD, MTTR, alert volume)
- [ ] Loki for log aggregation beyond Wazuh
- [ ] CIS benchmark hardening scripts (Ansible roles)
- [ ] Backup & disaster-recovery runbook

### Phase 5 — Public release
- [ ] Public GitHub repo with polished README, screenshots, demo GIFs
- [ ] Architecture Decision Records (ADR)
- [ ] Threat model (STRIDE)
- [ ] Blog post series / talk submission

Full roadmap with milestones & deliverables in [`docs/06-roadmap.md`](docs/06-roadmap.md).

---

## Portfolio context

This project is part of my **portfolio as a network & system engineer specialised in SOC / Blue Team** work. It demonstrates:

- **Architecture thinking** — translating business risk into layered controls.
- **Open-source fluency** — selecting, integrating and tuning community tools.
- **Detection engineering** — building rules, hunts and IR playbooks.
- **Documentation discipline** — writing for future operators, not just for myself.
- **Cost-aware design** — delivering enterprise-class capability on SMB budgets.

It complements adjacent portfolio pieces on system administration, networking and automation.

---

## Disclaimer

This project is for **educational and defensive security purposes only**. All components must be operated within the law and on systems you own or are authorised to monitor. The author is not liable for misuse. SafeLine™ and Wazuh™ are trademarks of their respective owners; this project is not affiliated.

---

*Last updated: design & feasibility phase. Next milestone: reproducible lab.*