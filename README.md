# RvB Blue Team Runbooks

Operational defense runbooks authored for the **SWIFT × ISACA Red vs Blue (RvB)** competition at **Cal Poly Pomona**, February 28, 2026.

I served as **Team Lead** for Team 2. This repository contains the two machine runbooks I personally wrote ahead of the competition; the remaining machines in our environment were covered by runbooks authored by my teammates using the same structure.

---

## What is Red vs Blue?

Red vs Blue is a live cyber-defense competition. Each blue team is handed a set of intentionally vulnerable machines and must keep specific **scored services** (SSH, DNS, web, FTP, etc.) online while a **red team** actively attempts to compromise, disrupt, and maintain persistence on those machines.

A scoring engine checks each service on a fixed interval. Teams earn points for every successful check and lose points for downtime, so the core challenge is twofold: **harden each box fast enough** to survive the opening minutes, then **monitor and respond** to live intrusion attempts without ever taking your own scored service offline. Teams are also graded on **injects** — business-style tasks (memos, reports, technical questions) delivered mid-competition and scored separately from service uptime.

---

## Environment

Our team defended a five-machine Proxmox cluster:

| Machine | IP | OS | Scored Service | Runbook |
|---|---|---|---|---|
| Tweek Bros Coffeehouse | `192.168.1.13` | Debian 12 | SSH (port 22) + DB | [📄 Runbook](./runbooks/Team2-TweekBros-Runbook.md) — *mine* |
| Tegridy Farms | `192.168.1.15` | Kubuntu | DNS (port 53) | [📄 Runbook](./runbooks/Team2-TegridyFarms-DNS-Runbook.md) — *mine* |
| UFO | `192.168.1.14` | Debian 12 | Flask (port 80) | teammate |
| City Wok DC | `192.168.1.111` | Win Server 2022 | WordPress (port 80) | teammate |
| City Sushi FS | `192.168.1.12` | Win Server 2022 | FTP (20/21) | teammate |

> The Tweek Bros box also hosted the **database that the UFO Flask app depended on**, so defending it effectively protected two scored services — a key reason it received the deepest runbook.

---

## Runbooks in this repository

### [Tweek Bros Coffeehouse — SSH + Database (Debian 12)](./runbooks/Team2-TweekBros-Runbook.md)

The most in-depth runbook of the set, reflecting the machine's dual role (scored SSH service plus the database backing a second team's web app). Covers:

- Time-boxed Phase 1 hardening sequence (inventory baseline, SSH hardening, password rotation + PCR, firewall, Fail2Ban, database lockdown, backdoor sweep, binary integrity hashing)
- Ongoing Phase 2 monitoring loop (authorized-keys and crontab persistence checks, process diffing against baseline, connection monitoring, rootkit scanning)
- A dedicated **Living Off The Land (LOTL) defense** section covering bash `/dev/tcp` reverse shells, socat, key drops, cron persistence, and Python one-liners — threats that automated reverse-shell killers miss
- Emergency procedures and a quick-command reference

### [Tegridy Farms — DNS (Kubuntu)](./runbooks/Team2-TegridyFarms-DNS-Runbook.md)

A focused runbook for a single scored DNS service. Covers the Phase 1 hardening sequence, a DNS-specific health and troubleshooting loop (`bind9` status, `named-checkconf`, port 53 verification), persistence checks, and emergency procedures.

> **A note on depth:** runbook detail was deliberately scaled to each machine's attack surface and criticality rather than written to a fixed template. The database-backed SSH box warranted LOTL defense and integrity baselining; a single-service DNS box on Kubuntu did not. Matching effort to risk was an explicit triage decision as team lead.

---

## Design principles

These runbooks were written to be **executed under pressure by a teammate**, not just read. That shaped a few consistent choices:

- **Strict ordering.** Phase 1 steps run in a fixed sequence because order matters — for example, rotating passwords before submitting a PCR, or opening a second SSH session to verify connectivity *before* applying a firewall that could lock you out.
- **Self-recovery built in.** Every destructive change (firewall, SSH config) is paired with a verification step and a documented way back in via the Proxmox console.
- **Scoring-aware.** Because a password change without a corresponding PCR submission silently fails service checks, that dependency is called out at every relevant step.
- **Plain language for non-specialists.** The DNS runbook in particular is written so a teammate less familiar with that service can keep it up and know exactly when to escalate.

---

## Repository structure

```
RvB-BlueTeam-Runbooks/
├── README.md
└── runbooks/
    ├── Team2-TweekBros-Runbook.md        # SSH + DB (Debian 12)
    └── Team2-TegridyFarms-DNS-Runbook.md # DNS (Kubuntu)
```

---

## Notes

- Internal addresses, credentials, and competition-specific script URLs have been removed or replaced with placeholders. The runbooks reference automation scripts (`firstrun.sh`, `inventory.sh`, etc.) by role rather than including their full source.
- This repository documents **defensive** operations within an authorized, sanctioned competition environment.
