# Runbook: init, unseal, and break-glass

> Audience: operators of a bao-stack deployment.
> Default unseal mode is **Shamir** (design D3). Unsealing is a deliberate
> human action; bao-stack never stores unseal keys on cluster nodes.

## 1. First-time initialization

`bao.yml` (role `bao_ha`) initializes the cluster exactly once, on the first
node, and writes the init output — key shares + initial root token — to the
**controller machine** at `secrets/<cluster>-init.json`.

Immediately after init:

1. Distribute the key shares to distinct key holders (default 5 shares,
   threshold 3). One person holding all shares defeats the purpose.
2. **Delete `secrets/<cluster>-init.json`** once the shares are distributed.
3. Revoke or vault away the initial root token after setting up auth methods.

## 2. Unsealing a node

Every process restart (reboot, upgrade, cert rotation) leaves the node sealed.
On the node (or via the API from anywhere that can reach it):

```bash
export BAO_ADDR=https://<node-ip>:8200
export BAO_CACERT=/etc/openbao/tls/ca.crt
bao operator unseal   # repeat until threshold reached (3 by default)
bao status            # verify: Sealed = false
```

Standbys re-join raft automatically (`retry_join` in openbao.hcl) once
unsealed.

## 3. Whole-cluster restart drill

Practice this before you need it (e.g. datacenter power event):

1. Start openbao on all nodes: `systemctl start openbao` — all come up sealed.
2. Unseal each node per §2. The first quorum of unsealed nodes elects a leader.
3. Verify: `bao operator raft list-peers` shows all nodes, one leader.
4. Smoke test a read through the LB endpoint (port 8443).

## 4. Auto-unseal (optional)

`bao_unseal: auto` is a placeholder in v0.1. Sovereignty caveat: auto-unseal
against a US cloud KMS undermines the self-hosting story; the PKCS#11 /
softhsm route is under evaluation (design §12.2). Until that lands, Shamir +
this runbook is the supported path.

## 5. Break-glass notes

- **Lost quorum of key shares**: unrecoverable by design. This is why shares
  must be distributed and inventoried (who holds what, verified quarterly).
- **Lost root token, keys intact**: generate a new root token with
  `bao operator generate-root` (requires key-share quorum).
- **Upstream break-glass tooling** is on the OpenBao Operator Experience
  roadmap; this runbook tracks it (design §7).
