# bao-stack

> **Status: design / lab phase (pre-alpha).** Nothing here is production-ready yet.
> The full design rationale lives in [docs/design-v0.1.md](docs/design-v0.1.md) (Chinese).

**bao-stack** is a batteries-included production distribution for
[OpenBao](https://openbao.org) — bare-metal / VM first, one command to bring up an
HA cluster with backups, monitoring, TLS, and idempotent IaC.
Think *Pigsty for OpenBao*: we don't touch the kernel, we ship the missing
operations layer around it.

```
./configure          # generate config.yml inventory interactively
ansible-playbook install.yml   # INFRA -> NODE -> BAO, end to end
```

## Why

OpenBao's kernel is solid, but the production operations layer is a known gap:
the official Helm chart explicitly excludes monitoring, backup, and upgrades,
and HA init/unseal/join is a manual affair. bao-stack fills the *artifact* slot:
deployment, HA, backup/restore, monitoring/alerting, certificate management, and
rolling upgrades as declarative, idempotent, auditable deliverables.

## Design principles

1. **Bare-metal / VM first** — no Docker, no Kubernetes required.
2. **Declarative + idempotent** — one YAML inventory describes desired state;
   Ansible converges to it; re-runs are safe.
3. **Air-gap ready** — local package repository first; zero external network
   dependency during install.
4. **Upstream first** — generic capabilities go into OpenBao upstream; the
   distribution keeps only the orchestration/integration layer.
5. **Never touches secret content** — we manage process and cluster lifecycle,
   never user secrets; backup artifacts are encrypted by default.
6. **Every module independently adoptable** — use only the dashboards, or only
   the backup playbook, if that's all you need.

## Architecture

Three modules, following Pigsty's modular layout:

| Module | Scope | Contents |
|---|---|---|
| **INFRA** | one per deployment (reusable) | VictoriaMetrics + Grafana + vmalert, local package repo (air-gap source), deployment-level self-signed CA |
| **NODE** | one per host | node_exporter, time-sync checks, system tuning (file handles, mlock) |
| **BAO** | 3- or 5-node cluster | OpenBao server (integrated Raft storage), HAProxy active-node routing, scheduled encrypted snapshots, audit device to disk |

Key decisions (see the design doc for rationale):

- **D1** — Raft integrated storage is the only supported backend in v0.1.
- **D2** — HAProxy routes to the active node via `/v1/sys/health` checks.
- **D3** — Shamir unseal + written runbook by default; auto-unseal optional
  (sovereignty-friendly PKCS#11 route under evaluation).
- **D4** — Backups: scheduled raft snapshot → client-side encryption → local
  path or any S3-compatible endpoint, plus a **restore rehearsal** playbook.
  An untested backup is not a backup.
- **D5** — Monitoring specialized for a secrets service: seal-status flips,
  leader loss, raft peer anomalies, snapshot age, TLS expiry, lease growth,
  audit-device write failures.

## Repository layout

```
configure         # interactive inventory generator -> config.yml
install.yml       # full install: infra -> node -> bao
infra.yml         # INFRA module only
node.yml          # NODE module only
bao.yml           # BAO cluster install / scale-out
backup.yml        # trigger a snapshot now (cron jobs are set up by bao.yml)
restore.yml       # restore (with rehearsal mode)
upgrade.yml       # rolling upgrade: standbys first, then leader step-down
cert.yml          # certificate issue & rotation
roles/            # infra_* / node_* / bao_* roles
files/dashboards/ # Grafana dashboards
files/alerts/     # alert rule sets
docs/             # design doc, runbooks, security model
ci/               # multi-OS matrix integration tests
```

## Non-goals (v0.1)

- Kubernetes deployment (that's the official Helm chart's territory)
- DR / cross-region replication (upstream 2026–2027 roadmap; not promised before GA)
- Multi-cluster federation
- Web management console (Grafana only in v0.1)
- Plugin packaging ecosystem
- Windows

## Security model

The distribution never reads, caches, or forwards secret plaintext; backup
artifacts are encrypted with user-owned keys; every playbook is auditable
(no compiled black boxes); no phone-home. See
[docs/security-model.md](docs/security-model.md).

## License

TBD — under evaluation (AGPLv3 + commercial subscription vs Apache-2.0; see
design doc §12). Until a license is added, all rights reserved.
