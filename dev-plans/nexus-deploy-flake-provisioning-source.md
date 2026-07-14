# Derive VM Target, Username, and Host-Key Pin from the Deploy Flake

## Tracking Issue

[allod/nexus#5](https://forge.anarch.diy/allod/nexus/issues/5) — "Derive VM target
and host-key pin from the deploy flake in rebuild-vm-from-host and
provision-vm-from-host". Single implementation PR expected; it carries
`Closes allod/nexus#5`. If the diff grows past comfortable review size, split into
(1) `rotation-common.sh` new resolvers + unit tests and (2) wiring the two scripts +
their tests, with the closing keyword on the second (final) PR only.

## Goal

Invoking `rebuild-vm-from-host <vm>` or `provision-vm-from-host <vm>` derives the
target IP, username, and host-key pin from a single deploy-flake reference and needs
no manual `INVENTORY=`, `MACHINE_PROFILES=`, or `IDENTITY_CONFIG=`.

## Scope

In scope:

- `scripts/lib/rotation-common.sh`: add pinned-rev resolvers for IP and username; change
  the host-key pinning helpers to take a resolved secrets revision instead of deriving
  it from a profiles checkout. Exact function contracts in Interface Contracts.
- `scripts/rebuild-vm-from-host`: replace the three-checkout env block with one
  `DEPLOY_FLAKE` reference; derive IP, username, and the presented-key pin check from
  the deploy flake's locked `inventory` and `secrets` revisions; build
  `--flake "${DEPLOY_FLAKE}#${VM_NAME}"`.
- `scripts/provision-vm-from-host`: same env replacement; read the target IP and the
  age-encrypted host-key secret at the pinned `inventory`/`secrets` revisions; keep the
  fail-closed pin assert before `nixos-anywhere`; build `--flake "${DEPLOY_FLAKE}#..."`.
- Tests: `tests/rotation-common.sh`, `tests/rebuild-vm-from-host.sh`,
  `tests/provision-vm-from-host.sh`, and the shared `tests/provisioning-contract.sh`,
  extended per the test matrix. `tests/rotate-token.sh` must still pass unchanged.

Out of scope:

- `scripts/nexus-host-key`, `scripts/vm-ssh-host-key`, `scripts/forge-ssh-key`,
  `scripts/bootstrap-vm-from-host.sh`, `scripts/verify-vm-from-host`, and `scripts/new-vm`.
  The first five share the global-based `resolve_target_ip` / `resolve_target_user` /
  `resolve_forge_connection` helpers, which this plan deliberately leaves signature-stable
  (see Interface Contracts); `new-vm` reads the inventory working tree directly for the
  DHCP reservation. Migrating them to the deploy-flake source is a follow-up, not this
  issue — but because provision invokes `new-vm`, bootstrap, and verify as children inside
  an in-scope run, the per-script wiring below keeps one provisioning run coherent (IP
  preflight, `INVENTORY` export, pinned target passed through) without touching their code.
- The follow-up "expose IP and host keys as flake outputs and drop side checkouts
  entirely" alternative from the issue. Rejected for now: it needs read-only outputs
  added to the profiles, inventory, and secrets flakes — a larger cross-repo change.
- The deploy-flake composition itself, the profiles `profileDefinitions` model, and the
  inventory and secrets data schemas.
- Host-side live rebuild/provision against a real machine (Agent Gate below).

## Risk Assessment

Residual risk: R3 High.

Why:

- Blast radius is provisioning tooling on a security/auth boundary: these scripts decide
  which host `nixos-rebuild switch` / `nixos-anywhere` land on, and the host-key pinning
  preflight is the control (per architecture principle 7, host keys are pinned, never
  TOFU) that stops a rebuild from trusting an unexpected or absent SSH host key. The
  change re-sources the inputs of that control, so a regression could silently weaken it.
- Identity now converges: after this change the target IP (inventory pin), the username
  (secrets pin), the presented/derived host-key material check (secrets pin), and — in
  provision — the injected age host-key secret (secrets pin) all flow from one deploy
  flake. That is the intended single-source-of-truth win (principle 8) but concentrates
  the failure surface into the lock-resolution and pinned-read path.
- Validation lowers this substantially: the existing harness already exercises the pin
  path in fixtures (fixture git repos + a `flake.lock` JSON + stubbed `nix`/`git`/
  `ssh-keyscan`/`nixos-rebuild`), and the plan requires a fail-closed regression test on
  the derived path plus an anti-drift test (working tree on a different commit than the
  pin still yields pinned content). The security property is testable without a live host.
- Rollback confidence is high: the change is tracked shell scripts and tests; revert is a
  straight `git revert` with no persistent state, secrets, or generated artifacts to
  unwind. Not R4: the boundary is preserved and regression-tested, the one irreversible
  action (a live rebuild) is an explicit, human-gated, observable step, and nothing here
  mutates unique or hard-to-reconstruct state.

Human scrutiny:

- First, the generated/derived pin path: confirm `assert_any_vm_host_key_material_pinned`
  and `assert_vm_host_key_material_pinned` still fail closed when the presented or derived
  key is absent from, or mismatched against, the material at the pinned secrets rev — read
  the new test assertions, not just that the suite is green.
- Before the first live run: against a real (private) deploy flake, confirm the derived
  IP, username, and host-key material at the pinned revs match the intended target.
- The username eval path (`nix eval` at a pinned git rev) and the dropped implicit
  `git pull` of the build flake (see Interface Contracts) — confirm both behave as
  intended on the host.

## Interface Contracts

### Environment surface (both scripts)

Replace the current `INVENTORY` / `MACHINE_PROFILES` / `IDENTITY_CONFIG` /
`REGISTRY` / `SPECS` block with, using the same `${VAR:-default}` override shape:

```sh
DEPLOY_FLAKE="${DEPLOY_FLAKE:-$HOME/work/allod/profiles}"          # build target + pin source
INVENTORY_CHECKOUT="${INVENTORY_CHECKOUT:-$HOME/work/allod/inventory}"  # location only; content read at pin
SECRETS_CHECKOUT="${SECRETS_CHECKOUT:-$HOME/work/allod/secrets}"        # location only; content read at pin
INVENTORY_REV="$(resolve_flake_input_locked_rev "${DEPLOY_FLAKE}/flake.lock" inventory)"
SECRETS_REV="$(resolve_flake_input_locked_rev "${DEPLOY_FLAKE}/flake.lock" secrets)"
```

- `DEPLOY_FLAKE` defaults to the public `profiles` checkout, which already pins both
  `inventory` and `secrets` as direct root git inputs and builds every machine — so the
  public template invocation `rebuild-vm-from-host <vm>` needs zero env. An operator
  points `DEPLOY_FLAKE` at their private deploy flake to deploy real machines.
- Checkout locations are resolved by convention/env, not from the lock, because the
  public deploy flake pins `inventory`/`secrets` by **remote forge URL**, not a local
  path. Revisions come from the lock; `ensure_git_commit_available` fetches a pinned rev
  into the local checkout when it is missing (same pattern the host-key path already
  uses). Drift is gone because content is read at the pin, never from the working tree.
- These two scripts no longer source `scripts/lib/resolve-repos.sh` and no longer read
  `repositories.json`; the env overrides above replace the registry indirection. The
  out-of-scope scripts keep their current registry usage.
- Drop the implicit `git pull` of the build flake that the scripts do today (rebuild
  `cd "$MACHINE_PROFILES"; git pull`; provision `git -C "$MACHINE_PROFILES" pull` and
  `git -C "$IDENTITY_CONFIG" pull`). The deploy flake is the operator's composition and
  its lock is authoritative; auto-advancing it past the tested pin at rebuild time
  contradicts single-source-of-truth. The operator updates the deploy flake deliberately.

### `rotation-common.sh` functions

Unchanged, now also exercised for `inventory` and for the pinned age-secret read:

- `resolve_flake_input_locked_rev(flake_lock, input_name)` — already a generic resolver;
  now called for both `secrets` and `inventory`.
- `ensure_git_commit_available(repo, rev, label)` — now also called for the inventory
  checkout and before provision's pinned age-secret read.

Changed signatures (only callers are the two in-scope scripts, so safe to change):

| Function | Old inputs | New inputs |
|---|---|---|
| `machine_host_key_materials_at_secrets_pin` | `(vm, profiles_checkout, secrets_checkout)` — derives secrets rev from `profiles_checkout/flake.lock` | `(vm, secrets_checkout, secrets_rev)` — takes the resolved rev |
| `die_vm_host_key_material_not_pinned` | `(vm, material, profiles_checkout, secrets_checkout)` | `(vm, material, secrets_checkout, secrets_rev)` |
| `assert_any_vm_host_key_material_pinned` | `(vm, materials, profiles_checkout, secrets_checkout)` | `(vm, materials, secrets_checkout, secrets_rev)` |
| `assert_vm_host_key_material_pinned` | `(vm, material, profiles_checkout, secrets_checkout)` | `(vm, material, secrets_checkout, secrets_rev)` |

The pin logic and refusal message are otherwise preserved verbatim, including the
"Update the secrets input in profiles, commit, push, then retry" guidance and the
`secrets@<rev>` / `flake.lock` context. This decouples the security assert from
flake-lock parsing: the script resolves the rev once, the assert verifies against it.

New functions (used only by the two in-scope scripts):

- `resolve_target_ip_at_pin(vm, inventory_checkout, inventory_rev)` — `ensure_git_commit_available`
  then `git show "${rev}:scripts/vm-specs.json"` piped to `jq -r --arg v "$vm" '.[$v].ip'`;
  dies with an "unknown VM" message when the entry is absent and a "no target IP" message
  when the IP is null/empty (folding in the existence checks the scripts do inline today).
- `resolve_target_user_at_pin(vm, secrets_checkout, secrets_rev)` — `ensure_git_commit_available`
  then `nix eval --raw "git+file://${secrets_checkout}?rev=${secrets_rev}#lib.vmUsernames.${vm}"`;
  dies on eval failure. Reads username at the pinned rev for consistency with IP and
  host-key material. Rationale: `lib.vmUsernames` lives in the secrets flake and the
  public profiles flake does not re-export it, so evaluating at the pin avoids imposing a
  re-export contract on the deploy flake (which would be the out-of-scope flake-outputs
  route). Tradeoff: a git-rev `nix eval` is heavier than evaluating a working-tree path;
  acceptable for an occasional provisioning command.

Left signature-stable on purpose (shared with out-of-scope scripts
`nexus-host-key`, `vm-ssh-host-key`, `forge-ssh-key`): `resolve_target_ip(target)`
(reads global `$SPECS`), `resolve_target_user(target)` (evals `$IDENTITY_CONFIG`),
`resolve_forge_connection()` (evals `$IDENTITY_CONFIG`; not called by either in-scope
script). Adding `_at_pin` siblings rather than mutating these keeps this issue's blast
radius to the two provisioning scripts.

### Per-script wiring

- `rebuild-vm-from-host`: `TARGET="${2:-$(resolve_target_ip_at_pin "$VM_NAME" "$INVENTORY_CHECKOUT" "$INVENTORY_REV")}"`;
  `VM_USERNAME="$(resolve_target_user_at_pin "$VM_NAME" "$SECRETS_CHECKOUT" "$SECRETS_REV")"`;
  scan presented keys as today, then
  `assert_any_vm_host_key_material_pinned "$VM_NAME" "$PRESENTED..." "$SECRETS_CHECKOUT" "$SECRETS_REV"`;
  `nixos-rebuild switch --flake ".#${VM_NAME}"` becomes `--flake "${DEPLOY_FLAKE}#${VM_NAME}"`
  (no `cd`). Strict SSH options (`-i`, `StrictHostKeyChecking=yes`, the VMs known_hosts)
  are unchanged.
- `provision-vm-from-host`: `TARGET="$(resolve_target_ip_at_pin "$VM_NAME" "$INVENTORY_CHECKOUT" "$INVENTORY_REV")"`;
  resolve the bootstrap gate at the pin in the same breath — provision decides near its end
  whether to run bootstrap/verify from `FORGE_KEY=$(jq -r ".\"${VM_NAME}\".forge_key" "$SPECS")`,
  a working-tree `$SPECS` read this plan deletes from the environment surface, so replace it with
  `FORGE_KEY="$(git -C "$INVENTORY_CHECKOUT" show "${INVENTORY_REV}:scripts/vm-specs.json" | jq -r --arg v "$VM_NAME" '.[$v].forge_key')"`
  resolved up front next to `TARGET` (null still means privacy VM: skip bootstrap/verify as today;
  leaving the late read unrewired would abort every dev-VM provision on an unbound `$SPECS` after
  `nixos-anywhere` has already installed);
  read the age host-key secret at the pin —
  `git -C "$SECRETS_CHECKOUT" show "${SECRETS_REV}:secrets/vm-host-keys/${VM_NAME}-ssh.age"`
  piped into `age --decrypt` (preserve piping, never command substitution, so the trailing
  newline survives) after `ensure_git_commit_available`; derive the public key and call
  `assert_vm_host_key_material_pinned "$VM_NAME" "$DERIVED_HOST_MATERIAL" "$SECRETS_CHECKOUT" "$SECRETS_REV"`
  before `nixos-anywhere`; build `--flake "${DEPLOY_FLAKE}#${VM_NAME}"`. Reading the age
  secret at the pin (rather than the working tree) means a machine whose key material is
  not yet in the pinned secrets rev fails loud here — correct, and the pin assert remains
  the fail-closed backstop regardless.
- `provision-vm-from-host`, single-run coherence with its children: `new-vm` derives the
  DHCP reservation (IP + MAC) from the inventory **working tree**, while `TARGET` is now
  pinned — on a drifted checkout, `--replace` would destroy the existing VM and then hang
  forever at "Waiting for installer SSH" on an address the VM never receives. Today's
  script cannot hit that split-brain (both reads are the same working tree); the pin change
  introduces it, so preflight before `new-vm`: compare the pinned IP against
  `jq -r --arg v "$VM_NAME" '.[$v].ip' "${INVENTORY_CHECKOUT}/scripts/vm-specs.json"` and
  die on mismatch naming both IPs and the fix (pull/sync the inventory checkout, or update
  the deploy flake). Export `INVENTORY="$INVENTORY_CHECKOUT"` so `new-vm`,
  `bootstrap-vm-from-host.sh`, and `verify-vm-from-host` (which keep their own `INVENTORY`
  defaults) read exactly the checkout the preflight validated, and pass the pinned
  `TARGET` as the second argument to bootstrap and verify (both already accept a target-ip
  override) so one run cannot install to the pinned IP and then bootstrap a drifted one.
  The children's remaining working-tree reads (username, forge key state, known_hosts
  material) stay with the follow-up migration.

