# microVM Persistent Volumes

## Tracking Issue

[allod/archetypes#25: Declare the persistent volumes a selected microvm archetype requires](https://forge.anarch.diy/allod/archetypes/issues/25)

Second slice of milestone 4 in [microvm-framework-adoption.md](microvm-framework-adoption.md), tracking [allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20). It implements that plan's **contract 13** and the archetypes half of **contract 6a**, and inherits 2, 9, 14 and 21 as constraints. One PR in `allod/archetypes` carrying `Closes allod/archetypes#25` and `Refs allod/strategy#20`; it does not close allod/strategy#20.

## Goal

A machine that selects the microvm runtime declares the disks it needs to keep state, so it evaluates and builds instead of failing on allod/vm's store-volume contract.

## Scope

In scope, all in `allod/archetypes` `flake.nix`:

- A per-archetype required-mount set: `/nix/var/nix` for every microvm guest, plus the dev user's home for dev.
- Derived `microvm.volumes` entries with every load-bearing field stated explicitly rather than inherited from an upstream default — see Interface Contracts 4.
- `fileSystems.<home>.neededForBoot = true`. Upstream sets this only for the store-overlay mount.
- A validated image root, and validation of every derived image path.
- Archetypes-side assertions over the merged `config.microvm.volumes`: required mounts present, exactly one entry each, writable, never auto-created, expected label and filesystem type, no duplicate mount points, no unsafe image path.
- Rework of the `runtime-module-selection` fixtures the declarations invalidate.

Out of scope, later slices of milestone 4: credential delivery and the guest credential root (contracts 7-12), guest networking (15), the `extendModules` host integration (8b), `vmFacts.<name>.runtime` (contract 1), and the nested-boot store-lifecycle tests that settle contract 6a.

Not in scope anywhere in this repo: creating, formatting, labelling or owning an image. Contract 13 keeps that an explicit deployer action, and `allod/nexus` `nix/microvm/launcher.nix:260-273` already refuses a missing, non-regular or inaccessible image before QEMU starts.

## Risk Assessment

Residual risk: **R2 Medium**, conditional on the sequencing gate below. Without that gate it is R3.

Why R2 holds:

- Every libvirt machine is untouched. Declarations are gated on `runtime == "microvm"` at the Nix level, so a libvirt machine's module list is unchanged and its derivation must not move.
- No public machine selects microvm, so the new path's public coverage is fixtures only.
- Rollback is a straight revert of one PR in one repo. Nothing creates or mutates an image, so a revert cannot strand state.

**The gate that keeps it R2:** this slice declares volumes but validates none of their runtime behavior. Nothing here proves a guest boots, mounts, or retains a byte. `microvm-test` must not be enabled or booted as a microVM on that basis — the store-lifecycle boots in a later slice are what license that, and the parent plan already blocks on them. If this lands and a machine is enabled before those pass, the honest score is R3.

**Correction to an earlier draft of this plan, kept because it changes what the tests are for.** That draft claimed omitting `neededForBoot` on the home volume causes silent data loss — a machine that comes up healthy and discards work. That is wrong, and review caught it. Measured: without it the home entry renders `defaults` with no `nofail`, so systemd's generator makes `local-fs.target` require the mount and a corrupt image fails the boot loudly into emergency. The real reason `neededForBoot` is required is contract 13's own: it moves validation into the initrd, "before sshd, user sessions, or any writer can use the tmpfs directory underneath". Omitting it leaves a window in which activation-time writers land on the tmpfs root beneath an unmounted mount point. That is a narrower defect than data loss and it is still worth closing, but the test exists to close a write window, not to prevent silent loss.

Human scrutiny, in order:

1. The merged `config.microvm.volumes` for a dev microvm fixture: two entries, distinct labels, distinct image paths, both writable, neither auto-created.
2. The generated `/etc/fstab`: both entries carry `x-initrd.mount`, which is what `neededForBoot` renders to.
3. The libvirt derivation paths, unchanged.

## Sequencing

The private `microvm-test` machine already declares `runtime = "microvm"` (allod/inventory#11, closed). Since allod/archetypes#23 merged it cannot evaluate, failing on contract 6a. **This PR is what unblocks evaluation**, and the private deploy lock should advance only after it lands.

It does not make that machine bootable. Image creation, formatting, labelling and ownership are deployer actions, and the store-lifecycle boots remain ahead. See the gate in Risk Assessment.

## Interface Contracts

Inherited and not restated: contract 6a, 13, 14, 21.

1. **The required set is per archetype, and `/nix/var/nix` is not dev-only.** `allod/vm` `modules/microvm-guest.nix:34` imports `microvm-store.nix` into *every* microvmGuest and its assertions carry no archetype guard — verified. A privacy machine selecting microvm requires `/nix/var/nix` exactly as a dev machine does; home is dev-only per contract 13. So a privacy microvm declares one volume, a dev microvm two.

2. **Gating happens outside the module system, on the module list.** `microvm.volumes` is not declared under a qemuGuest, so an ungated definition is an unmatched-option error. `lib.mkIf` does not help: it defers the value, not the definition. Gate with `lib.optional (runtime == "microvm")` on the builder's module list, using the `runtime` the builder already holds, so a libvirt machine's list is byte-identical to today's. A lexical `if`/`optionalAttrs` that discards the definitions before module evaluation is equally correct; list-level gating is chosen for symmetry with `guestModuleFor`.

   The same gating applies to the `fileSystems` entry for a different reason: `fileSystems` *does* exist under qemuGuest, so an ungated home entry would not error — it would silently render a bogus mount.

3. **Two distinct error classes, and only one escapes `tryEval`.** An unmatched option definition produced by `evalModules` *is* catchable by `tryEval`, including under `mkIf false`. A raw projection of an absent attribute — `config.microvm` on a libvirt machine — is not. An earlier draft conflated them. Consequence for the tests: probe the libvirt boundary with `lib.hasAttrByPath`, never by trying to catch `config.microvm`, which is the idiom the hypervisor guard already uses.

4. **Every load-bearing volume field is stated, not inherited.** Upstream defaults are not a contract; a default that moves silently changes what the deployer must produce. Each entry declares:
   - `readOnly = false` — load-bearing, and the launcher's writability probe keys on it.
   - `fsType = "ext4"` — this is the deployer's formatting contract, and the value the fstab entry renders.
   - `label` — `nix-state` for the state volume, `home` for home.
   - `autoCreate = false` — contract 13.
   - `size` — upstream makes the field mandatory but consumes it only inside the `autoCreate` path, so under `autoCreate = false` it is ignored. Set it to `0` and say so in a comment. Do not derive a fake capacity from `disk_gb`; a number that looks like a size but provisions nothing is worse than an obvious zero.

5. **Volumes are labelled, and the label is the deployer's contract.** Without a label upstream mounts by `/dev/vd<letter>` assigned by list position. The risk is not that appending a volume renumbers an existing one — it may not. It is that any later reorder or insertion silently remaps two valid ext4 images onto each other's mount points, corrupting state with no error. A wrong or missing label fails loudly at boot instead. Loud beats silent, so both volumes are labelled and the deployer applies those labels when formatting, which contract 13 already makes their job. `allod/nexus`'s unlabelled single-volume host fixture sets no policy for a two-volume archetype.

6. **The home mount point is the user's actual home.** `users.users.<username>.home` resolves to `/home/<username>` from nixpkgs' `isNormalUser` default; nothing in this repo set overrides it. Read the username from the identity the builder holds, per machine — privacy usernames are per-machine even though dev usernames currently are not.

7. **The image root is validated before any path is built from it.** Derived shape is `<imageRoot>/<machine>/<volume>.img`, defaulting to `/var/lib/allod-microvm-volumes` — the root `allod/nexus` already uses. Prefer a typed, microvm-only NixOS option over a bare builder argument: `runtime` had to be a builder argument because it selects the option tree before the module system exists, and that reason does not apply here. If `profileData` overridability requires a builder argument instead, it carries the same validation.

   Reject, with a named error, a value that is a Nix path, relative, non-normalized, contains `..` or a trailing slash, is under the Nix store, or is under `/run/allod`. `allod/nexus` `nix/microvm/host.nix:393-402` enforces the last two, but only after host composition, which this slice excludes — so archetypes cannot lean on it and must assert them itself.

8. **Assertions read the merged result, not the inputs.** Every check is over the final `config.microvm.volumes`, so a profile that overrides an entry is caught. Each required mount has exactly one entry; no two entries share a mount point; each is writable, not auto-created, and carries the expected label and filesystem type; every image path passes the root validation above.

## Agent Gates

None for implementation. Every acceptance test runs locally without host access or real credentials.

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

Then the whole set, which already builds every check on this system:

```sh
nix flake check --print-build-logs
```

`runtime-module-selection` must additionally assert, each with a paired sabotage fixture failing for the diagnostic it names:

1. **A dev microvm fixture evaluates with no placeholder.** The allod/archetypes#23 fixture that supplied its own `/nix/var/nix` volume is deleted; the builder's declarations must carry it. This is the positive proof the PR exists for.
2. **Its merged `config.microvm.volumes` is exactly right** — two entries; mount points the home path and `/nix/var/nix`; distinct labels `home` and `nix-state`; `fsType = "ext4"`; `readOnly = false`; `autoCreate = false`; distinct image paths both containing the machine name.
3. **Its generated `/etc/fstab` marks both volumes `x-initrd.mount`.** Read the built system's fstab, not the `fileSystems` attribute — that option is what `neededForBoot` renders to. Sabotage: drop `neededForBoot` from the home entry and require the fstab assertion to fail.
4. **A privacy microvm fixture evaluates with exactly one volume**, at `/nix/var/nix`, with no home volume.
5. **A dev microvm with volumes forced empty fails, and both diagnostics appear** — the archetypes mount-specific message and allod/vm's contract 6a message. NixOS collects every failed assertion into one combined throw, so neither precedes the other; assert both are present and make no ordering claim. This reuses the reworked `microvmWithoutVolume` fixture rather than adding another.
6. **A dev microvm missing only the home volume fails**, pinned to the archetypes diagnostic naming the home path. allod/vm has no home assertion, so archetypes is the only reporter here.
7. **Per-field sabotage on the home volume**, each pinned to its own diagnostic: a duplicate entry for the same mount point, `autoCreate = true`, and `readOnly = true`.
8. **A non-default image root moves both images**, and the derived paths still contain the machine name.
9. **Invalid image roots are rejected**, each pinned: a Nix path, a relative path, a store-backed path, one under `/run/allod`, and a non-normalized one containing `..`.
10. **A libvirt machine composes no `microvm` option at all** — `lib.hasAttrByPath` is false for `microvm` on a libvirt machine, proving the gating is outside the module system.

Pin diagnostics with the `config.assertions` idiom already in the check — readable without forcing the toplevel, so a mismatch reports as itself rather than as whatever the wrong composition breaks on first.

The existing `microvmWithoutVolume` fixture inverts once the builder declares volumes: it currently proves allod/vm's contract 6a fires. Rework it to `microvm.volumes = lib.mkForce []` so it still proves that, and reuse it for test 5.

## Rollback Plan

Revert the PR. The declarations are additive and gated on a runtime no public machine selects, so a revert restores the exact prior derivations for every libvirt machine and returns a microvm machine to failing on allod/vm's contract 6a — the state allod/archetypes#23 deliberately left.

No rollback step touches a disk image. Nothing here creates, formats, relabels or removes one, so there is no partial state to unwind on the host. A machine already provisioned with images keeps them; they simply stop being referenced.
