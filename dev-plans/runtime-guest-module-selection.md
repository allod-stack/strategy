# Runtime-Selected Guest Module

## Tracking Issue

[allod/archetypes#22: Select the guest module from the inventory runtime fact](https://forge.anarch.diy/allod/archetypes/issues/22)

This is a scoped slice of milestone 4 in [microvm-framework-adoption.md](microvm-framework-adoption.md), which tracks [allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20). It implements that plan's **contract 2** only, and inherits contracts 1, 1a, and 21 as constraints it must not violate.

One PR in `allod/archetypes`. It carries `Closes allod/archetypes#22` and `Refs allod/strategy#20`. It does **not** close allod/strategy#20 — the parent plan reserves that for the final integration PR.

## Goal

A VM composes the guest module its inventory `runtime` fact names, so `runtime` decides what gets built instead of labelling a build it cannot affect.

## Scope

In scope:

- `allod/archetypes` `flake.nix:60-71` — `sharedModules` selects `vm.nixosModules.qemuGuest` or `vm.nixosModules.microvmGuest` from the machine's runtime, replacing the hard-coded `qemuGuest` at `flake.nix:61`.
- `allod/archetypes` `flake.nix:159`, `:219` — `mkDevVm` and `mkPrivacyVm` gain one optional argument, `runtime ? machines.${name}.runtime`, threaded to `sharedModules`. This is the seam that lets a check drive the production selector with a runtime no real machine declares. See contract 2.
- `allod/archetypes` `flake.lock` — bump `inventory` `efe4a7c` → `15ad552` and `vm` `2acc559` → `8a1eb0f`. Both are prerequisites, not incidental refreshes: see Sequencing.
- `allod/archetypes` — a new `runtime-module-selection` check pairing each machine's declared runtime against its built system's `allod.vm.guestRuntimes`, with paired sabotage fixtures.

Out of scope, deferred to milestone 4 proper:

- **Persistent volume declarations** (parent contract 13). Consequence stated under Sequencing: a real machine selecting `microvm` fails evaluation until that milestone lands. That is the intended fail-closed state, not an oversight.
- **`vmFacts.<name>.runtime`** (parent contract 1). The issue scopes this PR to `sharedModules` and one check. Exporting the fact through `vmFacts` changes a public output consumed host-side by `nexus` and interacts with the `vm-facts-coherence` projections at `flake.nix:693-708`; it belongs with the consumers that need it.
- Credential delivery, guest credential root, networking, `extendModules` host integration, nested boot (parent contracts 7-12, 15, 8b).
- Adding a machine with `runtime = "microvm"` to `allod/inventory`. The parent plan requires the first real selection to be a purpose-made machine landing with its own provisioning change; `allod/inventory` `flake.nix:31-38` records the same decision. The microvm branch here is proven by fixture.

## Sequencing

Three facts govern the order, and getting them wrong produces a false no-op proof.

**The two input bumps are prerequisites.** The locked `inventory` (`efe4a7c`) predates the runtime fact — no machine at that revision has a `runtime` attribute, so `machines.${name}.runtime` cannot resolve. The locked `vm` (`2acc559`) predates the guest-module split and exports `qemuGuest` alone — there is no `microvmGuest` to select. Neither the selection nor the check can be written against the current lock.

**The bump is already proven to be a no-op**, measured before writing this plan. With `vm` and `inventory` overridden to the target revisions and no source change, every machine's system derivation is unchanged:

| Machine | `config.system.build.toplevel.drvPath`, locked and bumped |
|---|---|
| `allod-dev` | `/nix/store/1lhcxfhy9iddfssp3df3r2zwkwnd3ilm-nixos-system-allod-dev-25.11.20260630.b6018f8.drv` |
| `privacy-1` | `/nix/store/zzzaralwv032jdmbqc1q6ms9fm15w7yf-nixos-system-privacy-1-25.11.20260630.b6018f8.drv` |
| `nexus` | `/nix/store/fad4pq24f8iavzr3q369alz8932jq9q3-nixos-system-nexus-25.11.20260630.b6018f8.drv` |

`nixpkgs` (`b6018f87`) and `disko` (`ff8702b4`) are identical at both `vm` revisions, which is why the bump lands clean. `archetypes` sets `nixpkgs.follows = "vm/nixpkgs"`, so a `vm` bump that *had* moved nixpkgs would have moved the whole fleet and made byte-identity unachievable.

**Therefore the no-op proof must be taken across the selection commit, not across the PR.** Commit the lock bump first and record the three derivation paths; then commit the selection change and require the same three paths. Comparing the PR's merge-base against its head conflates the two and cannot attribute a difference to either. The table above is the expected value at both points.

One lock-graph note for review: bumping `vm` adds `microvm.nix` and its own `spectrum` input (`git+https://spectrum-os.org/git/spectrum`, rev `24c4346e`) to the archetypes lock graph. That is upstream microvm.nix's dependency arriving through allod/vm's sole pin, as parent contract 1a intends. `nexus` still resolves its own `vm` input separately; `nexus.inputs.vm.follows = "vm"` is parent contract 1a's requirement and belongs to the final integration PR, not here.

## Risk Assessment

Residual risk: **R2 Medium.**

Why:

- The libvirt path — the entire current fleet — is a measured byte-identical no-op across the input bump, and the selection change must preserve that. Worst credible libvirt failure after validation is a derivation change the acceptance test is built to catch.
- The microvm branch adds a code path no real machine can reach. `allod/inventory` has no machine with `runtime = "microvm"`, and a machine that acquired one would fail evaluation on allod/vm's contract 6a before producing a guest. The new path's blast radius is the check fixture.
- Both guest modules are already contract-checked in `allod/vm` (`checks/guest-module-contracts.nix`, 24 mutation fixtures). This PR selects between two proven modules; it does not define guest behavior.
- Rollback is a straight revert of one commit pair in one repo. No secrets, host-only commands, provisioning, activation, or persistent state.

This is deliberately scored below the parent plan's R3 row for "archetypes guest integration", because that row covers all of milestone 4 — credentials, volumes, networking, `extendModules`, nested boot. Review pass 1 accepted R2 conditional on the check actually being able to fail: with the pass-by-construction avenues in the original draft, the score was not earned. Contracts 2, 4, and 5 and the rewritten acceptance tests are what close that gap. R2 flips to R3 if a fresh evaluation moves any current derivation, or if real inventory data ever makes the microvm branch reachable before contract 13's volumes land.

Human scrutiny:

- The three derivation paths across the selection commit. This is the whole libvirt safety argument.
- Whether the sabotage fixtures fail for the diagnostic they name, rather than for some other evaluation error. `allod/inventory` `flake.nix:117-138` records why that distinction is load-bearing here specifically.
- The lock-graph delta, particularly the new `spectrum` input arriving transitively.

## Interface Contracts

Inherited from the parent plan and not restated: contract 1 (runtime is a required machine fact), contract 1a (one VM-owned upstream pin), contract 2 (runtime selects one guest module, never two), contract 21 (libvirt remains first-class).

This PR adds:

1. **Selection is total and defaulted nowhere.** `sharedModules` maps the runtime to exactly one of `vm.nixosModules.qemuGuest` (`"libvirt"`) or `vm.nixosModules.microvmGuest` (`"microvm"`). An unrecognized value throws a named archetypes-side error. There is no `or "libvirt"`, no `if runtime == "microvm" then … else qemuGuest`, and no fallback branch — a fallback would re-create the silent-label failure the issue exists to remove.

2. **Fixtures drive the production selector, not a copy of it.** `machines` is closed over lexically at `flake.nix:39`, so a separately constructed `nixosSystem` would exercise a fixture helper and prove nothing about `machineConfigurations`. Instead `mkDevVm` and `mkPrivacyVm` take `runtime ? machines.${name}.runtime`, threaded into `sharedModules`; the default preserves production behavior exactly and a check overrides only that one value. This matches the existing idiom in `mkDevVm`, whose `runtimeIdentity`, `tokenFile`, and `httpsTokenFile` arguments are already optional-with-default precisely so `dev-forge-opt-out` (`flake.nix:823-828`) can vary one field against the real builder.

   A heavier refactor — passing whole machine records, or extracting a `machineConfigurationsFor machines` constructor — was considered and rejected as ceremony for this scope. One optional argument reaches the same code path.

3. **The marker is derived independently of the selector's input.** The check compares `config.allod.vm.guestRuntimes` against the runtime the machine declares. These are not the same value threaded twice: the marker is set by the guest module itself (`vm/modules/qemu-guest.nix:14` sets `[ "libvirt" ]`, `vm/modules/microvm-guest.nix:35` sets `[ "microvm" ]`), so a selector that ignored its input and returned `qemuGuest` would report `[ "libvirt" ]` against a declared `"microvm"` and fail. That is exactly today's bug, and it is what the check detects.

4. **Every fixture forces `system.build.toplevel`.** Reading `config.allod.vm.guestRuntimes` alone does **not** force NixOS module assertions — at the pinned nixpkgs they are reached only through `baseSystemAssertWarn` on the toplevel path. A fixture that read the marker alone would report `[ "microvm" ]` while allod/vm's contract 6a volume assertions and `guest-base.nix`'s exclusivity assertion went unevaluated, so a broken or absent volume would pass. Positive fixtures force `config.system.build.toplevel.drvPath`; negative fixtures `tryEval` a forced toplevel and `deepSeq` the assertion messages. `allod/vm` `checks/examples.nix:301-317` (`probe`) is the established idiom and does both — follow it rather than inventing a variant.

5. **Hypervisors are not guests, and the check must exclude them.** `mkHypervisor` does not call `sharedModules`, and hypervisor machines carry no `runtime` by inventory's design (`inventory/flake.nix:40-49`, actively asserted). `nexus` therefore composes no guest module and does not declare `allod.vm.guestRuntimes` at all — reading it is an undeclared-option error, verified against the tree. The check iterates `lib.filterAttrs (_: m: m.type != "hypervisor") machines`. Because `checkedMachines` returns machines unfiltered, a future refactor routing the hypervisor through `sharedModules` would hit `attribute 'runtime' missing`; guard or assert that, do not leave it to call-site discipline.

6. **The microvm fixture supplies its own volume.** Composing `microvmGuest` requires a `microvm.volumes` entry at `/nix/var/nix` with `autoCreate = false` (allod/vm `modules/microvm-store.nix:121-127`), which this PR does not add to the builder. The fixture therefore appends a placeholder module declaring one, mirroring `allod/vm` `checks/examples.nix:34-54`. This is the seam between "the selection works" and "a machine can carry state", and it is what lets the first be proven without the second. When contract 13's real volume declarations land, this placeholder becomes redundant and should be deleted rather than left to shadow them.

7. **Enum validation belongs to inventory; archetypes owns only the mapping gap.** A missing, non-string, or unknown `runtime` surfaces `allod/inventory`'s own diagnostic — reading `machines.${name}.runtime` forces the full four-assertion chain through the `checkedMachines` trip-wire (`inventory/flake.nix:183-192`). Archetypes must add no default that could mask it.

   Archetypes cannot *re-prove* that from its own side, and must not pretend to: `runtimeDiagnostics` and `mkVmSpecs` are lexical internals of inventory (`inventory/flake.nix:139`, `:158`), and overriding `inventory.machines` downstream with `//` lands after the `builtins.seq` trip-wire has already run, so a mutated value reaches only the archetypes selector. The archetypes-side throw in contract 1 therefore covers exactly one case — a value inventory's enum admits but archetypes has no module for, i.e. the two repos disagreeing, which is the case inventory cannot catch. Enum coverage is `inventory.checks.<system>.runtime-fact-mutations`'s job; bind it as a build dependency of the archetypes check so the linkage is real rather than asserted in prose.

## Agent Gates

None. The agent can push to `allod/archetypes`, and every acceptance test below runs locally in the dev VM without host access, real credentials, or a rebuild.

Note — a sequencing constraint, not a gate: after this lands, no real machine can select `microvm` until parent contract 13's volume declarations land. Setting `runtime = "microvm"` on a real machine before then fails evaluation with allod/vm's contract 6a message. That is the designed state.

## Acceptance Tests

The no-op proof, run across the selection commit with the lock bump already in:

```sh
cd /path/to/archetypes
# At the lock-bump commit, before the selection change:
for m in allod-dev privacy-1 nexus; do
  nix eval --raw ".#nixosConfigurations.$m.config.system.build.toplevel.drvPath"
done | tee /tmp/drvpaths-before

# After the selection change — must be byte-identical:
for m in allod-dev privacy-1 nexus; do
  nix eval --raw ".#nixosConfigurations.$m.config.system.build.toplevel.drvPath"
done | diff -u /tmp/drvpaths-before - && echo "libvirt path unchanged"
```

Expected values are in the Sequencing table. A difference here is a blocker, not a rebase artifact.

Full check set and the new check:

```sh
cd /path/to/archetypes
nix flake check --print-build-logs
nix build .#checks.x86_64-linux.runtime-module-selection --print-build-logs
```

`runtime-module-selection` must assert the following. Every fixture forces `config.system.build.toplevel.drvPath` per contract 4 — reading the marker alone proves nothing about the assertions.

1. **Fleet positive control.** For every non-hypervisor machine — `lib.filterAttrs (_: m: m.type != "hypervisor") machines`, per contract 5 — `machineConfigurations.<name>.config.allod.vm.guestRuntimes == [ machines.<name>.runtime ]`. Today that is `allod-dev` and `privacy-1`, both `[ "libvirt" ]`, read off the real production configurations. `nexus` is excluded and covered by the derivation-path comparison instead.

2. **The microvm branch.** `mkDevVm { name = "allod-dev"; runtime = "microvm"; … }` — the real builder, one overridden argument per contract 2, plus the placeholder volume module from contract 6 — composes the microvm guest, forces its toplevel, and reports `[ "microvm" ]`. This is the branch no real machine covers, and it is the fixture that fails today: a hard-coded `qemuGuest` returns `[ "libvirt" ]` against a declared `"microvm"`. No separate "restore the hard-code" sabotage is needed, because this fixture *is* that sabotage's inverse — state that explicitly so a later reader does not add a redundant one.

3. **Unmapped-but-admitted runtime.** A runtime inventory's enum admits but archetypes has no module for fails through contract 1's named archetypes-side throw, observed via `tryEval`. This is the repos-disagree case and the only enum-adjacent failure archetypes owns.

4. **Enum coverage is delegated, not duplicated.** The check takes `inventory.checks.${system}.runtime-fact-mutations` as a build dependency so inventory's missing/non-string/unknown diagnostics are actually exercised in this repo's check run. Per contract 7, archetypes cannot re-prove them locally, and a fixture that appeared to would be testing its own mutation rather than inventory's validator.

5. **Exclusivity still fires.** Composing both guest modules fails on `guest-base.nix`'s exclusivity assertion, observed through a forced toplevel. This guards the selector against ever returning both — the failure mode parent contract 2 names.

Where a fixture pins to a specific diagnostic, use `allod/inventory` `flake.nix:339-346`'s `pinnedTo` shape — `attrNames hit == [ machine ] && all (others == {})` — rather than a bare "evaluation failed", for the reason recorded at `inventory/flake.nix:126-138`.

Known harness bound, carried forward from `allod/inventory` commit `530073c`: `tryEval` catches `AssertionError` and `ThrownError` but not a raw `EvalError`, so a missing-attribute case aborts evaluation rather than reporting. Loud rather than silent, but it bounds what fixtures 3 and 5 can assert — state which cases are `tryEval`-observable and which abort.

## Rollback Plan

Revert the PR's two commits, selection first, then lock bump. Each is independently revertible and the pair restores the exact pre-change lock graph and derivation paths in the Sequencing table.

No rollback step touches a credential, image, domain, registry, or persistent volume — the change creates none. Reverting only the selection commit and keeping the bump is also valid and leaves the fleet on the measured-identical derivations.

## Findings Against the Issue Text

Three claims in allod/archetypes#22 do not survive checking. Recorded here so the issue can be corrected rather than implemented as written.

1. **`inventory.lib.vmRuntimes` does not exist.** Not on `master`, not in history, not on the unmerged `agent/inventory-runtime-fact` branch. `allod/inventory` exports `machines`, `lib.machines`, `lib.supportedPlatforms`, `lib.vmSpecsJson`, and `checks`. The capability the issue wants it for is real and available: reading `inventory.machines.<name>.runtime` forces the full validation chain via the `checkedMachines` trip-wire. The plan uses that path.

2. **The issue's scope boundary contradicts its own validation criterion.** It puts "the persistent volume declarations a microvm guest needs" out of scope while requiring that "a machine declaring `microvm` composes the microvm guest, and its built system reports `allod.vm.guestRuntimes = [ "microvm" ]`". Composing `microvmGuest` without a `/nix/var/nix` volume fails evaluation on allod/vm contract 6a, so the second is unreachable under the first. Resolved by contract 5 above: the fixture supplies a placeholder volume, the builder does not.

3. **The issue understates the prerequisites.** "Every piece this needs already exists and is merged" is true of the source repos but not of `archetypes`, whose lock pins both `inventory` and `vm` at revisions predating the runtime fact and the microvm guest module. Two input bumps are required before a line of the selection can be written.

One adjacent defect found in `allod/inventory`, **not fixed here** — it is that repo's to own:

- `inventory/flake.nix:37-38` claims "The runtime enum's microvm branch is covered by the mutation fixtures below." It is not. The only fixture carrying `"microvm"` sets it on `nexus`, a hypervisor, and `runtimeDiagnostics` never compares hypervisor runtimes against `validRuntimes`. Deleting `"microvm"` from `validRuntimes` (`flake.nix:115`) leaves every inventory check green. `inventory/flake.nix:306-308` and `:438` and `README.md:71-77` carry the same stale claim that both enum values are covered by the public examples; both examples are `libvirt` since commit `15ad552`. Worth its own issue.
