# Architecture

This document expands the diagrams shown in the [README](../README.md) with the
detailed data flows, VLAN/IP plan and traffic matrices.

## 1. Global architecture

### Layers

| # | Layer | Components | Responsibility |
|---|-------|------------|----------------|
| 0 | **Edge** | Cloudflare (DNS, CDN, DDoS, Tunnel) | Absorb volumetric attacks, hide origin IP, expose only needed services |
| 1 | **Perimeter** | pfSense (FW, NAT, VPN, IDS/IPS) | North-south filtering, segmentation, site-to-site VPN |
| 2 | **DMZ** | SafeLine, reverse-proxy, public apps | L7 filtering, bot mitigation, TLS termination |
| 3 | **Internal** | Servers, users, IoT | Production workload, segmented by VLAN |
| 4 | **SOC platform** | Wazuh, Velociraptor, TheHive, Grafana | Detection, forensics, response, observability |

### Data flows

```
[Edge: Cloudflare] --HTTPS--> [pfSense WAN]
                                  │
                                  ▼
                            [pfSense routing]
                                  │
                ┌─────────────────┼─────────────────┐
                ▼                 ▼                 ▼
        [DMZ VLAN 30]    [Servers VLAN 10]   [Users VLAN 20]
                │                 │                 │
                │ SafeLine        │ Wazuh agent     │ Wazuh agent
                │ (L7 filter)     │ Velociraptor    │ Velociraptor
                ▼                 ▼                 ▼
        [Reverse proxy]    [App / DB]          [Endpoints]
                │
                ▼
        [Public web app]
```

### Telemetry flows

```
Endpoints / Network devices
    │
    │  (Wazuh agent / Syslog / NetFlow)
    ▼
Wazuh Manager ──► Wazuh Indexer ──► Wazuh Dashboard
    │
    │  (custom integration on alert)
    ▼
Velociraptor (auto-hunt) ──► Velociraptor GUI / notebook
    │
    ▼
TheHive (case) ──► Cortex (enrichment) ──► Analyst
```

---

## 2. Lab architecture

### Topology (single host, Proxmox VE)

```
┌──────────────────────────────────────────────────────────┐
│                 Host: Proxmox VE 8.x                     │
│   CPU: 8 vCPU   RAM: 32 GB   SSD: 500 GB                │
│                                                          │
│   VMID  Name              vCPU  RAM  Disk  Role          │
│   ────  ────────────────  ────  ───  ────  ────────────  │
│   100   pfsense-fw         2     2    8   Firewall/WAN  │
│   101   soc-platform       6    12   80   Wazuh+VR+SL   │
│   102   win-victim         2     4    40   Win10 victim │
│   103   lin-victim         2     2    20   Ubuntu vict. │
│   104   web-target         2     2    20   DVWA+JShop   │
│   105   kali-attacker      2     2    30   Kali         │
│                                                          │
│   Networks:                                              │
│     vmbr0  WAN            192.0.2.0/24  (NAT to host)   │
│     vmbr1  LAN / SOC      10.10.10.0/24                 │
│     vmbr2  DMZ            10.10.30.0/24                 │
│     vmbr3  User-LAN       10.10.20.0/24                 │
└──────────────────────────────────────────────────────────┘
```

### VLAN / IP plan

| VLAN ID | Name | Subnet | Gateway | Purpose |
|---------|------|--------|---------|---------|
| — | WAN | 192.0.2.0/24 | 192.0.2.1 (host) | Egress to internet via host NAT |
| 10 | Servers | 10.10.10.0/24 | 10.10.10.1 | Linux/Windows servers, infra |
| 20 | Users | 10.10.20.0/24 | 10.10.20.1 | Workstations, BYOD |
| 30 | DMZ | 10.10.30.0/24 | 10.10.30.1 | Web apps behind SafeLine |
| 40 | SOC mgmt | 10.10.40.0/24 | 10.10.40.1 | Wazuh, Velociraptor, TheHive GUI |
| 50 | IoT | 10.10.50.0/24 | 10.10.50.1 | Isolated, no lateral to LAN |

### Lab hardware budget

- Lab reuse: 1 x mini-PC (Intel NUC / Beelink) + USB-Ethernet dongle.
- Production-lite: refurbished Dell OptiPlex / HP ProDesk (i5–i7, 16 GB).
- Avoid consumer routers with custom firmware in production: pfSense needs ≥ 2 GB RAM and AES-NI.

### Network access control matrix

| Source → Dest | Servers | Users | DMZ | SOC | IoT | Internet |
|---------------|---------|-------|-----|-----|-----|----------|
| Servers | ✅ | ✅ | ✅ | ✅ | 🚫 | ⚠️ filtered |
| Users | ⚠️ auth | ✅ | ✅ | 🚫 | 🚫 | ✅ |
| DMZ | 🚫 | 🚫 | ⚠️ egress | 🚫 | 🚫 | ✅ |
| SOC | ✅ (agents) | ⚠️ opt-in | ✅ | ✅ | ✅ | ⚠️ updates only |
| IoT | 🚫 | 🚫 | 🚫 | 🚫 | 🚫 | ✅ filtered |
| Internet | 🚫 | 🚫 | 🚫 | 🚫 | 🚫 | — |

✅ allow · ⚠️ conditional · 🚫 deny.

---

## 3. Production deltas vs lab

When moving from lab to production, the following adjustments are required:

1. **pfSense** on a dedicated appliance (Netgate 1100 / 2100 / 5100) or 1U server with Intel NICs (igb/ix).
2. **Wazuh manager cluster** — 3 manager nodes + load-balanced indexer cluster (3 masters, ≥ 2 data nodes).
3. **Velociraptor multi-server** with a front-end load-balancer and offline-capable agents.
4. **Storage tiering** — Wazuh hot/warm/cold indexes via ISM policies; cold tier on NFS or S3-compatible.
5. **Out-of-band management** — separate VLAN + console (IPMI, iLO, iDRAC) unreachable from production.
6. **Backup** — Velociraptor datastore, Wazuh manager config, TheHive DB, daily snapshots, off-site copy.
7. **Identity** — SSO (Keycloak / Authentik / Cloudflare Access) on every GUI.

---

## 4. Failure-domain analysis

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| Cloudflare outage | Service unreachable from internet | Direct IP via secondary DNS provider; pfSense can switch to direct WAN |
| pfSense crash | Whole network down | CARP pair + spare box with identical config |
| Wazuh manager down | No new alerts | Local Wazuh agent buffer; agents reconnect; indexer keeps historical data |
| Velociraptor down | No new artefacts, but Wazuh still records | Server statelessness designed for fast redeploy; agents buffer |
| SafeLine down | Web traffic unprotected | Direct passthrough to nginx; rate limiting fallback at pfSense |
| Internet loss | Edge tools unusable | Local-only mode; SOC continues with internal data |

---

## 5. Open questions (to address in phase 1)

- Should Cloudflare Tunnel terminate TLS at the tunnel or at SafeLine? (decided: SafeLine, for L7 inspection)
- Which MISP instance to use as primary feed? (TBD — see roadmap)
- Single-tenant vs multi-tenant Wazuh for an MSSP pivot? (out of scope for SMB/startup)
- Storage retention policy: hot 30 d / warm 90 d / cold 365 d? (defaults proposed in implementation doc)