## Agent Gates

- The agent cannot perform the live end-to-end confirmation. Provisioning is Nexus-only
  and host-side (architecture principle 6; `vm-provisioning.md` "Host-side only"), and
  agents run inside dev VMs. The human runs the first `rebuild-vm-from-host <vm>` and
  `provision-vm-from-host <vm>` with the new derivation against a real deploy flake and a
  real target. This blocks final sign-off but not implementation: the fixture-based
  acceptance tests below are the agent's ceiling and cover the security-critical pin path.
- No repo creation, no secrets changes, no permission grants, no host-only commands.

## Acceptance Tests

All fixture-based, extending the existing harness. Run before declaring complete:

```sh
cd <allod/nexus checkout>
for t in tests/rotation-common.sh tests/rebuild-vm-from-host.sh \
         tests/provision-vm-from-host.sh tests/provisioning-contract.sh \
         tests/rotate-token.sh; do
  echo "== $t =="; bash "$t" || exit 1
done
```

Test matrix (new or adapted cases):

- `tests/rotation-common.sh`:
  - `resolve_flake_input_locked_rev` resolves `inventory` (not only `secrets`) from a lock
    that pins both; existing negative lock-shape cases keep passing.
  - `resolve_target_ip_at_pin` returns the IP recorded at the pinned inventory rev even
    when the inventory working tree is checked out at a different commit (anti-drift), and
    dies with "unknown VM" (absent entry) and "no target IP" (null IP).
  - `resolve_target_user_at_pin` evaluates `lib.vmUsernames.<vm>` at the pinned secrets rev
    (stub `nix` keyed on the `?rev=` reference) and dies on eval failure.
  - Host-key helpers under the new `(…, secrets_checkout, secrets_rev)` signature:
    accept active and staged material; **fail closed** with the stale-pin refusal when the
    presented/derived material is absent from, mismatched against, or has no entry at the
    pinned rev (the required security regression — assert the refusal message and that no
    build ran); missing `machine-host-keys.json` at the pin is an infra failure, not a
    "not pinned" refusal.
  - `ensure_git_commit_available` fetches a missing pinned rev once and dies after a fetch
    miss (existing cases, now reused for inventory).
