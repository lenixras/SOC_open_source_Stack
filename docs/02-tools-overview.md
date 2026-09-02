# Tooling deep-dive

Per-tool description, capabilities, deployment model, known limitations and our
usage in this project.

## 1. Wazuh — SIEM & XDR

### What it is
Open-source security platform combining **SIEM** (log management + correlation)
and **XDR** (endpoint detection + response). Born as a fork of OSSEC in 2015,
now maintained by Wazuh Inc. with a strong community edition.

### Architecture
```
   ┌────────────┐    TCP/1514    ┌───────────────┐
   │  Agent     │──────────────► │ Wazuh Manager │
   │ (endpoint) │                │  - analysisd  │
   └────────────┘                │  - monitord   │
                                 │  - logcollec. │
   ┌────────────┐    Syslog      │  - modulesd   │
   │ Syslog src │──────────────► │               │
   └────────────┘                └───────┬───────┘
                                         │
                                         ▼
                                 ┌───────────────┐
                                 │ Wazuh Indexer │
                                 │ (OpenSearch)  │
                                 └───────┬───────┘
                                         │
                                         ▼
                                 ┌───────────────┐
                                 │   Dashboard   │
                                 │ (OpenSearch   │
                                 │   Dashboards) │
                                 └───────────────┘
```

### Capabilities used
- **Log collection** — agent-based (push) or agentless (syslog).
- **File Integrity Monitoring (FIM)** — `syscheck` watches files/registry, diffs, who-data.
- **Vulnerability detection** — package inventory → CVE match (NVD feed).
- **Configuration assessment** — CIS benchmark scans (SCA policy).
- **MITRE ATT&CK mapping** — built-in tags + custom rules.
- **Active response** — block IP, kill process, quarantine file (per-agent policy).
- **API** — REST + Python SDK for automation / SOAR.

### Deployment in this stack
- Manager + indexer + dashboard on a single VM or as a Compose bundle (dev).
- Agents on every endpoint (Linux/Windows/macOS).
- Centralised syslog intake from pfSense, SafeLine, Cloudflare Logpush.

### Limits & gotchas
- Indexer (OpenSearch) is RAM-hungry: plan ~ 4 GB per indexer node minimum.
- Upgrades sometimes require full re-indexing — budget a maintenance window.
- Default ruleset is noisy; expect 50 %+ false-positive reduction work.
- Active response must be tested in lab first — it can lock you out.

---

## 2. Velociraptor — DFIR

### What it is
Open-source endpoint visibility, digital forensics and incident response tool.
Written in Go, agent-server model, VQL query language. Acquired by Rapid7 in
2024, remains open source.

### Architecture
```
   ┌──────────────┐    gRPC/TLS    ┌──────────────────┐
   │ Velociraptor │──────────────► │ Velociraptor     │
   │ Agent        │   :8000       │ Server (GUI:8889)│
   │ (Windows,    │               │  - filestore     │
   │  Linux, mac) │               │  - indexer       │
   └──────────────┘               │  - notebooks     │
                                  └──────────────────┘
```

### Capabilities used
- **Hunt artefacts** — prebuilt & custom VQL packages (scheduled hunts).
- **Live response** — interactive shell, file upload/download, memory dump.
- **Forensic acquisition** — disk triage (NTFS MFT, registry hives, EVTX).
- **Persistence & lateral-movement detection** — autoruns, scheduled tasks, SSH keys.
- **Notebooks** — shareable, replayable investigation timelines.
- **Permissions** — ACL labels on artefacts for least-privilege analyst access.

### Deployment in this stack
- Server: containerised (single binary in a thin container) with persistent `/data` volume.
- Agents: native binaries, enrolled with a one-time config (`client.config.yaml`).
- Inter-component: Wazuh custom integration triggers Velociraptor hunts on high-severity alerts.

### Limits & gotchas
- Agents are powerful — strict certificate pinning & ACLs are mandatory.
- Large acquisitions consume bandwidth; schedule bulk acquisitions off-hours.
- Server-side filestore must be backed up regularly.

---

## 3. pfSense — Network security

### What it is
Open-source FreeBSD-based router and firewall. Free edition is fully featured;
pfSense Plus is the commercial variant. Project born from m0n0wall.

