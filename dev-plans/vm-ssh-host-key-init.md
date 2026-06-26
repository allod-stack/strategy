## Implementation Plan — vm-ssh-host-key init command

### Tracking Issue

https://forge.anarch.diy/vnprc/nexus/issues/56

### Goal

Add an `init` command to `vm-ssh-host-key` that generates the first SSH host key for a new VM, so initial provisioning uses the same tooling as key rotation instead of ad-hoc manual steps.

### Scope

**In scope:**
- `nexus/scripts/vm-ssh-host-key` — add `init` command alongside existing `stage`/`activate`/`retire`

**Out of scope:**
- Changes to `stage`, `activate`, or `retire` behavior
- Test infrastructure for vm-ssh-host-key (no existing test file; adding one is a separate effort)

### Interface Contracts

```
vm-ssh-host-key init <target>
```

Preconditions:
- Target must exist in `vm-specs.json` (same as other commands)
- Target IP from `vm-specs.json` must resolve to a non-empty, non-`null` value before key generation or any repo/trust files are written
- Target must NOT exist in `machine-host-keys.json` (no active or staged key)
- `profiles/secrets/${TARGET}-ssh.age` and `.pub` must NOT already exist (inconsistent state if present without JSON entry)
- `/dev/shm` must be available
- `profiles` and `secrets` working trees must be clean

Behavior (reuse existing helpers such as `read_json_field`, `key_material`, and `update_known_hosts`; do not refactor `stage` unless the implementation genuinely needs it):
1. Resolve and validate target IP
2. Generate ed25519 keypair in `/dev/shm` tmpfs
3. Age-encrypt private key → `profiles/secrets/${TARGET}-ssh.age`
4. Copy public key → `profiles/secrets/${TARGET}-ssh.pub`
5. Create entry in `machine-host-keys.json`: `{ "active": "<pubkey>", "staged": null }`
6. Re-encrypt secrets repo with `agenix -r`
7. Update `KNOWN_HOSTS_VMS` with the new key for the target IP

Exit output: print changed files and required manual steps (same format as `stage`). No `credentials.nix` edit is needed — VM host entries are derived from `machine-host-keys.json` automatically.

Differences from `stage`:
- Refuses if target already has an entry (inverse of stage's check)
- Sets key as `active`, not `staged`
- No activate/retire cycle needed afterward
- Known hosts gets one entry (active only), not two (active + staged)

### Agent Gates

- Do not run `vm-ssh-host-key init <real-target>` as part of implementation review. A successful real run generates SSH host key material, rewrites `allod/secrets`, writes `allod/profiles` secrets, and mutates `KNOWN_HOSTS_VMS`; that is a host-side human operation after the command is merged.
- Agents may run package, lint, dispatcher, and isolated temporary-fixture tests that do not touch real Allod secrets or host trust files.

### Acceptance Tests

```bash
cd ~/work/allod/nexus
nix flake check
```

`nix flake check` already runs `bash -n`, `shellcheck -x`, and `test -x` on `vm-ssh-host-key` via the `provisioning-contract` check.

Verify the dispatcher accepts `init` (reaches the inventory validation path, not "unknown command"):

```bash
nix build .#packages.x86_64-linux.provisioning-scripts
err=$(mktemp)
if result/bin/vm-ssh-host-key init __definitely_missing_vm__ 2> "$err"; then
  echo "expected missing inventory failure"
  exit 1
fi
grep -q "not in inventory" "$err"
! grep -q "unknown command" "$err"
```

Run a non-host smoke test against temporary repos so the mutating path is exercised without touching real secrets or `KNOWN_HOSTS_VMS`:

```bash
tmp=$(mktemp -d)
trap 'rm -rf "$tmp"' EXIT

mkdir -p "$tmp/inventory/scripts" "$tmp/profiles/secrets" "$tmp/secrets" "$tmp/bin"
cat > "$tmp/inventory/scripts/repositories.json" <<'JSON'
{"repositories":{}}
JSON
cat > "$tmp/inventory/scripts/vm-specs.json" <<'JSON'
{"init-vm":{"ip":"192.0.2.55"}}
JSON
printf '{}\n' > "$tmp/secrets/machine-host-keys.json"

git -C "$tmp/profiles" init
git -C "$tmp/secrets" init
git -C "$tmp/profiles" -c user.name=test -c user.email=test@example.invalid commit --allow-empty -m init
git -C "$tmp/secrets" add machine-host-keys.json
git -C "$tmp/secrets" -c user.name=test -c user.email=test@example.invalid commit -m init

cat > "$tmp/bin/age" <<'SH'
#!/usr/bin/env bash
set -euo pipefail
out=
while [[ $# -gt 0 ]]; do
  case "$1" in
    -o) out="$2"; shift 2 ;;
    *) shift ;;
  esac
done
[[ -n "$out" ]]
cat > "$out"
SH
chmod +x "$tmp/bin/age"

ssh-keygen -t ed25519 -N "" -C host -f "$tmp/host" -q

PATH="$tmp/bin:$PATH" \
  INVENTORY="$tmp/inventory" \
  MACHINE_PROFILES="$tmp/profiles" \
  IDENTITY_CONFIG="$tmp/secrets" \
  AGE_IDENTITY="$tmp/host" \
  KNOWN_HOSTS_VMS="$tmp/known_hosts_vms" \
  AGENIX=: \
  result/bin/vm-ssh-host-key init init-vm

jq -e '."init-vm".active | startswith("ssh-ed25519 ")' "$tmp/secrets/machine-host-keys.json"
jq -e '."init-vm".staged == null' "$tmp/secrets/machine-host-keys.json"
test -s "$tmp/profiles/secrets/init-vm-ssh.age"
test -s "$tmp/profiles/secrets/init-vm-ssh.pub"
grep -q '^192[.]0[.]2[.]55 ssh-ed25519 ' "$tmp/known_hosts_vms"
```

### Rollback Plan

Revert the commit in nexus. The existing `stage`/`activate`/`retire` commands are unchanged.