- `tests/rebuild-vm-from-host.sh`: fixture provides a deploy-flake `flake.lock` pinning
  both inventory and secrets. With **no** `INVENTORY`/`MACHINE_PROFILES`/`IDENTITY_CONFIG`
  set (only `DEPLOY_FLAKE` + default checkout locations), rebuild derives IP + username +
  pin, preserves the strict SSH options and `--flake "${DEPLOY_FLAKE}#<vm>"` target, honors
  the positional IP override, and refuses before `nixos-rebuild` on stale/empty/failed/
  malformed/multiple-no-match keyscans (existing cases ported to the deploy-flake fixture).
- `tests/provision-vm-from-host.sh`: with only `DEPLOY_FLAKE` set, provision resolves the
  IP at the inventory pin, reads and decrypts the age host-key secret at the secrets pin,
  fails closed via `assert_vm_host_key_material_pinned` before `nixos-anywhere` on a
  mismatched derived key, and builds `--flake "${DEPLOY_FLAKE}#<vm>"`. The bootstrap gate
  reads `forge_key` at the inventory pin: the existing success case keeps asserting
  bootstrap/verify ran, plus an anti-drift case — working-tree `forge_key` nulled while the
  pinned rev carries it (IP unchanged) — where bootstrap still runs. The DHCP-seam
  preflight: a fixture whose working-tree IP differs from the pinned IP refuses before
  `new-vm` (assert the both-IPs message and `assert_new_vm_not_called`). The success case
  also asserts bootstrap and verify were invoked with the pinned `TARGET` argument.
- `tests/provisioning-contract.sh`: unchanged, still green.
- `tests/rotate-token.sh`: unchanged, still green — proves the signature-stable shared
  helpers and the token-rotation flow that also sources `rotation-common.sh` are
  unaffected.

## Rollback Plan

Revert the implementation commit(s) on `nexus` `master` with `git revert`; the scripts
and tests return to the three-checkout model with no residue (no persistent state, no
secrets, no generated artifacts outside the repo). If the work landed as two PRs, revert
the wiring PR before the `rotation-common.sh` PR. Because the change is inert until a
human runs a live rebuild/provision, a revert before that first live run has zero
operational blast radius.
