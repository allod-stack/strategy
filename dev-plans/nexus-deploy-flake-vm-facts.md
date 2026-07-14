# Derive VM Facts from Deploy-Flake Outputs

## Tracking Issue

[allod/nexus#6](https://forge.anarch.diy/allod/nexus/issues/6) — "Read VM
target/username/host-key from deploy-flake outputs, dropping side checkouts".
Follow-up to #5 (landed as nexus `cb0758a`). Three PRs, merged in order:

1. **PR1 `secrets`** — expose `lib.machineHostKeys` and `lib.vmHostKeySecretFiles`.
   `Refs allod/nexus#6`.
2. **PR2 `profiles`** — add the `vmFacts` output builder, wire it to the template
   flake, add checks, bump the `secrets` input lock past PR1. `Refs allod/nexus#6`.
3. **PR3 `nexus`** — rewire both scripts and the shared lib to consume the outputs;
   delete the git-show pin machinery. Carries `Closes allod/nexus#6`.

PR3 is test-green independently (its suites stub `nix`), but merges last so the
default deploy flake already exposes the outputs the scripts demand — an operator
who pulls nexus mid-chain must not get scripts that fail against the current
profiles master.

## Goal

`rebuild-vm-from-host <vm>` and `provision-vm-from-host <vm>` derive every
deployment fact — target IP, username, host-key pin material, host-key ciphertext,
and the bootstrap gate — from `DEPLOY_FLAKE` alone via `nix eval`, eliminating
`INVENTORY_CHECKOUT`/`SECRETS_CHECKOUT` from the fact path (provision keeps one
checkout knob solely for its unmigrated children).

## Scope

In scope:

- `secrets` flake: additive `lib.machineHostKeys` (parsed
  `machine-host-keys.json`) and `lib.vmHostKeySecretFiles` (VM name → age blob
  path, from `builtins.readDir ./secrets/vm-host-keys`).
- `profiles` flake: a data-in/facts-out builder (new `nix/vm-facts.nix`), exported
  as `lib.mkVmFacts` and instantiated as a top-level `vmFacts` output; new checks
  validating completeness, coherence against the raw JSON the scripts read today,
  and negative (sabotaged-input) behavior; `flake.lock` bump of the `secrets`
  input.
- `nexus` `scripts/lib/rotation-common.sh`: new deploy-flake facts helpers; the
  host-key asserts re-sourced to take pinned materials directly; deletion of the
  now-orphaned git-show pin machinery (`resolve_flake_input_locked_rev`,
  `ensure_git_commit_available`, `machine_host_key_materials_at_secrets_pin`,
  `resolve_target_ip_at_pin`).
- `nexus` `scripts/rebuild-vm-from-host` and `scripts/provision-vm-from-host`:
  replace lock parsing, pin fetches, and `git show`/working-tree reads with facts
  from `nix eval`; env surface reduced per Interface Contracts.
- `nexus` tests: `tests/rotation-common.sh`, `tests/rebuild-vm-from-host.sh`,
  `tests/provision-vm-from-host.sh` reworked per the test matrix;
  `tests/rotate-token.sh` and the other suites must stay green unchanged.

Out of scope:

- `inventory`: no change — `machines.<vm>.{ip,forge_key}` is already exposed, and
  the facts builder consumes it as data.
- The `vm` repo. The builder lives in `profiles`, not `vm`: profiles is already
  the composition point of inventory+secrets and the default deploy flake, and a
  `vm`-side lib would add a third repo plus a `vm` lock-bump to the chain for no
  consumer benefit. Private deploy flakes get the logic either by forking profiles
  or by calling `profiles.lib.mkVmFacts` with their own data inputs.
- The children `new-vm`, `bootstrap-vm-from-host.sh`, `verify-vm-from-host`, and
  the other rotation scripts (`nexus-host-key`, `vm-ssh-host-key`,
  `forge-ssh-key`, `rotate-token`). The children still read the inventory/secrets
  working trees (DHCP reservation, username, forge key state, known_hosts
  material); migrating them is the follow-up #5 already named. Consequence
  accepted here: provision keeps `INVENTORY_CHECKOUT` for the DHCP seam and the
  `INVENTORY` export, and a dev-VM provision still needs conventional
  inventory/secrets checkouts for bootstrap — the "one pointer" end state is fully
  reached by rebuild, and by provision's own fact reads but not its children's.
- The deploy-flake composition model, machine-data schemas beyond the two secrets
  lib attrs, baking `DEPLOY_FLAKE` into the host environment, and private-side
  adoption (tracked privately).
- Host-side live rebuild/provision (Agent Gate).

## Risk Assessment

Residual risk: R3 High overall, per PR:

| PR | Risk | Reason | Human scrutiny |
|---|---|---|---|
| PR1 secrets | R1 | Additive lib attrs exposing already-public material (public key JSON, age ciphertext paths); no machine behavior change; straight revert | Confirm nothing identity-bearing beyond the already-public files is exposed |
| PR2 profiles | R2 | Additive outputs plus checks; no `nixosConfigurations` change; wrong derivation is caught by the coherence check before any consumer exists; straight revert | Read the coherence check: it must compare `vmFacts` against the same JSON files the scripts read today, and the mutation checks must fail on sabotage |
| PR3 nexus | R3 | Re-sources the fail-closed host-key pin (principle 7: pinned, never TOFU) and the target-IP derivation on the provisioning/auth boundary; a regression could silently weaken the control | New assert signatures and refusal tests first; then the first live run against a real deploy flake |

Why R3 and not R4:

- The security property stays fixture-testable: the asserts become pure
  comparisons over materials the script fetched once, and the suites assert
  refusal-plus-no-build on mismatch, absence, and malformed facts.
- The one irreversible act (a live rebuild/provision) remains an explicit,
  human-gated step; the change is inert until then.
- Rollback is a straight revert of tracked scripts and flake outputs; no
  persistent state, no secrets mutation, no generated artifacts outside the repos.
- Store exposure is not new: the secrets source (age ciphertext included) already
  enters the world-readable Nix store today as a flake input of every deploy-flake
  build; age blobs are public-safe by design (principles 3 and 5).

Trade concentrated on purpose: the deploy flake was already the pin authority
(#5); this removes the last parallel read path (checkout content at a rev the
script resolved itself), so a compromised or misconfigured deploy flake is now the
single point of trust for *facts* as it already was for *what gets built*. That is
principle 8 working as intended — but it means fact-derivation bugs and build
targeting bugs now share one root, which is exactly where review attention and the
coherence check belong.

Human scrutiny, ordered:

1. PR3's refusal-path tests: mismatched derived/presented key, VM absent from
   pinned material, malformed `hostKeys` shape, missing `vmFacts` output — each
   must die before `ssh-keyscan`-gated builds (`nixos-rebuild`) or VM mutation
   (`new-vm`/`nixos-anywhere`). Read the assertions, not the green.
2. PR2's checks: coherence and sabotage cases.
3. First live run: `nix eval <private-deploy>#vmFacts.<vm> --json` by hand,
   compare against the intended target, then a real rebuild.

## Interface Contracts

### secrets flake (PR1)

```nix
lib.machineHostKeys = builtins.fromJSON (builtins.readFile ./machine-host-keys.json);
# { <vm> = { active = "<type> <b64> <comment>"; staged = <same> | null; }; ... }

lib.vmHostKeySecretFiles =
  # { <vm> = ./secrets/vm-host-keys/<vm>-ssh.age; ... } — one entry per
  # "<vm>-ssh.age" file found by builtins.readDir; nothing else matches.
```

Both are data the repo already publishes; the existing `credential-inventory`
check already parses `machine-host-keys.json`, so PR1 also switches that check to
read `lib.machineHostKeys` instead of re-reading the file (one parse, one owner).

### profiles builder and outputs (PR2)

```nix
# nix/vm-facts.nix — pure data-in/facts-out, no flake types in the signature,
# so mutation checks and private composition need no fixture flakes:
mkVmFacts = { machines, vmUsernames, machineHostKeys, vmHostKeySecretFiles }:
  # attrset over every machine with type != "hypervisor":
  # <vm> = {
  #   ip;                 # string, from machines.<vm>.ip
  #   username;           # string, from vmUsernames.<vm>
  #   forgeKey;           # string | null, from machines.<vm>.forge_key
  #   hostKeys;           # { active; staged; } from machineHostKeys.<vm>
  #   hostKeySecretFile;  # string — toString of vmHostKeySecretFiles.<vm>
  # }
```

- Each `vmFacts.<vm>` asserts **its own** completeness lazily (missing username,
  missing host-key entry, missing age file, missing/null ip are eval errors naming
  the VM and the missing fact — principle 11). Deliberately not strict across VMs:
  one machine's broken fact-level data must not brick `nix eval` for every other
  machine mid-incident. The laziness boundary: the names set itself forces
  `.type` on every machine (the hypervisor filter), so a machine with a missing
  or non-string `type` bricks the probe for every VM — accepted, that is
  structurally corrupt machine data failing loud, and it must not be "fixed"
  with a lazy `m.type or` default that would silently classify a typeless
  machine as provisionable. The check below forces the whole set, so global
  completeness still fails CI closed.
- Guard mechanics, forced by what `builtins.tryEval` can catch (only `throw` and
  `assert`): every per-VM completeness rule is an explicit `throw` naming the VM
  and the fact. A bare attribute access (`vmUsernames.${vm}`) on sabotaged input
  is an uncatchable abort that kills the whole `vm-facts-negative` check instead
  of failing one case.
- `hostKeySecretFile` is `toString` of the path at construction, never an
  interpolated path: the secrets input is already a store path, `toString` names
  the file inside it without a copy, and a plain string serializes identically
  across `nix eval --json` versions. The age payload is binary, so the contract is
  path-out, never `readFile`-content-out.
- The builder asserts its data arguments are present with adoption-pointing
  messages (e.g. a secrets input lacking `lib.machineHostKeys` dies with "bump the
  secrets input past the rev that exposes lib.machineHostKeys", not an attribute
  error). Mechanics — both halves are load-bearing: the four arguments take
  `? null` defaults and the builder throws on null at the top of its body (so
  even the names probe reports it), and the flake wiring passes
  `secrets.lib.<attr> or null`. Either shortcut defeats the message: a required
  destructured argument dies uncatchably ("called without required argument")
  when omitted, and a bare `secrets.lib.machineHostKeys` at the wiring site is
  passed as a lazy thunk whose raw attribute-missing error fires inside the
  builder when forced — in both cases the adoption message never runs and
  `vm-facts-negative` aborts instead of catching a failure.
- Flake wiring: `vmFacts = mkVmFacts { machines = inventory.machines; vmUsernames
  = secrets.lib.vmUsernames; machineHostKeys = secrets.lib.machineHostKeys;
  vmHostKeySecretFiles = secrets.lib.vmHostKeySecretFiles; }` as a top-level
  output, and `lib.mkVmFacts` exported. `nix flake check` warns on the unknown
  top-level output name; that is cosmetic and accepted.
- New profiles checks (eval-level, no VM builds):
  - **vm-facts-coherence** — `builtins.toJSON vmFacts` field-compared (jq) against
    `${inventory}/scripts/vm-specs.json` (ip, forge_key) and
    `${secrets}/machine-host-keys.json` (active/staged), plus attr-name equality
    with the non-hypervisor machine set, plus `hostKeySecretFile` existing in the
    secrets source. This proves the new derivation reads the same facts the
    scripts read via `git show` until now.
  - **vm-facts-negative** — `builtins.tryEval` sabotage cases through
    `mkVmFacts`: a machine absent from `machineHostKeys`, an absent username, an
    absent age file, a null ip, and an omitted data argument (must fail with the
    adoption message, exercising the `? null` guard) — each must fail (a
    validation that cannot fail on sabotage does not count). Each case forces
    the specific field under test: `tryEval` of an unforced lazy attrset
    succeeds trivially and proves nothing.

### Deploy-flake contract (replaces #5's)

Old: "pin `inventory` and `secrets` as direct root git inputs in `flake.lock`"
(that shape was what `resolve_flake_input_locked_rev` could parse). New, strictly
weaker: **expose `vmFacts.<vm>` with the field contract above**. How the deploy
flake composes them — forking profiles, calling `profiles.lib.mkVmFacts`, nesting
inputs, `follows` chains — is its own business; the scripts never look at
`flake.lock` again. A deploy flake without the output fails loud at the probe
(below) with an adoption-pointing message.

### nexus env surface (PR3)

```sh
# rebuild-vm-from-host:
DEPLOY_FLAKE="${DEPLOY_FLAKE:-$HOME/work/allod/profiles}"   # the only pointer
# INVENTORY_CHECKOUT and SECRETS_CHECKOUT are deleted, not deprecated.

# provision-vm-from-host:
DEPLOY_FLAKE="${DEPLOY_FLAKE:-$HOME/work/allod/profiles}"
INVENTORY_CHECKOUT="${INVENTORY_CHECKOUT:-$HOME/work/allod/inventory}"
# Retained ONLY for the unmigrated children: the worktree-vs-pin IP preflight
# and the INVENTORY export new-vm/bootstrap/verify read. Never used for facts.
# SECRETS_CHECKOUT is deleted (bootstrap keeps its own IDENTITY_CONFIG default).
```

No fallbacks: a checkout-based fallback would resurrect the second read path for
the same fact (the drift #5/#6 exist to kill) and fall back silently on exactly
the evals that should fail loud (principles 8 and 11). Removal is deletion of the
env knobs and the machinery, in one PR, tests re-pinned.

### rotation-common.sh (PR3)

New helpers:

- `deploy_flake_vm_names(deploy_flake)` — `nix eval --json
  "${deploy_flake}#vmFacts" --apply builtins.attrNames` (fixed `--apply` string;
  no interpolation into Nix code). Eval failure dies: "deploy flake
  `<deploy_flake>` does not expose vmFacts outputs; compose a profiles rev that
  provides them (see allod/nexus#6)". Both helpers leave nix's stderr connected
  to the operator (no `2>/dev/null` on the evals): a missing output, a cold
  store with no network, a dirty-checkout syntax error, and a broken transitive
  input all fail this same probe, and nix's own diagnostic is the only thing
  that tells them apart — the die line classifies and points at adoption, it
  must not replace the diagnostic.
- `deploy_flake_vm_facts(deploy_flake, vm)` — probes `deploy_flake_vm_names`
  first and dies "unknown VM '<vm>' in deploy flake vmFacts" when absent (clean
  message instead of raw nix attrpath stderr — the unknown-VM path never runs a
  per-VM eval, so passthrough and the clean message coexist), then `nix eval
  --json "${deploy_flake}#vmFacts.\"${vm}\""`, dying on eval failure. Prints
  compact JSON. The VM name reaches nix only as an attrpath argument, never
  spliced into an expression. Probe-first costs a second eval per run but keeps
  the failure classes disjoint by construction: membership is settled before
  the per-VM fetch, so a per-VM eval failure is a completeness throw whose
  passed-through stderr already names the VM and the missing fact; the die adds
  only the remediation pointer. A single fetch-first eval would save the probe
  on the success path at the price of classifying failures by parsing nix
  stderr — rejected.
- Field extractors over that JSON (jq, `-e`, die on null/missing with the
  existing message classes): `vm_fact_target_ip` ("no target IP"),
  `vm_fact_username` ("cannot resolve username"), `vm_fact_forge_key` (prints
  `null` for privacy VMs), `vm_fact_host_key_secret_file`, and
  `vm_fact_host_key_materials` — the last reuses today's jq material filter
  (active + optional staged, "type b64" shape enforced, error on bad shape) so
  material validation strength is unchanged, only the JSON's source moved. An
  entry with `active` and `staged` both null passes through the filter as empty
  materials and succeeds — presence judgment stays with the asserts, which
  refuse empty pinned materials with the security refusal exactly as today's
  no-material case does; the extractor must not preempt that with a generic
  extraction error, which would downgrade the message class.

Changed signatures (callers are only the two in-scope scripts):

| Function | Old inputs | New inputs |
|---|---|---|
| `assert_any_vm_host_key_material_pinned` | `(vm, materials, secrets_checkout, secrets_rev, flake_lock)` | `(vm, materials, pinned_materials, deploy_flake)` — pure comparison; extraction happens once in the script |
| `assert_vm_host_key_material_pinned` | same shape | same change |
| `die_vm_host_key_material_not_pinned` | `(vm, material, secrets_rev, flake_lock)` | `(vm, material, deploy_flake)` |

Deleted with their unit tests (no remaining callers, verified by grep; the
rotation scripts and `rotate-token` never used them): `resolve_flake_input_locked_rev`,
`ensure_git_commit_available`, `machine_host_key_materials_at_secrets_pin`,
`resolve_target_ip_at_pin`. Reverting PR3 restores them wholesale.

Refusal rewording (deliberate message-class change, tests re-pinned): the assert
no longer receives a secrets rev to interpolate. New refusal, same fail-closed
die: `VM '<vm>' SSH host key <observed> is not in the deploy flake's pinned
secrets. Facts derive from <deploy_flake>'s locked inputs (inspect with: nix
flake metadata <deploy_flake>). Update the secrets input in the deploy flake,
commit, push, then retry.` The remediation sentence is unchanged from #5's. The
rev disappears from the message rather than keeping lock parsing alive for
message decoration; `nix flake metadata` is the replacement pointer.

### Per-script wiring (PR3)

- `rebuild-vm-from-host`: one `FACTS_JSON=$(deploy_flake_vm_facts ...)` up front;
  `TARGET="${2:-$(vm_fact_target_ip ...)}"`; `VM_USERNAME` from the facts; scan
  presented keys as today; `assert_any_vm_host_key_material_pinned "$VM_NAME"
  "$PRESENTED..." "$PINNED_MATERIALS" "$DEPLOY_FLAKE"`; build
  `--flake "${DEPLOY_FLAKE}#${VM_NAME}"` and strict SSH options unchanged.
  Message-class change: an unknown VM now dies at the facts probe even when a
  positional IP is supplied (#5 had it surface late at the host-key assert);
  earlier and cleaner, still fail-closed before any build.
  Semantic change, chosen: the username is now **pinned** (it rides the facts
  eval through the deploy flake's lock) instead of #5's deliberate working-tree
  read. #5 rejected pinning because the only mechanism was a heavy, untestable
  `git+file://?rev=` eval; composition makes it free, and one atomic facts
  snapshot beats one deliberately-drifting field. Operator consequence: a
  username change takes effect on the next deploy-flake lock bump, like every
  other fact.
- `provision-vm-from-host`: same single facts fetch; `TARGET`, `FORGE_KEY`
  (bootstrap gate; `null` still means privacy VM, skip bootstrap/verify) from the
  facts; `assert_inventory_worktree_ip_matches_pin` and `export
  INVENTORY="$INVENTORY_CHECKOUT"` stay exactly as they are (the DHCP seam:
  `new-vm` reads the working tree, so the split-brain preflight remains
  load-bearing); host-key ciphertext read becomes `age --decrypt -i
  "$AGE_IDENTITY" "$(vm_fact_host_key_secret_file ...)"` — the store path from
  the facts, with a readability guard dying in the "No SSH host key secret for
  '<vm>'" class naming the deploy flake, and the "Cannot decrypt" class kept on
  the decrypt itself; derive the public key and
  `assert_vm_host_key_material_pinned` against the facts materials before
  `nixos-anywhere`; build `--flake "${DEPLOY_FLAKE}#${VM_NAME}"`; pass pinned
  `TARGET` to bootstrap/verify as today. `read_forge_key_at_inventory_pin` and
  the secrets `ensure_git_commit_available`/`cat-file`/`git show` block are
  deleted.

Operational notes, stated so they are chosen rather than discovered:

- The facts eval forces fetching the deploy flake's pinned `inventory`, `secrets`,
  and `nixpkgs` sources into the store. On any host that has ever built the
  deploy flake these are warm; on a cold store the first eval downloads what the
  immediately following `nixos-rebuild`/`nixos-anywhere` would fetch anyway. The
  eval replaces `ensure_git_commit_available`'s fetch as the network touchpoint.
- A dirty deploy-flake checkout evaluates as its working tree — the same
  semantics the build step (`--flake "${DEPLOY_FLAKE}#..."`) already has, so
  facts and build now read one snapshot by construction.

## Agent Gates

- Live end-to-end confirmation is human-only (agents run in dev VMs; provisioning
  is host-side — principle 6). The human runs the first
  `rebuild-vm-from-host`/`provision-vm-from-host` after PR3, ideally preceded by
  a by-hand `nix eval <deploy>#vmFacts.<vm> --json` against the real deploy
  flake. Blocks final sign-off, not implementation.
- Merge sequencing is human-gated: PR2's lock bump can only pin a `secrets` rev
  containing PR1 after PR1 is merged, and PR3 should merge after PR2. The agent
  parks between PRs; nothing else blocks.
- `secrets` and `inventory` are protected repos — PR1 goes through
  `allod change begin`. No secrets *values* change; no repo creation; no host-only
  commands.

## Acceptance Tests

Per repo, all runnable by the agent in the dev VM:

```sh
# PR1 — secrets
cd ~/work/allod/secrets && nix flake check
nix eval .#lib.machineHostKeys --json | jq -e 'keys | length > 0'
nix eval .#lib.vmHostKeySecretFiles --json | jq -e 'has("allod-dev") and has("privacy-1")'

# PR2 — profiles (includes the new coherence + negative checks)
cd ~/work/allod/profiles && nix flake check
# Genuine no-checkout derivation on template data — the real-eval proof:
nix eval .#vmFacts --apply builtins.attrNames --json   # non-hypervisor set only
nix eval .#vmFacts.allod-dev --json | jq -e '
  .ip and .username and .hostKeys.active and .hostKeySecretFile'
test -r "$(nix eval --raw .#vmFacts.allod-dev.hostKeySecretFile)"
! nix eval .#vmFacts.nexus 2>/dev/null          # hypervisor excluded
! nix eval .#vmFacts.no-such-vm 2>/dev/null

# PR3 — nexus: the full gate (all ten suites + bash -n + shellcheck -x)
cd ~/work/allod/nexus && nix flake check
```

The nexus suites stub `nix` (the suites run inside the flake-check sandbox where
recursive nix is unavailable), so the script-level cases prove the scripts'
handling of facts; the real derivation is proven by PR2's checks and the
`nix eval` commands above; the human's live run bridges the two.

PR3 test matrix (new or adapted cases; the `nix` stub keys on the `#vmFacts`
attrpaths):

- `tests/rotation-common.sh`: unit cases for `deploy_flake_vm_names` /
  `deploy_flake_vm_facts` (missing-output die, unknown-VM die, eval-failure die;
  the die cases also assert the stub nix's stderr reached the caller's stderr
  alongside the die message — pinning the no-swallow contract);
  field extractors (null ip, missing username, `null` forge key passthrough,
  malformed hostKeys shape dies in the material filter, an all-null entry yields
  empty materials and succeeds); the asserts under the new pure signatures —
  accept active and staged, **fail closed** with the reworded refusal (greps pin
  the new message including the `nix flake metadata` pointer) on mismatched
  material and on empty pinned materials — the no-entry case and the
  present-but-all-null entry both collapse to the same empty-materials refusal
  today, and must keep that single home. Unit tests of the four deleted
  helpers are deleted with them.
- `tests/rebuild-vm-from-host.sh`: fixture drops the inventory/secrets git repos
  and the deploy-flake `flake.lock` (facts come from the stub); the zero-env case
  sets none of the data vars and relies on the fixture-`$HOME` conventional
  `profiles` path; asserts derived IP + username + pin from one facts eval,
  positional-IP override still consults facts for username+pin, unknown VM dies at
  the probe (also with a positional IP), missing-vmFacts deploy flake dies before
  keyscan, and all existing keyscan refusal modes (stale/empty/fail/malformed/
  multiple-no-match) keep `assert_nixos_rebuild_not_called` with re-pinned
  refusal wording; `--flake "${DEPLOY_FLAKE}#<vm>"` and `NIX_SSHOPTS` unchanged.
  Every refusal case in the suite — the probe and facts-level dies included, not
  only the keyscan modes — asserts `assert_nixos_rebuild_not_called`; that
  blanket refusal-plus-no-build assertion on the facts-fed path is what the
  R3-not-R4 case rests on.
- `tests/provision-vm-from-host.sh`: inventory working-tree fixture stays (DHCP
  seam: preflight-mismatch case refusing before `new-vm` is unchanged, as are the
  `INVENTORY` export and pinned-`TARGET`-to-children assertions); secrets git
  fixture is replaced by a stub-served facts JSON whose `hostKeySecretFile`
  points at a fixture ciphertext file. Cases: dev-VM success asserts decrypt from
  exactly that path, injected material matching, bootstrap+verify invoked (and
  privacy VM with `forgeKey: null` skips both); absent/unreadable secret file
  dies in the "No SSH host key secret" class before `new-vm`; garbage ciphertext
  dies in the "Cannot decrypt" class; mismatched derived key refuses via the
  assert before `nixos-anywhere`; a garbage age file planted in the conventional
  secrets checkout path is never read (facts-path success — the anti-drift
  successor of the old ciphertext-skew case); facts `forgeKey` wins over a
  worktree-nulled `forge_key` and bootstrap+verify still run (adapting today's
  pinned-forge-key-wins case to the facts source — the stated non-IP split-brain
  consequence, unchanged in kind from #5). All refusal cases keep
  `assert_new_vm_not_called`.
- `tests/provisioning-contract.sh`, `tests/rotate-token.sh`, and the shared-helper
  suites (`nexus-host-key.sh`, `vm-ssh-host-key.sh`, `forge-ssh-key.sh`,
  `bootstrap-orchestration.sh`, `registry-resolver.sh`): unchanged and green —
  the flake-check gate runs them all, which is what makes deleting lib functions
  safe to declare done.

## Rollback Plan

- PR3: `git revert` on nexus master restores the #5 model wholesale — scripts,
  deleted lib helpers, and tests return in one commit; the conventional checkouts
  it needs are still on the host. The change is inert until a human runs a live
  command, so a revert before that has zero operational blast radius.
- PR2/PR1: additive outputs with no consumer after a PR3 revert; revert in reverse
  order (PR2 then PR1) only if a full unwind is wanted — leaving them merged is
  harmless.
- Partial-land states are safe by construction: PR1 alone and PR1+PR2 change no
  script behavior; the only coupled edge is PR2 before PR1 (its lock bump has no
  rev to pin), prevented by the merge order in Agent Gates.
