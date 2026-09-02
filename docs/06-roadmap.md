# Roadmap

The project is split into 6 phases. Each phase ends with a **verifiable
deliverable** and a demo entry in the README "Status" badge.

## Phase 0 — Analysis & feasibility ✅ (current)

Goal: prove the design, no implementation.

Deliverables:
- [x] Repository skeleton
- [x] Global README with stack overview, architecture, lab plan, integration
      matrix, implementation strategy, prerequisites, roadmap
- [x] Per-tool deep-dive (`docs/02-tools-overview.md`)
- [x] Integration matrix (`docs/03-integration.md`)
- [x] Implementation plan (`docs/04-implementation.md`)
- [x] Prerequisites checklist (`docs/05-prerequisites.md`)
- [x] Phased roadmap (this file)
- [x] MIT licence

Exit criteria:
- README renders correctly on GitHub.
- An experienced engineer can read only the README + `docs/01-architecture.md`
  and rebuild the design without further input.

## Phase 1 — Reproducible lab

Goal: spin up the full stack in a single-host lab, end-to-end telemetry works.

Tasks:
- [ ] Ansible playbook to provision the Docker host + Proxmox VMs
- [ ] pfSense VM template + Suricata tuning guide
- [ ] Wazuh single-node Compose + first agent on a Linux endpoint
- [ ] Velociraptor server + 1 hunt artefact + agent on the same endpoint
- [ ] SafeLine in front of DVWA / Juice Shop, public demo through Cloudflare
      Tunnel
- [ ] Cloudflare account + DNS delegated + Tunnel configured
- [ ] End-to-end test: an attack on the victim endpoint appears in Wazuh and
      triggers a Velociraptor hunt.

Exit criteria:
- `make lab-up` brings the stack from zero to fully running in < 30 min.
- Demo GIF / screenshots included in README.

## Phase 2 — Detection engineering

Goal: ship useful, tested detection content.

Tasks:
- [ ] Custom Wazuh rules (≥ 20) mapped to MITRE ATT&CK, with unit tests.
- [ ] Velociraptor hunt artefacts (≥ 5): persistence, lateral movement,
      credential dumping, suspicious services, suspicious binaries.
- [ ] MISP feed integration + IOC matching in Wazuh.
- [ ] SafeLine tuning playbook (false-positive analysis, custom rules).
- [ ] Purple-team exercises with Atomic Red Team.

Exit criteria:
- Detection catalogue documented (`docs/detections/`).
- ≥ 5 Atomic Red Team techniques fire alerts end-to-end.

## Phase 3 — SOAR & incident response

Goal: respond, not just detect.

Tasks:
- [ ] TheHive + Cortex wired to Wazuh.
- [ ] Shuffle SOAR playbooks: phishing triage, account lockout, host
      isolation, brute-force response.
- [ ] IR runbooks (top 5 scenarios: ransomware, BEC, web defacement, DDoS,
      insider threat).
- [ ] Tabletop scenarios and lessons-learned template.

Exit criteria:
- A simulated phishing email → ticket in TheHive → Cortex enriches →
  SOAR playbook auto-isolates the endpoint.

## Phase 4 — Hardening & observability

Goal: prove we measure what we operate.

Tasks:
- [ ] Grafana SOC dashboard (MTTD, MTTR, alert volume, top rules, top
      assets, top attackers).
- [ ] Loki for log aggregation beyond Wazuh (pfSense, SafeLine).
- [ ] CIS benchmark hardening scripts (Ansible roles) for Ubuntu, Windows,
      pfSense.
- [ ] Backup & disaster-recovery runbook, tested quarterly.
- [ ] Threat model (STRIDE) for the stack itself.

Exit criteria:
- All KPIs visible in a single Grafana view.
- DR drill restores the SOC platform in < 4 h from cold backup.

## Phase 5 — Public release & continuous improvement

Goal: showcase on portfolio, accept community contributions.

Tasks:
- [ ] Public GitHub repo with polished README, architecture diagrams
      (Mermaid / Draw.io), screenshots, demo GIFs.
- [ ] Architecture Decision Records (`docs/adr/`).
- [ ] Threat model (`docs/threat-model.md`).
- [ ] Blog post series (or talk submission): "Building an SMB SOC for €0".
- [ ] Issue templates & contribution guide.
- [ ] CI: lint Compose files, validate Wazuh rules XML, link-check Markdown.

Exit criteria:
- Public repo with ≥ 50 stars and ≥ 1 external contribution.
- One published blog post referencing real metrics from the lab.

---

## Milestones summary

| Milestone | Target | Status |
|-----------|--------|--------|
| Design published | end of Phase 0 | ✅ |
| Lab reproducible | end of Phase 1 | ⏳ |
| First detection content | end of Phase 2 | ⏳ |
| First automated response | end of Phase 3 | ⏳ |
| First dashboard + DR drill | end of Phase 4 | ⏳ |
| Public release | end of Phase 5 | ⏳ |

## How to contribute (planned)

- File an issue with a clear description and reproduction steps.
- Open a PR against `main`; CI will run lint + Compose validation.
- For new detection rules, attach a unit test under `tests/detections/`.
- For new integrations, follow the template in `docs/03-integration.md`.

## Risks & mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Upstream breaking changes (Wazuh / OpenSearch) | Medium | High | Pin versions, test in lab before prod |
| Hardware failure (single host lab) | Low | Medium | Daily VM snapshots + off-site backup |
| Contributor availability (one-person project) | Medium | Medium | Document everything; favour stable, boring tools |
| Cloudflare policy / pricing change | Low | Medium | DNS can move to another provider; tunnel optional |
| Community edition EOL | Low | High | Track upstream roadmap; have migration plan |