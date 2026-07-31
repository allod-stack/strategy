# vmFacts Runtime Export

## Tracking Issue

[allod/archetypes#27: Export the inventory runtime fact through vmFacts](https://forge.anarch.diy/allod/archetypes/issues/27)

The remaining half of **contract 1** in [microvm-framework-adoption.md](microvm-framework-adoption.md), tracking [allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20). Inventory's half — the required fact, its enum, `lib.vmSpecsJson`, and the committed `scripts/vm-specs.json` — already landed; this slice adds the third source of truth that contract makes agree, `archetypes.vmFacts.<name>.runtime`.

It is sequenced ahead of milestone 5 rather than bundled into milestone 4's other slices because **contract 18** makes both rotation tools dispatch on this exact attribute, so allod/nexus#21 and allod/nexus#22 cannot start until it exists. One PR in `allod/archetypes` carrying `Closes allod/archetypes#27` and `Refs allod/strategy#20`; it does not close allod/strategy#20.

## Goal

`archetypes.vmFacts.<name>.runtime` reports the machine's inventory runtime, so a host-side tool can tell a libvirt VM from a microvm without evaluating a NixOS configuration.

## Scope

In scope, all in `allod/archetypes`:

- `nix/vm-facts.nix`: one new `runtime` field in `factFor`, with presence and type validation in the file's existing `throw`-helper style.
- `flake.nix` `vm-facts-coherence`: extend the inventory projection to compare `runtime`; add the negative proof that the comparison actually detects drift; depend on `inventory.checks.<system>.vm-specs-json` so the committed file this check trusts is known fresh.
- `flake.nix` `vm-facts-negative`: `runtime` added to the fixture `baseData`, plus one sabotage fixture per new validation and one positive fixture.
- `README.md`: enumerate the `vmFacts` per-machine field set in the export bullet, `runtime` included. The bullet at `README.md:46` names no fields today; naming only the new one would leave five undocumented.

Out of scope:

- Consuming the fact. Runtime-aware rotation is allod/nexus#21 and allod/nexus#22, milestone 5.
- Guest-module selection. `flake.nix:73-81`, `:434` and `:496` already read `machines.<name>.runtime` directly and are untouched; this slice does not reroute them through `vmFacts`.
- `allod/inventory`. Its half of contract 1 is merged, and it owns the runtime enum — see Interface Contract 3.
- allod/nexus's hand-written vmFacts stub payloads (`nexus/tests/rotation-common.sh:345`, `:419`, `:428-436`; `nexus/tests/provision-vm-from-host.sh:211-222`; `nexus/tests/rebuild-vm-from-host.sh:130-138`, `:141-147`). Adding a field upstream does not break them — every extractor is a guarded single-field read — but it does make them a less faithful mirror. Whichever nexus PR first dispatches on `runtime` updates them; nothing here does.

## Risk Assessment

Residual risk: **R2 Medium**.

Why:

- `vmFacts` is a public flake output that `allod/nexus` reads host-side as `<flake>#vmFacts` during provisioning, rebuild, and key rotation, and `mkVmFacts` is re-exported in `lib` for deploy flakes. Changing its shape is a public interface change even when purely additive, and it carries cross-repo ordering because milestone 5 blocks on it. That is a literal match for R3's "cross-repo interfaces or sequencing" bullet, and the score declines it deliberately on the guideline's "lowest level whose description still matches the residual worst case" rule.
- What holds it at R2: the change is additive-only, and every consumer across the seven repos was enumerated and re-derived independently before planning. No consumer iterates the per-machine attrset's keys, destructures it without an ellipsis, pattern-matches it, or compares it as a whole object, and no schema, key-set, or key-count assertion exists anywhere. The only whole-attrset serialization is archetypes' own `builtins.toJSON vmFacts` at `flake.nix:963`, whose every downstream jq step projects named fields. `allod/deploy` is a bare passthrough (`deploy/flake.nix:38`). nexus reaches it only through JSON: `nix eval --json ...#vmFacts --apply builtins.attrNames` (`nexus/scripts/lib/rotation-common.sh:81`) and per-field guarded reads (`:109`, `:118`, `:127`, `:136`, and a type-guarded `hostKeys` read at `:147-168`), called from `nexus/scripts/provision-vm-from-host:62-76` and `nexus/scripts/rebuild-vm-from-host:52-56`. `$FACTS_JSON` is never re-serialized, hashed, or compared.
- No host command, secret, provisioning action, or persistent state is touched. Rollback is a straight revert of one PR in one repo.
- The libvirt no-op is checkable exactly, and a lazily-added field cannot move a derivation that never forces it.

Human scrutiny, in order:

1. That the new coherence comparison would actually fail on drift, and fails **for the diff** rather than for any other reason — see Interface Contract 4. This is the one place where a green check could mean nothing.
2. Interface Contract 3's accepted gap: `lib.mkVmFacts` driven with non-inventory data gets no enum validation from anything in this repo.
3. The four libvirt drvPaths.

## Interface Contracts

Inherited and not restated: contracts 1, 18, 21.

1. **`runtime` is a passthrough fact, not a derived one.** `vmFacts.<name>.runtime` is exactly `machines.<name>.runtime` as inventory declares it. `factFor` (`nix/vm-facts.nix:76-87`) gains one field; nothing computes, normalizes, lowercases, or defaults it. A default is what made the fact decorative before — see the comment at `flake.nix:59-63`.

2. **Hypervisors are unchanged and acquire nothing.** `includeMachine` (`nix/vm-facts.nix:45-51`) filters `type == "hypervisor"` out before `factFor` runs, so no hypervisor gets a fact, a fake runtime, or a new failure mode. This matches inventory, which rejects a hypervisor that declares `runtime` (`inventory/flake.nix:163-164`). Do not add a hypervisor branch to `factFor`; the filter is the mechanism.

3. **`mkVmFacts` validates structure. Inventory owns the enum, and the delegation is made real by depending on inventory's checks.** In `nix/vm-facts.nix`, a missing `runtime` and a non-string `runtime` each `throw` in the file's existing style — `missingVmFact vm "runtime"` for absence, matching `requireIp` (`:21-25`), and a distinct `throw "vmFacts.${vm}: runtime must be a string"` for the type error, matching `:30-31` and `:49`. `mkVmFacts` does **not** re-declare inventory's `[ "libvirt" "microvm" ]` enum.

   That is this repo's already-recorded position, not a new judgement: the comment at `flake.nix:1708-1712` states plainly that inventory owns the runtime enum and archetypes cannot re-prove it, because inventory's diagnostics are lexical internals and a downstream `//` override lands after the `checkedMachines` trip-wire has run — so `runtime-module-selection` depends on `inventory.checks.<system>.runtime-fact-mutations` instead (`flake.nix:1713`). Structural-only validation is also the existing standard in `vm-facts.nix`: `requireIp` checks presence and non-null and never the shape of an address.

   For the real export this is airtight. `inventory/flake.nix:445` exports `machines = checkedMachines`, and `:192` is `builtins.seq (mkVmSpecs machines) machines`; forcing that to WHNF runs all four asserts including the enum at `:174-175`. Archetypes reads it at `flake.nix:39`/`:43`, and `requireData` (`nix/vm-facts.nix:12-13`) compares against `null`, forcing WHNF. So `nix eval .#vmFacts` cannot produce an out-of-enum runtime.

   **The accepted gap, stated rather than papered over:** `lib.mkVmFacts` accepts any attrset, so a caller passing its own `machines` — as this repo's own fixtures do at `flake.nix:1021-1040` — gets presence and type validation but no enum, and `runtime = "bhyve"` would evaluate cleanly. This is inert today: `lib.mkVmFacts` has no caller outside archetypes, and `deploy/flake.nix:38` is a passthrough of `archetypes.vmFacts`. It is recorded here because contract 18's consumers are the mitigation: **allod/nexus#21 and allod/nexus#22 must fail closed on an unrecognized `runtime` rather than defaulting to the libvirt branch.** If that turns out to be insufficient, the fix is to give `mkVmFacts` an enum parameter defaulted from the caller rather than a third hard-coded copy of the value list.

4. **The coherence check must be able to fail, and today's cannot.** The existing projections enumerate fields — `{ ip: .value.ip, forge_key: .value.forgeKey }` at `flake.nix:980-988` — so a new key is projected away and passes unverified. Add `runtime` to both sides of that projection so `vmFacts` is diffed against `${inventory}/scripts/vm-specs.json`, which already carries it (`inventory/scripts/vm-specs.json:20`, `:31`, both `"libvirt"`).

   Extending the projection is not sufficient on its own. Both sides are real data that already agree, so deleting the added field from the projection leaves the check green — a comparator never observed to fail. The projection-and-diff step therefore becomes a shell function taking a facts file, invoked twice inside the same derivation: once on the real facts, once on a `builtins.toFile` fixture identical to the real facts except for one machine's `runtime`. Folding both into the one `runCommand` is so the function can be shared, not for memory reasons.

   The mechanics are load-bearing, because the obvious implementation is wrong. The builder runs under the stdenv's `set -eu -o pipefail` with `inherit_errexit`, and the existing failure idiom is `diff … || { echo "ERROR: …" >&2; exit 1; }` (`flake.nix:989-990`). Lifted verbatim into a function, the sabotage invocation would `exit 1` the whole builder. Reaching for `|| true` or `set +e` to fix that is exactly how the proof goes vacuous. So:

   - The function signals with `return 1`, never `exit`.
   - The positive invocation is a bare call, so errexit propagates a real regression.
   - The negative invocation is `if project_and_diff "$sabotaged" > drift.log 2>&1; then echo "ERROR: runtime drift fixture did not fail" >&2; exit 1; fi`.
   - The negative invocation is then **pinned to its reason**: `grep -q` for the specific runtime-diff ERROR string in `drift.log`. Without that, a jq parse error, an unreadable fixture, or a name-set mismatch all satisfy "it failed."
   - The sabotaged value is `"microvm"` — a *valid* enum member — so the fixture cannot be satisfied by anything except the diff itself.

   `inventory/flake.nix:416-431` already carries an errexit-safe idiom of this shape; match it rather than inventing one.

   **Do not add an in-check enum assertion.** Contract 1's acceptance test `all(.value.runtime == "libvirt" or "microvm")` is kept as a named manual command below, but not as a `nix flake check` assertion: given the trip-wire in contract 3, it cannot fail on real data by construction, and an unfalsifiable assertion is the exact defect this contract exists to prevent. The enum's falsifiable proof is inventory's own `runtime-fact-mutations`, already depended on at `flake.nix:1713`.

   Add one more dependency in the same idiom: `inventory.checks.<system>.vm-specs-json`. Contract 1 requires `lib.vmSpecsJson`, the committed `scripts/vm-specs.json`, and `vmFacts` to agree. This check diffs vmFacts against the committed file, but `inventory/flake.nix:186-187` records that a downstream `nix flake check` does not evaluate an input's checks — so without this the committed file's own freshness is assumed, not verified, and the three-way agreement has a hole.

5. **The negative fixtures prove the condition, and cannot prove the text.** `vm-facts-negative`'s `expectFailure` (`flake.nix:1041-1043`) is `builtins.tryEval (builtins.deepSeq value true)`, which observes *that* evaluation failed and never *why*. There is no pure-Nix way to read a `throw`'s message, so the `config.assertions` diagnostic pinning used in `runtime-module-selection` (`flake.nix:1371-1375`) does not transfer to `vm-facts.nix`'s lexical throws. Each condition therefore gets its own fixture forcing the single attribute path it guards, plus a positive fixture proving valid data still passes through.

   `baseData.machines.alpha-dev` (`flake.nix:1023-1027`) gains `runtime = "libvirt"` so the fixture set has a valid baseline. Its `nexus` hypervisor entry gains nothing. This disturbs none of the five existing fixtures: fixtures 1-4 each deep-seq exactly one field of `factFor`'s result, and fixture 5 (`flake.nix:1061-1064`) forces only `builtins.attrNames` and throws at `checkedData` (`nix/vm-facts.nix:63-72`, `:89`) before `factFor` runs at all.

6. **The presence check's job is to name the error, not to create it.** `tryEval` does not catch a raw missing-attribute `EvalError` — this repo already records that at `flake.nix:1359-1362`. So a `factFor` that reads `machine.runtime` with no presence guard does not silently succeed; it aborts the whole check with `attribute 'runtime' missing`. The guard converts an opaque abort into `vmFacts.<vm>: missing runtime`, which is precisely why `requireIp` and `requireAttr` are written the way they are. Acceptance test 4 is written against that actual behavior.

## Agent Gates

None. Every acceptance test runs locally with no host access, real credentials, or network.

## Acceptance Tests

The libvirt no-op, taken across the implementing commit:

```sh
cd /path/to/archetypes
for m in allod-dev privacy-1 nexus installer; do
  nix eval --raw ".#nixosConfigurations.$m.config.system.build.toplevel.drvPath"
done
```

Must be byte-identical before and after:

| Machine | drvPath hash |
|---|---|
| `allod-dev` | `1lhcxfhy9iddfssp3df3r2zwkwnd3ilm` |
| `privacy-1` | `zzzaralwv032jdmbqc1q6ms9fm15w7yf` |
| `nexus` | `fad4pq24f8iavzr3q369alz8932jq9q3` |
| `installer` | `2yj5038jnsgzbv6ghhmyafs5kspn9jrg` |

Then contract 1's own named test, and the full set:

```sh
nix eval .#vmFacts --json | jq -e '
  to_entries
  | all(.value.runtime == "libvirt" or .value.runtime == "microvm")
'
nix eval --json '.#vmFacts' | jq -e 'has("nexus") | not'
nix flake check --print-build-logs
```

The first is the parent plan's acceptance command and is run manually; contract 4 explains why it is not also a check assertion. The second confirms the new field did not come with a hypervisor entry.

`nix flake check` peaks near 5.7 GB on a 7 GiB box with no swap. Its peak is dominated by NixOS module-system evaluation — four toplevels plus the `runtime-module-selection` fixtures — and this slice adds no derivation that evaluates a NixOS configuration. Two `builtins.toFile` JSON strings derived from the already-computed `builtins.toJSON vmFacts`, a few `jq` runs, and one more interpolated inventory check drvPath cannot move it. If anything beyond that becomes necessary, measure with `\time -v` before keeping it.

Required assertions, each with the sabotage that proves it:

1. **`vmFacts.<name>.runtime` matches inventory for every non-hypervisor machine** — the extended coherence diff against `scripts/vm-specs.json`. Sabotage is test 3.
2. **The coherence comparison detects runtime drift, for the right reason** — contract 4's negative invocation on the `"microvm"` fixture must make the derivation fail, and `drift.log` must contain the runtime-diff ERROR string. Sabotage: drop `runtime` from the projection expression and require this invocation to stop failing. Second sabotage: make the fixture identical to the real facts and require it to stop failing.
3. **A missing `runtime` on a non-hypervisor machine fails evaluation** — `expectFailure` on `(mkVmFacts (baseData // { machines = … alpha-dev without runtime … })).alpha-dev.runtime`. Sabotage: delete the presence check in `factFor`. Per contract 6 the expected red is **not** the named `assertMsg`: it is `vm-facts-negative` aborting with an uncaught `attribute 'runtime' missing`. Confirm that exact text, and confirm the fixture is green again once the check is restored.
4. **A non-string `runtime` throws** — `expectFailure` on the same path with `runtime = 42`. Sabotage: delete the type check and require `vm-facts-negative` to go red naming this fixture in its `assertMsg`. This one does produce the named diagnostic, because presence still holds.
5. **Valid fixture data evaluates and passes the value through** — `(mkVmFacts baseData).alpha-dev.runtime == "libvirt"`, asserted alongside `mutationFailures`. Its unique value is narrow and should be described as such: an unconditionally-throwing `factFor` would satisfy tests 3 and 4 within `vm-facts-negative` alone, but would already be caught three other ways in the same check set — `vm-facts-coherence` deep-forces every fact via `builtins.toJSON vmFacts` (`flake.nix:963`), and `pi-integration` (`:1743`) and `agent-vm-status` (`:1772`) force `vmFacts.allod-dev.username`. What this fixture uniquely covers is value passthrough on fixture-shaped data, independent of real inventory.
6. **The committed `scripts/vm-specs.json` the diff trusts is fresh** — the interpolated `inventory.checks.<system>.vm-specs-json` dependency from contract 4. Sabotage: point the interpolation at a nonexistent check attribute and require the `throw` fallback to fire, matching the idiom at `flake.nix:1713-1714`.

The existing name diff at `flake.nix:975-978` is unchanged and keeps its pre-existing coverage that vmFacts holds exactly the non-hypervisor names; it needs no new sabotage because this slice does not modify it.

Verification protocol for tests 2 through 6: apply each sabotage on its own, confirm the named check goes red with the outcome the test predicts, then restore. A fixture that stays green under its own sabotage does not count, and neither does one that goes red only under a different fixture's sabotage.

## Rollback Plan

Revert the PR. The change is one additive field plus check coverage in one repo, with no generated artifact, host action, or persistent state behind it, so a revert restores the exact prior `vmFacts` shape and the exact prior derivations.

Partial states are not reachable in a way that needs unwinding: the field and the checks land in the same commit, and no consumer exists yet — allod/nexus#21 and allod/nexus#22 are unstarted and `lib.mkVmFacts` has no external caller. Reverting after milestone 5 has begun would break those tools instead, which is the ordering reason this slice lands first.
