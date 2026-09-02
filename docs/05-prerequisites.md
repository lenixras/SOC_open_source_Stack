# Prerequisites & constraints

Before starting the implementation phases, ensure you meet the following.

---

## 1. Hardware

### 1.1 Lab (single host)
| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 4 vCPU, AES-NI | 8 vCPU / 8 threads, AES-NI |
| RAM | 16 GB | 32 GB |
| Storage | 200 GB SSD | 500 GB NVMe |
| Network | 1 Gbps Ethernet | 2.5 Gbps + USB-Ethernet dongle for pfSense WAN |

### 1.2 Production-lite (SMB, 50 endpoints)
| Component | Spec |
|-----------|------|
| SOC VM | 8 vCPU, 24 GB RAM, 1 TB NVMe (RAID-1) |
| pfSense | Netgate 2100 / 5100 or 1U server with Intel NICs |
| Endpoints | any x86 device with ≥ 4 GB RAM |
| Network switch | VLAN-aware (L2+, e.g. Mikrotik, Ubiquiti, Cisco) |

### 1.3 Production (SMB+, 50–500 endpoints)
- 3-node Wazuh cluster (16 vCPU / 32 GB / 500 GB per node).
- 2-node Velociraptor HA (8 vCPU / 16 GB / 500 GB per node).
- NFS or S3-compatible bucket for cold indexes.
- Out-of-band management (IPMI / iDRAC).

---

## 2. Software

| Tool | Version | Purpose |
|------|---------|---------|
| Linux x86_64 | Ubuntu 22.04 LTS or Debian 12 | SOC VM, Docker host |
| Proxmox VE | 8.x | Optional hypervisor |
| Docker Engine | ≥ 24.0 | Container runtime |
| Docker Compose | v2.x | Stack orchestration |
| Git | ≥ 2.34 | Repo, version control |
| Ansible | ≥ 7.x | Configuration management |
| OpenSSL | ≥ 3.0 | Cert management |
| Terraform (opt.) | ≥ 1.5 | Infrastructure as code (Cloudflare / Proxmox) |

---

## 3. Network prerequisites

- Public IPv4 (or IPv6) for Cloudflare origin.
- Domain name with DNSSEC optional.
- Ability to delegate DNS to Cloudflare (`ahmed.ns.cloudflare.com`, etc.).
- Switch supporting 802.1Q VLANs.
- At least one spare Ethernet port for pfSense WAN.

---

## 4. Accounts & services

| Service | Account | Notes |
|---------|---------|-------|
| Cloudflare | Free (DNS/CDN/DDoS) or Pro (Logpush) | Pro recommended for full logs |
| Let's Encrypt | DNS-01 via Cloudflare API | Auto-renewal |
| GitHub | Public repo | Portfolio visibility |
| MISP instance | Phase 2 | Self-hosted or community feed |
| VirusTotal / AbuseIPDB | API keys for Cortex (phase 2) | Free tier fine |

---

## 5. Skills

- **Linux administration** — services, systemd, networking.
- **Networking fundamentals** — VLAN, NAT, DNS, TLS, routing.
- **Docker / Compose** — multi-container stacks, volumes, networks.
- **Reading logs** — JSON, syslog, pcap.
- **Basic Python** — for custom integrations.
- **Detection engineering mindset** — translating adversary TTPs to rules.

Nice-to-have (not blocking):
- Go (Velociraptor VQL deep-dive).
- Lua (Wazuh custom rules).
- Ansible / Terraform.

---

## 6. Constraints & known limitations

### 6.1 Operational
- **No SIEM replaces process.** Detection engineering, IR runbooks and regular
  purple-team exercises remain mandatory.
- **Staffing.** A full SOC needs ≥ 2 trained analysts; this stack is designed
  to be operable by a single part-time engineer (SMB / startup scale).
- **Vendor EOL.** Community editions (Wazuh, TheHive, Cortex, SafeLine) can
  evolve or pivot. Always track upstream.

### 6.2 Technical
- **Wazuh indexer (OpenSearch)** is RAM-hungry. Plan ≥ 4 GB per indexer node.
- **Suricata on pfSense** requires careful NIC tuning; AES-NI recommended.
- **SafeLine** requires ≥ 2 vCPU + 4 GB RAM for the semantic engine.
- **Velociraptor** agents are powerful — strict certificate management and
  ACLs are mandatory before any production rollout.
- **Cloudflare free tier** lacks Logpush — Pro plan required for full edge
  log ingestion, or use the Analytics API polled by a script.

### 6.3 Legal & regulatory
- **Data residency.** Cloudflare's regional services may be required for EU.
- **Lawful monitoring.** All endpoint agents must respect employee privacy
  (works council / DPO notice where applicable).
- **Encryption-at-rest.** Indexes can contain sensitive data — disk
  encryption (LUKS / ZFS native) is mandatory in production.

---

## 7. Pre-flight checklist

Before starting Phase 1, confirm:

- [ ] Hardware spec meets the lab minimum (§1.1).
- [ ] Proxmox or Docker host installed and reachable.
- [ ] Public domain registered and DNS delegated to Cloudflare.
- [ ] pfSense VM provisioned with 2 NICs.
- [ ] Wazuh single-node Compose stack starts and the dashboard is reachable.
- [ ] Velociraptor server starts and the GUI loads.
- [ ] SafeLine container starts and proxies a test app.
- [ ] A representative endpoint (Linux + Windows) has Wazuh + Velociraptor
      agents and reports data.
- [ ] Backups configured (§6 of implementation doc).
- [ ] A test attack (e.g. Atomic Red Team T1059.004) generates an alert in
      Wazuh.