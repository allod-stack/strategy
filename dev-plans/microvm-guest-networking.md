# microVM Guest Networking

## Tracking Issue

[allod/archetypes#28: Declare the network interface a selected microvm archetype needs, and no address](https://forge.anarch.diy/allod/archetypes/issues/28)

Third slice of milestone 4 in [microvm-framework-adoption.md](microvm-framework-adoption.md), tracking [allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20). It implements that plan's **contract 15** and inherits 2, 13, 16b and 21 as constraints. One PR in `allod/archetypes` carrying `Closes allod/archetypes#28` and `Refs allod/strategy#20`; it does not close allod/strategy#20.

## Goal

A machine that selects the microvm runtime declares the one network interface it needs, carrying its inventory MAC and nothing else, so the host can attach a TAP to it without any address reaching the public guest definition.

## Scope

In scope, all in `allod/archetypes` `flake.nix`:

- One `microvm.interfaces` entry per selected microvm archetype: `type = "tap"`, a derived and overridable host interface `id`, and the inventory `mac` fact.
- Every load-bearing field stated rather than inherited, on the same reasoning as the volumes slice — see Interface Contracts 3.
- Archetypes-side assertions over the merged `config.microvm.interfaces`, reported per entry: exactly one entry, the expected type, the expected and structurally valid `id`, the expected and structurally valid `mac`, and the fields upstream leaves unchecked.
- An address scanner over the evaluated guest configuration, the rendered QEMU command line, the generated host TAP scripts, and the generated network units, with the sabotages that introduce an address and must fail it.
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

**The one soundness caveat, stated rather than buried.** The scanner reads the rendered QEMU command line by re-importing the pinned upstream renderer against the guest's own merged configuration (Interface Contracts 7). That is a faithful reconstruction of the artifact, not the artifact itself: if upstream later changes how `declaredRunner` assembles the command, the reconstruction can diverge from what the host executes. Interface Contracts 7 requires a guard that makes that divergence loud, and parent acceptance test 8 reads the real runner in the nested-boot slice. The alternative — building `declaredRunner` here — is not available: `lib/runner.nix:230-232` symlinks the guest `toplevel` into the runner output and the `microvm-run` script embeds `${storeDisk}`, so building it builds the entire dev guest system and its erofs store image.

Human scrutiny, in order:

1. The merged `config.microvm.interfaces` for a dev and a privacy microvm fixture: one entry each, `type = "tap"`, distinct `id`s carrying the machine name, each machine's own inventory MAC.
2. The reconstructed QEMU command file and its guard — specifically whether the guard would actually catch a reconstruction built from the wrong configuration.
3. The scanner's needle set and allowlist: small, named, and anchored, rather than a widening list of exceptions.
4. The libvirt derivation paths, unchanged.

## Interface Contracts

Inherited and not restated: contract 13, 16b, 21.

1. **Gating happens outside the module system, on the module list.** Identical to the volumes slice and for the identical reason: `microvm.interfaces` is not declared under `vm.nixosModules.qemuGuest`, so an ungated definition is an unmatched-option error on every libvirt machine, and `lib.mkIf` does not help because it defers the value while still producing the definition. Gate with `lib.optional (runtime == "microvm")` on the builder's module list, using the `runtime` the builder already holds, next to the existing volume gates at `flake.nix:485` and `flake.nix:517`. `lib.optional` is lazy in its second argument, so a libvirt machine never forces `machines.<name>.mac`. `runtime-module-selection`'s existing `microvmOptionLeaks` probe already proves the result with `lib.hasAttrByPath`, which is the required idiom because an undeclared option is a raw attribute error `builtins.tryEval` does not catch.

   `sharedModules name runtime` (`flake.nix:321-333`) takes the runtime but not the machine record, so it cannot carry the MAC without a signature change. The volumes precedent passes per-machine data as an explicit argument to the gated module, and this follows it.

2. **The MAC is inventory data read the same way `runtime` is, and it is the only machine value that crosses.** The builders read `machines.<name>.mac` and hand it to the new module as an argument rather than as a builder argument: `runtime` needed a builder argument because it selects the option tree before the module system exists, and that reason does not apply here. Sabotage reaches the value through `microvm.interfaces = lib.mkForce [...]`, which is also what makes every assertion read the merged result rather than the definition.

   `machines.<name>.ip` sits immediately above `mac` in the same inventory attrset — `allod/inventory` `flake.nix:24,25` for `allod-dev`, `:56,57` for `nexus`, `:89,90` for `privacy-1` — and both are exported together by `lib.vmSpecsJson` (`flake.nix:179`, `inherit (m) memory_mb vcpus disk_gb ip mac forge_key repos runtime;`). That adjacency is the specific accident contract 15 exists to prevent, and it is what the scanner's exact needles are keyed on.

   Inventory validates `platform` and `runtime` and does not validate `mac` at all; its `checkedMachines` trip-wire forces `mkVmSpecs`, not `mkVmSpecsJson`, so a machine without a `mac` is a raw attribute error rather than a named diagnostic. Archetypes owns the *shape* of the value it hands to QEMU and asserts that. Closing the presence gap is inventory's business and is not in this slice.

3. **Every load-bearing interface field is stated, not inherited.** Upstream's submodule (`nixos-modules/microvm/options.nix:314-371` at the pinned `39a499ab85311b56dddb09ec43351cc3658f22c1`) declares `type`, `id`, `mac`, `bridge`, `macvtap.link`, `macvtap.mode` and `tap.vhost`. The entry states:
   - `type = "tap"` — the only type this arc supports. `user` is a userspace NAT with no host device, `bridge` invokes `qemu-bridge-helper`, and `macvtap` needs a host link name, which is deployment data.
   - `id` — the host TAP device name, per Interface Contracts 4.
   - `mac` — the inventory fact.
   - `tap.vhost = false` — stated because it is a host dependency, not a guest preference: `allod/nexus` `nix/microvm/launcher.nix:170-173` binds `/dev/vhost-net` into the VM's namespace only when it exists, and this slice boots nothing that could measure the throughput upstream claims for it. Enabling it is a separate, measured decision.
   - `bridge` is left at its `null` default and asserted null, because upstream asserts that a non-bridge interface must not define one (`nixos-modules/microvm/asserts.nix:32-49`).
   - `macvtap.link` and `macvtap.mode` are left undefined, which is legal for a `tap` entry because upstream reads them only under a `type == "macvtap"` guard. They are the reason a whole-entry `builtins.toJSON` throws — see Interface Contracts 7.

4. **The host interface ID is derived, overridable, bounded, and a valid Linux interface name.** Default shape is `vm-<machine>`, exposed through a typed archetypes option so a machine whose name does not fit can supply its own, exactly as `allod.archetypes.microvm.volumeImageRoot` already does for images. The validator runs on the **final merged value**, not on the default.

   The prefix is three characters and not four for a concrete reason: the first machine that selects this runtime is named `microvm-test` (twelve characters), which the volumes slice explicitly unstranded. Under a `tap-` prefix its derived id is sixteen characters and fails both the archetypes length rule and upstream's, so the slice that gives it an interface would re-strand it on the same day, and the parent plan (`microvm-framework-adoption.md:160`) makes renaming a human-only action because per-machine encrypted secret filenames are keyed to the name. `vm-` leaves a twelve-character machine-name budget, which that name exactly fits, and the override exists for the next one that does not. State the budget in the option description so it is discoverable before a deployer hits it.

   `tap-<machine>` appears in `allod/nexus` `checks/microvm/fixtures.nix:82,172-174` and `checks/microvm/isolation.nix:87`, but those are nexus's own fixtures setting their own ids. Nexus reads whatever the guest declares (`nix/microvm/host.nix:89-91`) and couples to nothing but the value, so the prefix is archetypes' to choose.

   Validation, with a named archetypes diagnostic per rule and reported per entry:
   - Non-empty, and at most 15 characters. Upstream asserts the length too (`asserts.nix:51-58`), but its message names only the interface, so a deployer whose *machine name* is too long learns nothing about the cause. Both diagnostics are pinned; NixOS collects every failed assertion into one combined throw, so neither precedes the other and no ordering is claimed.
   - Not `.` or `..`, and containing no `/`, no `:`, and no whitespace. These are the kernel's own `dev_valid_name` rules; upstream checks none of them, and an invalid name reaches `ip tuntap add` in a generated host script.

   Two things this slice deliberately does not check, recorded so they are decisions: cross-VM `id` uniqueness, which follows from machine-name uniqueness because the default is derived per machine and nothing else in the arc mints one; and collision with a real host link, which archetypes cannot see. The second is worth a comment at the derivation, because upstream's `tap-up` deletes any pre-existing link of that name as root (`nixos-modules/microvm/interfaces.nix`), so a collision is destructive rather than an error.

5. **The guest declares an interface, not a network.** The module sets `microvm.interfaces` and nothing else. It does not set `networking.interfaces`, `networking.defaultGateway`, `networking.nameservers`, `networking.search`, `networking.domain`, `systemd.network.networks`, or any address, route, gateway or DNS value anywhere. The parent plan assigns all of those to deployment integration.

   Two defaults are deliberately left alone and are not treated as violations, because neither is an address:
   - Upstream sets `networking.useNetworkd = lib.mkDefault true` for every microvm guest (`nixos-modules/microvm/optimization.nix:48`), so the guest already uses systemd-networkd. Bounding upstream's guest module is `allod/vm`'s job, not this slice's.
   - `networking.useDHCP` keeps its nixpkgs default. DHCP is a method for obtaining an address, not an address, and it is what makes the interface usable once deployment attaches the TAP.

   Measured on a microvm guest at the pinned revision with one interface and no explicit configuration, so the scanner's expectations are facts rather than guesses: exactly two network units render — `99-ethernet-default-dhcp` (`[Match] Type=ether, Kind=!*`; `[Network] DHCP=yes, IPv6PrivacyExtensions=kernel`) and `99-wireless-client-dhcp` — plus `systemd/networkd.conf`, and none carries an `Address=`, `Gateway=`, `DNS=` or `Domains=` directive with a value. `defaultGateway`, `defaultGateway6` and `domain` are null; `nameservers`, `search` and `networking.interfaces` are empty; `localCommands` is `""`; `networking.hosts` is `{ "127.0.0.2" = [ <hostname> ]; }`; and the generated `/etc/hosts` is three lines — `127.0.0.1 localhost`, `::1 localhost`, `127.0.0.2 <hostname>`. The generic-needle allowlist is therefore exactly `127.0.0.1`, `127.0.0.2` and `::1`, and it is expected to stay that size.

6. **The libvirt path is byte-identical and the addressing it uses is untouched.** Libvirt guests get their network stack from `allod/vm` `modules/qemu-guest.nix:24-25` (NetworkManager, which sets `useDHCP = false` and drives the wired NIC with `ipv4.method=auto`) and their addressing at runtime from libvirt's dnsmasq, bound by a MAC-keyed reservation that `allod/nexus` `scripts/new-vm` writes with `virsh net-update … ip-dhcp-host`. That script is the only consumer of the `mac` fact in the whole stack today, and there is no literal domain XML in any public repo — `virt-install` synthesizes it.

   `allod/archetypes` assigns no address anywhere: the only networking assignments in the repo are `networking.hostName` at `flake.nix:327` and `flake.nix:548`, and a read of `osConfig.networking.hostName` in `modules/ai-agents.nix:19`. This slice adds none on the libvirt path. The four drvPaths in Acceptance Tests are the proof, not this paragraph.

7. **The address scanner reads the artifact, and names what it cannot read.** Three surfaces, all required, all reachable at evaluation.

   **(a) The rendered QEMU command line.** This is where the interface finally lands, and it is reachable without building anything. Locate the pinned upstream source from the evaluated guest itself — the option's own declaration path, `dirOf (dirOf (dirOf (builtins.head sys.options.microvm.interfaces.declarations)))` — then import that source's `lib/runners/qemu.nix` against the guest's merged `config.microvm`, matching exactly the argument set `nixos-modules/microvm/default.nix` passes it, and write the resulting `command` string to a file with `builtins.unsafeDiscardStringContext`. Discarding the context is what keeps it from *building*: it strips every `inputDrv`, so the `writeText` builds instantly and nothing realises the store disk.

   It is not, however, free to *evaluate*. **Corrected against the implementation's measurement, which wins:** one command render costs about **0.5 GB**, not the ~40 MB an earlier probe suggested (2.78 s / 635 MB against 2.88 s / 678 MB). The command embeds `config.microvm.storeDisk`, and the divergence guard below asserts on that exact path, so rendering or guarding forces the erofs store-image derivation and with it a walk of the guest's whole store closure. Discarding string context removes the build, not the evaluation that produced the path. The earlier probe measured a shape that never forced `storeDisk`.

   That cost is what shapes the implementation: the rendered command is passed to every reader as an argument rather than re-rendered per reader, so each scanned configuration pays for exactly one render; and the passing scan is spent on the privacy fixture rather than the dev one, with the address sabotage spliced onto a dev fixture that already exists for the image-root findings.

   Deriving the source path from the guest's own option declarations, rather than from a new flake input or a transitive-input lookup, is what keeps this from being a second pin.

   This is a reconstruction, so it needs a guard that makes divergence loud rather than silent. Assert that the rendered command contains the exact `config.microvm.storeDisk` path and the exact `config.microvm.kernelParams` string the merged configuration reports, so a reconstruction assembled from the wrong configuration fails instead of scanning a plausible-looking string that no host will ever run. A signature change in upstream's renderer fails the import outright, which is the loud case; a semantic change is what parent acceptance test 8 covers when it reads the real runner.

   **(b) The generated host TAP scripts.** `config.microvm.binScripts` is `attrsOf lines` (`options.nix:1017-1023`) — plain strings, free to read. On a probe guest it held the full upstream `tap-up`/`tap-down` text including `ip tuntap add name 'vm-<machine>' mode tap user 'microvm' vnet_hdr`. That is a host-side generated artifact carrying the interface id, and nothing else in the arc scans it at this stage.

   **(c) An explicitly projected JSON of the remaining guest surface**, plus the structural assertions. `builtins.toJSON config.microvm` **does not work** and must not be attempted: measured, it throws on five attributes — `interfaces` (because `macvtap.link` and `macvtap.mode` have no default, so a `tap` entry hits "used but not defined"), `runner` (`lib.genAttrs` over eight hypervisors, of which `vfkit` throws `"vfkit only works on macOS (Darwin)"` as soon as its `drvPath` is demanded), `vmHostPackages` (`types.pkgs`, the whole nixpkgs set), `vfkit`, and `balloonMem`. Project instead, in the shape `volumesOf` already uses at `flake.nix:1416-1420`: the interface entries field by field, plus `preStart`, `socket`, `vsock`, `forwardPorts`, `devices`, `credentialFiles`, `kernelParams` and `extraArgsScript`. Name the five excluded attributes in a comment so the exclusion is visible rather than a silent hole.

   Also scanned: `config.boot.kernelParams`, the rendered systemd network unit text, and the generated `/etc/hosts`, on the cheap `environment.etc` route the volumes slice already uses for `/etc/fstab`. `config.microvm.kernelParams` is a strict superset of `boot.kernelParams` (`nixos-modules/microvm/system.nix:31-39` appends `init=${toplevel}/init`, and `allod/vm` adds `allod.regInfo=…`), so scanning both is correct and not redundant; note that forcing it forces `system.build.toplevel`.

   **Two routes that must be closed by assertion rather than by scanning.** `config.microvm.extraArgsScript` appends its *stdout at VM start* to the QEMU command line (`lib/runner.nix:179-185`), so no eval-time scan can see what it produces; assert it stays null. And `config.networking.hostName` is injected into `microvmConfig` by `nixos-modules/microvm/default.nix:32-34` and renders as `-name <hostName>` — the one input to the command that is neither under `config.microvm` nor in `boot.kernelParams`. There is no top-level `microvm.extraArgs`; the merged surface has `qemu.extraArgs` and `extraArgsScript`.

   **Needles are two kinds, and both are required.**
   - The exact `ip` string of every machine in the inventory set. This is the specific, high-value needle: it catches the actual accident, it cannot false-positive, and it follows a deploy's own inventory when substituted.
   - A generic dotted-quad, CIDR and IPv6 shape with the three-entry allowlist from Interface Contracts 5. This one needs anchoring: the scanned text carries store paths such as `init=/nix/store/…-nixos-system-…-25.11.20260630.b6018f8/init`, and four-component version strings are not rare. Exclude store paths or anchor matches to non-digit, non-dot delimiters. If a legitimate match ever appears, the fix is to narrow the scan target or the anchor, never to widen the allowlist quietly.

   The scanner must also assert the **positive** half: the inventory MAC appears in the rendered command. A scanner that would pass on a guest with no interface at all is not testing this contract.

8. **New coverage extends `runtime-module-selection` rather than adding a check, and the fixture merges are planned, not discovered.** That check already evaluates and forces `asMicrovm` and `asPrivacyMicrovm`; a second check would evaluate its own copies. Its own comments record that the sabotage fixtures are most of what takes `nix flake check` from a ~4 GB peak to over 5, and it already carries sixteen of them (`flake.nix:1555-1634`), so doubling that is not acceptable. Consequences, all binding:

   - Every new sabotage takes a thunk and is consumed exactly once, matching the existing `sabotages` list.
   - Interface rules are archetype-independent — `mkPrivacyVm` composes the identical module through the identical gate at `flake.nix:517` — so every interface sabotage is driven from the **privacy** microvm fixture, which costs a fraction of a dev one. This mirrors the image-root decision recorded in-line at `flake.nix:1510-1516`.
   - The archetypes validator reports **per entry**, mapping over `config.microvm.interfaces` rather than reading `builtins.head`. That is what lets one fixture with several entries reach several diagnostics, and it is the reason the merges below are legitimate rather than a way of hiding cases.
   - Planned merges, taking the Acceptance Tests list from sixteen fixtures to nine: the three invalid ids into one fixture of three entries; the malformed, multicast and zero MACs into one fixture of three entries; the `user` and `bridge` types into one; and the wrong-id and wrong-MAC equality cases onto one entry.

     **Corrected against the implementation, which wins: nine was still unaffordable, and the slice shipped four.** At the render cost in Interface Contracts 7 the merge went further than planned, and the per-entry reporter is what made it sound rather than a way of hiding cases. What shipped: one privacy fixture carrying fourteen interface entries and reaching nineteen separately pinned diagnostics, which absorbs every id, MAC, type and equality case above plus upstream's duplicate-id and length assertions and the `extraArgsScript` rule; one fixture for the empty interface list, which is the single shape that cannot share a fixture with any entry; one for the reconstruction guard; and the address sabotage, which adds no evaluation at all because it is spliced onto the dev fixture the image-root findings already build. Three of the four cost a new evaluation.

     The rule that bounds this, stated so a later slice does not read the merge as licence: a fixture may absorb another only while every needle stays anchored to its own reporter, so deleting one rule turns the fixture red for that rule alone. A merge that would leave a case provable only by the fixture failing for some reason is not affordable at any memory price.
   - Peak RSS is measured with `\time -v` before and after, and the numbers go in the PR body. `nix eval --raw .#checks.x86_64-linux.runtime-module-selection.drvPath` forces every fixture and every sabotage without building a toplevel, and is the right loop while iterating; the full `nix flake check` is the gate. Measured on the merged result: 44 s and 4.83 GB peak.

9. **The scanner needs its own harness, because an address evaluates cleanly.** The existing `checkSabotage` (`flake.nix:1640-1644`) requires the fixture's `system.build.toplevel.drvPath` to *fail* and the needle to appear in `config.assertions`. Measured: a guest with `networking.interfaces.eth0.ipv4.addresses = [{ address = "192.0.2.10"; prefixLength = 24; }]` evaluates cleanly, passes every assertion, and renders `40-eth0.network` containing `Address=192.0.2.10/24`. So the address sabotages cannot ride the `sabotages` list. Build them as a bound findings list asserted at check level, in the shape of the existing `movedRootFindings` and `unmountedVolumeFindings` (`flake.nix:1445-1491`): the scanner is a function from a system to a list of findings, applied to the passing fixtures (must be empty) and to each address fixture (must be non-empty and must name the value it found).

   One consequence for the sabotage that removes the interface entirely: the archetypes "exactly one entry" assertion does fail evaluation, so that half belongs on the `sabotages` list; the scanner's positive-MAC half belongs on the findings harness. Split it rather than trying to reach both through `needles`.

   Note for the implementer: `config.environment.etc."systemd/network/40-eth0.network".source` resolves to a file *inside* a directory derivation, so interpolating it at the top level of an expression errors on string context. Inside a `runCommand`, the way `fstabOf` is used at `flake.nix:1721`, it works.

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

`runtime-module-selection` must additionally assert the following. Every sabotage needs a paired fixture that provably fails for the diagnostic it names, verified by deleting the assertion and confirming the check goes red, then restoring it. Evaluation sabotages pin their diagnostics through the existing `config.assertions` idiom; scanner sabotages use the findings harness of Interface Contracts 9.

**Positive:**

1. **A dev microvm's merged `config.microvm.interfaces` is exactly one entry** — `type = "tap"`, `id = "vm-<dev machine>"`, `mac` equal to that machine's inventory MAC, `bridge = null`, `tap.vhost = false`. The expected MAC is read from `machines.<name>.mac`, not back off the fixture: an expectation taken from the configuration under test moves with it and proves nothing.
2. **A privacy microvm gets its own entry with its own MAC and its own `id`**, proving the derivation is per machine rather than shared.
3. **A non-default host interface ID moves the entry**, and the derived default is not baked into the check — the same shape as the existing `movedRootFindings`.
4. **A libvirt machine composes no `microvm` option tree at all** — the existing `microvmOptionLeaks` probe, confirmed to still hold with the interface declarations present.
5. **The scanner finds nothing on both passing fixtures, and finds the MAC and the interface id in the rendered QEMU command.** Includes the reconstruction guard: the command contains the exact `microvm.storeDisk` path and `microvm.kernelParams` the merged configuration reports.

**Evaluation sabotages, all driven from the privacy fixture. These are the rules that must each reach their own diagnostic, not a count of fixtures — Interface Contracts 8 records how far they were merged and why:**

6. **A second interface entry** — pinned to the archetypes "exactly one" diagnostic, and to upstream's duplicate-`id` message when the fixture reaches it.
7. **A wrong type** — one fixture carrying a `user` entry and a `bridge` entry, each pinned to the archetypes type diagnostic. The `bridge` case must pin the archetypes needle specifically, not pass merely because upstream's bridge assertion fires first.
8. **A wrong but structurally valid `id`, and a wrong `mac`, on one entry** — pinned to the two archetypes equality diagnostics separately. The MAC used is well-formed, unicast and non-zero (for example `52:54:00:00:00:99`) so it trips the equality reporter alone; upstream performs no MAC validation at all, so nothing else can catch it.
9. **Structurally invalid `id`s** — one fixture of three entries: containing `/`, containing whitespace, and `..`. Each pinned to its own needle.
10. **A 16-character `id`** — pinned to both the archetypes length diagnostic naming the machine and upstream's own 15-character message, with no ordering claim between them.
11. **Invalid MACs** — one fixture of three entries: not six colon-separated hex octets, multicast (low bit of the first octet set), and all-zero. Each pinned to its own needle. Upstream and QEMU accept the last two and the guest strands at boot, which is exactly the failure class this arc cares about.
12. **The interface removed entirely** — `microvm.interfaces = lib.mkForce []`, pinned to the archetypes "exactly one" diagnostic.

**Scanner sabotages, on the findings harness, each pinned to the value it must report:**

13. **A static address** — `networking.interfaces.<if>.ipv4.addresses`. Use the exact inventory `ip` of a machine in the set, so the exact-needle kind is proven to bite.
14. **A gateway and a nameserver** — `networking.defaultGateway = { address = "..."; interface = "..."; }` and `networking.nameservers`. The attrset form is required: measured, the bare-string form fails on nixpkgs' own `"networking.defaultGateway.interface is not optional when using networkd."` assertion and never reaches the scanner. Use an address that appears nowhere in inventory, so the generic-needle kind is proven to bite.
15. **An `ip=` kernel parameter** — `boot.kernelParams`, which reaches the rendered QEMU `-append` argument. This is the case that proves surface (a) is really being read.
16. **The interface removed entirely**, on the scanner side — the positive MAC assertion must fail. Without it, item 5 would pass a guest with no networking at all.

Memory, reported in the PR body rather than asserted:

```sh
\time -v nix flake check --print-build-logs 2>&1 | tail -40
```

Record the maximum resident set size before and after the change and state the headroom against the 7 GiB box.

## Rollback Plan

Revert the PR. The declaration is additive and gated on a runtime no public machine selects, so a revert restores the exact prior derivations for every libvirt machine and returns a microvm machine to declaring no interface, which is the state before this slice: it evaluates and builds, and its generated QEMU command carries no `-netdev` argument.

No rollback step touches a host network device. Nothing here creates, renames, attaches or addresses a TAP — `allod/nexus`'s launcher does that at VM start from whatever the guest declares — so there is no partial host state to unwind. A host that has already started a VM under this declaration keeps its TAP until that VM's unit stops, and the unit's own `ExecStop` removes it.
