# microVM Persistent Volumes

## Tracking Issue

[allod/archetypes#25: Declare the persistent volumes a selected microvm archetype requires](https://forge.anarch.diy/allod/archetypes/issues/25)

Second slice of milestone 4 in [microvm-framework-adoption.md](microvm-framework-adoption.md), tracking [allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20). It implements that plan's **contract 13** and the archetypes half of **contract 6a**, and inherits 2, 9, 14 and 21 as constraints. One PR in `allod/archetypes` carrying `Closes allod/archetypes#25` and `Refs allod/strategy#20`; it does not close allod/strategy#20.

## Goal

A machine that selects the microvm runtime declares the disks it needs to keep state, so it evaluates and builds instead of failing on allod/vm's store-volume contract.

## Scope

In scope, all in `allod/archetypes` `flake.nix`:

- A per-archetype required-mount set: `/nix/var/nix` for every microvm guest, plus the dev user's home for dev.
- Derived `microvm.volumes` entries for those mounts — image path, label, size, `autoCreate = false`.
- `fileSystems.<home>.neededForBoot = true` for the home volume. Upstream sets this only for the store-overlay mount; see Interface Contracts 3.
- An overridable image-root builder argument, defaulted and threaded like `runtime`.
- An archetypes-side assertion naming any required mount without a volume.
- Rework of the `runtime-module-selection` fixtures the declarations invalidate.

Out of scope, later slices of milestone 4: credential delivery and the guest credential root (contracts 7-12), guest networking (15), the `extendModules` host integration (8b), `vmFacts.<name>.runtime` (contract 1), and the nested-boot store-lifecycle tests that settle contract 6a.

Not in scope anywhere in this repo: creating, formatting, labelling or owning an image. Contract 13 keeps that an explicit deployer action, and `allod/nexus` `nix/microvm/launcher.nix:260-273` already refuses a missing, non-regular or inaccessible image before QEMU starts.

## Risk Assessment

Residual risk: **R2 Medium**, with one R3 signal called out below.

Why:

- Every libvirt machine is untouched. The declarations are gated on `runtime == "microvm"` at the Nix level, so a libvirt machine's module list is identical and its derivation must not move. Same acceptance test as allod/archetypes#23.
- No real machine in the public inventory selects microvm, so the new path's only public coverage is fixtures. The private `microvm-test` machine does select it — see Sequencing.
- Rollback is a straight revert of one PR in one repo. Nothing creates or mutates a disk image, so a revert cannot strand state.

The R3 signal, stated plainly rather than averaged away: **a missing `neededForBoot` on the home volume loses user data silently.** Without it a corrupt, unformatted or mislabelled home image is skipped at boot and the guest writes to the tmpfs root underneath, so the machine comes up looking healthy and discards the user's work on the next restart. Nothing in `allod/vm`, `allod/nexus` or upstream asserts it for a non-overlay mount — the auto-add at `mounts.nix:102-117` is guarded on `mountPoint == config.microvm.writableStoreOverlay`. This plan's assertion and its paired sabotage fixture are the only thing standing between that setting and silent loss.

Human scrutiny, in order:

1. The generated `/etc/fstab` for a microvm dev fixture: the home entry must carry `x-initrd.mount`, which is what `neededForBoot` renders to and what allod/vm's own contract check greps for on the state volume.
2. The libvirt derivation paths, unchanged.
3. Whether every new assertion has a sabotage fixture that fails for the diagnostic it names.

## Sequencing

The private `microvm-test` machine already declares `runtime = "microvm"` (allod/inventory#11, closed). Since allod/archetypes#23 merged, that machine cannot evaluate — it fails on contract 6a. **This PR is what unblocks it**, and the private deploy lock should advance only after this lands.

That also means this PR's blast radius on the private side is larger than on the public side: it is the change that first makes a real machine composable as a microVM. It does not make it bootable — image creation and the store-lifecycle boots remain ahead.

## Interface Contracts

Inherited and not restated: contract 6a (one coherent writable Nix store at `/nix/var/nix`), 13 (every required persistent path maps to an explicitly provisioned writable volume), 14 (persistence and secrets never overlap), 21 (libvirt stays first-class).

This PR adds:

1. **The required set is per archetype, and `/nix/var/nix` is not dev-only.** `allod/vm` `modules/microvm-guest.nix:34` imports `microvm-store.nix` into *every* microvmGuest, and its assertions carry no archetype guard. So a privacy machine selecting microvm requires `/nix/var/nix` exactly as a dev machine does. Home is dev-only, per contract 13. A privacy microvm therefore declares one volume, a dev microvm two.

2. **Gating happens outside the module system.** `microvm.volumes` does not exist in a qemuGuest's option tree, and an unmatched definition is raised while building that tree — before any config read, and not catchable by `tryEval`. `lib.mkIf` does not help: it defers the value, not the definition. So the volume modules are appended with `lib.optional (runtime == "microvm")` on the builder's module list, using the `runtime` value the builder already holds. The same applies to the `fileSystems` entry: `fileSystems` does exist under qemuGuest, so an ungated home entry would not error — it would silently render a bogus mount.

3. **The home volume sets `neededForBoot` itself.** Upstream adds it only for the mount equal to `microvm.writableStoreOverlay` (`mounts.nix:102-117`). Archetypes sets `fileSystems."/home/<username>".neededForBoot = true`. It composes cleanly — upstream never defines the option for a non-overlay volume, so there is no conflict and no `mkForce`.

4. **The home mount point is the user's actual home.** `users.users.<username>.home` resolves to `/home/<username>` from nixpkgs' `isNormalUser` default; nothing in this repo set overrides it. The username comes from the identity the builder already has, not from a literal. Privacy usernames are per-machine, dev usernames are not, so read it per machine rather than assuming one value.

5. **Volumes are labelled, and the label is the deployer's contract.** Without a label upstream mounts by `/dev/vd<letter>` assigned by list position, so adding a second volume silently renumbers devices. Labelling both makes the device path stable and order-independent. The framework declares the label it expects; contract 13 already makes applying it the deployer's job when formatting. `label = "nix-state"` matches what allod/vm's own fstab check greps for; home takes `label = "home"`.

6. **Image paths are derived, unique per machine, and never per-machine data.** `<imageRoot>/<machine>/<volume>.img`, with `imageRoot` a builder argument defaulting to `/var/lib/allod-microvm-volumes` — the root `allod/nexus` already uses in its own fixtures. It is overridable the same way `runtime` is, so a deploy can move it through `profileData` without a second declaration. `allod/nexus` `nix/microvm/host.nix:165` rejects duplicate image paths across all VMs on the host, which the `<machine>` component satisfies; `:393-402` requires absolute, outside the Nix store, and outside `/run/allod`.

7. **The archetypes-side assertion names the missing mount.** Every required mount for the archetype has exactly one `microvm.volumes` entry, or evaluation fails naming the path. This is the diagnostic issue #25 asks for, replacing allod/vm's lower-level store-volume message as the first thing a misconfigured machine reports.

## Agent Gates

None for implementation. Every acceptance test below runs locally without host access or real credentials.

One sequencing note that is not a gate: nothing here makes a microVM bootable. Image creation, formatting, labelling and ownership are deployer actions on the hypervisor, and the store-lifecycle boots that settle contract 6a are a later slice.

## Acceptance Tests

The libvirt no-op, taken across the implementing commit:

```sh
cd /path/to/archetypes
for m in allod-dev privacy-1 nexus installer; do
  nix eval --raw ".#nixosConfigurations.$m.config.system.build.toplevel.drvPath"
done
```

Must be byte-identical before and after. Expected values are the current ones:

| Machine | drvPath hash |
|---|---|
| `allod-dev` | `1lhcxfhy9iddfssp3df3r2zwkwnd3ilm` |
| `privacy-1` | `zzzaralwv032jdmbqc1q6ms9fm15w7yf` |
| `nexus` | `fad4pq24f8iavzr3q369alz8932jq9q3` |
| `installer` | `2yj5038jnsgzbv6ghhmyafs5kspn9jrg` |

Full check set, and the extended selection check:

```sh
nix flake check --print-build-logs
nix build .#checks.x86_64-linux.runtime-module-selection --print-build-logs
```

`runtime-module-selection` must additionally assert, each with a paired sabotage fixture failing for the diagnostic it names:

1. **A dev microvm fixture evaluates with no placeholder.** The fixture from allod/archetypes#23 that supplied its own `/nix/var/nix` volume is deleted; the builder's declarations must carry it. This is the positive proof the whole PR exists for.
2. **Its generated `/etc/fstab` marks both volumes `x-initrd.mount`.** Read the built system's fstab rather than the `fileSystems` attribute — `neededForBoot` renders to that option, and it is what allod/vm's own check greps for. Sabotage: drop `neededForBoot` from the home entry and require the fstab assertion to fail. This is the silent-data-loss guard and the single most important test here.
3. **A privacy microvm fixture evaluates with exactly one volume**, at `/nix/var/nix`, with no home volume.
4. **A dev microvm with the home volume removed fails**, pinned to the archetypes-side diagnostic naming the home path — not to allod/vm's store message, which would not fire for a missing home.
5. **A dev microvm with the state volume removed fails**, pinned to the archetypes-side diagnostic naming `/nix/var/nix`, and specifically *before* allod/vm's contract 6a message, since contract 7 makes archetypes the first reporter.
6. **Both volumes carry distinct labels and distinct image paths**, and both image paths contain the machine name.
7. **A libvirt machine composes no `microvm` option at all** — the existing hypervisor-boundary idiom, extended. Reading `config.microvm` on a libvirt machine must stay an undeclared-option error, proving the gating is outside the module system.

The existing `microvmWithoutVolume` negative fixture inverts once the builder declares volumes: it currently proves allod/vm's contract 6a fires. Rework it to force `microvm.volumes = lib.mkForce []` so it still proves that, rather than deleting the only coverage of allod/vm's own assertion.

Pin diagnostics with the `config.assertions` idiom already in the check — readable without forcing the toplevel, so a mismatch reports as itself rather than as whatever the wrong composition breaks on first.

## Rollback Plan

Revert the PR. The declarations are additive and gated on a runtime no public machine selects, so a revert restores the exact prior derivations for every libvirt machine and returns a microvm machine to failing on allod/vm's contract 6a — the state allod/archetypes#23 deliberately left.

No rollback step touches a disk image. Nothing in this PR creates, formats, relabels or removes one, so there is no partial state to unwind on the host. A machine already provisioned with images keeps them; they simply stop being referenced.
