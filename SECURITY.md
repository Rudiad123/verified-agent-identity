# Security Policy

This document describes the security model of the `verified-agent-identity` skill, the threats it does and does not defend against, and the rationale behind design decisions that may surface in automated security scans.

## Scope

`verified-agent-identity` is a **local CLI skill**. It runs on a single operator's host, creates a decentralized identity (DID) for an AI agent, signs challenges with the agent's private key, and persists state under `~/.openclaw/billions/`. It is not a network service, has no listening port, and does not provide multi-tenant trust boundaries.

The only secret it manages is the agent's identity private key, stored in `~/.openclaw/billions/kms.json`.

## Threat Model

**In scope:**

- Preventing the identity key from being accidentally committed into the workspace or read by tools that operate inside the project directory.
- Protecting the key against casual disclosure on a single-user host (e.g. shoulder-surfing, accidental file sharing, careless backups).
- Preventing operator mistakes that would let an identity key double as an asset-holding wallet key.
- Providing opt-in at-rest encryption for shared/multi-user hosts and for environments where compliance requires it.

**Out of scope:**

- An attacker with read access to the operator's home directory or process memory. This is equivalent to full host compromise; no local secret-storage scheme defends against it without an external HSM or OS keystore, and integrating those would expand the dependency surface beyond what this skill commits to.
- Full-disk forensic recovery on a host the attacker physically controls.
- Hostile code already running with the operator's privileges.

## Storage Modes

Private keys are written to `~/.openclaw/billions/kms.json` in one of two formats, selected by the presence of the `BILLIONS_NETWORK_MASTER_KMS_KEY` environment variable.

| `BILLIONS_NETWORK_MASTER_KMS_KEY` | `provider` on disk | `key` value on disk     | Posture                      |
| --------------------------------- | ------------------ | ----------------------- | ---------------------------- |
| Not set                           | `"plain"`          | Raw hex string          | Acceptable on a single-user host with `chmod 700 ~/.openclaw/billions`. |
| Set                               | `"encrypted"`      | `iv:authTag:ciphertext` | **Recommended for all deployments.** AES-256-GCM at rest. |

Mode is selected per-write, so an operator can switch from `plain` to `encrypted` at any time by exporting the variable before the next key creation or import — no migration step is required.

## Compensating Controls

The following mitigations are present in the codebase and the documented installation flow:

- **Out-of-workspace storage.** Keys live under `~/.openclaw/billions/`, never inside the project directory. Tools (and the agent itself) that operate inside the workspace cannot read or exfiltrate them.
- **Filesystem hardening.** The README instructs the operator to run `chmod 700 ~/.openclaw/billions` after the first run (`README.md` → "Key Storage and Isolation").
- **Dedicated-key warning.** The README warns the operator never to import an Ethereum wallet key that holds assets, only a dedicated identity key (`README.md` step 2 warning under the Human CTA).
- **At-rest encryption available behind one env var.** AES-256-GCM is provided via `BILLIONS_NETWORK_MASTER_KMS_KEY`. No code change, no migration, no extra dependency.
- **Versioned on-disk format.** Each `kms.json` entry carries a `version` and `provider` field, so future format upgrades (e.g. an OS-keystore provider) can ship without breaking existing installs. Legacy entries auto-migrate on next write (see `scripts/shared/storage/keys.js`, `_decodeEntry` legacy branch).

## Scanner Findings — Acknowledged Risks

### Identity and Privilege Abuse — `scripts/shared/storage/keys.js` (plaintext storage branch)

> "When no master KMS key is configured, the key-storage code writes the private key value directly to disk as plaintext."

**Status: acknowledged, accepted with documented mitigations.**

This is the documented `provider: "plain"` mode described in the [Storage Modes](#storage-modes) section above. It is the **default only because the variable is unset**; setting `BILLIONS_NETWORK_MASTER_KMS_KEY` switches the same code path to AES-256-GCM with no further action by the operator. The README places an explicit `> Note` block before any key-creation command instructing the operator to set the variable.

The threat the plaintext mode enables — **local read of `~/.openclaw/billions/kms.json` on the operator's own host** — falls under the out-of-scope items above. An attacker with that level of access has equivalent access to the operator's shell history, SSH agent, browser-stored secrets, and process memory; defending only the identity key under that threat model is not coherent without an external keystore, which this skill does not depend on.

The plaintext mode is retained because:

1. It allows zero-config local development and CI smoke tests without committing or fetching a master secret.
2. It preserves backward compatibility with existing `kms.json` files written by earlier versions of the skill.
3. The same code path becomes encrypted at rest the moment the env var is set — there is no separate "secure mode" to migrate to.

The `Operator Checklist` below is the recommended deployment posture.

## Operator Checklist

1. **Set the master key first.**
   ```bash
   export BILLIONS_NETWORK_MASTER_KMS_KEY="<a strong secret>"
   ```
   Do this **before** the first `node scripts/createNewEthereumIdentity.js`. Keys created without it are written as `provider: "plain"`.
2. **Use a dedicated identity key.** Never reuse an Ethereum private key that holds assets. If the `kms.json` file is exposed, every key inside it should be revocable / disposable.
3. **Restrict the storage directory.**
   ```bash
   chmod 700 ~/.openclaw/billions
   ```
4. **Back up the master key out of band.** If `BILLIONS_NETWORK_MASTER_KMS_KEY` is lost, every entry written under `provider: "encrypted"` is unrecoverable.

## Reporting a Vulnerability

Please report suspected vulnerabilities privately to the Billions Network security contact rather than filing a public issue. Open an issue marked `security` requesting a private disclosure channel if you do not already have one.
