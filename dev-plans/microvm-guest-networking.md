# microVM Guest Networking

## Tracking Issue

[allod/archetypes#28: Declare the network interface a selected microvm archetype needs, and no address](https://forge.anarch.diy/allod/archetypes/issues/28)

Third slice of milestone 4 in [microvm-framework-adoption.md](microvm-framework-adoption.md), tracking [allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20). It implements that plan's **contract 15** and inherits 2, 13, 16b and 21 as constraints. One PR in `allod/archetypes` carrying `Closes allod/archetypes#28` and `Refs allod/strategy#20`; it does not close allod/strategy#20.

## Goal

A machine that selects the microvm runtime declares the one network interface it needs, carrying its inventory MAC and nothing else, so the host can attach a TAP to it without any address reaching the public guest definition.

## Scope

In scope, all in `allod/archetypes` `flake.nix`:

- One `microvm.interfaces` entry per selected microvm archetype: `type = "tap"`, a derived host interface `id`, and the inventory `mac` fact.
- Every load-bearing field stated rather than inherited, on the same reasoning as the volumes slice — see Interface Contracts 3.
- Archetypes-side assertions over the merged `config.microvm.interfaces`: exactly one entry, the expected type, the expected and structurally valid `id`, the expected and structurally valid `mac`, and the fields upstream leaves unchecked.
- An address scanner over the evaluated guest configuration and generated network units, with a named diagnostic, plus the sabotage that introduces an address and must fail it.
- Extension of the existing `runtime-module-selection` fixtures rather than a new check, for the memory reason in Interface Contracts 8.

Out of scope, later slices of milestone 4 and other repos: host TAP attachment, addressing, routes, DNS and `systemd-networkd` address configuration, which stay deployment inputs; credential delivery and the guest credential root (contracts 7-12); the `extendModules` host integration (8b); `vmFacts.<name>.runtime` (contract 1); and the nested-boot test that proves the interface reaches a fixture network (parent acceptance test 8), which needs a boot slice that does not exist yet.

Not in scope anywhere in this repo: creating, attaching, bridging or addressing the host TAP device. `allod/nexus` `nix/microvm/host.nix:89-91` already reads the guest's declared interfaces and `nix/microvm/launcher.nix:347-364` already creates each TAP owned by that VM's own principal, because upstream's generated `tap-up` hardcodes `user = "microvm"` (contract 16b).

## Risk Assessment

Residual risk: **R2 Medium**, on the same gate as the volumes slice.

Why R2 holds:

- Every libvirt machine is untouched. The declaration is gated on `runtime == "microvm"` at the Nix level, outside the module system, so a libvirt machine's module list is unchanged and its derivation must not move.
- No public machine selects microvm, so the new path's public coverage is fixtures only.
- Rollback is a straight revert of one PR in one repo. Nothing attaches, addresses or configures a host device, so a revert cannot strand host network state.

**The gate that keeps it R2:** this slice declares an interface and validates none of its runtime behavior. Nothing here proves a guest boots, that the TAP appears, or that a packet moves. Parent acceptance test 8 is what licenses that claim and it is not in this slice. If a machine is enabled on the strength of this PR, the honest score is R3.

**The residual defect this slice cannot close by itself.** The QEMU argument list is where the interface finally lands, and this slice does not read it. Building the runner realises the guest's erofs store disk, which is the whole system closure, and that is not affordable inside `nix flake check` on the public builder. Interface Contracts 7 states what replaces it and why the substitute is complete for the contract's negative half; the built-artifact scan belongs to the nested-boot slice, which builds a runner anyway. That deferral is recorded here so it is a decision rather than an omission.

Human scrutiny, in order:

1. The merged `config.microvm.interfaces` for a dev and a privacy microvm fixture: one entry each, `type = "tap"`, distinct `id`s carrying the machine name, each machine's own inventory MAC.
2. The rendered guest network units and the address scanner's finding list — specifically what the scanner reads, and whether its allowlist is small and named.
3. The libvirt derivation paths, unchanged.

## Interface Contracts

Inherited and not restated: contract 13, 16b, 21.

1. **Gating happens outside the module system, on the module list.** Identical to the volumes slice and for the identical reason: `microvm.interfaces` is not declared under `vm.nixosModules.qemuGuest`, so an ungated definition is an unmatched-option error on every libvirt machine, and `lib.mkIf` does not help because it defers the value while still producing the definition. Gate with `lib.optional (runtime == "microvm")` on the builder's module list, using the `runtime` the builder already holds. `runtime-module-selection`'s existing `microvmOptionLeaks` probe already proves the result with `lib.hasAttrByPath`, which is the required idiom because an undeclared option is a raw attribute error `builtins.tryEval` does not catch.

2. **The MAC is inventory data read the same way `runtime` is, and it is the only machine value that crosses.** The builders read `machines.<name>.mac` directly, as a module argument to the new module rather than as a builder argument: `runtime` needed a builder argument because it selects the option tree before the module system exists, and that reason does not apply here. Sabotage reaches the value through `microvm.interfaces = lib.mkForce [...]`, which is also what makes every assertion read the merged result rather than the definition.

   `machines.<name>.ip` sits one attribute away from `mac` in the same inventory attrset (`allod/inventory` `flake.nix:25,57,90`) and both are exported together by `lib.vmSpecsJson` (`flake.nix:179`). That adjacency is the specific accident contract 15 exists to prevent, and it is what the scanner's needles are keyed on.

   Inventory owns presence of the fact and does not currently give `mac` a named missing-value diagnostic the way it does `runtime`; a machine without one is a raw attribute error from the projection. Archetypes owns the *shape* of the value it hands to QEMU and asserts that. Closing the presence gap is inventory's business and is not in this slice.

3. **Every load-bearing interface field is stated, not inherited.** Upstream's submodule (`nixos-modules/microvm/options.nix:314-371` at the pinned `39a499ab85311b56dddb09ec43351cc3658f22c1`) declares `type`, `id`, `mac`, `bridge`, `macvtap.*` and `tap.vhost`. The entry states:
   - `type = "tap"` — the only type this arc supports. `user` is a userspace NAT with no host device, `bridge` invokes `qemu-bridge-helper`, and `macvtap` needs a host link name, which is deployment data.
   - `id` — the host TAP device name, per Interface Contracts 4.
   - `mac` — the inventory fact.
   - `tap.vhost = false` — stated because it is a host dependency, not a guest preference: `allod/nexus` `nix/microvm/launcher.nix:170-173` binds `/dev/vhost-net` into the VM's namespace only when it exists, and this slice boots nothing that could measure the throughput upstream claims for it. Enabling it is a separate, measured decision.
   - `bridge` is left at its `null` default and asserted null, because upstream asserts that a non-bridge interface must not define one (`nixos-modules/microvm/asserts.nix:32-49`) and archetypes should not be the thing that trips that.

4. **The host interface ID is derived, bounded, and a valid Linux interface name.** Shape is `tap-<machine>`, which is what `allod/nexus` `checks/microvm/fixtures.nix:82,172-174` already assumes and what `nix/microvm/launcher.nix:360` hands to `ip tuntap add name`. It is validated with a named archetypes diagnostic rather than left to upstream:
   - Non-empty, and at most 15 characters. Upstream asserts the length too (`asserts.nix:51-58`), but its message names only the interface, so a deployer whose *machine name* is too long learns nothing about the cause. Both diagnostics are pinned; NixOS collects every failed assertion into one combined throw, so neither precedes the other and no ordering is claimed.
   - Not `.` or `..`, and containing no `/`, no `:`, and no whitespace. These are the kernel's own `dev_valid_name` rules; upstream checks none of them, and an invalid name reaches `ip tuntap add` in a generated host script.

5. **The guest declares an interface, not a network.** The module sets `microvm.interfaces` and nothing else. It does not set `networking.interfaces`, `networking.defaultGateway`, `networking.nameservers`, `networking.search`, `networking.domain`, `systemd.network.networks`, or any address, route, gateway or DNS value anywhere. The parent plan assigns all of those to deployment integration.

   Two nixpkgs and upstream defaults are deliberately left alone and are not treated as violations, because neither is an address:
   - Upstream sets `networking.useNetworkd = lib.mkDefault true` for every microvm guest (`nixos-modules/microvm/optimization.nix:48`), so the guest already uses systemd-networkd. Bounding upstream's guest module is `allod/vm`'s job, not this slice's.
   - `networking.useDHCP` keeps its nixpkgs default, which renders a match-all DHCP `.network` unit. DHCP is a method for obtaining an address, not an address; it is the same default every libvirt guest carries, and it is what makes the interface usable once deployment attaches the TAP. The scanner's job is to reject *values*, not to reject the mechanism.

6. **The libvirt path is byte-identical and the addressing it uses is untouched.** Libvirt guests get their network stack from `allod/vm` `modules/qemu-guest.nix:24-25` (NetworkManager) and their addressing from the host's libvirt XML, which lives outside this repo. `allod/archetypes` sets no address today — `flake.nix:327`'s `networking.hostName` is the only networking assignment in the repo — and this slice adds none on the libvirt path. The four drvPaths in Acceptance Tests are the proof, not this paragraph.

7. **The address scanner reads what is reachable at evaluation, and says what it does not read.** Two halves, both required:

   **Structural.** Over the merged guest configuration: the interface entry matches Interface Contracts 3 and 4 exactly; `networking.defaultGateway` and `networking.defaultGateway6` are null; `networking.nameservers`, `networking.search`, `networking.interfaces.*.ipv4.addresses` and `.ipv6.addresses` are empty; `networking.domain` is null; and no `networking.localCommands`, `networking.hosts` entry beyond nixpkgs' own loopback names, or static route definition exists.

   **Textual.** Over artifacts, not over prose:
   - A JSON rendering of the merged `config.microvm` attrset, which is every route by which a value can reach the QEMU command line — `interfaces`, `kernelParams`, `forwardPorts`, `devices`, `extraArgs`, `qemu.extraArgs`, `socket`, `vsock`, `volumes`, `shares`. Render the whole attrset rather than a hand-picked subset, so an address route added by a future upstream option is scanned by default. Where an attribute is not JSON-renderable, project it out explicitly *and name it in a comment*, so the exclusion is visible rather than a silent hole.
   - `config.boot.kernelParams`, where an `ip=` argument is the classic way an address reaches a guest.
   - The rendered systemd network unit text, which is a cheap derivation. No unit may carry an `Address=`, `Gateway=`, `DNS=`, `Domains=` or route directive with a value.
   - The generated `/etc/hosts`, on the same cheap-`environment.etc` route the volumes slice already uses for `/etc/fstab`.

   Needles are two kinds, and both are required:
   - The exact `ip` string of every machine in the inventory set. This is the specific, high-value needle: it catches the actual accident, it cannot false-positive, and it fails loudly if a deploy's own inventory is substituted.
   - A generic dotted-quad, CIDR and IPv6 shape, with a small explicitly named allowlist — the loopback and unspecified addresses nixpkgs itself emits. If a legitimate match ever appears, the fix is to narrow the scan target or name the artifact, never to widen the allowlist quietly.

   The scanner must also assert the **positive** half: the inventory MAC is present in the scanned surface. A scanner that would pass on a guest with no interface at all is not testing this contract.

   **What it does not read, and why.** It does not read the built runner. `microvm.declaredRunner` is a `runCommand` whose scripts reference `microvm.storeDisk`, the guest's erofs store image, so building it builds the entire guest system closure — not affordable inside `nix flake check` here. The structural and JSON halves above cover every option that feeds the QEMU renderer (`lib/runners/qemu.nix:307-357` reads only `type`, `id`, `mac`, `bridge` and `tap.vhost` from each entry), so the deferral costs coverage of upstream's renderer *changing*, not of an address being introduced. The nested-boot slice builds a runner already and is where the built-artifact scan belongs; record that requirement there rather than leaving it implied.

8. **New coverage extends `runtime-module-selection` rather than adding a check.** That check already evaluates and forces `asMicrovm` and `asPrivacyMicrovm`; a second check would evaluate its own copies. Its own comments record that the sabotage fixtures are most of what takes `nix flake check` from a ~4 GB peak to over 5, so a new fixture is not free. Consequences, all of them binding:
   - Every new sabotage takes a thunk and is consumed exactly once, matching the existing `sabotages` list.
   - Interface rules are archetype-independent — the same module and validator run on both builders — so every interface sabotage is driven from the **privacy** microvm fixture, which costs a fraction of a dev one. This is the same decision the image-root rejections already record.
   - Where one fixture legitimately reaches two diagnostics, it pins both through the existing `needles` list rather than being split into two fixtures.
   - Peak RSS is measured with `\time -v` before and after, and the numbers go in the PR body. If the addition pushes the peak past the headroom on a 7 GiB box with no swap, fixtures are merged until it does not.

## Agent Gates

None for implementation. Every acceptance test runs locally without host access, a real TAP device, or real credentials. The agent cannot attach a TAP, configure a host bridge, or prove packet flow; parent acceptance test 8 and the private integration plan own that, and nothing in this slice claims it.

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

Then the whole set:

```sh
nix flake check --print-build-logs
```

`runtime-module-selection` must additionally assert the following. Every item whose text says "sabotage" needs a paired fixture that provably fails for the diagnostic it names, verified by deleting the assertion and confirming the check goes red, then restoring it. Diagnostics are pinned through the existing `config.assertions` idiom, which is readable without forcing the toplevel so a mismatch reports as itself.

1. **A dev microvm's merged `config.microvm.interfaces` is exactly one entry** — `type = "tap"`, `id = "tap-<dev machine>"`, `mac` equal to that machine's inventory MAC, `bridge = null`, `tap.vhost = false`. The expected MAC is read from `machines.<name>.mac`, not back off the fixture: an expectation taken from the configuration under test moves with it and proves nothing.
2. **A privacy microvm gets its own entry with its own MAC and its own `id`**, proving the derivation is per machine rather than shared.
3. **A libvirt machine composes no `microvm` option tree at all** — the existing `microvmOptionLeaks` probe already covers this; confirm it still holds with the interface declarations present, so the gating is proven outside the module system for the new module too.
4. **Sabotage: a second interface entry** — pinned to the archetypes "exactly one" diagnostic and to upstream's duplicate-`id` message when the fixture reaches both.
5. **Sabotage: the entry's type forced to `user`, and separately to `bridge`** — pinned to the archetypes type diagnostic. The `bridge` case must not be allowed to pass merely because upstream's bridge assertion fires first; pin the archetypes needle.
6. **Sabotage: a wrong but structurally valid `id`** — pinned to the archetypes "must be the derived host interface name" diagnostic alone, so it defends the equality check rather than the validity check.
7. **Sabotage: a structurally invalid `id`** — one case per rule that has no other reporter: containing `/`, containing whitespace, and `..`. Each pinned to its own needle.
8. **Sabotage: a 16-character `id`** — pinned to both the archetypes length diagnostic naming the machine and upstream's own 15-character message, with no ordering claim between them.
9. **Sabotage: a malformed `mac`** — a value that is not six colon-separated hex octets, pinned to the archetypes format diagnostic.
10. **Sabotage: a multicast `mac`** — first octet with its low bit set, and separately the all-zero MAC. Upstream and QEMU accept both and the guest strands at boot, which is exactly the failure class this arc cares about.
11. **Sabotage: a `mac` that is not the machine's inventory MAC** — pinned to the archetypes equality diagnostic, proving the value is bound to inventory rather than to any well-formed string.
12. **The address scanner finds nothing on the passing fixtures**, and finds the MAC. Run over both the dev and the privacy microvm fixture.
13. **Sabotage: an address introduced into the guest** — at least three independent routes, each pinned, because a scanner that reads only one surface would pass the others: a `networking.interfaces.<if>.ipv4.addresses` entry, a `networking.defaultGateway` or `networking.nameservers` value, and an `ip=` entry in `boot.kernelParams`. At least one case must use the exact inventory `ip` of a machine in the set, and at least one must use an address that appears nowhere in inventory, so both needle kinds are proven to bite.
14. **Sabotage: the interface removed entirely** — `microvm.interfaces = lib.mkForce []`. This must fail, both on the archetypes "exactly one" diagnostic and on the scanner's positive MAC assertion. Without it, item 12's scanner would pass a guest with no networking at all.

Memory, reported in the PR body rather than asserted:

```sh
\time -v nix flake check --print-build-logs 2>&1 | tail -40
```

Record the maximum resident set size before and after the change and state the headroom against the 7 GiB box.

## Rollback Plan

Revert the PR. The declaration is additive and gated on a runtime no public machine selects, so a revert restores the exact prior derivations for every libvirt machine and returns a microvm machine to declaring no interface, which is the state before this slice: it evaluates and builds, and its generated QEMU command carries no `-netdev` argument.

No rollback step touches a host network device. Nothing here creates, renames, attaches or addresses a TAP — `allod/nexus`'s launcher does that at VM start from whatever the guest declares — so there is no partial host state to unwind. A host that has already started a VM under this declaration keeps its TAP until that VM's unit stops, and the unit's own `ExecStop` removes it.
