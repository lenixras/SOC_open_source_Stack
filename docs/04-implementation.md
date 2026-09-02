# Implementation plan (Linux + Docker + VM)

How each component will be deployed in the lab, and what changes in production.

---

## 1. Deployment principles

1. **Reproducible** — every component is launched by a script (Ansible /
   Compose) checked into this repo. Manual clicks are forbidden.
2. **Idempotent** — re-running the deploy must converge, not duplicate.
3. **Hardened by default** — TLS everywhere, no default passwords, syslog
   traffic on private VLAN only.
4. **Observable** — every container exports metrics + structured logs.
5. **Backup-friendly** — all stateful data lives in named volumes or mounts,
   never inside container layers.

---

## 2. Lab deployment (single host, Proxmox + Docker)

### 2.1 Base host

```bash
# Ubuntu 22.04 minimal install
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent chrony curl git

# Docker + Compose
curl -fsSL https://get.docker.com | sudo bash
sudo usermod -aG docker $USER
```

### 2.2 Container plan

```
docker/
├── soc/
│   ├── docker-compose.yml         # Wazuh manager + indexer + dashboard
│   ├── velociraptor/
│   │   └── docker-compose.yml     # VR server + GUI
│   ├── safeline/
│   │   └── docker-compose.yml     # SafeLine community
│   ├── thehive/
│   │   └── docker-compose.yml     # TheHive + Cortex + Postgres
│   └── observability/
│       └── docker-compose.yml     # Grafana + Prometheus + Loki
├── edge/
│   └── cloudflared/
│       └── docker-compose.yml     # Cloudflare Tunnel
└── data/
    └── docker-compose.yml         # Shared Postgres / Redis
```

### 2.3 Wazuh single-node Compose (skeleton)

```yaml
# docker/soc/docker-compose.yml — simplified skeleton, illustrative only
version: "3.9"

volumes:
  wazuh_api_configuration:
  wazuh_etc:
  wazuh_logs:
  wazuh_queue:
  wazuh_var:
  wazuh_indexer_data:
  wazuh_dashboard_data:

services:
  wazuh.manager:
    image: wazuh/wazuh-manager:4.9.0
    hostname: wazuh-manager
    restart: unless-stopped
    ports:
      - "1514:1514"
      - "1515:1515"
      - "514:514/udp"          # syslog intake
    volumes:
      - wazuh_api_configuration:/wazuh-mount/api-configuration
      - wazuh_etc:/wazuh-mount/etc
      - wazuh_logs:/wazuh-mount/logs
      - wazuh_queue:/wazuh-mount/queue
      - wazuh_var:/wazuh-mount/var

  wazuh.indexer:
    image: wazuh/wazuh-indexer:4.9.0
    hostname: wazuh-indexer
    restart: unless-stopped
    environment:
      - OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g
    ulimits:
      memlock: { soft: -1, hard: -1 }
      nofile:  { soft: 65536, hard: 65536 }
    volumes:
      - wazuh_indexer_data:/var/lib/wazuh-indexer

  wazuh.dashboard:
    image: wazuh/wazuh-dashboard:4.9.0
    hostname: wazuh-dashboard
    restart: unless-stopped
    ports:
      - "443:443"
    depends_on: [wazuh.indexer]
```

> Production MUST tune the JVM heap, shard count and ISM retention policy.
> The skeleton above is for the lab; production values are documented in §5.

### 2.4 Velociraptor Compose (skeleton)

```yaml
# docker/soc/velociraptor/docker-compose.yml
services:
  velociraptor:
    image: ghcr.io/velociraptor/velociraptor:latest
    container_name: velociraptor
    restart: unless-stopped
    ports:
      - "8000:8000"   # agent
      - "8889:8889"   # GUI
    volumes:
      - ./config:/etc/velociraptor
      - velociraptor_data:/velociraptor
```

Config files (`server.config.yaml`, `client.config.yaml`) are generated with
`velociraptor config generate` and committed **without** keys/tokens.

### 2.5 SafeLine Compose (skeleton)

```yaml
# docker/soc/safeline/docker-compose.yml — see Chaitin docs for actual values
services:
  safeline:
    image: chaitin/safeline-ce:latest
    container_name: safeline
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "9443:9443"     # admin UI
    volumes:
      - safeline_data:/data
```

