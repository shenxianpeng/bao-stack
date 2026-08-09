# Runbook: backup & restore

> Product stance (design D4): **an untested backup is not a backup.**
> Run the rehearsal against a scratch cluster on a schedule, not once.

## 1. How backups work

- Cron on every bao node runs `/usr/local/bin/bao-snapshot`; only the
  **active** node actually snapshots, so the job survives failover with no
  coordination.
- Pipeline: `bao operator raft snapshot save` → client-side encryption with
  the user's **age** public key → copy to `bao_snapshot_target` (local dir in
  the v0.1 scaffold; S3-compatible endpoints planned — Garage/SeaweedFS
  friendly).
- Retention: newest `bao_snapshot_keep` artifacts (default 28 = 7 days at
  6-hour cadence).
- The decryption key belongs to **you**, not to bao-stack. Store it offline;
  losing it makes every backup worthless.

Trigger an immediate snapshot: `ansible-playbook backup.yml`.

## 2. Restore rehearsal (do this regularly)

```bash
ansible-playbook restore.yml -e snapshot_file=/var/backups/openbao/<artifact>
```

Without `confirm_restore=yes` the playbook only verifies the artifact and
prints the restore plan. The full rehearsal is: stand up a scratch cluster
(1 or 3 nodes), run the real restore against it, unseal, and verify a known
secret path is readable. Automating that end-to-end is a v0.1 CI-matrix
scenario (design §8).

## 3. Real restore (destructive)

```bash
ansible-playbook restore.yml \
  -e snapshot_file=/var/backups/openbao/<artifact> \
  -e confirm_restore=yes
```

This replaces the **entire raft state** of the cluster with the snapshot
contents. After restore:

1. Nodes may need re-unseal — see runbook-unseal.md. Note: restoring a
   snapshot from a *different* cluster requires the unseal keys of the
   cluster that **took** the snapshot.
2. Verify `bao operator raft list-peers` and smoke-read a known path.
3. Everything written after the snapshot timestamp is gone (snapshot-level
   granularity; there is no PITR for raft storage — by design).

## 4. Open items (lab phase)

- Snapshot consistency semantics under active writes (design §12.7) — to be
  verified and documented here.
- S3-compatible target wiring (rclone vs aws cli).
- Automatic decrypt step in restore.yml for `.age` artifacts.
