# microvm.nix Framework Adoption Review Prompt

You are the virtualization engineer people call when a NixOS module evaluates
perfectly, systemd reports green, and the guest still booted without its identity.
You read QEMU `fw_cfg` command lines, initrd ordering, mount namespaces, and
rotation state machines as one continuous execution trace. You know that
`User=microvm` is not tenant isolation, that `Requires=` is not a restart hook,
and that a writable image stops being disposable the moment it holds the only
copy of real work. This plan crosses all of those boundaries at once while
promising that libvirt stays boring. That is exactly your kind of problem. Do
not hold back.

## Your Task

Review the [microvm.nix framework adoption dev plan](../dev-plans/microvm-framework-adoption.md)
for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag
anything that will block implementation, create unnecessary work, or leave a
landmine for future changes.

Read the actual codebase and the pinned upstream microvm.nix source to ground
the review in reality. Do not review the plan in isolation. Trace the generated
QEMU runner, host and guest units, initrd and system credential flow, volume
mounts, Home Manager output, and rotation failure paths. Source-level
assertions are not enough for this change.

## Project Context

**Allod** is a self-sovereign NixOS stack in which one human-owned hypervisor
provisions isolated coding and privacy VMs from public framework code plus
private inventory, identity, and deployment data.

Key repos in play:

- `allod/strategy` - owns this public cross-repo plan and tracking issue
  `allod/strategy#20`. The final `allod/archetypes` integration PR is the sole
  PR that closes the strategy issue.
- `allod/vm` - owns shared guest behavior. It currently exports only
  `nixosModules.qemuGuest`, which imports disko, `disk.nix`, and
  `modules/qemu-guest.nix` as one unit.
- `allod/inventory` - owns machine facts and derives the non-hypervisor
  `lib.vmSpecsJson` projection plus committed `scripts/vm-specs.json`.
- `allod/archetypes` - composes inventory, identity, profile definitions, and
  guest builders; exports `vmFacts`, `nixosConfigurations`, and validation
  checks.
- `allod/profiles` - owns value-free public profile definitions. Its example
  dev Home Manager module currently hard-codes the Forge SSH key under
  `~/.ssh`.
- `allod/nexus` - owns the host NixOS module, provisioning scripts, and
  host-key and Forge-key rotation tools. Issues `allod/nexus#21`, `#22`, and
  `#23` are required parts of this arc.
- `microvm-nix/microvm.nix` - the proposed upstream runtime. The plan's spike
  uses revision `39a499ab85311b56dddb09ec43351cc3658f22c1` with nixpkgs
  `b6018f87da91d19d0ab4cf979885689b469cdd41`, which is also the nixpkgs
  revision currently pinned by `allod/vm`.

Current state (verify against the tree before trusting it; line numbers and
file layout drift):

- `allod/vm/flake.nix` exports one disk-installed guest module:
  `disko.nixosModules.disko`, `disk.nix`, and `modules/qemu-guest.nix`.
  `disk.nix` declares a GPT `/dev/vda` with `/boot` and `/`; the guest module
  imports nixpkgs's QEMU profile, enables systemd-boot, NetworkManager, and
  OpenSSH, and recursively fixes home ownership during activation. A microvm
  path cannot merely add an upstream module to this list.
- Inventory machines currently have `platform`, `type`, sizing, `ip`, `mac`,
  `forge_key`, and repo facts but no runtime fact. `vmSpecsJson` filters out
  hypervisors and projects a selected field set into JSON; the committed JSON
  is checked against it.
- Archetypes' `sharedModules` unconditionally imports
  `vm.nixosModules.qemuGuest` and agenix for both dev and privacy guests. It
  declares `/etc/ssh/<name>` as both the OpenSSH host key and age identity.
  `mkDevVm` then adds agenix consumers and an activation converter:
  `/root/.git-credentials`, `/etc/nix/netrc`, `/root/.netrc`,
  `/home/<user>/.netrc`, the Forge API token under the user's home, Forge SSH
  identity under `~/.ssh`, and registry-driven GitHub credential targets.
  Removing guest-local age therefore requires a complete runtime-specific
  consumer audit, not only deleting `age.identityPaths`.
- `archetypes/nix/vm-facts.nix` currently exports `ip`, `username`,
  `forgeKey`, `hostKeys`, and `hostKeySecretFile` for non-hypervisors. It
  fails loudly on missing required facts but knows nothing about runtime.
  Existing coherence checks compare these outputs with inventory and secrets.
- The Nexus host module currently enables libvirtd and installs host-side
  provisioning tools; it has no microvm.nix input or host module. The shared
  rotation library already contains deploy-flake `vmFacts` readers, but
  `vm-ssh-host-key` and `forge-ssh-key` still resolve target state from the
  working inventory/secrets checkouts and install activated keys directly into
  `/etc/ssh/<vm>` or `~/.ssh/<forge_key>` inside the guest.
- At the pinned microvm.nix revision, `microvm.credentialFiles` is an
  `attrsOf path` and QEMU renders each entry as
  `-fw_cfg name=opt/io.systemd.credentials/<name>,file=<host-path>`.
  Only the QEMU runner implements this transport. The x86_64 default QEMU
  machine is `microvm`; the guest root defaults to tmpfs; `storeOnDisk`
  defaults true when `/nix/store` is not shared.
