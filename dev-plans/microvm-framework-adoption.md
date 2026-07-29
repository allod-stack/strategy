# microvm.nix Framework Adoption: Public Half

## Tracking Issue

[allod/strategy#20: Adopt microvm.nix as a VM runtime in the framework](https://forge.anarch.diy/allod/strategy/issues/20)

That issue is the canonical goal statement for the public framework work and the private deployment integration. This plan is the public leaf: it contains no private machine, key, address, or host path, and a public-only agent can implement and validate it. The private plan links to the same issue and owns real values, full deployment builds, and cutover.

This is a multi-repo arc. Every prerequisite PR carries `Refs allod/strategy#20`. The final `allod/archetypes` integration PR, after all prerequisite revisions are pinned and its full check set passes, carries `Closes allod/strategy#20`.

Three `allod/nexus` issues are required parts of this arc rather than deployment workarounds:

- [allod/nexus#21](https://forge.anarch.diy/allod/nexus/issues/21): runtime-aware VM SSH host-key rotation.
- [allod/nexus#22](https://forge.anarch.diy/allod/nexus/issues/22): runtime-aware Forge SSH key rotation.
- [allod/nexus#23](https://forge.anarch.diy/allod/nexus/issues/23): per-VM host-path and runner-principal isolation for microvm guests.

The PR implementing each nexus issue closes that issue and references allod/strategy#20. Only the final archetypes integration PR closes allod/strategy#20.

## Goal

The framework can select microvm.nix for a machine and boot a networked guest alongside the unchanged libvirt runtime, with stable secrets delivered from bounded non-pageable host memory at every start, declared state surviving restart, no in-guest age identity, no shared folders, and generated plus nested-boot tests proving the lifecycle and its failure paths.

## Scope

In scope:

- `allod/vm`: pin microvm.nix to the framework's nixpkgs line; re-export its `nixosModules.host` as `nixosModules.microvmHost`; separate common guest behavior from the disk-installed `nixosModules.qemuGuest`; add an exclusive `nixosModules.microvmGuest` wrapper that directly imports the pinned upstream guest module without disko, a partition-backed root, or a bootloader; and add module-composition checks.
- `allod/inventory`: add the required runtime fact for non-hypervisor machines, export it in `lib.vmSpecsJson` and `scripts/vm-specs.json`, validate its enum, and keep public examples for both libvirt and microvm.
- `allod/archetypes`: select exactly one guest module from the runtime fact; export the same fact through `vmFacts`; declare the microvm runner, credential names, persistent mount points, guest interface, and guest-only plaintext root; move microvm credential consumption away from agenix and durable paths; extend each selected evaluated guest with host credential paths derived from the Nexus host option before wiring it into the host configuration; and add generated-artifact, mutation, and nested-boot checks.
- `allod/nexus`: take `allod/vm` as a flake input and import its re-exported host module; expose value-free public options for per-machine credential sources, writable-volume placement, and the one host plaintext root; publish the active root value for standalone rotation tools; prepare and remove per-VM runtime directories; run each selected microvm under its own principal and isolate it from sibling credentials and volumes; bound the host-wide settings the upstream host module brings with it; keep libvirt enabled; and make the two SSH-key rotation tools dispatch by runtime.
- `allod/profiles`: update only the public example home configuration that currently hard-codes the Forge SSH identity under `~/.ssh`, so the microvm example consumes the framework-provided runtime identity path. The profiles schema and private layering model do not change.
- Documentation in the changed repositories that describes the runtime fact, guest-module split, host options, runtime credential paths, persistent-volume ownership, rotation behavior, and the fact that the libvirt path remains supported.

The existing structured secret registries remain authoritative for credential aliases, owners, rotation state, and encrypted source files. This arc derives runtime delivery from them; it does not introduce a second credential inventory.

Persistent-state ownership is split deliberately:

- The framework declares which guest mount points an archetype must keep. For the dev archetype this includes the user's home, which contains checkouts, in-flight work, and agent state, and the writable Nix store overlay contract 6a requires.
- Deployment data supplies host placement through the public option shape. Initial image creation, filesystem formatting, ownership, and capacity are explicit private provisioning actions before a VM is enabled; the framework does not create volumes on start.
- A volume image is unique state once a guest has been used. It is not disposable merely because the VM root and store image are.

The network boundary is also explicit:

- The framework declares a guest TAP interface and its inventory-owned MAC.
- Deployment integration supplies addressing, host TAP attachment, routes, and systemd-networkd configuration.
- The public guest module carries no real address.

Out of scope:

- Real host configuration values, real secret material, real addressing, real volume locations and sizes, host rebuilds, and cutover of running machines. The private integration plan owns them.
- Ephemeral SSH host keys. This arc preserves the stable pinned host-key model while changing delivery. Reconsidering key lifetime is separable after per-boot delivery exists.
- Brokering short-lived service authority so a compromised guest holds less durable power. That is allod/strategy#13; per-boot delivery changes persistence, not the authority of a stolen credential.
- The privacy-VM Tor topology on microvm.nix. The public examples keep a libvirt privacy VM so this arc does not weaken the existing fail-closed topology while it is still expressed in libvirt XML.
- Removing libvirt, libvirtd, `nixos-anywhere`, or the existing provisioning scripts. Both runtimes coexist through migration, and the libvirt path remains a tested rollback target.
- A general volume manager, snapshot interface, backup policy, live migration mechanism, or multi-host scheduler.
- Changing Forgejo accounts, SSH pin registries, token registries, or age as the host-side encrypted source format.

## Risk Assessment

**Overall residual risk: R3 High.**

The arc crosses repository interfaces and changes generated boot, activation, systemd, credential, persistence, networking, and rotation behavior. Those are R3 areas even after strong local validation. It is not R4 for the public work: no PR operates on a real host, mutates deployed state, or handles real plaintext, and every framework contract is exercisable with synthetic fixtures and nested VMs. The private cutover remains R4 in its own plan because it acts on real host state and unique writable volumes.

The worst credible public failure after validation is a guest or host unit that evaluates cleanly but strands on boot, loses runtime credentials, writes derived plaintext to a durable volume, or exposes a sibling VM's files. Nested boots, generated-unit inspection, store scans, restart tests, and hostile cross-VM fixtures target those failures directly.

| PR or milestone | Risk | Reason | Human scrutiny |
|---|---|---|---|
| `allod/inventory` runtime fact | R2 Medium | Changes a machine-data contract consumed by host scripts and `vmFacts`; rollback is a revert and both values are checked. | Schema propagation into generated JSON and missing/unknown-value failures. |
| `allod/vm` guest-module split | R3 High | Changes boot and filesystem composition and adds an upstream runtime dependency. | Confirm microvm never composes disko, a disk root, or a bootloader, while `qemuGuest` output remains unchanged. |
| `allod/nexus` host runtime and nexus#23 isolation | R3 High | Adds generated host units around plaintext credentials, writable images, namespaces, and one runner principal per VM. | Inspect concrete unit properties and the behavioral sibling-access fixture, including `/proc/<pid>/root` and the shared-principal sabotage case. |
| `allod/archetypes` guest integration and credential consumers | R3 High | Selects runtime modules, changes secret consumption, and defines persistent and network behavior. | Inspect initrd ordering, missing-credential failure, SSH host-key startup, runtime-only destinations, and restart persistence. |
| `allod/profiles` public runtime identity path | R2 Medium | Localized consumer-path change with a generated Home Manager assertion. | Confirm libvirt still resolves the existing home path and microvm resolves only a `/run` path. |
| `allod/nexus` nexus#21 host-key rotation | R3 High | Changes a trust and anti-TOFU lifecycle while preserving two runtime paths. | Stage/activate/retire state transitions, restart failure recovery, and selected-endpoint pin behavior. |
| `allod/nexus` nexus#22 Forge-key rotation | R3 High | Changes deployment and verification of a durable Forge authority. | Prove no microvm path installs the key under a persistent home and both runtime verification paths fail closed. |
| Final `allod/archetypes` input integration | R3 High | Establishes the cross-repo revision set and closing validation signal. | Input ordering, full check output, generated host/guest artifacts, and absence of private values. |

Most useful human scrutiny is the boundary between the privileged host preparation unit, the unprivileged per-VM runner principal, and the guest's systemd consumers. Source review alone is not enough: review the rendered units and runner command, then the negative fixtures.

## Interface Contracts

The transport-specific rules below were established by a nested-boot spike on nixpkgs `b6018f87da91d19d0ab4cf979885689b469cdd41` and microvm.nix `39a499ab85311b56dddb09ec43351cc3658f22c1`. The implementation may advance those pins only by rerunning the same lifecycle checks.

1. **Runtime is a required machine fact.** Every non-hypervisor `inventory.machines.<name>` has `runtime = "libvirt"` or `runtime = "microvm"`. Missing, non-string, and unknown values fail evaluation. `lib.vmSpecsJson`, committed `scripts/vm-specs.json`, and `archetypes.vmFacts.<name>.runtime` agree exactly. Hypervisor entries do not acquire a fake guest runtime.

1a. **One VM-owned upstream pin supplies both module roles.** `vm` has the sole direct `microvm` input with `microvm.inputs.nixpkgs.follows = "nixpkgs"`, re-exports `microvm.nixosModules.host` as `vm.nixosModules.microvmHost`, and makes its exclusive `vm.nixosModules.microvmGuest` wrapper import `microvm.nixosModules.microvm` directly from the same lexical input. Nexus imports only `microvmHost`; no consumer needs a raw upstream guest export. `nexus` has a direct `vm` input and no `microvm` input or transitive-input lookup. In archetypes, `nexus.inputs.vm.follows = "vm"` and every existing common input follows the top-level equivalent. The final integration check inspects the lock graph and evaluated host/guest module origins to prove that the host and guest use the same `vm` revision, microvm revision, and nixpkgs revision.

2. **Runtime selects one guest module, never two.** Libvirt machines compose the existing `vm.nixosModules.qemuGuest` behavior. Microvm machines compose the new microvm guest module and the shared base, but not disko, `disk.nix`, the NixOS qemu disk profile, or a bootloader. Composing both runtime modules is an evaluation error with a named assertion rather than a precedence accident.

3. **The microvm runner is QEMU with the upstream machine default.** `microvm.hypervisor = "qemu"` and `microvm.qemu.machine` is not overridden (`"microvm"` on x86_64 at the pinned revision). `credentialFiles` is QEMU-only. A non-QEMU runner with a non-empty credential map fails evaluation.

4. **The microvm root is the upstream tmpfs root.** No microvm `fileSystems` entry resolves `/` or `/boot` to a partition or disk device, no EFI or systemd-boot installer is enabled, and no disko device is generated. A mutation that composes the disk guest module must fail evaluation.

5. **Credentials exist before initrd activation consumers.** `boot.initrd.kernelModules` contains `qemu_fw_cfg`, and `boot.blacklistedKernelModules` does not. Removing the initrd module proves an activation-time consumer cannot read the credential; blacklisting it is rejected at evaluation.

6. **There are zero shared folders.** `microvm.shares = []`, `microvm.storeOnDisk` resolves true, and the runner contains no `bin/virtiofsd-run`. The upstream `microvm-virtiofsd@` template may still exist, but its condition skips and no virtiofsd process runs.

6a. **A selected dev guest keeps a writable Nix store.** At the pinned revision the upstream guest module mounts `/nix/store` straight from the read-only erofs store disk, and sets `systemd.services.nix-daemon.enable` and its socket false, whenever `microvm.writableStoreOverlay` is null. A guest that sets only `storeOnDisk` therefore cannot realise, fetch, or collect a store path, while still carrying the shared guest base's flake settings and trusted users and contract 9's Nix netrc and Git credential wiring — a dev VM that cannot run `nix build`, `nix flake check`, or `nix flake update` does not do the job the archetype exists for. Every selected dev microvm therefore sets `microvm.writableStoreOverlay` to a mount point in contract 13's required persistent set, which makes `/nix/store` an overlay of the erofs lower plus that volume's upper, marks the volume `neededForBoot`, and re-enables the daemon. `nix.optimise.automatic` and `nix.settings.auto-optimise-store` stay false, as upstream asserts for that overlay. `microvm.storeOnDisk` still resolves true and `microvm.shares` stays empty, so contract 6 is unchanged, and the overlay upper is a persistent mount like any other, so contracts 13 and 14 govern it unaltered. A selected dev microvm with a null `writableStoreOverlay`, an overlay mount point outside the declared persistent set, or store auto-optimisation enabled fails evaluation.

7. **Credential names and host paths are closed contracts.** Each `microvm.credentialFiles` key is non-empty and at most 28 characters. In the host-integrated configuration, each value is the exact absolute string `<host-plaintext-root>/active/<machine>/<credential-name>` derived from the host option and the declared credential-name set; no guest module restates the default root. The path must not resolve under the Nix store prefix. A Nix path literal, relative path, overlong name, duplicate normalized name, non-derived value, source/guest-map mismatch, or unexpected extra credential fails evaluation.

8. **Host preparation is per start and uses only the mounted active tree.** `nexus` exposes a per-machine credential-source map whose values default to nothing and are supplied by deployment composition. It replaces upstream `microvm@<name>`'s direct runner `ExecStart` with one root launch helper and explicitly overrides that unit's `User` and `Group` to root, rather than adding a separate preparation oneshot. On every service start, including `Restart=always`, that helper first proves the host plaintext root satisfies contract 8a, then validates and atomically prepares that VM's own directory under its derived `active/<name>` child, creates the VM's mount namespace, performs the sole uid/gid drop to that VM's own `microvm-<name>` principal, and executes the runner. The guest leaves `microvm.user` unset so the generated QEMU command has no second `-run-with user=` transition. Mount verification, preparation, or privilege-drop failure prevents QEMU from starting without creating a fallback directory on the underlying `/run`. The declared absolute `microvm.socket` is in one per-VM QMP working directory outside the prepared credential directory and beneath a root-only source parent; the directory is owned `root:microvm-<name>` with mode `0770`, so only that VM's QEMU process can create the socket in it. The launcher removes and recreates that exact directory on every start, so a directory or socket left behind by an abnormal exit is cleared and logged rather than reused or treated as fatal — a fail-closed reading would wedge the VM in a restart loop that only a human on the host could clear — and it bind-mounts the fresh directory at the same absolute path in its namespace so the upstream host-namespace `ExecStop` reaches the socket QEMU created. The launcher cannot clean after the runner it execs has exited: Nexus instead appends one idempotent root `ExecStopPost` with `lib.mkAfter`, after every upstream `ExecStopPost`, to remove only the prepared credential and QMP directories. That final hook runs after preparation failure, QEMU failure, manual stop, rebuild stop, and rollback restart; systemd completes it before an automatic or requested next start can prepare fresh directories. It never removes the host plaintext mount or its derived `rollback/<name>` slot. The generated unit is the only supported start path: importing the host module also installs upstream's `microvm` command for every account on the host, and `microvm -r <name>` execs the runner directly with no unit, so no prepared credentials, no namespace, and no privilege drop. It fails closed only because the `active/<name>` directory the runner's `-fw_cfg` paths name exists solely while the unit runs, which is exactly what the post-stop hook above guarantees. `microvm -s <name>` execs `ssh` with `StrictHostKeyChecking=no` and `UserKnownHostsFile=/dev/null` into a root shell over VSOCK, so no selected microvm sets `microvm.vsock.cid`; a later revision wanting VSOCK console access pins that path's host key before enabling it. `microvm -u` is inert against the fully-declarative form, which writes the `toplevel` symlink it refuses. No secret bytes appear in argv, the Nix store, logs, or durable host storage.

8a. **The two plaintext roots have different owners.** The Nexus host module exposes `nexus.microvm.hostPlaintextRoot`, the one value-free readable host option (`/run/allod/microvm` by default); per-start credentials live only under its derived `active/<name>` child and rotation recovery state only under its root-owned `rollback/<name>` child, so the rollback area is not a second option or mount. Nexus renders that active option value as `hostPlaintextRoot` in `/etc/allod/microvm-runtime.json`, a non-secret derivative of the running host configuration; the packaged rotation tools read this file rather than a baked default or a possibly unactivated deploy-flake checkout, and their fixtures redirect only its path through `ALLOD_MICROVM_RUNTIME_CONFIG`. The Archetypes microvm guest integration owns `allod.microvm.guestCredentialRoot`, the independent guest option (`/run/allod/credentials` by default), and derives its materializer, Nix, Git, SSH, Forge, and Home Manager consumers from it. Generated units, launcher arguments, rotation tools, guest consumers, and checks read the owning option, its generated descriptor, or a derived child rather than restating a path literal. Both options are normalized absolute strings strictly below `/run/allod`, never Nix paths or store-prefixed strings; invalid values fail evaluation.

The host module declares the host root as a real systemd mount unit — a `fileSystems` or `systemd.mounts` entry, never `boot.specialFileSystems` — of type `tmpfs` with `noswap`, mode `0700`, and a fixed framework-owned 16 MiB limit. `ramfs` is not permitted: the pinned kernel cannot bound or resize it, so a bad source or writer bug could consume all host memory. The unit fragment is what makes the dependency real. Systemd derives `Requires=` from `RequiresMountsFor` only for a mount unit that has a fragment file; a `boot.specialFileSystems` mount exists only in `/proc/self/mountinfo`, so against one the same setting degrades to bare ordering, and when nothing is mounted at the path it degrades silently to `-.mount` and the service starts anyway. With the fragment in place the concrete `microvm@<name>` unit carries `RequiresMountsFor` on the host root and refuses to start when that mount unit fails. Its launcher independently verifies before any source read or plaintext write that the root itself — not merely an ancestor selected by `findmnt --target` — is the mounted `tmpfs`, has `noswap`, has the configured bound, and is mode `0700`. The two rotation tools perform the same preflight before decrypting, copying, or creating recovery state because they run outside any unit and cannot inherit systemd ordering. A missing, unmounted, wrong-type, swappable, over-capacity, or wrongly owned root makes every writer fail before it reads a source or creates a destination, so an empty mount point is the only thing that can appear on stock `/run`; no tmpfiles rule and no writer builds a fallback tree there. The host-global mount and rollback child survive VM stop, failure, and automatic restart. A host reboot or loss of that mount destroys both active and rollback bytes and follows contract 19's explicit lost-slot recovery rather than claiming persistence. No microvm guest declares a swap device, so the guest's tmpfs root is non-pageable too.

8b. **The host option crosses the guest-evaluation boundary once.** A standalone Archetypes microvm configuration declares the required credential-name set but no host root or `credentialFiles` value. When Archetypes composes the hypervisor, it uses pinned nixpkgs's `extendModules` interface on each selected exported guest `nixosConfiguration` to add one module whose `microvm.credentialFiles` map derives from the enclosing host's evaluated Nexus plaintext-root option and that guest's declared names, then assigns that extended result to upstream `microvm.vms.<name>.evaluatedConfig`. The runner installed and started by the host is built from that same extended result. Nexus does not import Archetypes or duplicate the name set, and the standalone unextended guest output is not accepted as the host runner. This keeps dependency direction intact while making a host-root override change the actual QEMU `-fw_cfg file=` arguments. Supplying `evaluatedConfig` rather than `config` also flips upstream's `microvm.vms.<name>.restartIfChanged` default to false, because that default keys off `microvm.vms.<name>.config != null`; the rendered drop-in then carries `X-RestartIfChanged=false` and `switch-to-configuration` skips the unit on every rebuild instead of stopping it. Each selected microvm therefore sets `restartIfChanged = true` explicitly.

9. **Guest-derived plaintext has one runtime root.** Microvm consumers use named files under the guest credential root; they do not write credential-derived plaintext under `/home`, `/root`, `/etc`, any declared persistent mount, or any other durable filesystem. The runtime module configures the Forge API token path, Git credential-store path, Nix netrc path, user netrc path if still needed, Forge SSH `IdentityFile`, and any registered GitHub credential target to those runtime files. The libvirt destinations remain unchanged.

10. **The materializer is a boot service, not the current activation script.** It imports or loads credentials with systemd, validates exact input shape, writes consumer files atomically with declared owner and mode, and orders before every consuming daemon or user session. It does not inherit the provisioning-era "missing is expected" branch. A required missing, empty, malformed, or wrongly owned credential makes the materializer and dependent consumer fail visibly.

11. **The stable SSH host key cannot auto-regenerate.** For microvm, `services.openssh.hostKeys` is empty and sshd is pointed at the delivered runtime key, or an equivalent ordering prevents `sshd-keygen` from ever minting a replacement. A boot without the credential leaves sshd failed and reports the missing key; it never generates a new identity. Libvirt keeps its existing `/etc/ssh/<name>` path and installer lifecycle.

12. **No microvm guest decrypts with age.** For every microvm archetype, `config.age.secrets == {}` and `age.identityPaths` is unset. Host-side encrypted source files and rotation registries remain authoritative. A closure and generated-config check rejects an age private key, a VM host private key, or an agenix deployment target in a microvm guest.

13. **Every required persistent path maps to an explicitly provisioned writable volume.** The framework declares a non-empty, duplicate-free set per selected microvm archetype. Every path has exactly one `microvm.volumes` entry with `autoCreate = false`; every image path is outside the Nix store and credential runtime root, and the root/store disk is not mistaken for persistence. The private deployer explicitly creates, formats, labels, owns by that VM's runner principal, and assigns the image before enabling the VM; the root launch helper refuses a missing, non-regular, or inaccessible declared image before QEMU starts. The guest marks every required persistent `fileSystems` entry `neededForBoot = true`, so an unformatted, corrupt, or wrongly labeled image fails in the initrd before sshd, user sessions, or any writer can use the tmpfs directory underneath. Empty sets, duplicate mount points, required paths without volumes, or `autoCreate = true` fail evaluation. This deliberately leaves upstream `truncate` and `mkfs` out of every framework runner.

14. **Persistence and secrets never overlap.** No runtime credential path is equal to or nested under a declared persistent mount. Generated activation text, Home Manager output, and unit definitions contain no command that copies a delivered credential into a persistent path. A mutation that adds such a copy must fail the validator.

15. **The guest declares networking but no address.** Each selected microvm has one TAP interface with its inventory MAC and a valid host interface ID. The guest definition contains no real IP, route, gateway, or DNS value. Host TAP attachment and addressing remain deployment inputs. The public example's nested test may use synthetic test-network values local to the fixture.

16. **Per-VM isolation needs a distinct principal, not only a namespace.** Each selected microvm runs as its own `microvm-<name>` system user with its own primary group, holding `kvm` only as a supplementary group for `/dev/kvm`. `microvm.user` stays unset so the generated QEMU command has no `-run-with user=` transition, and the observed runner and QEMU processes carry that per-VM uid and gid. The root launch helper prepares files before isolation, then creates a private mount namespace with a tmpfs root and exposes only the selected runner/current path read-only, the selected QMP working directory at its identical absolute host path, read-only `/nix/store`, the required device nodes including KVM and TAP access, a fresh `/proc`, the selected credential directory read-only, and the selected volume images read-write. No source parent of any of those objects is bound into the namespace. Before its single uid/gid transition the helper enters the new root and leaves no inherited working directory, old root, or open descriptor outside it: upstream sets the unit's `WorkingDirectory` to the state directory, so an inherited cwd would hand the runner the `booted` symlink that the root unit executes at stop. No path the root unit itself executes is writable by any runner principal, so the host module also tightens upstream's `0775 microvm:kvm` state-directory rule until no runner principal can create, rename, or delete an entry in it.

16a. **The namespace closes direct paths; only the principal closes procfs.** Through a direct path, symlink, or alternate bind path, one VM reaches no sibling object and no source parent — the root-only source parents and the tmpfs root are effective there. They are irrelevant to `/proc/<sibling-pid>/root`, which resolves inside the target's own root and never traverses them. At the pinned kernel that open is gated only by the same-credential comparison in `__ptrace_may_access`, so under one shared runner uid every sibling credential, QMP directory and socket, runner entry, and image stays readable, and the group-writable QMP directory makes the sibling's shutdown socket unlinkable — a cross-VM primitive for turning another guest's graceful stop into a kill. `hidepid` and `subset=pid` do not help, because each `has_pid_permissions` branch reduces to the same `PTRACE_MODE_READ_FSCREDS` check, and Yama only inspects `PTRACE_MODE_ATTACH`. The distinct principal is the control that denies all of it, and it equally denies sibling signals and `/proc/<sibling-pid>/mem`. No `kernel.yama.ptrace_scope` pin is part of this arc: it cannot close the path vector, and distinct principals already deny cross-VM attach in the credential check before Yama runs. Each VM retains only the corresponding access to its own declared objects, and isolation setup failure prevents VM start.

16b. **Per-VM ownership follows the principal.** The QMP working directory is `root:microvm-<name>` mode `0770`, the prepared credential directory is readable only by that VM's principal, and every declared volume image is owned by it. Upstream's `tap-up` hardcodes `user = "microvm"` at the pinned revision, so the host module supplies TAP ownership for the per-VM principal instead of inheriting it; a TAP the runner cannot open fails the VM's start rather than falling back.

17. **Host and guest declarations agree before boot.** For each selected microvm, the prepared host credential names equal the extended guest's `credentialFiles` keys, every value equals the exact host-option-derived `active/<machine>/<credential-name>` path from contract 7, the host installs the runner from that extended guest rather than the standalone base result, the declared volume files equal the guest volume attachments, and the runtime in host tooling equals inventory and `vmFacts`. The generated check evaluates and compares the two sides; it does not trust duplicated prose or a default-path text scan.

18. **Rotation dispatches on `vmFacts.<name>.runtime`.** Libvirt stage/activate/retire behavior and guest installation remain intact. Microvm activation refreshes only the target VM's host runtime credential, restarts its declared `microvm@<name>` unit, and verifies the presented SSH or Forge identity through the runtime path. It never copies a private key into the guest's persistent filesystem and never tells the operator to add the guest as an age recipient.

19. **Rotation failure preserves authoritative state.** Before replacing a microvm runtime credential, activation passes contract 8a's exact-mount preflight, then copies the currently running file to the root-only derived `rollback/<name>` slot outside `active/<name>` and verifies it against the registry's active public key. It keeps that slot until restart, identity verification, and registry promotion succeed. If refresh, restart, or verification fails first, the tool atomically arms a one-shot recovery selector beside that slot and restarts the unit. On that start the root launcher must materialize the selected credential from the verified rollback slot instead of the still-staged configured source; `ExecStopPost` must not remove either recovery object. The selector stays armed across automatic restart attempts and is removed with the slot only after the tool verifies the old presented identity. Staged/active registry and known-host state remain unchanged. A host reboot or loss of the host plaintext mount destroys the slot: the tool then stops unpromoted and gives an exact `git show <pre-stage-revision>:<encrypted-source-path>` recovery command from the committed pre-stage ciphertext, rather than claiming automatic recovery. Fixtures exercise failures at each boundary, including loss of the mounted runtime rollback slot.

20. **Rotation manages the selected endpoint only.** The tools preserve unrelated `known_hosts` lines and do not pretend to reconcile a second rollback endpoint during a dual-runtime migration. The private cutover freezes routine rotation while both endpoints or pins are live, then reconciles the retired endpoint before lifting the freeze. This avoids adding a permanent migration-state schema solely for a one-owner cutover.

21. **Libvirt remains a first-class tested path.** Existing inventory values continue to build and provision with disko, `nixos-anywhere`, guest-local agenix, stable key paths, and the current network flow. No default silently changes an omitted runtime to microvm. Host libvirt services and tools remain enabled after the microvm host module is added.

21a. **Importing the upstream host module is a host-wide act.** At the pinned revision its host module makes four assignments outside `microvm.*`, and the plan pins the ones that matter rather than inheriting them. `hardware.ksm.enable = lib.mkDefault true` beats the NixOS default of `false`, and QEMU marks guest RAM mergeable by default, so the import would switch on page merging across every QEMU process on the host, including the libvirt guests contract 21 keeps first-class; an arc whose isolation contract is about what one guest can learn about a sibling does not acquire a cross-guest memory-deduplication side channel by import, and merging also makes host capacity evidence recorded before adoption incomparable with evidence recorded after. The host module therefore sets `hardware.ksm.enable = false` explicitly, and a deployment that wants merging opts in with a recorded reason. `environment.systemPackages` gains upstream's `microvm` command, which contract 8 bounds. The remaining two are inert: `boot.kernelModules` gains `tap` and `vhost_net`, and the `security.pam.loginLimits` `memlock` grant never applies because the generated units set no `PAMName`, so the runner's locked memory comes from the unit's own `LimitMEMLOCK`. Two adjacent effects need no fix while libvirt is enabled — upstream's `mkDefault` `qemu/bridge.conf` loses to the libvirt module's own definition, and its `qemu-bridge-helper` wrapper is guarded by `!virtualisation.libvirtd.enable` — and both change on a host that later disables libvirt.

## Implementation Sequence

1. Land the inventory runtime fact and generated JSON update. This gives every later consumer one authoritative discriminator.
2. Land the `allod/vm` guest-module split, sole upstream pin, wrapped guest module, and re-exported host module. Prove disk and microvm composition independently before any archetype selects the new module.
3. Land the `allod/nexus` microvm host module with its `vm` input, host plaintext-root option and active-system descriptor, per-start root launcher, and nexus#23 isolation against synthetic guests. It may consume an injected runtime and credential-name map in tests before archetypes exports the real ones; it must not add a second microvm.nix pin.
4. Land the archetypes and public profiles consumer work together: pin the inventory and VM revisions, make `nexus.inputs.vm.follows = "vm"`, export runtime through `vmFacts`, select the guest module, declare credentials/state/networking and the guest credential root, extend each selected evaluated guest with host-option-derived `credentialFiles`, wire that exact result into the host, and boot the public microvm example in nested checks.
5. Land nexus#21 and nexus#22 after the runtime fact, host refresh interface, and active-system root descriptor exist. Each PR proves both libvirt and microvm rotation paths.
6. Update the final archetypes PR or follow-up integration PR to pin the completed nexus revision. Run the whole public check matrix and use this PR as the sole closing PR for allod/strategy#20.

Do not combine the source-repo PRs into one cross-repo change. Each step is independently reviewable and revertible, while the final pin records the coherent revision set.

## Agent Gates

The public implementation agent runs in a dev VM and may build every public flake, boot nested VMs, inspect closures and units, and use synthetic credential files. It cannot:

- Rebuild or run commands on the real hypervisor host.
- Create, decrypt, rotate, re-encrypt, or place real key material.
- Supply the private host credential sources, volume paths/sizes, TAP attachment, addresses, routes, or SSH client pins.
- Prove the real host's QEMU, KVM, TPM, storage, or network behavior.
- Start cutover, stop a real libvirt VM, delete a domain or disk, or enable microvm autostart for a real machine.
- Run routine key rotation during a private dual-endpoint migration window. The human must freeze it before cutover and reconcile the retired endpoint before unfreezing.

These gates do not block the public plan. They block the private plan's host rebuild, real nested-boundary validation, and cutover. The human performs those steps only after the public closing PR and private integration plan have passed review.

## Acceptance Tests

Every named check below is part of the implementation, not an ad hoc command that can be skipped.

**Repository and schema checks:**

```sh
cd /path/to/inventory
nix flake check --print-build-logs
nix eval .#lib.vmSpecsJson --raw | jq -e '
  to_entries
  | all(.value.runtime == "libvirt" or .value.runtime == "microvm")
'
diff -u <(nix eval .#lib.vmSpecsJson --raw | jq -S .) \
  <(jq -S . scripts/vm-specs.json)

cd /path/to/vm
nix flake check --print-build-logs

cd /path/to/profiles
nix flake check --print-build-logs

cd /path/to/nexus
nix flake check --print-build-logs

cd /path/to/archetypes
nix flake check --print-build-logs
nix eval .#vmFacts --json | jq -e '
  to_entries
  | all(.value.runtime == "libvirt" or .value.runtime == "microvm")
'
```

Inventory mutation checks must show that a missing runtime, an unknown runtime, and drift in committed JSON each fail. Archetypes must independently show that missing or unknown runtime data cannot be hidden by a default.

**Guest module composition:**

```sh
cd /path/to/vm
nix build .#checks.x86_64-linux.guest-module-contracts --print-build-logs

cd /path/to/archetypes
nix build .#checks.x86_64-linux.runtime-module-selection --print-build-logs
```

The generated checks assert that the libvirt example still has its disko GPT root and bootloader, while the microvm example has the upstream tmpfs root, no disk/partition root, no `/boot`, no bootloader, no disko devices, QEMU selected, the default QEMU machine, `qemu_fw_cfg` in the initrd, and zero shares. A deliberate both-modules mutation must fail.

**Nested microvm lifecycle:**

```sh
cd /path/to/archetypes
nix build .#checks.x86_64-linux.microvm-nested-boot --print-build-logs
```

The nested test must:

1. Start the host unit through the root launch helper with synthetic host-key, Forge token, Forge SSH key, and Git credential canaries.
2. Assert PID 1 reports receipt of each expected system credential and the materializer round-trips every canary byte-exact.
3. Assert the system credential directory is root-only on a `noswap` tmpfs, the guest declares no swap device, and every consumer file under the guest credential root has the declared owner and mode.
4. Assert sshd presents the supplied public host key and a boot with the host-key credential absent fails sshd without creating any new host key.
5. Assert the Forge CLI, Git HTTPS/netrc flow, and Git SSH configuration resolve their runtime files, while no corresponding file exists under the persistent home, `/root`, or `/etc`.
6. Pre-create, format, label, and hand the synthetic volume images to the fixture before enabling the microvm. Write a distinct canary under every declared persistent mount, restart the microvm host unit, and assert every canary survives. Separately prove an absent image fails before QEMU, an existing image is never truncated or relabeled, and a regular image with an invalid filesystem or expected label fails in the initrd without starting sshd, a user session, or a writer against the tmpfs mount point.
7. Write a canary outside the declared persistent mounts, restart, and assert it is gone. This proves persistence came from the volume rather than an accidentally durable root.
8. Assert the guest TAP interface exists and can reach the fixture network without any deployment address appearing in the public guest definition.
9. Realise or add a path into the guest's Nix store, restart the microvm host unit, and assert the path is still valid, that the nix daemon is running, and that `/nix/store` is an overlay whose upper is the declared persistent volume. A paired fixture with `writableStoreOverlay` unset must fail the store write against the read-only erofs mount instead of appearing to succeed.
10. Start cold with no prepared credential or QMP directory. Exercise a preparation failure, an early QEMU failure with `Restart=always`, manual stop and restart, a rebuild stop driven by a guest change that moves the `system.build.toplevel` the generated restart trigger keys on, and rollback restart. In every path assert the final Nexus `ExecStopPost` runs after the rendered upstream post-stop commands, removes any complete or partial credential and QMP directories only after QEMU exits, preserves the volume and rollback slot, and lets the next attempt recreate a different QMP-directory inode with a newly reachable socket. For the successful stop, install a guest shutdown unit that writes a unique sentinel to the persistent volume, ordered after that mount and before `shutdown.target` so it lands while the volume is still mounted; invoke the rendered upstream QMP `ExecStop` and require that sentinel before accepting directory cleanup as a graceful shutdown. The pinned stop path sends ctrl-alt-del over QMP and the runner carries `-no-reboot`, so the guest runs a reboot transaction that QEMU turns into an exit: the sentinel must survive that path, not only a poweroff. In a separate sabotage case remove or break the socket while QEMU is live, stop the unit, and prove the sentinel is absent when the pinned shutdown script falls through and systemd kills QEMU; directory removal or a successful later restart alone does not count as QMP-shutdown evidence.

**Negative and mutation checks:**

```sh
cd /path/to/archetypes
nix build .#checks.x86_64-linux.microvm-contract-mutations --print-build-logs
```

Each mutation must be demonstrated to fail for the intended reason:

- `qemu_fw_cfg` removed from `boot.initrd.kernelModules`.
- `qemu_fw_cfg` added to `boot.blacklistedKernelModules`.
- `credentialFiles` value supplied as a Nix path literal or store-prefixed string.
- Relative credential path, 29-character name, duplicate normalized name, missing required name, and unexpected extra name.
- Non-QEMU hypervisor with credentials.
- Empty persistent path set, duplicate mount, missing volume, store-backed volume, `autoCreate = true`, and a credential destination nested under persistence.
- A generated command copying credential-derived plaintext into a persistent path.
- Both runtime guest modules composed.
- Microvm sshd with host-key auto-generation enabled.
- Microvm `age.secrets` or `age.identityPaths` populated.
- Microvm guest declaring a swap device.
- Selected dev microvm with a null `writableStoreOverlay`, a store overlay outside the declared persistent set, or store auto-optimisation enabled.
- Selected microvm setting `microvm.vsock.cid`.
- A plaintext-root option supplied as a Nix path, a relative or store-prefixed string, or a value outside `/run/allod`.
- A host-integrated guest whose `credentialFiles` are absent, come from the standalone unextended result, or retain a default-root path after the host root changes.
- Guest interface omitted or a real address introduced.

The no-initrd-module case is behavioral: boot far enough to prove an activation-time consumer cannot read the canary and fails visibly. The other invalid configurations should fail evaluation before a runner is built.

**Generated artifacts and closure:**

```sh
cd /path/to/archetypes
nix build .#checks.x86_64-linux.microvm-generated-artifacts --print-build-logs

cd /path/to/nexus
nix build .#checks.x86_64-linux.microvm-host-contract --print-build-logs
```

The checks inspect the actual runner, guest activation text, Home Manager result, host units, and closures and assert:

- No credential source or derived plaintext path is under `/nix/store`.
- No credential destination appears under a persistent mount.
- The runner contains no `virtiofsd-run`, no share, and no private byte canary.
- The microvm guest closure contains no age identity, private host key, Forge private key, or agenix target.
- The root launch helper prepares credentials and a fresh `root:microvm-<name>` mode-`0770` QMP directory on every start, clears and recreates a leftover directory or socket instead of reusing it, invokes the matching runner only after preparation, and gives the root-namespace upstream `ExecStop` the same absolute per-VM QMP socket path that QEMU sees in its namespace. A final root `ExecStopPost` merged with `lib.mkAfter` follows the rendered upstream post-stop commands and idempotently removes only the credential and QMP directories, including after failed preparation or QEMU exit; neither command puts secret bytes in argv, environment, or logs.
- The one host plaintext root is an exact `noswap` `tmpfs` mount with mode `0700` and the fixed 16 MiB bound at the path its option reports, declared by a mount unit with a real fragment. The rendered `microvm@<name>` unit's resolved `Requires=` names that mount unit, and a fixture whose mount unit fails leaves the VM unstarted with a dependency failure rather than a running QEMU. Both the launcher and the standalone rotation tools reject an absent mount, an ancestor-only stock `/run` match, `ramfs`, swappable or oversized `tmpfs`, and a wrong mode before any plaintext source read, destination creation, or VM start; paired fixtures fail for each intended reason.
- Two independently evaluated integration fixtures use the defaults and safe non-default host and guest roots. In the non-default fixture the host option moves the early mount, active-system descriptor, unit dependency, launcher arguments, rotation-tool target, every exact `credentialFiles` value, and the built runner's `-fw_cfg file=` arguments; the guest option separately moves the materializer, Nix, Git, SSH, Forge, and Home Manager destinations. A corresponding artifact retaining an old default fails, and the check proves the host installed the extended runner rather than the standalone guest result.
- Host and guest credential-name sets, exact host-option-derived credential values, and volume declarations are identical.
- Each selected microvm's rendered `microvm@<name>` drop-in carries `X-RestartIfChanged=true`, so a guest change stops and restarts the VM on a host rebuild instead of being skipped.
- The generated runner contains no automatic volume-creation command, and a missing image is rejected before QEMU.
- The concrete microvm unit runs the root launch helper as root; the helper's namespace exposes only its declared paths, performs the sole drop to that VM's own `microvm-<name>` principal before the runner, and the observed runner and QEMU processes have that uid/gid with no QEMU `-run-with user=` argument. No two selected microvms share a runner uid, and no runner principal can write the state directory or any path the root unit executes.
- The libvirt services, packages, environment, and provisioning script outputs remain present.
- Importing the microvm host module leaves the host's rendered `hardware.ksm.enable` false and the libvirt bridge configuration and wrappers unchanged; an upstream default change fails this check rather than altering the host silently.

**Cross-VM host isolation:**

```sh
cd /path/to/nexus
nix build .#checks.x86_64-linux.microvm-host-isolation --print-build-logs
```

A NixOS test starts two credentialed guests under their own per-VM principals and inspects their real namespace mount tables. From each unit's namespace it uses direct paths, symlinks, alternate bind paths, an inherited working directory, and `/proc/<sibling-pid>/root` to attack the sibling's credential, its `/var/lib/microvms/<name>` entries including `current` and `booted`, its QMP directory and socket, its volume image, and each host source parent, and it signals the sibling's runner. Credential reads; runner and `booted` reads, execution, and replacement; state-directory entry creation, rename, and deletion; source-parent traversal; QMP connect, unlink, rename, and replacement; image read, write, truncate, rename, and replacement; and signal delivery all fail, while each unit retains exactly the required access to its own objects. Two independent sabotage fixtures must each reopen the attack they correspond to: removing the launcher namespace/chroot must reopen the direct-path attacks, and collapsing both guests onto one shared runner principal must reopen the `/proc/<sibling-pid>/root` and signal attacks. A fixture that proves only the direct-path half does not satisfy this check, and narrowing the procfs case is the specific outcome that would close it with the boundary still open.

**Rotation lifecycle:**

```sh
cd /path/to/nexus
nix build .#checks.x86_64-linux.provisioning-contract --print-build-logs
bash tests/vm-ssh-host-key.sh
bash tests/forge-ssh-key.sh
```

Fixtures cover libvirt and microvm `stage`, `activate`, and `retire`; dry-run output; source refresh; unit restart; presented-key and Forge access verification; preservation of unrelated known-host lines; and failures before refresh, after refresh, during restart, during verification, and after loss of the host plaintext mount and its rollback slot. They assert no registry promotion on failure; prove that an armed rollback restart presents the old identity even while the normal configured source still contains the staged key; exercise an automatic retry while the selector remains armed; prove both rotation commands reject every contract-8a mount sabotage before decrypting or writing; and require an exact pre-stage-revision ciphertext recovery command when the slot is gone. They also assert no microvm guest copy and no age-recipient instruction for microvm.

**Validator validation:**

Every assertion and generated-artifact scanner added by this arc has a paired sabotage fixture. A check that passes only the valid configuration, without proving it fails on the bad configuration it claims to catch, does not count.

## Rollback Plan

Public rollback is a reverse dependency unwind:

1. Revert the final archetypes integration pin so no public configuration selects the new nexus behavior.
2. Revert the nexus rotation PRs, then the archetypes/profile consumer PR, then the nexus host/isolation PR, then the VM and inventory PRs.
3. Keep libvirt selected and rebuild the public checks after each dependency boundary. The original disko, `nixos-anywhere`, guest-local agenix, network, and rotation paths remain present throughout the arc.

No rollback step deletes a credential source, writable image, libvirt disk, domain, pin, or registry entry.

For a private guest that has already run, stop microvm autostart and return the inventory selection to libvirt only after relaying or pushing all work from every declared persistent volume. A revert reconstructs the root and store image but cannot reconstruct unique files on a volume.

If a rotation fails after refreshing a runtime credential but before registry promotion, use the tool's tested recovery path to restore the active credential and restart the previous state. Do not hand-edit the staged/active registries or known-host pins. If the tool cannot prove restoration, preserve both the runtime directory and logs, leave the registries unpromoted, and follow its explicit recovery commands before any retry.

Host tmpfs credential directories are reconstructable derivatives of the encrypted sources and may be removed after the relevant VM is stopped. Writable volumes are authoritative user state and must never be removed as cleanup.
