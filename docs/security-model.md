# bao-stack security model

> The first document a customer's security team should read.
> Scope: what the distribution does, and — more importantly — what it can
> never do. (Design doc §10.)

## What bao-stack manages

The OpenBao **process and cluster lifecycle**: installation, configuration,
TLS, HA bootstrap, load balancing, scheduled snapshots, monitoring, upgrades.
Nothing else.

## Hard boundaries

1. **No access to secret content.** No playbook, role, script, or dashboard
   reads, caches, logs, or forwards secret plaintext. The access layer
   (HAProxy) runs in TCP passthrough mode — client TLS terminates at OpenBao
   itself, never at the LB.
2. **Unseal material stays with humans.** Shamir key shares and root tokens
   are produced once at init, written only to the operator's controller
   machine with instructions to distribute and delete. They are never stored
   on cluster nodes and never re-read by any playbook.
3. **Backups are encrypted with user-owned keys.** Snapshots are encrypted
   client-side (age) before leaving the node; the private key is generated
   and held by the user. bao-stack has no key escrow.
4. **Everything is auditable.** Pure Ansible YAML, Jinja templates, and shell
   scripts — no compiled black boxes. `grep` is a complete audit tool for
   this repository.
5. **No phone-home.** Nothing in the distribution calls out to any service
   operated by the bao-stack project. Air-gapped installs are a first-class
   path, which makes this externally verifiable.
6. **Kernel bytes untouched.** OpenBao is installed from packages; bao-stack
   never patches or wraps the binary (beyond the standard `cap_ipc_lock`
   file capability for mlock).

## Defense-in-depth defaults

- TLS everywhere: listener TLS from a deployment-scoped CA is mandatory, not
  optional; `cert.yml` rotates certificates without cluster downtime.
- mlock enforced (`LimitMEMLOCK=infinity` + `cap_ipc_lock`), swappiness
  minimized: secrets stay out of swap.
- systemd hardening on the service unit (`ProtectSystem`, `PrivateTmp`,
  `NoNewPrivileges`, capability bounding).
- Audit device on disk with rotation; audit write failures alert at
  `critical` because OpenBao blocks requests when auditing fails.

## Long-term goals

- Reproducible builds of every release artifact.
- Signed artifacts + SBOM for each release (design §8).

## Reporting

Until a dedicated security policy lands, report vulnerabilities via GitHub
security advisories on this repository.
