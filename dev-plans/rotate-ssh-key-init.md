## Implementation Plan — rotate-ssh-key init command

### Tracking Issue

TBD — create before implementation starts.

### Goal

Add an `init` command to `rotate-ssh-key` that generates the first SSH host key for a new VM, so initial provisioning uses the same tooling as key rotation instead of ad-hoc manual steps.

### Scope

**In scope:**
- `nexus/scripts/rotate-ssh-key` — add `init` command alongside existing `stage`/`activate`/`retire`

**Out of scope:**
- Changes to `stage`, `activate`, or `retire` behavior
- Test infrastructure for rotate-ssh-key (no existing test file; adding one is a separate effort)

### Interface Contracts

```
rotate-ssh-key init <target>
```

Preconditions:
- Target must exist in `vm-specs.json` (same as other commands)
- Target must NOT exist in `machine-host-keys.json` (no active or staged key)
- `/dev/shm` must be available
- `profiles` and `secrets` working trees must be clean

Behavior (reuses the same helpers as `stage`):
1. Generate ed25519 keypair in `/dev/shm` tmpfs
2. Age-encrypt private key → `profiles/secrets/${TARGET}-ssh.age`
3. Copy public key → `profiles/secrets/${TARGET}-ssh.pub`
4. Create entry in `machine-host-keys.json`: `{ "active": "<pubkey>", "staged": null }`
5. Re-encrypt secrets repo with `agenix -r`
6. Update `KNOWN_HOSTS_VMS` with the new key for the target IP

Exit output: print changed files and required manual steps (same format as `stage`). No `credentials.nix` edit is needed — VM host entries are derived from `machine-host-keys.json` automatically.

Differences from `stage`:
- Refuses if target already has an entry (inverse of stage's check)
- Sets key as `active`, not `staged`
- No activate/retire cycle needed afterward
- Known hosts gets one entry (active only), not two (active + staged)

### Agent Gates

None. All changes are to a single file in the nexus repo. The script runs on the host but isn't executed as part of this plan — it's tested structurally.

### Acceptance Tests

```bash
cd ~/work/allod/nexus
nix flake check
```

Verify the script parses and is packaged:

```bash
nix build .#packages.x86_64-linux.provisioning-scripts
test -x result/bin/rotate-ssh-key
```

Verify `init` is accepted by the dispatcher and usage text:

```bash
result/bin/rotate-ssh-key --help | grep -q init
```

### Rollback Plan

Revert the commit in nexus. The existing `stage`/`activate`/`retire` commands are unchanged.