### Capabilities used
- **Stateful firewall** — per-rule logging, aliases, virtual IPs.
- **IDS/IPS** — Suricata package (recommended over legacy Snort).
- **VPN** — WireGuard (community pkg) or OpenVPN / IPsec.
- **Captive portal** — guest networks, BYOD.
- **Traffic shaping** — ALTQ queues, per-IP throttling.
- **High availability** — CARP + XML-RPC config sync.
- **Packages** — pfBlockerNG (geo-IP + threat feeds), ntopng, FreeRADIUS, haproxy.

### Deployment in this stack
- VM with **two virtual NICs** (WAN bridged to host internet, LAN on private vmbr).
- Suricata in **inline IPS mode** on the LAN interface.
- Syslog forwarding to Wazuh (UDP 514 or reliable syslog via `syslog-ng` relay).

### Limits & gotchas
- Hardware compatibility list matters — Intel NICs (`em`, `igb`, `ix`) preferred.
- Suricata inline mode can drop legitimate traffic if rules are aggressive — start in IDS-only.
- WAN-side spoofing protection: enable bogons, block private nets on WAN.

---

## 4. SafeLine WAF — Web protection

### What it is
Self-hosted Web Application Firewall by Chaitin Tech. Built on a semantic
parsing engine + ML detection. Free community edition; commercial tiers add
clustering and SLA.

### Capabilities used
- **L7 protection** — SQLi, XSS, SSRF, RCE, path traversal, deserialization.
- **Bot mitigation** — challenge, captcha, JA3 fingerprinting.
- **Rate limiting** — per-IP / per-URI / per-token.
- **TLS** — built-in cert management (Let's Encrypt via DNS-01).
- **Dashboard** — live traffic, blocked requests, geo, top attackers.
- **Reverse-proxy** — replaces a basic nginx in front of web apps.

### Deployment in this stack
- Container behind pfSense, in front of one or more web apps (DVWA / Juice Shop in lab).
- HTTPS terminated at SafeLine, backend over private HTTP or mutual TLS.
- Syslog + webhook integration to Wazuh for every blocked request.

### Limits & gotchas
- Chinese documentation; community is smaller than ModSecurity / Coraza.
- Semantic engine consumes ~ 2 GB RAM and 2 vCPU minimum.
- Free edition limits clusters; single-node is fine for SMB.

---

## 5. Cloudflare — Edge / Anti-DDoS

### What it is
Global Anycast CDN + reverse-proxy + managed security. Free tier provides DNS,
CDN, basic DDoS and shared SSL; Pro/Business unlock WAF customisation and
Logpush.

### Capabilities used
- **Authoritative DNS** — primary nameservers at `*.ns.cloudflare.com`.
- **CDN** — caching of static assets, reducing origin load.
- **Always-on DDoS** — L3/L4/L7 mitigation, no extra config.
- **Managed WAF rules** — Cloudflare-managed + OWASP CRS ruleset.
- **Rate limiting / Bot Fight Mode** — cheap abuse mitigation.
- **Cloudflare Tunnel (`cloudflared`)** — expose services without public IP, ideal for SOC GUIs.
- **Zero Trust Access** — SSO in front of Wazuh dashboard, pfSense, Velociraptor.

### Deployment in this stack
- DNS managed by Cloudflare (orange-cloud for web properties).
- Tunnel (`cloudflared`) running as a sidecar next to SafeLine.
- Zero Trust Access policies for admin GUIs (Cloudflare Access + OTP via TOTP / IdP).
- Logpush (Pro+) → R2 bucket → Wazuh indexer (phase 2).

### Limits & gotchas
- Free tier lacks Logpush and full WAF customisation.
- Some EU/regulated sectors require data residency — verify Cloudflare's regional services.
- Tunnel outages = no remote admin; have an out-of-band fallback (IPMI, KVM, IPsec VPN).

---

## 6. Adjacent / future tools

| Tool | Role | Phase |
|------|------|-------|
| **Suricata** | IDS/IPS engine (run inside pfSense) | 1 |
| **MISP** | Threat intelligence platform | 2 |
| **TheHive** | Incident case management | 2 |
| **Cortex** | Observable enrichment (VirusTotal, AbuseIPDB) | 2 |
| **Shuffle** | SOAR automation | 3 |
| **Arkime** | Full packet capture | 3 |
| **Zeek** | Network security monitor (logs) | 3 |
| **Grafana + Prometheus + Loki** | Observability, dashboards | 2 |
| **OpenCTI** | Threat intel knowledge base (alternative to MISP) | 3 |

Each will be evaluated individually in its phase; none is locked in.