- The upstream host module creates one `microvm` system user in group `kvm`
  and runs every `microvm@<name>` instance as that same principal from
  `/var/lib/microvms/<name>/current/bin/microvm-run`. The template unit has
  `Restart=always` and depends on TAP, device, virtiofsd, and booted-link
  helpers. It does not provide per-instance filesystem isolation. Its state
  directory is `microvm:kvm` mode `0775`, and `tap-up` hardcodes
  `user = "microvm"`.
- That same import makes four assignments outside `microvm.*`:
  `hardware.ksm.enable = lib.mkDefault true` where the NixOS default is
  `false`; upstream's `microvm` command on every account, whose `-r` execs the
  runner with no unit and whose `-s` execs `ssh` with
  `StrictHostKeyChecking=no` into a root shell over VSOCK; a
  `boot.kernelModules` list merge; and a `security.pam.loginLimits` `memlock`
  grant that never applies because the generated units set no `PAMName`.
- At kernel 6.12.93, `/proc/<pid>/root` opens under `PTRACE_MODE_READ_FSCREDS`
  and is gated only by the same-credential comparison in
  `__ptrace_may_access`. Yama's hook inspects `PTRACE_MODE_ATTACH` only, and
  every `has_pid_permissions` branch reduces to that same read check, so no
  `ptrace_scope`, `hidepid`, or `subset=pid` value closes a same-uid read into
  another process's mount namespace. Measured downstream and re-derived from
  the pinned kernel source; do not relitigate it.
- Upstream `microvm.volumes` defaults `autoCreate = true`; creation touches,
  truncates, and formats the declared host image if it does not exist.
  Volumes are attached after the read-only store disk and are mounted by label
  or virtio block letter. `microvm.shares` defaults empty, but the upstream
  virtiofsd template still exists and is skipped by `ConditionPathExists` when
  its runner script is absent.
- The public/private boundary is load-bearing. Public repos may carry
  synthetic, buildable examples and value-free option shapes, but real
  credential sources, image paths and sizes, addresses, routes, TAP
  attachment, and cutover actions belong to private deployment composition.
  Agents run inside dev VMs; real host rebuild, key handling, and cutover are
  human-only.

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections
from `dev-plans.md`: Tracking Issue, Goal, Scope, Risk Assessment, Interface
Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Agent Gates apply
here and may not be omitted.

This is a multi-repo, multi-PR plan. Verify that each PR or milestone has a
defensible residual risk, prerequisite PRs use `Refs allod/strategy#20`, Nexus
issue PRs close their own issues, and only the final Archetypes integration PR
uses `Closes allod/strategy#20`. Confirm that every cross-repo contract is
available before the next consumer PR needs it and that rollback unwinds pins
in the reverse dependency order.

## Focus Areas

The plan text converged at pass 10. No further plan-text pass is scheduled:
the remaining verification targets are runtime questions that passes 7
through 10 could each advance only by static analysis of the pinned sources
and store-level measurement, and each such advance seeded the next finding in
the same subsystem. What remains is handed to implementation review, ordered
below; implementation can boot the nested fixtures, and that is where the
remaining defects will surface.

Handoff to implementation review, most-unproven first. Every item names
contracts that were reasoned about statically across passes 5-10 but never
executed:

1. **Store persistence (contract 6a, acceptance test 9).** The least-proven
   subsystem; passes 7, 8, 9, and 10 each found exactly one defect here.
   Never executed: the fresh-volume first boot and cache creation; the GC
   whiteout plus same-toplevel restart on the cached copy; the toplevel move
   with and without a surviving upper referrer into the old lower, including
   reconciliation convergence and the measured fate of lower-referencing
   survivors; the A-to-B-to-A rollback from the cached registration; the
   missing/corrupt-cache sabotage; the explicit pre-systemd termination in
   both initrd flavors, given that stage 2 ignores a failing
   `boot.postBootCommands` child in both; and the per-boot cost of whole-DB
   existence checks on a grown store.
2. **Nested lifecycle paths (acceptance test 10).** The final `ExecStopPost`
   ordering after rendered upstream post-stop hooks, the persistent-volume
   shutdown sentinel as graceful-stop evidence across the ctrl-alt-del plus
   `-no-reboot` path, and a rebuild stop driven by a real toplevel move
   through the pinned restart trigger.
3. **Store-volume initrd failure paths (contract 13, test 6).** Missing,
   unformatted, and mislabeled images failing in the initrd before sshd, a
   user session, or any writer against the tmpfs mount point.
