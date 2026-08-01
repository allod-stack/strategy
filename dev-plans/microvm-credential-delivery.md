# microVM Runtime Credential Delivery

## Tracking Issue

[allod/archetypes#29: Deliver microvm guest credentials at boot from host memory, not agenix](https://forge.anarch.diy/allod/archetypes/issues/29)

Third slice of milestone 4 in [microvm-framework-adoption.md](microvm-framework-adoption.md), tracking [allod/strategy#20](https://forge.anarch.diy/allod/strategy/issues/20). It implements that plan's **contract 7**, the guest half of **8a**, and contracts **9, 10, 11, 12, 14**, and inherits 2, 6, 13 and 21 as constraints. One PR in `allod/archetypes` carrying `Closes allod/archetypes#29` and `Refs allod/strategy#20`; it does not close allod/strategy#20.

## Goal

A machine that selects the microvm runtime receives its private key material from the host at every boot, holds no age identity, writes no credential-derived plaintext outside one runtime root, and fails visibly instead of minting a replacement SSH identity when a credential does not arrive.

## Scope

In scope, all in `allod/archetypes`:

- `allod.microvm.guestCredentialRoot`: a typed, validated, microvm-only option (default `/run/allod/credentials`) that every consumer path derives from.
- A closed credential-name set per machine, derived from the credentials that machine actually declares.
- A materializer systemd service that loads the delivered system credentials, validates each one, and writes the consumer files atomically with declared owner and mode.
- Microvm-only rewiring of six consumers: sshd's `HostKey`, the Nix netrc path, root's Git credential store, the user's Git credential store, the Forge API token path, and the Forge SSH `IdentityFile` in the generated Home Manager output.
- Microvm-only removal of agenix wiring: no `age.secrets` definition, no `age.identityPaths` definition, no `modules/netrc.nix`.
- Assertions over the merged configuration for every rule above, each with a paired sabotage fixture pinned to the diagnostic it names.
- Generated-artifact checks: the rendered materializer unit, its ordering edges and the `Requires=` edge sshd carries, the rendered `sshd_config`, the rendered `nix.conf`, the generated `gitconfig`, the generated Home Manager `programs.ssh` result, and a closure scan of a microvm guest.
- Whatever rework the existing `credential-profiles`, `netrc-activation` and `dev-forge-opt-out` checks need to stay true, which is to become explicitly libvirt-scoped rather than accidentally so.

Out of scope, other slices of milestone 4: contract 8b's `extendModules` host integration that supplies the actual `microvm.credentialFiles` *values*; contract 15's guest networking and `microvm.socket`; contract 1's `vmFacts.<name>.runtime`; contract 18's runtime-dispatched rotation in `allod/nexus`; and the nested-boot lifecycle tests.

Not in scope anywhere in this repo: the host-side preparation, mount, principal and namespace. `allod/nexus` `nix/microvm/host.nix` and `nix/microvm/launcher.nix` own all of it and are already merged.

**Explicitly not changed: `allod/profiles`.** Its `hosts/dev/allod-dev/home.nix:13` sets `identityFile = "~/.ssh/${identity.sshKeyName}"`, a persistent-mount path, for both runtimes. The parent plan's scope gives profiles its own line, and a second repo in this PR would put the diff across a boundary that the drvPath no-op cannot cover. This slice therefore *overrides* that value for microvm from the archetypes side with `lib.mkForce`, and proves the override in the generated Home Manager result. The profiles edit that makes the override unnecessary is a later, separable change; until it lands, a microvm guest's Forge identity path is correct because archetypes forces it, not because profiles agrees.

## Risk Assessment

Residual risk: **R3 High**, and it does not come down with the validation in this plan.

This is a security boundary change: it moves where a VM's private key material comes from and where its plaintext lands. The parent plan scores the archetypes guest integration R3 for exactly this row. Three properties keep the *blast radius* small and none of them lower the score:

- Every libvirt machine is untouched, proved by four unchanged derivation paths.
- No public machine selects microvm, so the new path's coverage is fixtures only.
- Rollback is a straight revert of one PR in one repo, and nothing here creates, moves or deletes key material.

What keeps it R3 is what the validation cannot reach. Nothing in this slice boots a guest. Every claim about credential *receipt* — that PID 1 imports the fw_cfg credentials, that `LoadCredential=` finds them, that the materializer runs early enough, that sshd starts with a `HostKey` under `/run` — is an inference from pinned upstream source, not a measurement. The parent plan's acceptance test 9 and the nested-boot slice are what settle those, and this PR must not be read as evidence for them.

**The gate:** no machine is enabled on the microvm runtime on the strength of this change. A guest whose materializer is subtly mis-ordered comes up with sshd failed and no way in; a guest whose materializer is mis-ordered the *other* way comes up with a working sshd and a stale or absent credential, which is worse because it looks fine. Neither is detectable from this repo's checks.

**The worst credible failure after this plan's validation passes** is a microvm guest that evaluates and builds cleanly but, on a real boot, either strands with sshd failed, or starts sshd against a credential the materializer wrote with the wrong owner or mode. Both are contained to a machine nothing depends on, per the parent plan's rule that the first microvm machine is purpose-made. Neither can affect a libvirt machine, a host, or any encrypted source.

Human scrutiny, in order:

1. The rendered materializer unit: each `LoadCredential=` source path, its `Before=`/`After=`/`RequiredBy=` edges, the `Requires=` edge sshd carries, and the exact `install`/`mv -T` invocations in its script.
2. The rendered `sshd_config` for a microvm fixture: exactly one `HostKey` line, pointing under the credential root, and `sshd-keygen.service` rendered as a mask.
3. The rendered `nix.conf`: exactly one `netrc-file =` line.
4. The four libvirt derivation paths.

## Contract contradictions found

Two, both recorded rather than worked around.

1. **Option namespace.** Contract 8a names the guest option `allod.microvm.guestCredentialRoot`. The volumes slice that landed after contract 8a was written chose `allod.archetypes.microvm.volumeImageRoot`, with a comment at `flake.nix:244-246` arguing the namespace should name the owning repo. Both are archetypes-owned options on the same configuration, so the repo now has two conventions. This plan follows contract 8a literally, because the contract names the exact string and nothing in the code depends on the other spelling. The reconciliation — most likely renaming the volumes option — is a separate cosmetic change, not this one.

2. **Contract 9 lists `/etc` as forbidden, and `/etc` on a microvm guest is tmpfs.** The microvm root is the upstream tmpfs root (contract 4), so `/etc/nix/netrc` on a microvm guest is no more durable than `/run/allod/credentials/netrc`. The contract is still followed literally: the value of one runtime root is that it is one *scannable* root, so a check can enumerate credential-derived plaintext by looking in exactly one place. A second location that happens to be non-durable defeats that even though it leaks nothing. No change requested.

## Interface Contracts

Inherited and not restated: contracts 2, 6, 13, 21.

### 1. The credential root is a validated, microvm-only option, gated outside the module system

`allod.microvm.guestCredentialRoot`, declared only by a module the builders append with `lib.optional (runtime == "microvm")`, exactly as `microvmVolumesModule` is appended at `flake.nix:485-488` and `517-518`. `lib.mkIf` does not help: it defers the value, not the definition, and every option this slice writes to under microvm either does not exist under `qemuGuest` (`microvm.*`) or would silently render wrong content for a libvirt machine (`systemd.services`, `services.openssh.extraConfig`, `environment.etc`).

Type is `lib.types.either lib.types.str lib.types.path`, for the reason stated at `flake.nix:248-252`: a bare `str` makes a Nix path fail with nixpkgs' generic type error while the option tree is built, pinnable to nothing, whereas accepting it lets a named assertion be the reporter. This deliberately diverges from `nexus.microvm.hostPlaintextRoot`'s `types.strMatching` (`nix/microvm/host.nix:321`), because acceptance test rule "every assertion has a paired sabotage fixture pinned to its diagnostic" cannot be satisfied by a type-check failure.

**The existing `pathErrors` cannot be reused whole, and reusing it whole is an evaluation failure on the default value.** Its sixth rule (`flake.nix:179-180`) rejects any path equal to or under `/run/allod`, because a volume image on the host's tmpfs runtime root cannot hold persistent state. This option's default *is* `/run/allod/credentials`, and contract 8b's supplied values are host paths under `/run/allod/microvm`. Reused unchanged, that rule rejects every correct value this slice produces.

So split it. Hoist only the five generic rules into a shared validator: the value is a plain string and not a Nix path, is absolute, has no empty segment (no trailing or doubled slash), has no `.` or `..` segment, and is not under the store. Those five are what the volume image root, the credential root and every supplied `microvm.credentialFiles` value genuinely share.

The `/run/allod` prohibition stays inside `microvmVolumesModule`, where it is one `lib.optional` on top of the shared list rather than a shared rule with an exception.

The credential root then adds the *opposite* rule, and only it has it: the value must be strictly below `/run/allod` — neither `/run/allod` itself nor anything outside it. Contract 8a requires that, and `allod/nexus` states the same shape for its host-side root as `types.strMatching "^/run/allod/[^/]+(/[^/]+)*$"` (`nix/microvm/host.nix:321`).

Supplied `microvm.credentialFiles` values get the generic rules plus one of their own (contract 3), and never the volume rule.

Hoisting the generic rules out of `microvmVolumesModule` to a shared `let` binding is in scope and is the point: two copies of a five-rule path validator in one file is the drift this contract exists to prevent. Copying the sixth rule along with them is the drift it would cause.

### 2. The credential-name set is closed and derived, never restated

The builder derives it from what the machine has, not from a literal list:

| Name | Length | Declared when | What the host supplies |
|---|---|---|---|
| `ssh-host-key` | 12 | always | the machine's ed25519 SSH host private key |
| `forge-token` | 11 | dev, `tokenFile != null` | the Forgejo REST API token |
| `forge-https-token` | 17 | dev, `httpsTokenFile != null` | one `https://user:pass@host` credential-store line |
| `forge-ssh-key` | 13 | dev, `machines.<name>.forge_key != null` | the Forge SSH private key |

A privacy microvm declares exactly `[ "ssh-host-key" ]`. A dev microvm with `forgeAccess = false` and no forge key declares the same. This is what keeps `allod/archetypes#17`'s opt-out meaning one thing: the three Forge names are gated on precisely the values that already gate `age.secrets.forgejo-https-token` (`flake.nix:465`) and `modules/agent-forgejo-token.nix` (`flake.nix:480`). A machine that opts out composes no Forge credential name, so the host is never asked to supply one.

Expose the set as a read-only internal option, `allod.microvm.credentialNames`, so the check reads a merged result rather than recomputing the builder's conditional. That is contract 4 of the runtime-selection slice applied here: a check that restates the generator proves nothing.

`machines.<name>.forge_key` is the same fact `vmFacts.<name>.forgeKey` already reads (`nix/vm-facts.nix:84`). Read it from `machines`, not from `vmFacts`, because `vmFacts` filters to non-hypervisors through its own trip-wire and the builders already read `machines.<name>` for `platform` and `runtime`.

### 3. Contract 7's rules apply to the declared set and to any supplied values

Assertions, all over the merged configuration:

- Every declared name is non-empty and at most 28 characters. Contract 7's bound is measured, not asserted by prose: pinned QEMU 10.1.5 `include/standard-headers/linux/qemu_fw_cfg.h:49-50` defines `FW_CFG_MAX_FILE_PATH 56`, and `system/vl.c:1163-1167` rejects `strlen(name) > FW_CFG_MAX_FILE_PATH - 1`. The name QEMU sees is the whole `opt/io.systemd.credentials/<cred>` string that `lib/runners/qemu.nix:156` builds; the prefix is 27 bytes, so `55 - 27 = 28`. A 29-character name makes QEMU exit at startup with `name too long (max. 55 char)` and `Restart=always` turns that into a restart loop with no framework diagnostic — the assertion is the only thing that reports it.
- **Names match `[A-Za-z0-9_-]+` and contain no `.`.** Two separate reasons, both measured. systemd's `credential_name_valid` (`src/shared/creds-util.c:49-53`) requires `filename_is_valid` and `fdname_is_valid`, so `/`, `:`, control bytes, `.` and `..` are out; a name failing it is **silently dropped** with only a `log_warning` in the guest journal, which is the worst possible failure shape. Independently, `,` and `=` would mis-split QEMU's own `name=…,file=…` option string. Forbidding `.` outright is what makes the rule cheap *and* closes the reserved-name hazard: every system credential systemd itself acts on is dotted — `ssh.authorized_keys.root`, `passwd.hashed-password.root`, `system.machine_id`, `tmpfiles.extra`, `sysusers.extra`, `fstab.extra`, `systemd.extra-unit.*`, `network.*`, `udev.rules.*` — and a host that supplied one of those names would silently reconfigure the guest rather than deliver a file the materializer reads. A dotless closed set cannot collide with any of them.
- The two name rules above apply to the **union** of the declared set and the keys of a non-empty `config.microvm.credentialFiles`, not to the declared set alone. The declared set is four literals in one builder and a read-only option, so a violation of it is a code change and no fixture can reach one; the supplied map is the untrusted half and is where a bad name actually arrives. Applying the rules to both is one `lib.unique` and is what gives every name rule a reachable sabotage.
- `config.microvm.credentialFiles` is either `{}` or has exactly the declared name set as its keys. `{}` is the standalone shape contract 8b requires; anything else must agree with the guest's own declaration, and a superset or subset fails.
- Every value in a non-empty `config.microvm.credentialFiles` is a plain string (not a Nix path), absolute, normalized, not under `builtins.storeDir`, **and contains no comma**. `microvm.credentialFiles` is typed `attrsOf path` upstream (`nixos-modules/microvm/options.nix:1070-1071`), which accepts a Nix path literal and would copy it into the world-readable store on interpolation at `lib/runners/qemu.nix:156`. The shared generic validator covers the first four. The comma rule is this option's own, for the same reason the name rules forbid one: `qemu.nix:156` interpolates the value straight into QEMU's comma-delimited `name=opt/io.systemd.credentials/<name>,file=<path>` string with no escaping, so one comma splits it into two options and QEMU exits at startup under `Restart=always`. `allod/nexus`' `hostPlaintextRoot` regex (`nix/microvm/host.nix:321`) permits commas in every segment, so a host-integrated configuration can pass every host-side check and still produce this; nexus rejecting it at the root is recorded for the 8b slice, and the guest asserting it is not redundant with a check that does not exist yet.

One value rule has no reachable fixture and stays anyway. `types.path` is `pathWith { absolute = true; }` (nixpkgs `lib/types.nix:671-673`, check at `:710-729`), so a *relative* value is rejected by the option's own type with nixpkgs' generic error before any assertion runs. The absolute rule stays in the shared validator because the credential root — typed to accept anything string-like precisely so a named assertion can report it — does need it. The value-side sabotage simply carries no relative case, and the plan does not claim one.

Values are only ever supplied by fixtures in this slice; contract 8b is where a real host supplies them. That is not a reason to defer the value rules — the assertion is the guest's half of contract 17 and the fixtures are how it is proved.

One fact for the 8b slice, recorded here because this is where it was measured: the machine component of the required value is the `microvm.vms.<name>` **attribute name**, which is also the systemd instance name, not `networking.hostName`. `allod/nexus` `nix/microvm/host.nix:249` compares against `"${vm.credentialDirectory}/${cred}"` derived from that attribute name, and the two coincide only by convention in today's fixtures. This slice's guest-side assertions never build that string, so nothing here depends on it.

### 4. One runtime root, and the consumer table is the contract

Every path below is `<root>/<name>` with `<root>` the option value. Nothing restates a literal.

| File | Owner | Mode | Consumed by |
|---|---|---|---|
| `<root>/ssh-host-key` | `root:root` | `0600` | sshd's `HostKey` |
| `<root>/forge-token` | `<user>:users` | `0600` | the `forge` CLI via `FORGE_TOKEN_FILE` |
| `<root>/netrc` | `root:users` | `0640` | the Nix daemon, root's git, the user's git |
| `<root>/git-credentials` | `root:users` | `0640` | `credential.helper = store --file=…` for root and the user |
| `<root>/forge-ssh-key` | `<user>:users` | `0600` | `programs.ssh` `IdentityFile` for the Forge host |

`<root>` itself is `0711 root:root`: traversable so the user can open its own files by exact path, not listable, so the set of credentials a guest holds is not enumerable by an unprivileged process.

Three notes on why this table is shorter than the libvirt one it replaces.

- **One netrc, not three.** Libvirt writes `/etc/nix/netrc`, `/root/.netrc` and `/home/<user>/.netrc` because three different readers need three different owners on a disk-installed guest. Under one `0640 root:users` file on tmpfs, the Nix daemon, root and the dev user all read the same bytes. Contract 9's "user netrc path if still needed" is answered: it is not.
- **`git-credentials` is `0640 root:users`, and that is not a new disclosure.** Today the dev user already holds the same Forge HTTPS password in `/home/<user>/.netrc` mode `0600`. Making the credential-store file group-readable changes no posture; it removes a second copy.
- **`forge-https-token` is delivered once and derived twice.** The host supplies one credential-store line; the materializer derives both the netrc and the credential-store file from it, exactly as `modules/netrc.nix:31-37` derives netrc from `/root/.git-credentials` today. Two host credentials for one secret would be a second inventory of the same fact.

### 5. Contract 14 is a property of the table, and is asserted as one

No path in the table above may be equal to, or nested under, any `mountPoint` in the merged `config.microvm.volumes`. Assert it over the merged volumes list rather than over the required set the volumes module declares, so a profile that adds a volume at `/run/allod` is caught. Assert it over the whole table, not over the root alone: a root outside the volumes with one consumer path re-anchored elsewhere is the failure this is for.

### 6. The materializer is a boot service with no optional branch

One `systemd.services.allod-microvm-credentials` oneshot with `RemainAfterExit = true`.

**Where the credential already is by the time this unit runs.** Measured on pinned systemd 258.7. PID 1 calls `kmod_setup()` (`src/core/main.c:3186`), which explicitly loads `qemu_fw_cfg` under KVM/QEMU with the comment *"would be loaded by udev later, but we want to import credentials from it super early"* (`src/core/kmod-setup.c:144-145`), and only then `initialize_runtime()` → `import_credentials()` (`main.c:3313`, `main.c:2453`). That reads `/sys/firmware/qemu_fw_cfg/by_name/opt/io.systemd.credentials` (`src/core/import-creds.c:375`) into `/run/credentials/@system` (`src/shared/creds-util.h:33`), files `0400 root:root` in a `0700 root:root` non-swappable tmpfs, then remounts it read-only.

This happens in the **initrd's** PID 1, because these guests run systemd-in-initrd: `microvm.optimize.enable` defaults true and `nixos-modules/microvm/optimization.nix:16-31` sets `boot.initrd.systemd.enable` for QEMU. It then survives switch-root — `src/shared/switch-root.c:38` carries `{ "/run/credentials", MS_BIND|MS_REC, 0 /* skip! */ }, /* Credential mounts should survive */` — and the real-root manager re-execs with `execv`, inheriting `$CREDENTIALS_DIRECTORY`, so `import_credentials()` short-circuits rather than re-importing.

**This corrects why contract 5 matters.** `boot.initrd.kernelModules = [ "qemu_fw_cfg" ]` is load-bearing because it puts the `.ko` in the initrd's `modulesClosure` at all — `qemu_fw_cfg` is a module, not built in, at kernel 6.12.93. Without it systemd's own early modprobe fails, `/sys/firmware/qemu_fw_cfg` never appears, and `import_credentials_qemu()` logs `"No credentials passed via fw_cfg."` and returns non-fatally. It is *not* about ordering against `systemd-modules-load.service`, which runs far too late to matter for this.

- **Input.** `LoadCredential = "<name>:/run/credentials/@system/<name>"` for each declared name. **The absolute-path form is deliberate and is the first fail-closed gate.** `systemd.exec(5)` makes a bare `LoadCredential=<name>` **non-fatal** when the credential is absent — the unit starts and the file simply is not there — whereas an absolute source path that cannot be read fails the unit with `EXIT_CREDENTIALS`. `ImportCredential=` is likewise soft-fail. So the bare form would let the materializer run to completion having written nothing, and sshd would fail two units away from the cause. The script then reads `$CREDENTIALS_DIRECTORY/<name>`, which for this system service is `/run/credentials/allod-microvm-credentials.service/<name>`, mode `0400`.
- **Nothing in the guest references a host path.** `microvm.credentialFiles` values exist only in QEMU's argv on the host; no guest-side code may name `/run/allod/microvm/...`.
- **Script validation is the second gate, and is still mandatory.** systemd's gate proves the bytes arrived; it says nothing about their shape. The host launcher additionally refuses a missing, non-regular or empty *source* before QEMU starts (`allod/nexus` `nix/microvm/launcher.nix:285-287`), but the guest cannot rely on that either: upstream's `microvm -r <name>` execs the runner with no unit and no preparation, which `allod/nexus` `docs/microvm-host.md:44-48` documents as unsupported but reachable.
- **Validation, per credential, before any write.** The file exists, is a regular file, and is non-empty. `ssh-host-key` parses as an OpenSSH private key (`ssh-keygen -y -f` succeeds). `forge-https-token` yields exactly one non-empty line matching the `https://user:pass@host` shape — reuse `modules/netrc.nix:31-37`'s `sed` expression, which already encodes it. `forge-ssh-key` parses as an OpenSSH private key. `forge-token` is a single non-empty line with no whitespace.
- **No skip branch.** Any failure is `exit 1` with a message naming the credential. There is no counterpart to `modules/netrc.nix:18`'s `"missing or empty, skipping (expected during provisioning)"`, and the check asserts that string is *absent* from the microvm materializer while remaining present in the libvirt activation script.
- **Atomic writes, and `install` alone is not one.** `install(1)` truncates the destination in place; it does not write-and-rename. The idiom is `install -m <mode> -o <owner> -g <group> <src> <dir>/.<name>.tmp && mv -T <dir>/.<name>.tmp <dir>/<name>` — the temporary must be in the destination directory so the rename is a same-filesystem `rename(2)`. `allod/nexus` `nix/microvm/launcher.nix:290-299` uses exactly this shape for the host side; follow it rather than the weaker `modules/netrc.nix:39-44` form, which is a plain in-place `install`.
- **Ordering: `Before=sysinit.target` with `DefaultDependencies=no`.** This is the single ordering that precedes *every* normal service, every socket unit, `systemd-user-sessions.service`, `user@.service` and `sshd.service`, because systemd gives all of them a default `After=sysinit.target` (`systemd.service(5)`, `systemd.socket(5)`). Enumerating consumers instead — `before = [ "sshd.service" "nix-daemon.service" … ]` — is a list that goes stale the moment a profile adds a reader, which on a machine whose whole point is composed profiles is not a hypothetical. The skeleton is the one `nixos/modules/security/wrappers/default.nix:305-320` already uses for the same problem:

  ```nix
  wantedBy = [ "sysinit.target" ];
  before = [ "sysinit.target" "shutdown.target" ];
  conflicts = [ "shutdown.target" ];
  unitConfig.DefaultDependencies = false;
  serviceConfig.Type = "oneshot";
  serviceConfig.RemainAfterExit = true;
  ```

  Keep an explicit `before = [ "sshd.service" ]` on top: it costs one list entry and survives someone later setting `DefaultDependencies=no` on sshd. Add `requiredBy = [ "sysinit-reactivation.target" ]` so `switch-to-configuration` re-runs it on a rebuild rather than leaving stale files, as `nixos/modules/services/system/userborn.nix:113-125` does.

  Additionally `systemd.services.sshd.requires` gains the materializer, so a failed materializer leaves sshd unstarted with a dependency failure rather than started against a missing file — contract 11's stated outcome, made a real edge rather than a coincidence.

- **Not `systemd-tmpfiles`.** The `f^` line type reads a credential into a file with declared owner and mode and is the idiomatic pinned-nixpkgs route — `nixos/modules/services/networking/ssh/sshd.nix:692-709` uses it for root's `authorized_keys`. It is rejected here for three concrete reasons, recorded so a later SIMPLIFY sweep does not reopen it: `systemd-tmpfiles-setup.service` imports a fixed credential allowlist (`ImportCredential=tmpfiles.*`, `ssh.authorized_keys.root`, …), so any other name is *silently skipped* unless a drop-in is added in three separate units nixpkgs keeps in sync; `f` never rewrites an existing file, and `f+` truncates in place rather than renaming; and the tmpfiles specifier table has no `%d`, so the `^` modifier is the only route and it cannot express per-credential validation at all.
- **The directory.** The script creates `<root>` itself with the declared mode. A `systemd.tmpfiles` rule would be a second declaration of the same path in a different language, ordered by a different unit.

### 7. sshd is pointed at the delivered key, and cannot mint one

Measured against pinned nixpkgs `nixos/modules/services/networking/ssh/sshd.nix`:

- Line 846 builds the `HostKey` directives solely from `cfg.hostKeys`. With `hostKeys = []` no `HostKey` line is rendered, and sshd then falls back to its **compiled-in** defaults. Measured directly against the pinned binary: `sshd -G -T -C lport=22 -f /dev/null` reports `hostkey /etc/ssh/ssh_host_rsa_key`, `…ssh_host_ecdsa_key`, `…ssh_host_ed25519_key`. So `hostKeys = []` on its own is not a safe state — it silently re-adopts any stale key sitting at one of those three paths, and exits with `no hostkeys available` if none does. An explicit `HostKey` directive is what discards the defaults, and it is mandatory, not merely convenient.
- Lines 779-802 define `services.sshd-keygen` with `ConditionFileNotEmpty = map (k: "|!${k.path}") cfg.hostKeys` and `script = concatMapStrings … cfg.hostKeys`. With `hostKeys = []` both collapse: no condition line is emitted, and `script = ""` means nixpkgs emits no `ExecStart` at all (`nixos/lib/systemd-unit-options.nix:501-508`). systemd then **refuses to load** the unit — `src/core/service.c:686-691`, `"Service has no ExecStart=, ExecStop=, or SuccessAction=. Refusing."` Because both sshd units only `Wants=` it (not `Requires=`), sshd still starts, so nothing is minted; but the guest boots with a load error in its journal on every boot. Mask it instead: `systemd.services.sshd-keygen.enable = false` renders a `/dev/null` symlink (`nixos/lib/systemd-lib.nix:74,99`), which is a real mask, is quiet, and is a stronger statement than an empty script.
- Keeping `hostKeys` pointed at the runtime path instead of emptying it is the option that must not be taken. The `ConditionFileNotEmpty=|!<path>` guard skips the unit only while the file is non-empty, so the first boot where the credential fails to arrive is exactly the boot where `sshd-keygen` mints a fresh ed25519 key at the runtime path and sshd comes up with an unknown identity — silently. That is the precise failure contract 11 exists to forbid.
- Line 826 defines the module's own `extraConfig` contribution with `lib.mkOrder 0`, so a plain `services.openssh.extraConfig` definition is appended after it. `services.openssh.settings.HostKey` also works for a single key, but throws on a list (`sshd.nix:69-71`), so `extraConfig` is the form that stays correct if a second key is ever added.

So for microvm: `services.openssh.hostKeys` defined as `[]` lexically inside `sharedModules` (not `mkForce`; the definition is the builder's own and is simply omitted for microvm), plus `services.openssh.extraConfig = "HostKey <root>/ssh-host-key"`, plus `systemd.services.sshd-keygen.enable = false`.

sshd refuses a host key file that is group- or world-accessible; the table's `0600 root:root` satisfies it. The public half is not delivered: sshd derives it from the private key, and the pinned module's build-time `sshd -G -T` check does not open host key files — proved today by `/etc/ssh/<name>`, which does not exist at build time either. **That same check also exits 0 with no host key configured at all**, so it is not a guard against the empty-`HostKey` state above; acceptance test 16 is the only thing that is.

### 8. No age, and it costs one omission rather than one removal

`agenix.nixosModules.default` stays in `sharedModules` for microvm. Its entire `config` is `mkIf (cfg.secrets != {})` (agenix `modules/age.nix:271`), including the `identityPaths != []` assertion at line 275, so a guest that declares no secrets composes no activation script, no systemd unit, and no assertion. Keeping the module imported keeps `config.age.secrets` a *declared* option, which is what lets the check assert `== {}` instead of probing for an absent attribute, and keeps `credential-profiles`' `machineConfigurations.<m>.config.age.secrets` read total.

`age.identityPaths` is not defined for microvm. Its default (agenix `modules/age.nix:241-254`) is derived from `config.services.openssh.hostKeys`, which contract 7 above makes `[]`, so the option resolves to `[]` with no definition of ours. Assert `config.age.identityPaths == []`.

What the builder omits for microvm: the `age.secrets.forgejo-https-token` attrset (`flake.nix:465-473`), `modules/agent-forgejo-token.nix` (`flake.nix:480`), `modules/netrc.nix` (`flake.nix:490`), and the `services.openssh.hostKeys` / `age.identityPaths` pair in `sharedModules` (`flake.nix:328-329`).

`modules/github-credentials.nix` (`flake.nix:491`) stays composed, and a microvm machine with a non-empty `secrets.lib.githubCredentialTargets.<name>` **fails evaluation** with a named diagnostic pointing at the follow-up. The public template declares none (`allod/secrets` `flake.nix:85`), so deriving credential names for them now would be untestable speculative generality; refusing them is honest, is one assertion, and has a fixture that injects a target.

### 9. The closure carries no key material

A generated check over the microvm guest's `system.build.toplevel` closure asserting it contains no `.age` file, no age or agenix identity, and no private key. The realistic failure this catches is a Nix path literal reaching an option that interpolates it — `microvm.credentialFiles` being `attrsOf path` is exactly that hazard, and `secrets.lib.*TokenFile` values are Nix paths into the `secrets` store path today.

Scan the *closure*, not the toplevel directory: an `age.secrets.<n>.file` reference lands as a store path in the activation script, whose closure includes the ciphertext. Compare against a libvirt dev machine, which must still contain them, so the scan cannot pass vacuously.

### 10. The libvirt path is byte-identical

Every change is gated on `runtime == "microvm"` outside the module system, so no libvirt machine's module list changes. Proved by derivation-path equality across the implementing commit, not by inspection.

## Agent Gates

None for implementation. Every acceptance test runs locally without host access or real credentials.

One gate on *consumption*: this change does not license enabling a machine on the microvm runtime. See the gate in Risk Assessment.

## Acceptance Tests

### The libvirt no-op

Taken across the implementing commit:

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

### Memory budget

`runtime-module-selection` already states its own constraint at `flake.nix:1543-1554`: sabotage fixtures are consumed as thunks one at a time because holding them live is a measured OOM kill, and they are most of what takes `nix flake check` from a ~4 GB peak to over 5 on a 7 GiB box with no swap. This slice adds more fixtures than the volumes slice did, so the budget is a design constraint, not a footnote:

- Every new fixture is a thunk consumed exactly once by `checkSabotage`.
- Drive every archetype-independent rule from the **privacy** builder, which composes no dev user environment. That covers the root validation, the name-length and name-shape rules, the `credentialFiles` value rules, and the contract 14 overlap rule.
- Reuse `asMicrovm` for every positive assertion rather than building a second dev fixture.
- Measure with `\time -v nix flake check` and record maximum resident set size in the PR body. If the peak exceeds the current measurement by more than ~15%, convert positive assertions that only read projections to share one fixture before adding anything.

### Contract assertions, each with a paired sabotage fixture

Every case below is `{ name; fixture = _: …; needles = [ … ]; }` in `runtime-module-selection`'s existing `sabotages` list, pinned through `missingDiagnostic` against `config.assertions`, and verified by deletion: removing the assertion must make the fixture stop failing for its named reason.

Positive, on `asMicrovm` and `asPrivacyMicrovm`:

1. A dev microvm's `config.allod.microvm.credentialNames` is exactly the four names, sorted; a privacy microvm's is exactly `[ "ssh-host-key" ]`.
2. A dev microvm with `agentTokenFile = null`, `forgeTokenFile = null` declares exactly `[ "ssh-host-key" ]`. This is the `forgeAccess = false` shape, driven through the same `withTokens` helper `dev-forge-opt-out` already uses at `flake.nix:1105-1110`.
3. `config.age.secrets == {}` and `config.age.identityPaths == []` on both microvm fixtures.
4. `config.services.openssh.hostKeys == []` on both.
5. A non-default `allod.microvm.guestCredentialRoot` moves every entry in the consumer table, the materializer's script, the `HostKey` line, the `netrc-file` line, and the Home Manager `IdentityFile`. Restate the default literally in the check, per `flake.nix:1396-1399`.

Sabotage, each pinned to its own diagnostic:

6. Root given as a Nix path, a relative path, a store-backed path, a non-normalized path, a path with a trailing slash, `/run/allod` itself, and a path outside `/run/allod`. Seven cases, driven from the privacy builder, every needle anchored to the option name for the reason at `flake.nix:1402-1409`.
7. A 29-character credential name; an empty credential name; a name containing `/`; a name containing `.`; a name containing `:`; two names colliding after normalization. The `.` case is the reserved-system-credential guard and its diagnostic must name that reason, not just "invalid character".
8. `microvm.credentialFiles` forced to a map with one extra key; with one key missing; with a Nix path value; with a store-prefixed value; with a relative value.
9. A `microvm.volumes` entry forced to `mountPoint = "/run/allod"`, so a consumer path is nested under a persistent mount.
10. `age.secrets` forced non-empty on a microvm fixture; `age.identityPaths` forced non-empty.
11. `services.openssh.hostKeys` forced non-empty on a microvm fixture.
12. A microvm machine with a non-empty `githubCredentialTargets` entry injected.

### Generated-artifact checks

These are the tests this slice exists for, and none of them may be replaced by an option read.

13. **The rendered materializer unit.** Build `config.systemd.units."allod-microvm-credentials.service".unit` and read the fragment file. Assert one `LoadCredential=` per declared name, each carrying the absolute `/run/credentials/@system/<name>` source rather than a bare name; `DefaultDependencies=no`; `Before=` naming `sysinit.target` and `sshd.service`; `RemainAfterExit=yes`. Paired sabotages: a fixture that drops the `before` list loses those needles, and a fixture that uses the bare `LoadCredential=<name>` form loses the absolute-path needle — the second is what defends the soft-fail distinction, which is invisible in the option value.
14. **sshd's real dependency edge.** In the rendered `sshd.service`, assert `Requires=` names the materializer. Paired sabotage: dropping the `requires` definition loses it.
15. **The materializer script has no optional branch.** Write the script to a store file and assert `missing or empty, skipping` is absent, and that each declared credential name appears in a validation command. Assert the write idiom is a rename, not a bare `install` — the needle is `mv -T`, because `install` alone truncates in place and would leave a window where a consumer reads a partial key. Paired positive control: the `missing or empty, skipping` needle must still be *present* in `machineConfigurations.allod-dev`'s `system.activationScripts.netrc.text`, which is what the existing `netrc-activation` check reads, so the two checks together prove the branch was removed from one path and kept on the other.
16. **The rendered `sshd_config`.** Read `config.environment.etc."ssh/sshd_config".source` for a microvm fixture. Assert exactly one `HostKey` line and that it points under the credential root. Paired sabotage: a fixture whose `extraConfig` is forced empty must have zero `HostKey` lines. This is the *only* guard against the compiled-in-default fallback: pinned `sshd -G -T` exits 0 with no host key configured, so nixpkgs' own build-time config check cannot catch it.
17. **`sshd-keygen` is masked** for a microvm fixture — its rendered unit is a symlink to `/dev/null` — and is a real unit with a non-empty script for `allod-dev`. This is the anti-regeneration proof; the option read `hostKeys == []` is not, because an empty `hostKeys` also produces an unloadable unit rather than a masked one, and the two are indistinguishable from the option value.
18. **The rendered `nix.conf`.** Read the built `/etc/nix/nix.conf` source. Assert exactly one `netrc-file =` line and that it points under the credential root. `nix.extraOptions` is `types.lines` and `allod/vm` `modules/guest-base.nix:65` already defines one, so the microvm module must `lib.mkForce` it; a second line would be a silent last-wins. The "exactly one" count is what defends against a future `allod/vm` change adding another `extraOptions` definition that `mkForce` would silently swallow.
19. **The rendered `gitconfig`.** Assert exactly one `helper =` line for microvm, pointing at `<root>/git-credentials`. Same `mkForce`-on-`lines` hazard as above, and git treats `credential.helper` as multi-valued, so two lines would run two helpers rather than the later one winning.
20. **The generated Home Manager result.** Read the microvm fixture's `home-manager.users.<user>.programs.ssh` merged config and assert the Forge host's `identityFile` is `<root>/forge-ssh-key`. Paired sabotage: dropping the `mkForce` must let `allod/profiles`' `~/.ssh/<key>` win, proving the override is what produces the value and not an accident of merge order.
21. **`FORGE_TOKEN_FILE`.** Assert the rendered `environment.sessionVariables` names `<root>/forge-token` for microvm and is unset for libvirt.
22. **Closure scan.** Build the closure of a microvm dev fixture's `system.build.toplevel` and assert no path under it is or contains a `.age` file, an `age`/`rage` identity file, or an OpenSSH private key. Positive control on the same check: `machineConfigurations.allod-dev`'s closure *must* contain the two `.age` ciphertexts, so a scan that silently matches nothing fails.
23. **No consumer path under a persistent mount, read off generated artifacts.** Scan the materializer script, the rendered `sshd_config`, `nix.conf`, `gitconfig` and the Home Manager result for any literal equal to or under a declared `microvm.volumes` mount point. This is the scanner contract 14 asks for, and it is what catches a copy command as well as a destination.

### Existing checks that must stay true

24. `netrc-activation` (`flake.nix:1073`) keeps reading `allod-dev` and keeps requiring the skip branch. Unchanged, and test 15 above turns it into half of a paired proof.
25. `dev-forge-opt-out` (`flake.nix:1100`) keeps its literal `expectedWithAccess` projection for the libvirt `allod-dev`. Add the microvm opted-out case as test 2 above rather than widening this check, so the libvirt contract stays pinned to a literal.
26. `credential-profiles` (`flake.nix:660`) hard-codes `/root/.git-credentials`, `/etc/nix/netrc`, `/root/.netrc`, `/home/<u>/.netrc` and `netrc-file = /etc/nix/netrc` in `localRefreshErrorsForRef` (`flake.nix:803-878`). It reads `machineConfigurations`, all of which are libvirt, so it passes unchanged. Make that scoping *explicit* — a comment and, if it costs one line, a filter — so the first microvm machine in a private inventory does not silently fail a check whose expectations were never runtime-aware.

## Rollback Plan

Revert the PR. Every declaration is additive and gated on a runtime no public machine selects, so the revert restores the exact prior derivation for every libvirt machine and returns a microvm machine to composing the agenix path it composed before.

No rollback step touches key material, a ciphertext, a registry, a host path, or a disk image. Nothing in this change creates, moves, rotates or deletes a credential; it changes only which paths a generated configuration names.

A partial state is not reachable: the change is one commit in one repo with no migration, no activation-time side effect on any machine that exists today, and no host counterpart. A machine that had already been enabled on the microvm runtime — which the Risk Assessment gate forbids — would return to failing evaluation on the missing agenix identity, which is loud.
