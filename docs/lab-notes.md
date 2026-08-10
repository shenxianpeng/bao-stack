# Lab verification notes

> The living ledger for design-doc §12 open questions. Each entry records
> what was tested, against which OpenBao version, and what changed in the
> repo as a result. Milestone 1 deliverable ("踩坑笔记 = playbook 需求规格").

## Environment

- OpenBao **v2.6.1** (linux_amd64 release binary, checksum-verified)
- 3-node raft cluster on localhost (ports 8200/8210/8220), TLS from a lab CA
  generated with the exact openssl flow used by `roles/infra_ca` and
  `roles/bao_config/tasks/tls.yml`

## Verified: health endpoint status codes (§12.3) ✅

| Node state | `GET /v1/sys/health` | notes |
|---|---|---|
| uninitialized | **501** | `?uninitcode=299` override works |
| initialized + sealed | **503** | `?sealedcode=299` override works |
| unsealed active | **200** | |
| unsealed standby | **429** | `?standbyok=true` → 200; `?standbycode=299` works |

Consequences applied:
- HAProxy `http-check expect status 200` (roles/bao_lb) is correct as-is —
  only the active node passes; Vault-era semantics carry over unchanged.
- The snapshot script's "only active proceeds" check (`health == 200`) is
  correct.
- `upgrade.yml` now reads the plain health code (200/429/501/503) to decide
  leadership instead of using override params.

## Verified: telemetry metric names (§12 / D5) ✅

OpenBao v2.6.1 keeps the **`vault_` prefix** (431 of 475 series in a fresh
cluster). Checked against a live `/v1/sys/metrics?format=prometheus`:

| Metric used by bao-stack | exists? |
|---|---|
| `vault_core_unsealed` | ✅ (gauge, label `cluster`) |
| `vault_core_active` | ✅ (1 on active, 0 on standby) |
| `vault_expire_num_leases` | ✅ |
| `vault_audit_log_request_failure` | ✅ |
| `vault_core_handle_request` | ✅ (summary) |
| `vault_raft_peers` | ✅ — **better than counting `up`**, adopted |
| `vault_autopilot_healthy` / `vault_autopilot_failure_tolerance` | ✅ — adopted |
| `vault_core_certificate_expiry_seconds` | ❌ **does not exist** |

Consequences applied:
- `BaoRaftPeerLoss` now uses `max(vault_raft_peers)`.
- Added `BaoAutopilotUnhealthy` (`vault_autopilot_healthy == 0`).
- TLS expiry and snapshot age are exported by a small textfile-collector
  script (`bao-stack-textfile`, cron every 5 min) as
  `bao_stack_cert_expiry_timestamp` / `bao_stack_last_snapshot_timestamp`,
  since the server exposes neither.

## Decided: metrics scrape auth (scrape.yml open item) ✅

`unauthenticated_metrics_access = true` in the listener `telemetry` block
works as documented and is now the bao-stack default
(`bao_metrics_unauthenticated: true`). Rationale: metrics carry no secret
material, deployments are on trusted/air-gapped networks, and a static
metrics token in the scraper is a worse secret-hygiene story. Set the var
to `false` to require a token.

## Verified: cluster mechanics ✅

- **retry_join**: standbys joined raft automatically once unsealed —
  no join playbook step needed, as designed.
- **Snapshot forwarding**: `bao operator raft snapshot save` also succeeds
  against a **standby** (request forwarding), byte-identical output. The
  active-only guard in the cron script stays — it exists to avoid 3×
  duplicate snapshots, not because standbys can't serve them.
- **step-down**: works; leadership moved within ~3 s and health codes
  flipped accordingly (old leader 429, new leader 200).
- **Voter status**: freshly joined followers report `Voter=false` until
  autopilot promotes them (~stabilization window). CI assertions must not
  expect immediate voter status.

## Packaging intel (for infra_repo / ci) 📦

- Upstream release assets are named `openbao_<ver>_linux_<arch>.deb|rpm|tar.gz`
  (project name, not `bao_`), with per-asset **SBOM JSON** and a
  `checksums.txt` — good inputs for our §8 signing/SBOM discipline.
- An **`openbao-hsm`** variant exists per platform — directly relevant to
  the PKCS#11/auto-unseal sovereignty track (§12.2). Not yet tested.

## Still open

- §12.1 storage-backend landscape (docs/community question, not lab)
- §12.2 auto-unseal: test `openbao-hsm` + softhsm2
- §12.7 snapshot consistency under concurrent writes (needs a write-load
  harness; snapshot restore rehearsal automation comes with it)
- S3-compatible snapshot targets (rclone vs aws cli)
- ~~Multi-node CI topology~~ → automated: 3-node cluster in systemd
  containers with leader-kill failure injection (ci/README.md)
