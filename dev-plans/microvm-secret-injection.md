# microvm.nix Secret Injection — Framework Half

## Tracking Issue

https://forge.anarch.diy/allod/strategy/issues/20

Multi-PR arc across `allod/vm`, `allod/archetypes`, `allod/nexus`, and `allod/inventory`. Earlier PRs carry `Refs allod/strategy#20`; the final integration PR carries the closing keyword.

This is the public half of a cross-boundary plan. It is a self-contained leaf: it names no private machine, path, or key, and a public-only agent can execute and validate all of it. The private plan owns integration against real machines and full-build validation, and links here.

## Goal

The VM framework can build and boot a microvm.nix guest whose secrets arrive from the host as systemd credentials at every start, with no age identity and no agenix decryption inside the guest, proven by nested VM boots and generated-artifact assertions.

## Scope

In scope:

- `allod/vm` — a microvm guest module: `boot.initrd.kernelModules` carrying `qemu_fw_cfg`, zero `microvm.shares`, and the QEMU hypervisor pin.
- `allod/archetypes` — a credential-consumption module replacing the agenix path in `modules/agent-forgejo-token.nix` and the agenix half of `modules/netrc.nix`; `sharedModules` stops setting `age.identityPaths` for the microvm archetypes; the `credentialFiles` assertions; and the checks plus their mutation fixtures.
- `allod/nexus` — the host framework module gains microvm.nix host wiring exposed as public options carrying no values, in the same shape as the existing `nexus.provisioning.deployFlake` option: the framework declares the interface, a deployment supplies the paths.
- `allod/inventory` — whichever machine fact selects the runtime for a machine, since that is a machine fact and inventory owns it. The public template machines exercise both values.

Out of scope, each with its owner:

- The real host configuration, real secret material, real machine cutover, and any absolute path on a deployed host — the private plan.
- Ephemeral per-boot host keys. Separable and cheaper to evaluate once per-boot delivery exists, because at that point a guest host key decrypts nothing and only authenticates one sshd to one client. Do not fold it in here.
- Brokering service credentials so a compromised guest holds less durable authority — allod/strategy#13, orthogonal to delivery.
- The privacy-VM Tor topology under microvm.nix. The topological fail-closed property is preservable with TAP plus netns plus nftables, but the existing design is written in libvirt XML and needs its own plan.
- Removing libvirt from the host. The two coexist through migration; a first microvm guest is provisioned alongside, not in place of, the libvirt path.

## Risk Assessment

**Residual risk: R3 High.**

Cross-repo interfaces with a sequencing constraint, generated lifecycle behavior including activation ordering and systemd units, and a change to how secret material reaches a machine. Those are R3 signals and the plan does not reduce them below that.

It is not R4. No PR in this arc mutates deployed state or authoritative secret material: the framework repos describe behavior, and the deployment that consumes them is the private plan's subject. Every contract rule below is exercisable by the implementing agent — in fixtures for the assertions, and in real nested microVM boots for the delivery path, which the spike demonstrated works inside a dev VM. Rollback is a revert against a libvirt path that this arc never modifies.

The single failure mode that would justify R4 is a `credentialFiles` value silently copied into the world-readable Nix store, which is a security-boundary mistake that could expose private material. That is why the store-prefix assertion and its mutation fixture are contract items rather than nice-to-haves, and why the arc should not land without them.

Most useful human scrutiny: whether the credential-consumption module's failure path is genuinely loud when a credential is absent, and whether the mutation fixtures actually fail on sabotaged input rather than passing vacuously.

## Interface Contracts

Every rule below was observed in a nested microVM boot on nixpkgs `b6018f87da91d19d0ab4cf979885689b469cdd41` with microvm.nix `39a499ab85311b56dddb09ec43351cc3658f22c1`, except where marked as source-derived.

1. **`microvm.hypervisor = "qemu"`.** `credentialFiles` is QEMU-only; every other microvm.nix runner throws on it, cloud-hypervisor explicitly. A machine configured for another runner with a non-empty `credentialFiles` must fail evaluation, not silently lose its secrets.

2. **`microvm.qemu.machine` stays at its default** (`"microvm"` on x86_64). Do not set it. `fw_cfg` works there and errors loudly when absent.

3. **`boot.initrd.kernelModules` must contain `"qemu_fw_cfg"`.** microvm.nix sets `boot.initrd.systemd.enable = true` by default, which relocates NixOS activation into the initrd ahead of switch-root. Without the module in the initrd, credentials are imported only by stage-2 PID 1, roughly a second and a switch-root after the activation script has already run — so anything consuming a credential during activation sees nothing. With it, the import happens in the initrd and the credential is readable at activation time.

4. **`boot.blacklistedKernelModules` must not contain `qemu_fw_cfg`.** Blacklisting it produces no credentials at all and the guest still boots successfully — a silent, total failure. Assert against it.

