# Integration

How every component in the stack exchanges data with the others. Each section
follows the same template: **direction · protocol · port(s) · payload · auth**.

---

## 1. Endpoints → Wazuh (agent-based)

- **Direction:** Endpoint → Wazuh Manager
- **Protocol:** Custom encrypted binary over TCP
- **Port:** `1514` (enrollment: `1515`)
- **Payload:** OS logs, FIM diffs, SCA results, vulnerability scan
- **Auth:** Pre-shared agent key + manager certificate

Manager-side config snippet:
```xml
<client_buffer>
  <disabled>no</disabled>
  <queue_size>10000</queue_size>
  <events_per_second>1000</events_per_second>
</client_buffer>
```

---

## 2. pfSense → Wazuh (syslog)

- **Direction:** pfSense → Wazuh (via `syslog-ng` relay or direct)
- **Protocol:** Syslog (RFC 5424) over UDP/TCP
- **Port:** `514` (relay preferred for reliability)
- **Payload:** firewall logs, Suricata alerts, DHCP, VPN
- **Auth:** Network ACL on Wazuh collector

pfSense: *Status → System Logs → Settings → Send to remote syslog server*.

---

## 3. Velociraptor agents → Velociraptor server

- **Direction:** Endpoint → Velociraptor server
- **Protocol:** gRPC over TLS
- **Port:** `8000` (client) / `8889` (GUI)
- **Payload:** VQL results, file uploads, hunt artefacts
- **Auth:** Client certificate pinned in `client.config.yaml`

---

## 4. Wazuh → Velociraptor (custom integration)

- **Trigger:** High-severity Wazuh alert (rule level ≥ 12 or specific group).
- **Mechanism:** Custom integration script (Python) calling Velociraptor
  `/api/v1/Hunt` endpoint with a hunt artefact targeting the affected agent.
- **Example integration:**
  `/var/ossec/integrations/custom-virustotal-vrlite.py`
- **Outcome:** Velociraptor launches a hunt, the notebook URL is appended to
  the Wazuh alert as a "reference" field.

```python
# pseudo-integration
import json, requests
alert = json.loads(open(sys.argv[1]).read())
agent_id = alert["agent"]["id"]
requests.post(
    f"{VR_API}/api/v1/Hunt",
    json={"artifacts": ["Windows.System.Persistence"], "hostname": alert["agent"]["name"]},
    headers={"Authorization": f"Bearer {VR_TOKEN}"},
)
```

---

## 5. SafeLine → Wazuh

- **Direction:** SafeLine → Wazuh
- **Protocol:** Syslog + Webhook
- **Port:** `514` (syslog) / HTTPS (webhook)
- **Payload:** Blocked requests, anomalies, challenge failures, audit logs
- **Auth:** Syslog network ACL / webhook HMAC

---

## 6. Cloudflare → Wazuh (Logpush)

- **Direction:** Cloudflare → R2 / S3 → Wazuh indexer (Filebeat input)
- **Protocol:** HTTPS push to bucket; Filebeat tails the bucket
- **Payload:** HTTP requests, WAF events, DNS queries, DDoS events
- **Auth:** Cloudflare API token, R2 access keys

Requires **Pro plan or higher** for Logpush. Free tier alternative: use the
Cloudflare Analytics API polled by a cron → push to Wazuh manager.

---

## 7. Wazuh → TheHive

- **Direction:** Wazuh manager → TheHive
- **Protocol:** HTTPS REST
- **Payload:** Alerts with MITRE ATT&CK tags, observables
- **Auth:** TheHive API key + per-integration org
- **Mechanism:** Official `wazuh/thehive` integration script (Lua + Python).

---

## 8. TheHive ↔ Cortex

- **Direction:** TheHive → Cortex analyzers / responders
- **Protocol:** HTTPS REST
- **Payload:** observables (IP, domain, hash, URL) for enrichment
- **Auth:** Cortex API key + per-user analyser ACL

---

## 9. MISP ↔ stack (phase 2)

- **Direction:** MISP feed export → Wazuh custom rules
- **Mechanism:** Cron pulls MISP events in STIX 2, converts to Wazuh CDB lists.
- **Direction:** Wazuh alerts → MISP events (via TheHive → MISP exporter).
- **Purpose:** Bidirectional threat intel sharing.

---

## 10. Prometheus / Loki / Grafana (phase 2)

- **Direction:** Every component → Prometheus exporters & Loki
- **Mechanism:** `wazuh-prometheus-exporter`, `node_exporter`, Velociraptor
  `/api/v1/metrics`, pfSense `syslog-ng → Loki`.
- **Visualisation:** Grafana SOC dashboard (alert volume, MTTD, MTTR, top rules).

---

## 11. Integration map (visual)

```
                  ┌──────────────┐
                  │  Endpoints   │
                  └──────┬───────┘
            Wazuh agent  │  Velociraptor agent
                ┌─────────┴─────────┐
                ▼                   ▼
        ┌──────────────┐    ┌──────────────┐
        │ Wazuh Mgr    │    │ Velociraptor │
        └──────┬───────┘    └──────┬───────┘
               │ custom integ.     │ artefacts
               └────────┬──────────┘
                        ▼
                ┌──────────────┐
                │  TheHive     │◄── Cortex enrich
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │  Analyst GUI │
                └──────────────┘

       pfSense ──syslog──► Wazuh Indexer
       SafeLine ──syslog──► Wazuh Indexer
       Cloudflare ──R2 ──► Wazuh Indexer (Filebeat)
       All ──Prom/Loki──► Grafana
```

---

## 12. Ports reference

| From → To | Port | Protocol | Purpose |
|-----------|------|----------|---------|
| Endpoint → Wazuh | 1514 | TCP/TLS | Agent logs |
| Endpoint → Wazuh | 1515 | TCP/TLS | Enrollment |
| Endpoint → Velociraptor | 8000 | gRPC/TLS | Agent comms |
| Analyst → Velociraptor | 8889 | HTTPS | GUI |
| Analyst → Wazuh | 443 | HTTPS | Dashboard |
| Analyst → TheHive | 9000 | HTTPS | UI |
| Analyst → Cortex | 9001 | HTTPS | UI |
| Analyst → pfSense | 8443 | HTTPS | GUI |
| Analyst → Grafana | 3000 | HTTPS | UI |
| pfSense → Wazuh | 514 | UDP/TCP | Syslog |
| SafeLine → Wazuh | 514 | UDP/TCP | Syslog |
| Cloudflare → R2 | 443 | HTTPS | Logpush |
| R2 → Wazuh | 9200 | HTTPS | Filebeat → OpenSearch |
| Cloudflare Tunnel | 7844 | TCP | Outbound to edge |

All admin GUIs are placed behind **Cloudflare Zero Trust Access** in production.