### 2.6 Cloudflare Tunnel

```yaml
# docker/edge/cloudflared/docker-compose.yml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=${CF_TUNNEL_TOKEN}
    restart: unless-stopped
```

`CF_TUNNEL_TOKEN` is read from `.env` (gitignored).

---

## 3. pfSense deployment

1. Download the latest pfSense CE ISO from Netgate.
2. Create a Proxmox VM: 2 vCPU, 2 GB RAM, 8 GB disk, 2 NICs (`vmbr0` WAN, `vmbr1` LAN).
3. Boot the ISO, run the installer, enable AES-NI if the CPU supports it.
4. Post-install: assign interfaces, set WAN/LAN IPs from the VLAN plan
   (see `docs/01-architecture.md` §2).
5. Install packages: **Suricata**, **pfBlockerNG**, **System Patches**, **Wake on LAN**.
6. Enable syslog forwarding to Wazuh collector (`Status → System Logs → Settings`).
7. Configure Suricata in **inline IPS** mode (start in IDS only for a week).
8. Snapshot the VM after first successful boot.

---

## 4. Endpoint agents

### Linux
```bash
# Wazuh
curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.9.0-1_amd64.deb
sudo WAZUH_MANAGER="10.10.40.10" dpkg -i ./wazuh-agent.deb
sudo systemctl enable --now wazuh-agent

# Velociraptor
sudo velociraptor --config /etc/velociraptor/client.config.yaml client
```

### Windows
- Wazuh agent: MSI from `packages.wazuh.com`, deployed via GPO / Intune.
- Velociraptor: `velociraptor.exe service install` + config push.

---

## 5. Production deltas

| Area | Lab | Production |
|------|-----|------------|
| Wazuh manager | 1 node | 3-node cluster behind HAProxy |
| Wazuh indexer | 1 node | 3 master + 2 data, ISM cold tier to NFS |
| Velociraptor | 1 server | 2 servers + LB, offline-capable agents |
| TheHive / Cortex | single instance | HA pair + nightly DB backup to offsite |
| Postgres | container | managed (RDS / Cloud SQL) or replicated VM |
| pfSense | 1 VM | CARP pair (active/passive) |
| TLS certs | self-signed / Let's Encrypt staging | Let's Encrypt DNS-01 + HSTS |
| Secrets | env vars in `.env` | HashiCorp Vault / SOPS-encrypted secrets |
| Observability | Grafana local | Grafana Cloud or self-hosted with SSO |

---

## 6. Backup & disaster recovery

| Data | Method | Frequency | Retention |
|------|--------|-----------|-----------|
| Wazuh manager config | `tar` + Ansible-pull | Daily | 30 days |
| Wazuh indexer snapshots | Curator / ISM snapshot | Daily | 7 days hot, 90 days cold |
| Velociraptor datastore | `velociraptor datastore backup` | Daily | 30 days |
| TheHive / Cortex DB | `pg_dump` | Hourly | 7 days |
| pfSense config | Auto Config Backup pkg | Daily | 90 days |
| SafeLine config | Volume snapshot | Daily | 30 days |
| Cloudflare Tunnel token | Vault / sealed-secret | On change | Indefinite |

A quarterly DR drill must restore the SOC platform from a cold backup in < 4 h.

---

## 7. Upgrade strategy

- **Wazuh:** minor upgrades monthly; major every 12–18 months. Always read
  release notes, test in lab, snapshot manager before upgrade.
- **Velociraptor:** weekly check of upstream; agent auto-updates.
- **OpenSearch / Wazuh Indexer:** sequential, never jump 2 majors.
- **TheHive / Cortex:** align versions; matrix on their docs.
- **pfSense:** CE updates ~ quarterly; only patch level on production.
- **SafeLine:** image tag pinned, monthly rebuild.
- **Cloudflared:** `latest` is fine — single binary.

---

## 8. Out-of-scope for now

- Multi-tenant / MSSP mode.
- Hardware Security Module (HSM) integration.
- Kubernetes-based deployment (Compose is sufficient for SMB scale).
- Air-gapped deployment (requires offline feeds repository).