5. **`microvm.shares` is empty**, so `microvm.storeOnDisk` resolves true and the store is a read-only image rather than a host share. Note precisely: the host module always emits the `microvm-virtiofsd@` template unit, but with zero shares its `ConditionPathExists` gate never matches, so it condition-skips and no virtiofsd process runs. A check must assert the *runner* has no `virtiofsd-run`, not that the template unit is absent.

6. **Every `credentialFiles` value is a quoted absolute string.** The option's type accepts a Nix path literal, which copies the file into the store at mode 0444 — world-readable plaintext that also propagates into any binary cache. An assertion must reject any value whose string form begins with the store prefix.

7. **Every `credentialFiles` key is at most 28 characters.** systemd caps the full `io.systemd.credentials/<name>` string at 55. Existing secret names must be checked against this before they are reused as credential names.

8. **An absent credential fails loud.** A unit consuming a credential that was not delivered must fail visibly rather than starting degraded. The tolerate-a-missing-optional-resource pattern used during `nixos-anywhere` provisioning is the wrong default here, because there is no provisioning window to tolerate.

9. **No guest holds an age identity.** For microvm archetypes, `config.age.secrets` is empty and `age.identityPaths` is unset. Consumption is via `ImportCredential=` or `LoadCredential=` in units.

10. **Host-side plaintext lives on a ramfs.** The framework's option documentation states this requirement; the deployment supplies the path. The framework must not default it to anything on durable storage.

## Agent Gates

The implementing agent is public-only and inside a dev VM. It can build every configuration, boot nested microVMs, and assert on generated artifacts. It cannot:

- Run any command on the hypervisor host — provisioning, `virsh`, microvm host units, or a host rebuild. `virsh` and `virt-install` are not installed in a dev VM by design.
- Determine the real host's QEMU version or TPM presence. Both are inputs the private plan needs; neither blocks this plan, because contracts 1–10 are host-version-independent on the pinned nixpkgs.
- Generate, re-encrypt, or place any real key material.
- Touch the operator's SSH configuration or `known_hosts` pinning, which live in the private encrypted home configuration.
- Provision or migrate any real machine.

What this blocks: nothing in this plan. What it defers: proving the mechanism against the real host, which is the private plan's first acceptance test.

## Acceptance Tests

Nested microVM boots are the primary evidence, because a happy-path build passing while first boot strands the machine is the failure this stack has already paid for once.

**Delivery path, positive:**

1. Build a microvm guest with one `credentialFiles` entry holding a recognizable canary. Boot it. Assert the console shows `Received regular credentials: <name>` from PID 1 and the canary round-trips byte-exact through a unit consuming it with `ImportCredential=`.
2. Assert `/run/credentials/@system/<name>` is mode 0400 and its parent is 0700, on a tmpfs mounted `noswap`.

**Delivery path, negative — each must fail, and be shown to fail:**

3. The same configuration with `boot.initrd.kernelModules` lacking `qemu_fw_cfg`: assert the credential is *not* readable during the activation script, and that whatever consumes it says so loudly rather than proceeding.
4. The same configuration with `qemu_fw_cfg` blacklisted: assert the build refuses, per contract 4.
5. A `credentialFiles` value given as a Nix path literal: assert the store-prefix assertion trips.
6. A `credentialFiles` key of 29 characters: assert the length assertion trips.
7. A non-QEMU hypervisor with non-empty `credentialFiles`: assert evaluation fails.

**Generated-artifact assertions:**

8. The generated runner script contains no credential path under the store prefix.
9. The runner's `bin/` contains no `virtiofsd-run`.
10. `config.age.secrets` is empty and `age.identityPaths` is unset for every microvm archetype.
11. No private-key or age-key material appears anywhere in the built closure.

**Validating the validators.** Tests 4–7 are mutation fixtures and must be shown to fail on sabotaged input, following the pattern already established in `allod/archetypes` by the mutated-registry checks and the `vmFacts` negative check, both of which build deliberately broken inputs and assert each mutation errors. A check that cannot be demonstrated to fail does not count.

## Rollback Plan

Revert the framework commits in reverse dependency order. The libvirt provisioning path is untouched by this arc and remains a pure function of the flake lock, so a revert restores the previous behavior exactly.

Partial states are benign by construction. Because the first microvm guest is provisioned *alongside* the libvirt path rather than replacing it, an abandoned arc leaves an unused module and an unused inventory value, and every existing machine continues to provision as before. A microvm guest that booted without an age identity holds no in-guest secret material; its recovery is destructive replacement, which is the intended lifecycle for a disposable machine.

The sequencing constraint is the one thing a revert must respect: `allod/archetypes` pins `allod/vm` as a flake input, so the consumer-side change lands before the input bump, and a revert unwinds the bump first.