4. **Two-runner namespace isolation (contracts 16/16a, nexus#23).** The
   direct-path and `/proc/<sibling-pid>/root` attack fixtures with both
   sabotage reopenings; the kernel gate was measured once, but the two-guest
   NixOS test has never run.
5. **Rotation failure boundaries (contracts 18/19).** The armed rollback
   selector across automatic restarts, loss of the mounted rollback slot, and
   both runtime verification paths failing closed.
6. **Host plaintext mount dependency (contract 8a).** The resolved
   `Requires=` on the real mount unit fragment, a failing mount leaving the
   VM unstarted, and `.mount` restart behavior mid-rotation.

Do not schedule another plan-text pass for these. A future plan-text pass is
warranted only if implementation changes the plan's contracts themselves.

## Pass Metadata

Pass 1 found three original-plan BLOCKERs, one original-plan GAP, one
original-plan SIMPLIFY, and one GAP introduced by `416bd94`. Commit `416bd94`
fixed the pin handoff, disabled automatic image creation, and removed the
generic volume manager, but its launch and rollback designs required
blocker-level repair in pass 2; it remains unstable. Commit `12c49d7` resolved
the raw/wrapped name collision but added an unused public raw guest export that
pass 2 removed, so it also is not fix-stable.

Pass 2 found two BLOCKERs and one SIMPLIFY introduced by the prior fixes, plus
one original-plan GAP missed by pass 1. Commit `10f3425` makes the root helper
the sole privilege transition, adds a recovery selector that overrides the
still-staged source, makes required volume mounts fail in the initrd, and
removes the raw guest export; its stability is pending the scoped next pass.
The SIMPLIFY sweep reconsidered the raw guest export, root launcher, rollback
slot, volume creation/capacity, and a generic mount validator. It removed only
the unused raw export because the other mechanisms serve concrete lifecycle or
security requirements. Convergence has not been reached: this is the first
pass where review-introduced findings outnumbered original-plan findings, and
the pass contains both BLOCKERs and an original-plan GAP.

Pass 3 found one BLOCKER introduced by `10f3425`: the new launch namespace did
not make QEMU's `microvm.socket` visible to the upstream host-namespace
`ExecStop`, so graceful stop and restart could fall through to a systemd kill.
Commit `e99f5e6` requires a root-owned, `microvm:kvm`-writable, per-VM QMP
working directory bound at the same absolute path inside the launcher and adds
generated plus nested coverage for stop, retry, rebuild, and rollback. The
SIMPLIFY sweep reconsidered a separate shutdown helper, a generic host-state
manager, a second service unit, and a durable QMP path; all add scope or weaken
the one-start lifecycle, so nothing was removed. Commit `10f3425` is unstable;
`e99f5e6` is pending an independent scoped diff review. The review-introduced
finding majority now spans two passes, which meets the plan-text convergence
heuristic, but convergence is not declared until the mandatory independent
review of the new structural fix verifies that the terminal lifecycle repair
does not introduce another blocker. Afterwards, hand remaining work to
implementation review unless that scoped diff exposes a new plan defect.

Pass 4 found one BLOCKER and two GAPs, all introduced by `e99f5e6`.
First, the execing root launcher could not perform cleanup after QEMU exit and
upstream post-stop hooks; `23e9704` assigns idempotent cleanup to a final root
`ExecStopPost` merged with `lib.mkAfter` and covers preparation failure, QEMU
failure, every stop/restart path, and rollback-slot preservation
([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).
Second, cleanup plus restart could falsely pass as QMP shutdown because the
pinned shutdown script accepts an absent socket before systemd kills QEMU;
`23e9704` requires a durable guest shutdown sentinel and a broken-socket
sabotage fixture. Third, the same-uid fixture omitted the new writable QMP
socket/directory, current runner entries, and their host source parents;
`23e9704` adds those objects and operations to every namespace access path
([allod/nexus#23](https://forge.anarch.diy/allod/nexus/issues/23)).
The SIMPLIFY sweep considered a cleanup service, a long-lived privileged
supervisor, a generic runtime-directory manager, and a second shutdown helper;
all were rejected in favor of one ordered post-stop command on the existing
unit. Commit `e99f5e6` is unstable. Commit `23e9704` is pending independent
scoped verification. The review-introduced finding majority continues for a
third pass, but convergence is not declared while this replacement
blocker-level lifecycle fix lacks its mandatory independent stability result.
If the scoped next pass finds no new blocker or original-plan GAP, stop
plan-text review and hand the remaining checks to implementation review.

Pass 5 was the scoped diff review of `23e9704`, widened by the user to also
dispose of three downstream integration reviews left unaddressed on
`allod/strategy` PR #22. `claude-fable-5` ran it: pass 4 recommended
`claude-opus-4-6`, which the Claude runner cannot instantiate, so the
substitute was chosen as the highest-capability Claude option, the roster's
role for terminal verification of cross-repo generated lifecycle behavior, and
not the author of any fix in this plan. Record substitutions like this one
rather than silently renaming the pass.

Two determinations, kept separate. First, `23e9704` is fix-stable: its ordered
final `ExecStopPost`, its `lib.mkAfter` merge, its stop-before-next-start
guarantee, and its sentinel-based graceful-stop evidence all survived scoped
verification against the pinned unit, the built runner argv, and the built
`microvm-shutdown`. Only its own new "refuses a stale socket" wording was
wrong, and that is a GAP, not a blocker. `gpt-5.6-sol` is therefore the first
model on this plan with a stable structural fix. Second, the plan text has not
converged: pass 5 found three BLOCKERs and four GAPs, five of them original
defects that four passes missed, so both stop conditions fail. The
review-introduced majority streak is broken, and this pass contains both new
BLOCKERs and original-plan GAPs.

Those five original defects share a cause worth naming: every earlier pass
reviewed the plan's own mechanisms and never audited what the pinned upstream
host module does to the host, nor whether a control the plan names actually
holds at the kernel. A sweep that asks only "did this change a pre-existing
default?" sees the overrides and misses the additions. The plan's isolation
contract asserted a boundary that measurement showed open, and its
"ramfs-only" title had no mount behind it.

Pass 5 replaced the shared runner uid with one principal per VM, put both host
plaintext roots on declared non-pageable storage behind readable options,
bounded the host-wide settings the upstream import brings, and made the
leftover-QMP-directory behavior explicit. That is a structural change, so pass
6 is its scoped diff review by a different model.

Pass 6 was the scoped diff review of `e315741`, `86dd6e5`, `d40adf9`,
`a7e2fca`, and `e95d704`, run by `gpt-5.6-sol` at `xhigh` effort. It found two
BLOCKERs, one GAP, one SIMPLIFY, and no QUESTIONs, all introduced by the
pass-5 root change.

1. [BLOCKER] Standalone rotation could bypass the plaintext mount.
   `86dd6e5` ordered systemd units but not `vm-ssh-host-key` or
   `forge-ssh-key`, so either command could create the configured path on
   stock swappable `/run` when the mount was absent; `2657827` requires the
   unit dependency plus an exact-mount preflight in every writer before any
   plaintext read or write. Origin: introduced by `86dd6e5`
   ([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).
2. [BLOCKER] The permitted ramfs could exhaust host memory. The pinned kernel
   cannot bound or resize ramfs, so a bad source or writer bug could OOM the
   public 16 GiB host while satisfying contract 8a; `2657827` permits only one
   fixed 16 MiB `tmpfs,noswap` host root. Origin: introduced by `86dd6e5`
   ([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).
3. [GAP] The host root did not reach the already evaluated guest.
   `86dd6e5`, with an aspirational assertion carried by `e95d704`, never
   defined how a Nexus host option could change `credentialFiles` inside an
   exported Archetypes `nixosConfiguration`; `34ef13d` assigns option
   ownership, extends each selected guest once at host composition, compares
   exact derived values, and builds default plus non-default fixtures. Origin:
   introduced by `86dd6e5` and carried forward by `e95d704`
   ([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).
4. [SIMPLIFY] The rollback root did not need its own option or mount.
   `86dd6e5` added a second host storage surface even though a root-owned
   `rollback/<name>` child beside `active/<name>` remains outside cleanup and
   every runner namespace; `2657827` removes the option, mount, ordering edge,
   and duplicated value. Origin: introduced by `86dd6e5`
   ([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).

The SIMPLIFY sweep also reconsidered the guest credential-root option, per-VM
principal, mount namespace, QMP directory, and active-system root descriptor.
Only the rollback-root option and mount were deleted. The guest option is the
single source for materializer and Home Manager destinations; the principal,
namespace, and QMP directory close independent tested boundaries; and the
minimal generated descriptor is how commands outside systemd read the active
host option instead of an unactivated checkout.

Fix stability for all five pass-5 commits: `e315741` is stable for the
runner-principal change; the pinned upstream's shared `microvm` account can
remain only for trusted install and `booted`-link work while `0755` state
directories exclude every per-VM runner's supplementary `kvm` group, and TAP
ownership is replaceable per instance. `86dd6e5` is unstable and required both
blocker-level storage repair and simplification. `d40adf9` is stable.
`a7e2fca` is stable. `e95d704` is stable for its principal and sentinel
carry-through but is not fix-stable overall because its root-propagation
assertion needed `34ef13d`. Commits `2657827` and `34ef13d` are pending
independent scoped verification.

Convergence: not converged. Review-introduced findings outnumber
original-plan findings four to zero, but this is only the first consecutive
pass after pass 5 broke that streak, and the two BLOCKERs also prevent the
second stop condition. The next scoped target is `2657827` and `34ef13d`, with
`claude-opus-5` recommended as a callable model that authored neither.

Pass 7 was the scoped diff review of `2657827` and `34ef13d`, run by
`claude-opus-5` at max effort. It found one BLOCKER and two GAPs, grounded
against pinned nixpkgs `b6018f87`, pinned microvm.nix `39a499ab`, systemd
258.7, kernel 6.12.93, and a synthetic host/guest evaluation built from those
store paths.

1. [BLOCKER] The selected dev guest had no writable Nix store. At the pinned
   revision the upstream guest module mounts `/nix/store` from the read-only
   erofs disk and disables `nix-daemon` and its socket whenever
   `microvm.writableStoreOverlay` is null, so a dev VM would boot unable to
   realise, fetch, or collect a store path while carrying contract 9's Nix
   netrc wiring; `3dc1a73` adds contract 6a, the overlay on a declared
   persistent mount point, a restart-survival test, and a paired mutation.
   Origin: original-plan defect, present since `236f1cb`
   ([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).
2. [GAP] `RequiresMountsFor` was not a requirement. Systemd derives
   `Requires=` only for a mount unit with a fragment file, so against a
   `boot.specialFileSystems` mount it degrades to ordering and, with nothing
   mounted, silently to `-.mount`; the check bullet `2657827` added could not
   be shown to fail. `94ca185` declares the root as a real mount unit and
   asserts the resolved `Requires=` plus a failing-mount fixture. Origin:
   introduced by `2657827`
   ([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).
3. [GAP] `evaluatedConfig` silently disabled restart-on-rebuild. Upstream
   defaults `restartIfChanged` to `config.config != null`, so the rendered
   drop-in carried `X-RestartIfChanged=false` and `switch-to-configuration`
   skipped the unit, making the nested test's rebuild-stop path vacuous;
   `f3c93bc` pins the value and drives the test with a toplevel-moving change.
   Origin: introduced by `34ef13d`
   ([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).

The SIMPLIFY sweep cut `boot.specialFileSystems` as the mount mechanism. It
considered and kept the `/etc/allod/microvm-runtime.json` descriptor and its
env override, because a rotation tool invoked from a deploy-flake checkout
would bake that checkout's value; the `RequiresMountsFor` edge, which was made
real rather than deleted because it is the only control that refuses the start
before the launcher runs; the standalone exported guest, which is the base
`extendModules` needs and the subject of six contract checks; the separate
`active/` and `rollback/` children, because `ExecStopPost` removes
`active/<name>` wholesale; and the 28-character credential-name limit, which is
the real fw_cfg constraint rather than ceremony.

Fix stability for the pass-6 commits: `2657827` is stable for the storage
decision itself — the ramfs ban, the fixed 16 MiB `tmpfs,noswap,mode=0700`
root, the exact-mount rather than `findmnt --target` preflight, and the
consolidated `active`/`rollback` children all survived measurement — but it is
not fix-stable overall, because its `RequiresMountsFor` claim and the check
built on it needed `94ca185`. `34ef13d` is stable for its central mechanism:
`extendModules` derives the exact `credentialFiles` map with no recursion, the
extended result is what `install-microvm-<name>` installs, and a non-default
host root moves every derived value and the runner path. It is not fix-stable
overall, because naming `evaluatedConfig` silently flipped `restartIfChanged`
and needed `f3c93bc`. Commits `3dc1a73`, `94ca185`, and `f3c93bc` await
independent scoped verification.

Convergence: not converged. New findings are 2 review-introduced against 1
original-plan, so the arithmetic of the first stop condition holds for a second
consecutive pass. It is not being used to declare convergence. The heuristic
assumes the plan's own defects are exhausted and the reviews are finding their
own churn; an original-plan BLOCKER that six passes missed falsifies that
assumption, and it is the same species as pass 5's — nobody had audited what
the pinned upstream guest module does to the guest, exactly as nobody had
audited the host module. `3dc1a73` is also a blocker-level structural change to
the store and persistence model, and this prompt requires an independent scoped
review of such a fix before it is trusted, which is the reasoning passes 3, 4,
and 6 used when they also met the arithmetic. The condition is primed: a scoped
pass over the three pass-7 commits that finds no original-plan BLOCKER closes
the arc.

Pass 8 was the scoped diff review of `3dc1a73`, `94ca185`, and `f3c93bc`, run
by `gpt-5.5` at `xhigh` effort. Findings new to pass 8: 1 BLOCKER, 0 GAP, 0
SIMPLIFY, 0 QUESTION. 1. [BLOCKER] Overlay-only store persistence forgot Nix's
database. `3dc1a73` made the selected dev guest's writable store overlay
persistent, but upstream keeps the Nix validity database, profiles, and gcroots
under `/nix/var/nix`; paired nested boots showed an added store path physically
survived when only the overlay upper/work persisted while `nix-store --verify-path`
failed after reboot. Fixing commit: `35e49dc`. Issue:
https://forge.anarch.diy/allod/strategy/issues/20. Origin: introduced by
pass-7 commit `3dc1a73`.

Focus area 3 completed the pinned guest-module audit. The wrapper imports the
upstream guest module and drags these non-`microvm.*` settings into a selected
guest: `assertions` and `warnings`; `boot.loader.grub.enable = false`;
`boot.initrd.kernelModules`, optional `boot.initrd.availableKernelModules`, and
the store-overlay `overlay` module; `boot.blacklistedKernelModules`;
`systemd.services.nix-daemon.enable`, `systemd.sockets.nix-daemon.enable`,
`systemd.services.mount-pstore.enable = false`,
`systemd.generators.systemd-gpt-auto-generator = "/dev/null"`, and the
non-storeOnDisk unmount-helper `systemd.mounts`; optional
`environment.etc."machine-id"` and `networking.hostId`; the root, store,
overlay, volume, and share `fileSystems`; closure-registration
`boot.postBootCommands`; optimization defaults for `documentation.enable`,
`boot.initrd.systemd.enable`, `boot.initrd.systemd.tpm2.enable`,
`boot.kernelParams = [ "8250.nr_uarts=1" ]`, `boot.swraid.enable`,
`nixpkgs.overlays`, `networking.useNetworkd`,
`systemd.network.wait-online.enable`, `systemd.tpm2.enable`, and
`system.switch.enable`; optional graphics `boot.kernelModules` and
`environment.systemPackages`; optional Rosetta; and optional VSOCK SSH
`services.openssh.enable`. The selected dev archetype can live with these
because it deliberately uses the tmpfs root, qemu, headless graphics, zero
shares, no VSOCK SSH, no PCI devices, explicit per-VM TAP ownership instead of
upstream's helper user, and now a persistent `/nix/var/nix` Nix state volume;
future graphics, PCI, VSOCK, share, or non-qemu use needs explicit review.

Fix stability: `3dc1a73` did not fully hold. Its overlay rendering, daemon
enablement, and auto-optimise guard held, but its persistence boundary needed
`35e49dc` because overlay-only state loses the Nix DB. `94ca185` held: the real
mount unit makes `RequiresMountsFor` a real dependency, launcher and rotation
preflight still gate every writer, and host mount loss is already modeled as
lost active and rollback bytes under contract 19. `f3c93bc` held: explicit
`restartIfChanged = true` renders `X-RestartIfChanged=true` and toplevel
restart triggers for `evaluatedConfig`, while the upstream default remains
false for evaluated guests. The SIMPLIFY sweep considered deleting
`restartIfChanged`, deleting the real host mount unit, deleting the standalone
guest export before `extendModules`, deleting `microvm.registerClosure`, and
keeping an overlay-only state volume separate from `/nix/var/nix`; only the
overlay-only state surface was cut, as part of the blocker repair.

Convergence: not converged. Pass 8 found no original-plan defect, and the guest
audit came back clean after the Nix-state repair, but the pass did produce a
BLOCKER introduced by the previous review pass. The next review should be the
scoped diff review of `35e49dc` and a final convergence check rather than
another full pass. Next pass: scoped diff review of `35e49dc`, not a full pass;
recommend `gpt-5.6-sol` at `xhigh`.

Pass 9 was the scoped diff review of `35e49dc`, run by `gpt-5.6-sol` at
`xhigh` effort. Findings new to pass 9: 1 BLOCKER, 0 GAP, 0 SIMPLIFY, and
0 QUESTION.

1. [BLOCKER] The persistent Nix DB diverged from each new erofs lower, and GC
   erased rollback's registration payload. A third same-shape boot and one
   toplevel move preserved a path added to the overlay, so `35e49dc`'s central
   database-persistence repair worked on its happy path. The wider A-to-B-to-A
   fixture exposed the missing lifecycle: after B booted, A's toplevel was
   physically absent while `nix path-info` still reported it valid from the
   persistent DB and `nix-store --verify-path` rejected it; meanwhile GC had
   whiteouted each lower store's registration file because that payload is not
   in its own registration. `nix-store --verify` silently removed the stale A
   entries, and rollback then booted A with both its current toplevel invalid
   and its registration source hidden. `2c847c4` disables the incompatible
   upstream registration path, caches each booted toplevel's registration on
   `/nix/var/nix`, reconciles the DB to the current merged store before replay,
   and adds same-toplevel GC plus A-to-B-to-A rollback and cache-sabotage
   coverage. Origin: introduced by `35e49dc`
   ([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).

The SIMPLIFY sweep reconsidered the writable overlay, guest garbage
collection, a separate registration volume, a generic registration manager, a
separate replay service, and unbounded historical metadata. It cut the
upstream `microvm.registerClosure` requirement as part of the blocker repair:
that is the incompatible mechanism. The writable overlay and GC are required
for a useful bounded dev store; the existing `/nix/var/nix` volume already
owns the database, gcroots, overlay, and small keyed registration copies; and
one pre-systemd replay is narrower than another service or manager.

Fix stability: `35e49dc` is not stable overall. Its `/nix/var/nix` volume,
overlay layout, daemon enablement, repeated-boot validity for upper-store
paths, initrd failure boundary, relay coupling, and credential exclusion all
held, but its explicit `microvm.registerClosure = true` requirement and its
assumption that a persistent DB stays coherent across changing lower stores
needed `2c847c4`. The pass-6 fixes `2657827` and `34ef13d` remained stable and
were not reopened.

Convergence: not converged. Pass 9 found no original-plan defect, but
`35e49dc` failed its required independent scoped review with a
review-introduced BLOCKER, and `2c847c4` is itself a structural replacement
that requires independent verification. The original-plan defect pool was not
shown to have reopened; the next scoped target is `2c847c4`, specifically its
persistent registration replay, DB reconciliation, explicit pre-systemd
termination, GC behavior, and A-to-B-to-A rollback.

Pass 10 was the scoped diff review of `2c847c4` plus the terminal convergence
check, run by `claude-fable-5` at `max` effort as pass 9 recommended. The
pass was interrupted once by a provider session limit and resumed rather than
restarted; the interruption lost no commits. Findings new to pass 10: 1
BLOCKER, 1 GAP, 0 SIMPLIFY, 0 QUESTION, both introduced by `2c847c4`. No
original-plan defect surfaced for a third consecutive pass.

1. [BLOCKER] Registration reconciliation could not converge. Contract 6a
   claimed one `nix-store --verify` run removes database entries whose paths
   are absent from the current merged store. Measured on the pinned
   `nixVersions.stable` line (Nix 2.31), `--verify` keeps any absent entry
   that still has a surviving valid referrer, reports it, and exits 1 on
   every run without converging, and after an ordinary toplevel move that
   referenced-absent state is the normal one because upper-overlay builds
   reference old lower-only paths. The fail-closed reading would terminate
   every boot after the first host rebuild that follows real dev work —
   `Restart=always`, `panic=-1`, and `-no-reboot` make that a tight restart
   loop only host intervention clears — while a fail-open reading silently
   preserves the stale validity pass 9 set out to remove. `8a21d50`
   redefines reconciliation as invalidating absent paths together with the
   valid entries whose reference closures reach them, makes a clean repeated
   verify the post-condition, adds the referenced-absent fixture to test 9,
   and scopes rooted-path survival to reference-complete paths. Origin:
   introduced by `2c847c4`
   ([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).
2. [GAP] The registration kernel argument's string context was unspecified.
   The closure-info payload references its entire subject graph (508
   references on the pass-9 spike guest), so interpolating its path into the
   runner command line with context would copy the whole guest closure into
   every host runner closure; upstream discards context for `init=` under
   `storeOnDisk` for the same reason. `c784e32` requires the discard and
   records why the erofs payload makes it safe. Origin: introduced by
   `2c847c4`
   ([allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20)).

Grounding for pass 10: the pinned microvm.nix store-disk, system, mounts,
options, and qemu-runner sources; pinned nixpkgs stage-2-init, the
systemd-initrd prepare-root handoff, overlayfs `depends` wiring, and the
`booted-system`/`current-system` gcroot rules; a synthetic evaluation of the
selected dev guest shape with `registerClosure = false` proving the exposed
`storeDisk.passthru.regInfo`, the absent upstream `regInfo=` parameter and
load-db hook, a framework parameter reaching `-append`, daemon enablement,
and the exact overlay layout; the pass-9 built store disk proving payload
packing is gated only on `nix.enable`; and a scratch-store measurement of
`--verify` against referenced and unreferenced absent paths. The structural
mechanism of `2c847c4` held everywhere static checking could reach:
bijective per-toplevel cache keying, first-boot copy-before-GC ordering, GC
whiteouts confined to the unregistered closure-info path because the booted
closure is gcroot-protected, kernel-command-line transport, contract
13/14/16 alignment, and the necessity of explicit pre-systemd termination.
No nested boot exercised the replay, because the replay exists only as plan
text; the defect found is a store-semantics fact a boot would reproduce, not
resolve.

Fix stability: `2c847c4` did not fully hold. Its cache, replay placement,
termination requirement, and transport survived scrutiny; its reconciliation
mechanism claim was disproven by measurement and needed `8a21d50`, and its
command-line sentence left string context open and needed `c784e32`. The
pass-6 and pass-7 fixes (`2657827`, `34ef13d`, `94ca185`, `f3c93bc`) and the
surviving parts of `35e49dc` were not reopened.

The pass-10 SIMPLIFY sweep considered deleting the per-toplevel registration
cache in favor of a current-only copy (breaks A-to-B-to-A rollback), adding
a cache-pruning mechanism (new scope for a few hundred kilobytes per
toplevel; left to implementation if growth measures as material), deleting
the reconciliation step to rely on load-db plus toplevel verification
(restores pass 9's stale-validity blocker), dropping store persistence
entirely to delete the subsystem (re-litigates the settled pass-7/8 design
and trades a bounded mechanism for a per-restart rebuild tax on the
archetype's core workflow), and trimming test 9's scenario list (each
scenario maps to a distinct measured failure). Nothing was cut.

Convergence: converged, with contract 6a explicitly flagged as an empirical
specification. Review-introduced findings outnumber original-plan findings
in passes 7 (2-1), 8 (1-0), 9 (1-0), and 10 (2-0), satisfying the first stop
condition for four consecutive passes, and no original-plan defect has
surfaced since pass 7. Passes 8 and 9 declined to declare convergence
because each had just authored a structural fix requiring independent
review, a loop that is self-perpetuating by construction; pass 10 ends it
deliberately after the fourth consecutive store-persistence defect showed
the subsystem cannot be settled by plan-text review. `8a21d50` corrects a
disproven factual claim to measured semantics rather than restructuring the
replay, `d20de3b` converts the subsystem into named empirical acceptance
criteria that implementation must prove in the real nested boot before the
archetypes milestone lands, and no fifth plan-text verification pass is
scheduled. Remaining areas are handed to implementation review in the Focus
Areas order, store persistence first.

Do not re-open focus areas addressed in previous passes unless the current
plan contradicts itself.

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves.
  Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. Public option
  and module interfaces may break, but the plan explicitly promises that
  selected libvirt guests keep working throughout this migration.
- **Do not overengineer.** If the plan introduces abstraction that is not
  needed for the concrete public examples and one private deployment, call it
  out. Three explicit runtime branches beat a premature framework. Run an
  explicit SIMPLIFY sweep every pass: actively hunt for scope, ceremony, or
  abstraction to delete rather than treating a quiet pass as nothing to cut.
- **Solo project, one human.** No team coordination overhead. No release
  process. No migration guides for other consumers.
- **Security matters, ceremony does not.** Host/guest credential isolation,
  no durable plaintext, stable SSH identity, fail-closed startup, and
  same-principal sibling isolation are real boundaries. Extra schemas and
  generalized managers are not.
- **Solve problems as they come.** If the plan front-loads snapshotting,
  scheduling, backup orchestration, multi-host behavior, or generic migration
  state, remove it.
- **Think operationally.** Walk cold boot, rebuild, restart, crash, rotation,
  missing credential, failed mount, and rollback in the order systemd and the
  scripts actually execute them.
- **Calibrate residual risk.** The public implementation remains R3 if
  synthetic nested and generated-artifact validation contains its worst
  failures. Challenge any milestone whose validation or rollback leaves
  unique volumes, credential authority, or cross-repo pins uncertain.
- **Inspect generated lifecycle artifacts.** Do not stop at Nix evaluation.
  Read the actual QEMU argv, unit dependency graph, service credentials,
  activation scripts, Home Manager files, mounts, closures, and failure logs.

The person implementing this is technically sharp. They do not need
hand-holding; they need the sharp edges they missed.

## Severity Rubric

Use `[BLOCKER]` only when following the plan literally is likely to:

- perform a destructive or unsafe operation;
- fail before implementation can complete;
- leave the resulting system nonfunctional;
- break first boot, activation, provisioning, rebuild, rotation, or rollback
  lifecycle behavior;
- violate a security or privacy boundary; or
- require missing human input that cannot be inferred from the repo or memory.

Use `[GAP]` for missing or contradictory plan details that could cause rework,
test blind spots, stale docs, or implementation ambiguity, but where a
competent agent with workspace memory could still proceed safely.

Use `[GAP]` when the Risk Assessment is missing, materially understated,
materially overstated, or unsupported by the acceptance tests and rollback
plan. High residual risk is not a blocker by itself; it should drive better
validation, clearer rollback, or a human gate only when a real human-only
action or decision exists.

Use `[SIMPLIFY]` for unnecessary scope, ceremony, or abstraction. Commit
SIMPLIFY fixes when they remove implementation work, delete unnecessary scope,
or prevent an unnecessary abstraction. Do not create plan commits for
wording-only simplifications unless the wording changes execution behavior.

Use `[QUESTION]` only when the plan cannot be corrected from repo context. If
the answer is inferable, resolve it as `[GAP]` or `[SIMPLIFY]`.

Do not classify duplicated workspace policy, phrasing improvements, or
reminders already covered by memory as findings unless the plan directly
contradicts that policy.

## Deliverable

The deliverable is not a report. Every review pass ends with:

1. Plan-file commits for findings that require plan changes, or an explicit
   no-findings result.
2. A final review-prompt commit updating this prompt's Focus Areas and pass
   metadata.
3. A push to the remote.

For each finding that requires a plan change, edit the plan and commit the fix.
Group changes into logical, self-contained commits.

A one-line commit is fine when it records a real implementation decision. Fold
or skip commits that only rephrase already-correct guidance.

## Output Format

Give your review as a numbered list of findings, each tagged as one of:
`[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, `[QUESTION]`. Start with blockers, end with
questions. Be blunt.

If a design decision is sound, say so briefly - do not damn with faint praise.
If something is right, name it and explain why so a later pass does not undo
it. QUESTIONs must be resolved in the plan, not left as open items. If the
answer is clear from the codebase, update the plan and commit. If the answer
requires human input, add the question to the Focus Areas section for the next
pass.

## After Each Pass

As a final commit, update this prompt's Focus Areas section:

1. Remove focus areas that were fully addressed.
2. Refine any focus areas that were partially addressed.
3. Add new focus areas discovered during the review.

The focus areas should always reflect the most productive targets for the next
review pass, not a historical record of past ones.

Include a plain-text findings summary in this commit's message:

- Count only findings new to this pass, by tag. Carried-over unresolved items
  are not findings; move them to Focus Areas rather than re-counting them, so
  the per-pass counts track a real severity trend instead of inflating it.
- Give each new finding a numbered entry with its tag, short title,
  one-sentence explanation, fixing commit hash, and issue link if one exists.
- Classify each finding's origin: an original-plan defect, or introduced by an
  earlier review pass (name the commit that introduced it). Origin is what
  makes the convergence heuristic below and any trend read possible.
- State what the SIMPLIFY sweep considered for deletion this pass, even when
  nothing was cut. Two or more consecutive passes with zero SIMPLIFY on a
  growing plan is a smell to call out, not a clean bill: pure accretion is what
  breeds internal contradictions.
- Put a final `Model: <exact model>` footer at the bottom (for example,
  `Model: claude-fable-5` or `Model: gpt-5.5`). Use the exact identifier of the
  model that actually ran, not the agent framework, the product name, or the
  model a previous pass recommended. This is review-pass metadata, not
  authorship attribution.

When a pass commits a structural or design change (a blocker-level fix), the
next pass should be a scoped diff review of that change, not a full re-review,
run by a different model than the one that wrote the fix. Structural fixes are
where new blockers enter, and the author model tends not to catch its own
gaps. Passes are launched manually one at a time, so you cannot pick or start
the next model yourself; make the handoff explicit instead. In the Focus Areas
update, add a `Next pass:` line that names the commit(s) to review, says
whether it is a scoped diff or a full pass, and recommends a model other than
the fix's author (preferably the most fix-stable one on record; see Agent
Rotation in `dev-plans.md`). Whoever starts the next pass reads that line and
the previous `Model:` footer and picks accordingly. Record in the same update
how the fix held up so its stability stays traceable.

Stop the plan-text review when either condition holds:

- Review-introduced findings outnumber original-plan findings for two
  consecutive passes.
- Two consecutive passes produce no BLOCKER and no original-plan GAP.

At that point the plan text has converged; hand any remaining focus areas to
implementation review, and resolve remaining SIMPLIFYs during implementation.
The old zero-BLOCKER/zero-GAP/zero-QUESTION rule could grind a token budget to
nothing without terminating, because each accretive fix tended to seed the
next finding.

## Before Final Response

- Plan fixes are committed, or the pass explicitly found no plan changes.
- This review prompt's Focus Areas are updated and committed.
- The final review-prompt commit message includes the findings summary and
  `Model:` footer.
- The repo is pushed to the remote.
- `git status` is clean.
