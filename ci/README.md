# CI matrix (design §8)

Release discipline is the core differentiator versus consulting-only vendors:
**every new OpenBao release → full CI matrix regression → compatibility
statement → version-pinned bao-stack release.**

## Planned matrix

| Axis | Values (v0.1) |
|---|---|
| OS | Debian 12, Ubuntu 22.04/24.04, RHEL-compatible 9 |
| Topology | single node, 3-node HA |
| Scenario | fresh install, scale-out, backup + restore rehearsal, rolling upgrade, node failure injection |

## Scenario definitions

1. **fresh install** — `./configure` + `install.yml` on clean hosts; assert
   cluster initialized, all nodes join raft, LB routes to active.
2. **scale-out** — add 2 hosts to a 1-node inventory, re-run `bao.yml`;
   assert 3 raft voters.
3. **backup/restore rehearsal** — write a marker secret, `backup.yml`,
   restore onto a scratch cluster, assert the marker is readable.
4. **rolling upgrade** — `upgrade.yml` from previous pinned version; assert
   zero failed requests through the LB during the roll (single in-flight
   blip allowed at step-down; exact SLO defined in lab phase).
5. **failure injection** — kill the active node; assert leader election and
   LB re-route within the health-check window.

## Release artifacts

All release artifacts are signed and ship with an SBOM. Index publication is
collect-then-swap (no half-published releases), following the immutable-
artifact discipline referenced in the design doc.

## Status

- **Running today** (`.github/workflows/lint.yml` + `integration.yml`):
  - lint: yamllint + ansible-lint (production profile) + shellcheck.
  - **fresh-install, single node** (`ci/inventory-ci.yml`) — real playbooks
    against the runner: CA bootstrap → TLS issuance → install → init →
    unseal → LB routes to active → token-less metrics → encrypted
    snapshot → decrypt & verify.
  - **HA cluster, 3 nodes** (`ci/inventory-ci3.yml`) — three systemd
    containers on a fixed-IP docker network standing in for VMs (docker is
    the CI harness here, not a product dependency): cluster forms via
    retry_join → exactly one active + two standbys → autopilot promotes
    3 raft voters → HAProxy routes to active → **leader kill** → survivor
    elected within the timeout → surviving node's HAProxy re-routes.
- **Next**: scale-out, restore rehearsal, rolling upgrade; OS axis beyond
  Ubuntu (milestone gate 2: "MVP comes up with one command *and passes the
  CI matrix*").
