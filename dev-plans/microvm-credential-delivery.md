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
- **Two** materializer systemd services, not one: `allod-microvm-host-key.service` owns the SSH host key alone and is what sshd depends on; `allod-microvm-forge-credentials.service` owns every other credential and nothing requires it. Each loads the delivered system credentials it owns, validates them, and writes the consumer files atomically with declared owner and mode. Contract 6 states why the split is mandatory rather than tidy.
- Microvm-only rewiring of six consumers: sshd's `HostKey`, the Nix netrc path, root's Git credential store, the user's Git credential store, the Forge API token path, and the Forge SSH `IdentityFile` in the generated Home Manager output.
- A runtime parameter on `modules/home-shared.nix`, so the *user's* generated `git/config` carries the same credential-helper reset the system file does. It is the only archetypes file other than `flake.nix` this slice edits, and its libvirt output is unchanged.
- Microvm-only removal of agenix wiring: no `age.secrets` definition, no `age.identityPaths` definition, no `modules/netrc.nix`.
- Assertions over the merged configuration for every rule above. Path and name validation is hoisted into pure functions and exhaustively table-tested; each *assertion family* keeps one full-NixOS sabotage fixture pinned to the diagnostic it names, rather than one fixture per invalid input. See the memory budget.
- Generated-artifact checks: the two rendered materializer units, their disjoint `LoadCredential=` sets, their ordering edges, the `Requires=` edge sshd carries on the host-key unit *and only on it*, the rendered `sshd_config`, the rendered `nix.conf`, both rendered Git config files, the generated Home Manager `programs.ssh` result, and a closure scan of a microvm guest.
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

What keeps it R3 is what the validation cannot reach. Nothing in this slice boots a guest. Every claim about credential *receipt* — that PID 1 imports the fw_cfg credentials, that `LoadCredential=` finds them, that both materializers run early enough and in the right order relative to each other, that sshd starts with a `HostKey` under `/run` — is an inference from pinned upstream source, not a measurement. The parent plan's acceptance test 9 and the nested-boot slice are what settle those, and this PR must not be read as evidence for them.

**The gate:** no machine is enabled on the microvm runtime on the strength of this change. A guest whose materializer is subtly mis-ordered comes up with sshd failed and no way in; a guest whose materializer is mis-ordered the *other* way comes up with a working sshd and a stale or absent credential, which is worse because it looks fine. Neither is detectable from this repo's checks.

Splitting the materializer in two (contract 6) narrows the first of those: a broken *Forge* credential can no longer take sshd down with it, so the guest stays reachable for repair. It does not narrow the second at all, and it adds a second unit whose ordering can itself be wrong. The score does not move.

**The worst credible failure after this plan's validation passes** is a microvm guest that evaluates and builds cleanly but, on a real boot, either strands with sshd failed, or starts sshd against a credential the host-key materializer wrote with the wrong owner or mode. Both are contained to a machine nothing depends on, per the parent plan's rule that the first microvm machine is purpose-made. Neither can affect a libvirt machine, a host, or any encrypted source.

Human scrutiny, in order:

1. The two rendered materializer units: each `LoadCredential=` source path, that the two sets are disjoint and together are exactly the declared set, each unit's `Before=`/`After=`/`RequiredBy=` edges, the `Requires=` edge sshd carries on the host-key unit and the absence of any edge to the Forge unit, and the exact `install`/`mv -T` invocations in each script.
2. The rendered `sshd_config` for a microvm fixture: exactly one `HostKey` line, pointing under the credential root, and `sshd-keygen.service` rendered as a mask.
3. The rendered `nix.conf`: the *last* `netrc-file =` line is the runtime path, and every unrelated inherited `extraOptions` line is still there.
4. Both rendered Git config files — `/etc/gitconfig` and the dev user's `git/config` — for the helper reset, the runtime helper after it, and no bare `store` helper surviving in either.
5. The four libvirt derivation paths.

## Contract contradictions found

Three, all recorded rather than worked around.

1. **Option namespace.** Contract 8a names the guest option `allod.microvm.guestCredentialRoot`. The volumes slice that landed after contract 8a was written chose `allod.archetypes.microvm.volumeImageRoot`, with a comment at the option's declaration in `microvmVolumesModule` arguing the namespace should name the owning repo. Both are archetypes-owned options on the same configuration, so the repo now has two conventions. This plan follows contract 8a literally, because the contract names the exact string and nothing in the code depends on the other spelling. The reconciliation — most likely renaming the volumes option — is a separate cosmetic change, not this one.

2. **Contract 9 lists `/etc` as forbidden, and `/etc` on a microvm guest is tmpfs.** The microvm root is the upstream tmpfs root (contract 4), so `/etc/nix/netrc` on a microvm guest is no more durable than `/run/allod/credentials/netrc`. The contract is still followed literally: the value of one runtime root is that it is one *scannable* root, so a check can enumerate credential-derived plaintext by looking in exactly one place. A second location that happens to be non-durable defeats that even though it leaks nothing. No change requested.

3. **Contract 7 requires rejecting a "duplicate normalized name", and no normalization exists to duplicate through.** Measured on the pinned sources, not argued: nothing between the guest declaration and the guest's `/run/credentials/@system` entry ever folds two distinct names together. `microvm.credentialFiles` is an `attrsOf`, so its keys are Nix attribute names — already unique by construction, and a Nix attrset literal with a repeated key is an evaluation error, not a silent merge. microvm.nix's runner interpolates each key verbatim into `name=opt/io.systemd.credentials/<name>,file=<path>` with no case folding, trimming, or substitution. systemd's `credential_name_valid` (`src/shared/creds-util.c:49-53`) *validates* the exact string and rejects what fails; it does not rewrite one name into another, and `import-creds.c` writes the file under the name it read. So "two names that collide after normalization" has no reachable input on the guest side: to construct one, an implementer would have to invent a normalization step the framework does not perform, purely so a fixture could violate it.

   What this slice does instead: it asserts the name rules that *are* reachable — non-empty, at most 28 characters, `[A-Za-z0-9_-]+`, no `.` — over the union of the declared set and any supplied `credentialFiles` keys, and it carries no collision fixture. Acceptance test 7 previously demanded one; that case is removed.

   What it implies for the parent contract: contract 7's "duplicate normalized name" clause is either unreachable and should be dropped, or it names a normalization operation the framework has yet to define — most plausibly on the *host* side, where contract 8b derives a value per machine and per name and a real folding step could be introduced. **This plan does not edit the parent plan**, which converged at its own review pass 10; correcting or deleting that clause goes through the parent's normal change process, and this section is the empirical input to it.

## Interface Contracts

Inherited and not restated: contracts 2, 6, 13, 21.

**How to read the citations below.** Anything in a pinned store path — nixpkgs, systemd, QEMU, microvm.nix, agenix, the git documentation — is cited by file and line, and those are stable because the revision is. Anything in a working tree is cited by **identifier and quoted construct** instead, because `allod/archetypes` `flake.nix` moves under every slice: it grew from ~1879 to over 2000 lines between this plan being written and its first review, and three line citations in the original draft had already drifted onto unrelated code by then. Locate working-tree constructs by searching for the name, and re-derive any line number before quoting one in the implementation PR.

### 1. The credential root is a validated, microvm-only option, gated outside the module system

`allod.microvm.guestCredentialRoot`, declared only by a module the builders append with `lib.optional (runtime == "microvm")`, exactly as `microvmVolumesModule` is appended in `mkDevVm` and `mkPrivacyVm`. `lib.mkIf` does not help: it defers the value, not the definition, and every option this slice writes to under microvm either does not exist under `qemuGuest` (`microvm.*`) or would silently render wrong content for a libvirt machine (`systemd.services`, `services.openssh.extraConfig`, `environment.etc`).

Type is `lib.types.either lib.types.str lib.types.path`, for the reason the comment above `options.allod.archetypes.microvm.volumeImageRoot` already gives ("The type accepts a Nix path it will then reject on purpose"): a bare `str` makes a Nix path fail with nixpkgs' generic type error while the option tree is built, pinnable to nothing, whereas accepting it lets a named assertion be the reporter. This deliberately diverges from `nexus.microvm.hostPlaintextRoot`'s `types.strMatching` (`nix/microvm/host.nix:321`), because acceptance test rule "every assertion has a paired sabotage fixture pinned to its diagnostic" cannot be satisfied by a type-check failure.

**The existing `pathErrors` cannot be reused whole, and reusing it whole is an evaluation failure on the default value.** Its sixth rule — the `lib.optional` whose message is `"must not be under ${hostRuntimeRoot}: that root is the host's tmpfs runtime state"` — rejects any path equal to or under `/run/allod`, because a volume image on the host's tmpfs runtime root cannot hold persistent state. This option's default *is* `/run/allod/credentials`, and contract 8b's supplied values are host paths under `/run/allod/microvm`. Reused unchanged, that rule rejects every correct value this slice produces.

So split it. Hoist only the five generic rules into a shared validator: the value is a plain string and not a Nix path, is absolute, has no empty segment (no trailing or doubled slash), has no `.` or `..` segment, and is not under the store. Those five are what the volume image root, the credential root and every supplied `microvm.credentialFiles` value genuinely share.

The `/run/allod` prohibition stays inside `microvmVolumesModule`, where it is one `lib.optional` on top of the shared list rather than a shared rule with an exception.

The credential root then adds the *opposite* rule, and only it has it: the value must be strictly below `/run/allod` — neither `/run/allod` itself nor anything outside it. Contract 8a requires that, and `allod/nexus` states the same shape for its host-side root as `types.strMatching "^/run/allod/[^/]+(/[^/]+)*$"` (`nix/microvm/host.nix:321`).

Supplied `microvm.credentialFiles` values get the generic rules plus one of their own (contract 3), and never the volume rule.

Hoisting the generic rules out of `microvmVolumesModule` to a shared `let` binding is in scope and is the point: two copies of a five-rule path validator in one file is the drift this contract exists to prevent. Copying the sixth rule along with them is the drift it would cause.

Hoist to the **top-level `let` of `flake.nix`**, the same scope `checks` is evaluated in, so the checks can call the validator as an ordinary function on an ordinary string. That is what makes the memory budget below work: exhaustively table-testing a pure function over thirty bad inputs costs nothing, while proving the same thirty rules through thirty forced NixOS fixtures is the measured OOM. The name validator of contract 3 is hoisted the same way and for the same reason.

### 2. The credential-name set is closed and derived, never restated

The builder derives it from what the machine has, not from a literal list:

| Name | Length | Declared when | What the host supplies |
|---|---|---|---|
| `ssh-host-key` | 12 | always | the machine's ed25519 SSH host private key |
| `forge-token` | 11 | dev, `tokenFile != null` | the Forgejo REST API token |
| `forge-https-token` | 17 | dev, `httpsTokenFile != null` | one `https://user:pass@host` credential-store line |
| `forge-ssh-key` | 13 | dev, `forgeKey != null` | the Forge SSH private key |

**Two gates, not one, and they are independent.** The two API/HTTPS names follow the two token files; the SSH name follows the machine's forge key. Nothing ties them together, and the plan must not pretend otherwise:

- `forge-token` and `forge-https-token` are gated on precisely the values that already gate `modules/agent-forgejo-token.nix` (`lib.optional (tokenFile != null)`) and `age.secrets.forgejo-https-token` (`lib.optionalAttrs (httpsTokenFile != null)`). Both default off `identity.agentTokenFile` and `identity.forgeTokenFile`, which is what `allod/archetypes#17`'s `forgeAccess = false` opt-out clears.
- `forge-ssh-key` is gated on the machine's `forge_key` inventory fact, which the identity template does not touch. `allod/inventory` `flake.nix` declares `allod-dev.forge_key = "allod_vm"`, so **an `allod-dev` fixture with both token files nulled still declares `[ "forge-ssh-key" "ssh-host-key" ]`** — Git-over-SSH keeps working when the REST API credentials are gone, which is the honest shape.

A privacy microvm declares exactly `[ "ssh-host-key" ]`, because the privacy builder composes no Forge anything. A dev microvm reaches `[ "ssh-host-key" ]` only when both token files *and* `forge_key` are null. That is a real machine shape — `allod/inventory` already declares `forge_key = null` machines — but no dev machine has it today, so the fixture has to construct it.

**So `mkDevVm` takes `forgeKey ? machines.${name}.forge_key`.** It already takes `runtime ? machines.${name}.runtime`, `tokenFile ? identity.agentTokenFile` and `httpsTokenFile ? identity.forgeTokenFile` on exactly this pattern: a machine fact with an override the checks use to build a shape inventory does not contain. One more defaulted argument is what lets a fixture reach the no-Forge-at-all case without inventing a machine, and it changes no libvirt derivation because every real call omits it.

`forge-ssh-key` stays in this slice. Parent contract 9 explicitly moves the Forge `IdentityFile` off persistent home for microvm, and `allod/nexus` `scripts/forge-ssh-key` installing it imperatively today is the thing that path replaces. Leaving the name out would strand Git-over-SSH on the first microvm guest with no credential to point at.

Expose the set as a read-only internal option, `allod.microvm.credentialNames`, so the check reads a merged result rather than recomputing the builder's conditional. That is contract 4 of the runtime-selection slice applied here: a check that restates the generator proves nothing.

`machines.<name>.forge_key` is the same fact `vmFacts.<name>.forgeKey` already reads (`nix/vm-facts.nix`, `forgeKey = machine.forge_key or null`). Read it from `machines`, not from `vmFacts`, because `vmFacts` filters to non-hypervisors through its own trip-wire and the builders already read `machines.<name>` for `platform` and `runtime`.

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

Four notes on this table.

- **One netrc, not three.** Libvirt writes `/etc/nix/netrc`, `/root/.netrc` and `/home/<user>/.netrc` because three different readers need three different owners on a disk-installed guest. Under one `0640 root:users` file on tmpfs, the Nix daemon, root and the dev user all read the same bytes. Contract 9's "user netrc path if still needed" is answered: it is not.
- **`git-credentials` is `0640 root:users`, and that is not a new disclosure.** Today the dev user already holds the same Forge HTTPS password in `/home/<user>/.netrc` mode `0600`. Making the credential-store file group-readable changes no posture; it removes a second copy.
- **`forge-https-token` is delivered once and derived twice.** The host supplies one credential-store line; the Forge unit derives both the netrc and the credential-store file from it, exactly as `modules/netrc.nix:31-37` derives netrc from `/root/.git-credentials` today. Two host credentials for one secret would be a second inventory of the same fact.
- **`git-credentials` has two readers and therefore two config files to fix, and pointing only one of them at it is a silent leak.** This is the note the rest of this section is about.

**The Git helper list is reset, not replaced, and it is reset in both files.**

`credential.helper` is multi-valued and *accumulates* across git's config files in read order — system, then user, then repository. Replacing one file's value does not remove another file's. Git's own documentation (pinned git 2.51.2, `gitcredentials.adoc:199-200`) states the escape hatch: *"If `credential.helper` is configured to the empty string, this resets the helper list to empty (so you may override a helper set by a lower-priority config file by configuring the empty-string helper, followed by whatever set of helpers you would like)."*

Both Allod files set a bare `store`, whose default file is `~/.git-credentials` — a **persistent** path on a microvm guest, and the one contract 9 forbids:

- `allod/vm` `modules/guest-base.nix` writes `environment.etc."gitconfig".text` containing `[credential]` / `helper = store`. Root reads this and nothing else.
- `allod/archetypes` `modules/home-shared.nix` sets `programs.git.settings.credential.helper = "store"`, rendered into the dev user's `git/config`. The user reads *both* files, and this one is read later.

So each file gets the reset plus the runtime helper, in that order:

- `/etc/gitconfig`: append with `lib.mkAfter` on `environment.etc."gitconfig".text`, which is `types.lines`. The appended block is an empty `helper =` followed by `helper = store --file=<root>/git-credentials`. Not `mkForce`: forcing a `lines` option discards every other definition, and the reset is precisely the mechanism that makes discarding unnecessary. Acceptance test 18 records the same hazard for `nix.extraOptions`, where the original draft's defence against it did not work.
- The user's `git/config`: give `modules/home-shared.nix` a `credentialRoot ? null` argument and set `credential.helper = [ "" "store --file=<root>/git-credentials" ]` when it is non-null, keeping today's `"store"` when it is null. Home Manager's `gitIniType` is `attrsOf (either multipleType sectionType)` with `multipleType = either primitiveType (listOf primitiveType)`, so a list renders one `helper = ` line per element, in order. No `mkForce`: this is the only definition of that attribute, and forcing it would be forcing our own value.

The two files render the same two settings **differently**, which is a trap for any check that greps for one string. `environment.etc."gitconfig".text` is raw text, so it renders `helper = store --file=/run/allod/credentials/git-credentials` unquoted. Home Manager renders through `lib.generators.toGitINI`, whose `mkValueString` quotes every string, so the same setting renders `helper = "store --file=/run/allod/credentials/git-credentials"` and the reset renders `helper = ""`. Git strips the quotes on read and both mean the same thing; a needle built for one file does not match the other. Acceptance test 19 reads each file in its own form.

Root needs no user-side reset because Home Manager manages only the dev user; `/etc/gitconfig` is the whole of root's Git configuration.

### 5. Contract 14 is a property of the table, and is asserted as one

No path in the table above may be equal to, or nested under, any `mountPoint` in the merged `config.microvm.volumes`. Assert it over the merged volumes list rather than over the required set the volumes module declares, so a profile that adds a volume at `/run/allod` is caught. Assert it over the whole table, not over the root alone: a root outside the volumes with one consumer path re-anchored elsewhere is the failure this is for.

### 6. Two materializer boot services, no optional branch in either

**Two** oneshot units with `RemainAfterExit = true`, not one, and the split is a correctness requirement rather than tidiness:

| Unit | Composed when | Owns | Required by |
|---|---|---|---|
| `allod-microvm-host-key.service` | always | `ssh-host-key` only, plus `<root>` itself | sshd, and only sshd |
| `allod-microvm-forge-credentials.service` | the machine declares at least one Forge name | `forge-token`, `forge-https-token`, `forge-ssh-key` and every file derived from them | nothing |

**Why one unit is wrong.** Validation is per credential and any failure is fatal to the unit (below), and sshd `Requires=` the unit that carries the host key. Put all four credentials in one unit and a missing, empty or malformed *Forge* credential — every one of them optional, every one of them supplied by a host-side path this repo cannot see — stops the host key from ever being published, so sshd never starts and the guest is unreachable. The credential whose absence must stop sshd is the host key; nothing else is. Splitting the unit is what makes "fail visibly" and "stay reachable for repair" both true, and it costs one extra unit.

The failure the split accepts is the correct one: a broken Forge credential leaves the guest up, reachable, with `allod-microvm-forge-credentials.service` failed in `systemctl --failed` and Git, Nix-over-HTTPS and the `forge` CLI broken at the point of use. That is louder in the place it matters and quieter in the place it does not.

**What differs between the two units, and nothing else does.** Both use the same skeleton, the same fail-closed absolute `LoadCredential=` form, the same per-credential validation, the same atomic write idiom, and the same ordering block. The differences are exactly four:

- Their `LoadCredential=` sets are **disjoint**, and their union is exactly `config.allod.microvm.credentialNames`.
- The host-key unit creates `<root>`; the Forge unit does not, and carries `Requires=`/`After=` on the host-key unit so the directory has exactly one creator. That dependency runs the harmless direction — a failed host key already means sshd is down, so failing the Forge unit with it strands nothing extra. The forbidden edge is sshd depending on the Forge unit, not the Forge unit depending on the host key.
- The Forge unit also carries `After=systemd-sysusers.service`, because it writes files owned by the dev user and `install -o <user>` needs that user to exist. Under classic activation the user is already there; under `systemd.sysusers.enable` or `services.userborn.enable` it is created by a unit that is itself only `Before=sysinit.target`, so without this edge two `DefaultDependencies=no` units race. One `After=` covers both paths: pinned nixpkgs gives `userborn.service` `aliases = [ "systemd-sysusers.service" ]` (`nixos/modules/services/system/userborn.nix`), so the ordering resolves to whichever exists, and to nothing when neither does. The host-key unit writes `root:root` only and needs no such edge.
- Only the host-key unit carries the explicit `Before=` on sshd and is named in sshd's `Requires=`.

**Where the credential already is by the time these units run.** Measured on pinned systemd 258.7. PID 1 calls `kmod_setup()` (`src/core/main.c:3186`), which explicitly loads `qemu_fw_cfg` under KVM/QEMU with the comment *"would be loaded by udev later, but we want to import credentials from it super early"* (`src/core/kmod-setup.c:144-145`), and only then `initialize_runtime()` → `import_credentials()` (`main.c:3313`, `main.c:2453`). That reads `/sys/firmware/qemu_fw_cfg/by_name/opt/io.systemd.credentials` (`src/core/import-creds.c:375`) into `/run/credentials/@system` (`src/shared/creds-util.h:33`), files `0400 root:root` in a `0700 root:root` non-swappable tmpfs, then remounts it read-only.

This happens in the **initrd's** PID 1, because these guests run systemd-in-initrd: `microvm.optimize.enable` defaults true and `nixos-modules/microvm/optimization.nix:16-31` sets `boot.initrd.systemd.enable` for QEMU. It then survives switch-root — `src/shared/switch-root.c:38` carries `{ "/run/credentials", MS_BIND|MS_REC, 0 /* skip! */ }, /* Credential mounts should survive */` — and the real-root manager re-execs with `execv`, inheriting `$CREDENTIALS_DIRECTORY`, so `import_credentials()` short-circuits rather than re-importing.

**This corrects why contract 5 matters.** `boot.initrd.kernelModules = [ "qemu_fw_cfg" ]` is load-bearing because it puts the `.ko` in the initrd's `modulesClosure` at all — `qemu_fw_cfg` is a module, not built in, at kernel 6.12.93. Without it systemd's own early modprobe fails, `/sys/firmware/qemu_fw_cfg` never appears, and `import_credentials_qemu()` logs `"No credentials passed via fw_cfg."` and returns non-fatally. It is *not* about ordering against `systemd-modules-load.service`, which runs far too late to matter for this.

- **Input.** `LoadCredential = "<name>:/run/credentials/@system/<name>"`, for the host key on the host-key unit and for each declared Forge name on the Forge unit. **The absolute-path form is deliberate and is the first fail-closed gate.** `systemd.exec(5)` makes a bare `LoadCredential=<name>` **non-fatal** when the credential is absent — the unit starts and the file simply is not there — whereas an absolute source path that cannot be read fails the unit with `EXIT_CREDENTIALS`. `ImportCredential=` is likewise soft-fail. So the bare form would let a materializer run to completion having written nothing, and sshd would fail two units away from the cause. Each script then reads `$CREDENTIALS_DIRECTORY/<name>`, which for a system service is `/run/credentials/<unit name>/<name>`, mode `0400` — a different directory per unit, which is why the two `LoadCredential=` sets have to be disjoint rather than merely correct in aggregate.
- **Nothing in the guest references a host path.** `microvm.credentialFiles` values exist only in QEMU's argv on the host; no guest-side code may name `/run/allod/microvm/...`.
- **Script validation is the second gate, and is still mandatory.** systemd's gate proves the bytes arrived; it says nothing about their shape. The host launcher additionally refuses a missing, non-regular or empty *source* before QEMU starts (`allod/nexus` `nix/microvm/launcher.nix:285-287`), but the guest cannot rely on that either: upstream's `microvm -r <name>` execs the runner with no unit and no preparation, which `allod/nexus` `docs/microvm-host.md:44-48` documents as unsupported but reachable.
- **Validation, per credential, before any write.** The file exists, is a regular file, and is non-empty. `ssh-host-key` parses as an OpenSSH private key (`ssh-keygen -y -f` succeeds). `forge-https-token` yields exactly one non-empty line matching the `https://user:pass@host` shape — reuse `modules/netrc.nix:31-37`'s `sed` expression, which already encodes it. `forge-ssh-key` parses as an OpenSSH private key. `forge-token` is a single non-empty line with no whitespace.
- **No skip branch.** Any failure is `exit 1` with a message naming the credential. There is no counterpart to `modules/netrc.nix:18`'s `"missing or empty, skipping (expected during provisioning)"`, and the check asserts that string is *absent* from both microvm materializers while remaining present in the libvirt activation script.
- **Atomic writes, and `install` alone is not one.** `install(1)` truncates the destination in place; it does not write-and-rename. The idiom is `install -m <mode> -o <owner> -g <group> <src> <dir>/.<name>.tmp && mv -T <dir>/.<name>.tmp <dir>/<name>` — the temporary must be in the destination directory so the rename is a same-filesystem `rename(2)`, and the leading dot keeps a partial file out of a `<root>` listing. `allod/nexus` `nix/microvm/launcher.nix` establishes the same *principle* on the host side and is worth reading for it, but it is not the same implementation: the launcher stages an entire fresh directory (`stage=…/active/.stage-<name>`, `install`ing every credential into it) and publishes that whole directory once with `mv -T -- "$stage" <credentialDirectory>`. Per-file staging is chosen here because the two guest units publish into a directory the other one is also writing, so there is no single moment either could rename the whole thing. What both share, and what the check enforces, is that no consumer ever observes a partially written file. Either way, follow it rather than the weaker `modules/netrc.nix:39-44` form, which is a plain in-place `install`.
- **Ordering: `Before=sysinit.target` with `DefaultDependencies=no`.** This is the single ordering that precedes *every* normal service, every socket unit, `systemd-user-sessions.service`, `user@.service` and `sshd.service`, because systemd gives all of them a default `After=sysinit.target` (`systemd.service(5)`, `systemd.socket(5)`). Enumerating consumers instead — `before = [ "sshd.service" "nix-daemon.service" … ]` — is a list that goes stale the moment a profile adds a reader, which on a machine whose whole point is composed profiles is not a hypothetical. The skeleton is the one `nixos/modules/security/wrappers/default.nix:305-320` already uses for the same problem, plus the reactivation edges:

  ```nix
  wantedBy = [ "sysinit.target" ];
  requiredBy = [ "sysinit-reactivation.target" ];
  before = [ "sysinit.target" "sysinit-reactivation.target" "shutdown.target" ];
  conflicts = [ "shutdown.target" ];
  unitConfig.DefaultDependencies = false;
  serviceConfig.Type = "oneshot";
  serviceConfig.RemainAfterExit = true;
  ```

  **`requiredBy` alone is not the reactivation wiring, and getting this wrong is a rebuild-time race.** `switch-to-configuration` starts `sysinit-reactivation.target` to re-run changed early units in the right order relative to each other; `Requires=` creates the pull but no ordering at all. Pinned `nixos/modules/services/system/userborn.nix` — the precedent this plan cites — carries `"sysinit-reactivation.target"` in **both** `requiredBy` and `before`, and the `before` half is the load-bearing one. Without it, a rebuild that changes a materializer can reactivate a consumer before the materializer has rewritten the file, which produces a running guest holding stale credentials and no error anywhere. Both units carry both edges.

  Keep an explicit `before = [ "sshd.service" ]` on the **host-key unit only**: it costs one list entry and survives someone later setting `DefaultDependencies=no` on sshd.

- **sshd's real dependency, and nothing else's.** `systemd.services.sshd.requires` gains `allod-microvm-host-key.service`, so a failed host-key materializer leaves sshd unstarted with a dependency failure rather than started against a missing file — contract 11's stated outcome, made a real edge rather than a coincidence. Two constraints on how that is written:

  - **Nothing anywhere `Requires=` the Forge unit.** Not `nix-daemon.service`, not `sshd`, not a user session, not a `home-manager-<user>.service`. Ordering from `Before=sysinit.target` is all those consumers get and all they need; a dependency would re-create exactly the stranding the split exists to prevent.
  - **Follow sshd's own conditional shape.** Pinned `nixos/modules/services/networking/ssh/sshd.nix` defines `systemd.services.sshd` under `lib.mkIf (!cfg.startWhenNeeded)` and defines `systemd.sockets.sshd` plus `systemd.services."sshd@"` under `lib.mkIf cfg.startWhenNeeded`. `startWhenNeeded` defaults false and nothing in Allod sets it, so the socket path is unreachable today — but writing an unconditional `systemd.services.sshd.requires` would *materialize* an `sshd.service` unit in the socket-activated case, where nixpkgs deliberately renders none. Gate the wiring on `config.services.openssh.startWhenNeeded` the same way sshd does, adding the edge to `sshd.socket` and `sshd@.service` on the socket branch, so a profile that flips the option neither loses the guarantee nor gains a stray unit.

- **Not `systemd-tmpfiles`.** The `f^` line type reads a credential into a file with declared owner and mode and is the idiomatic pinned-nixpkgs route — `nixos/modules/services/networking/ssh/sshd.nix:692-709` uses it for root's `authorized_keys`. It is rejected here for three concrete reasons, recorded so a later SIMPLIFY sweep does not reopen it: `systemd-tmpfiles-setup.service` imports a fixed credential allowlist (`ImportCredential=tmpfiles.*`, `ssh.authorized_keys.root`, …), so any other name is *silently skipped* unless a drop-in is added in three separate units nixpkgs keeps in sync; `f` never rewrites an existing file, and `f+` truncates in place rather than renaming; and the tmpfiles specifier table has no `%d`, so the `^` modifier is the only route and it cannot express per-credential validation at all.
- **The directory.** The host-key unit's script creates `<root>` itself with the declared mode, and it is the only creator. A `systemd.tmpfiles` rule would be a second declaration of the same path in a different language, ordered by a different unit.

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

What the builder omits for microvm, all in `mkDevVm` and `sharedModules`: the `age.secrets.forgejo-https-token` attrset guarded by `lib.optionalAttrs (httpsTokenFile != null)`, the `lib.optional (tokenFile != null) ./modules/agent-forgejo-token.nix` entry, the `./modules/netrc.nix` entry, and the `services.openssh.hostKeys` / `age.identityPaths` pair `sharedModules` sets to `/etc/ssh/${name}`.

`(mkGithubCredentialModule name)` stays composed, and a microvm machine with a non-empty `secrets.lib.githubCredentialTargets.<name>` **fails evaluation** with a named diagnostic pointing at the follow-up. The public template declares none, so deriving credential names for them now would be untestable speculative generality; refusing them is honest, is one assertion, and has a fixture that injects a target.

### 9. The closure carries no key material

A generated check over the microvm guest's `system.build.toplevel` closure asserting it contains no `.age` file, no age or agenix identity, and no private key. The realistic failure this catches is a Nix path literal reaching an option that interpolates it — `microvm.credentialFiles` being `attrsOf path` is exactly that hazard, and `secrets.lib.*TokenFile` values are Nix paths into the `secrets` store path today.

Scan the *closure*, not the toplevel directory: an `age.secrets.<n>.file` reference lands as a store path in the activation script, whose closure includes the ciphertext.

**Three detectors, three positive controls, and one of the controls is not the others.** The check runs three independent predicates — a filename ending in `.age`, an age or rage identity in the file contents, and an OpenSSH private key header in the file contents — and each one has to be shown capable of rejecting. The libvirt dev machine's closure supplies exactly one of those proofs: it contains the two `.age` ciphertexts, which proves traversal happens and that the *filename* predicate fires, and nothing more. Both content predicates could be deleted outright and the check would stay green.

So add two synthetic canary files, built by the check itself, and assert each content detector rejects its own canary:

- a file containing the literal `AGE-SECRET-KEY-1` prefix followed by filler, for the age/rage identity detector;
- a file containing an `-----BEGIN OPENSSH PRIVATE KEY-----` header followed by filler, for the private-key detector.

They are strings in the check, not key material: no real secret and no generated key goes anywhere near a fixture. They are also not a second NixOS closure — building another full system to carry a canary is the memory cost this budget cannot pay. Run each predicate over the canary directly and require a rejection, then run all three over the microvm closure and require silence.

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

`runtime-module-selection` already states its own constraint, in the comment above its `sabotages` list: fixtures are consumed as thunks one at a time because *"Holding them all live at once instead is an OOM kill on an 8 GB builder — measured, not feared"*, and even consumed one at a time they are most of what takes `nix flake check` from a ~4 GB peak to over 5 on a 7 GiB box with no swap.

**The budget is what shapes the fixture list below, not a footnote appended to it.** The original draft of this plan listed more than twenty full-system sabotages and then said what to do if the peak rose too far — which targets the cheap half (a handful of positive assertions that only read projections) and leaves the expensive half (every forced negative system) untouched. Forcing a system is what runs `config.assertions`, so a negative fixture cannot be made cheap; it can only be made rarer.

So the count comes down first, by moving what does not need a system out of the systems:

- **Table-test the pure validators exhaustively.** Contract 1 hoists the path validator and contract 3 the name validator into `flake.nix`'s top-level `let`. Every invalid *shape* — Nix path, relative, store-prefixed, trailing slash, `.`/`..` segment, `/run/allod` itself, outside `/run/allod`, empty name, 29 characters, `/`, `.`, `:`, `,` — is checked by calling the function on a string and comparing the returned message list. No `nixosSystem`, no forcing, effectively no memory. This is where exhaustiveness lives.
- **Keep one full-NixOS sabotage per assertion family, not per input.** The families are: the credential root is rejected; a supplied `credentialFiles` key is rejected; a supplied `credentialFiles` value is rejected; the key *set* disagrees with the declared set; a consumer path overlaps a persistent mount; agenix wiring is present; `hostKeys` is non-empty; a GitHub credential target exists; a generated unit was mutated. Each family's fixture proves the wiring — that the validator is actually called on the real merged value and its message reaches `config.assertions` — which is exactly what a pure-function table cannot prove and the only thing a system is needed for.
- **Load each surviving fixture up.** One bad `credentialFiles` map can carry several invalid keys *and* several invalid values at once; one map can be both missing a required key and carrying an extra one; the generated-unit mutations of tests 13 to 15 can share a fixture, since `checkSabotage` already takes a list of needles per fixture for precisely this reason. Combine wherever the needles stay individually pinned.
- **Every new fixture is a thunk consumed exactly once by `checkSabotage`.**
- **Drive every archetype-independent family from the privacy builder**, which composes no dev user environment: the root validation, the name and value rules, and the contract 14 overlap rule. Reuse `asMicrovm` for every positive assertion rather than building a second dev fixture.
- **Measure after each small batch, not after the list is complete.** `\time -v nix flake check`, maximum resident set size, recorded in the PR body. A batch that moves the peak more than ~5% stops and gets merged into an existing fixture before the next batch starts. Discovering the OOM after twenty fixtures means unpicking twenty fixtures.

### Contract assertions, each with a paired sabotage fixture

Every fixture case below is `{ name; fixture = _: …; needles = [ … ]; }` in `runtime-module-selection`'s existing `sabotages` list, pinned through `missingDiagnostic` against `config.assertions`, and verified by deletion: removing the assertion must make the fixture stop failing for its named reason.

**0. The validator tables come first, and they carry the exhaustiveness.** Two pure-function tables, no `nixosSystem` anywhere in them:

- The path validator, over: a Nix path, a relative path, a store-prefixed path, a trailing slash, a doubled slash, a `.` segment, a `..` segment, `/run/allod` itself, a path outside `/run/allod`, and a valid path. Each case asserts the exact expected message list, not merely that it is non-empty.
- The name validator, over: empty, 28 characters (valid), 29 characters, `/`, `.`, `..`, `:`, `,`, `=`, a leading dash (valid), and a valid name. The `.` case asserts the reserved-system-credential wording specifically, so it cannot be satisfied by a generic "invalid character" message.

Tests 6, 7 and 8 below then each keep **one** system fixture whose only job is to prove the validator is wired to the real merged value and its message reaches `config.assertions`.

Positive, on `asMicrovm` and `asPrivacyMicrovm`:

1. A dev microvm's `config.allod.microvm.credentialNames` is exactly `[ "forge-https-token" "forge-ssh-key" "forge-token" "ssh-host-key" ]`; a privacy microvm's is exactly `[ "ssh-host-key" ]`.
2. **The two gates are independent, and it takes three fixtures to show it.** Built through the same `withTokens` helper `dev-forge-opt-out` already defines, extended with the new `forgeKey` argument:
   - both token files null, `forge_key` left at inventory's `"allod_vm"` → `[ "forge-ssh-key" "ssh-host-key" ]`. This is the real `forgeAccess = false` shape for `allod-dev`, and it is **not** `[ "ssh-host-key" ]`: the identity template does not touch `forge_key`, so Git-over-SSH survives the opt-out.
   - both token files present, `forgeKey = null` → `[ "forge-https-token" "forge-token" "ssh-host-key" ]`.
   - both token files null and `forgeKey = null` → `[ "ssh-host-key" ]`. This is the only way to reach the no-Forge-at-all shape from a dev builder, and it is why `mkDevVm` takes the argument.

   A fixture that changed only the token files and expected `[ "ssh-host-key" ]` would be asserting a machine shape that cannot exist; that is what the first case exists to prevent recurring.
3. `config.age.secrets == {}` and `config.age.identityPaths == []` on both microvm fixtures.
4. `config.services.openssh.hostKeys == []` on both.
5. A non-default `allod.microvm.guestCredentialRoot` moves every entry in the consumer table, both materializer scripts, the `HostKey` line, the appended `netrc-file` line, **both** Git config files, and the Home Manager `IdentityFile`. Restate the default literally in the check, for the reason the existing `expectedImageRoot` comment gives: a check that derived the default from the option it is checking could not notice the default moving.

Sabotage, each pinned to its own diagnostic:

6. **Credential root, one fixture.** The privacy builder with the root forced to a store-prefixed path. Needle anchored to `allod.microvm.guestCredentialRoot` by name, for the reason the `imageRootOption` comment already records: the same validator runs over several values, so an unanchored needle like "must be an absolute path" is satisfied by another reporter's message and stays green with this assertion deleted. Every other invalid root shape is a row in the table above.
7. **Credential names, one fixture.** `microvm.credentialFiles` forced to a map whose keys include a 29-character name, an empty name, a name containing `/`, and a name containing `.`, all at once; four needles on one fixture. The `.` needle must name the reserved-system-credential reason.

   **No collision case.** The original draft asked for "two names colliding after normalization" to match parent contract 7. There is no normalization step anywhere on this path to collide through, and Nix attribute keys are unique by construction, so the fixture cannot be written without inventing the operation it violates. See contradiction 3.
8. **Credential values and key set, one fixture each.** One map carrying a Nix path value, a store-prefixed value and a value containing a comma; one map that both omits a declared key and adds an extra one.

   **No relative-value case.** Contract 3 already establishes why: `microvm.credentialFiles` is `attrsOf path`, `types.path` is `pathWith { absolute = true; }`, and a relative value is rejected by the option's own type with nixpkgs' generic error before any assertion runs. Upstream's type error *is* the enforcement evidence for that rule on the value side. The absolute rule keeps its named assertion and its sabotage on `guestCredentialRoot`, where the permissive option type makes it reachable — that is test 6's table row.
9. A `microvm.volumes` entry forced to `mountPoint = "/run/allod"`, so a consumer path is nested under a persistent mount.
10. `age.secrets` forced non-empty on a microvm fixture; `age.identityPaths` forced non-empty.
11. `services.openssh.hostKeys` forced non-empty on a microvm fixture.
12. A microvm machine with a non-empty `githubCredentialTargets` entry injected.

### Generated-artifact checks

These are the tests this slice exists for, and none of them may be replaced by an option read.

13. **The two rendered materializer units, and the split itself.** Build `config.systemd.units."allod-microvm-host-key.service".unit` and `…"allod-microvm-forge-credentials.service".unit` and read both fragment files. Assert:
    - `allod-microvm-host-key.service` carries exactly one `LoadCredential=`, `ssh-host-key:/run/credentials/@system/ssh-host-key`, and the Forge unit carries exactly one per declared Forge name. The two sets are **disjoint** and their union is exactly `config.allod.microvm.credentialNames`.
    - Each carries `DefaultDependencies=no`, `RemainAfterExit=yes`, `Before=` naming both `sysinit.target` **and** `sysinit-reactivation.target`, and `Conflicts=shutdown.target`.
    - `sysinit-reactivation.target`'s rendered unit carries a `Requires=` edge to both.
    - Only the host-key unit's `Before=` names `sshd.service`. Only the Forge unit carries `After=systemd-sysusers.service` and `Requires=`/`After=` on the host-key unit.
    - For `asPrivacyMicrovm`, and for the dev fixture with no Forge names, `config.systemd.services` has **no** `allod-microvm-forge-credentials` attribute at all.

    Paired sabotages, sharing fixtures where the needles stay pinned: a fixture that drops the `before` lists loses the `sysinit.target`, `sysinit-reactivation.target` and `sshd.service` needles together; a fixture that uses the bare `LoadCredential=<name>` form loses the absolute-path needle, which is what defends the soft-fail distinction that is invisible in the option value; and a fixture that **merges or swaps the two credential sets** — moving `ssh-host-key` onto the Forge unit — must fail the disjointness and union assertions. That last one is the sabotage for the split itself: a two-unit design whose fixtures cannot tell the units apart is worse than one unit.
14. **sshd's real dependency edge, and the edge that must not exist.** In the rendered `sshd.service`, assert `Requires=` names `allod-microvm-host-key.service` and does **not** name `allod-microvm-forge-credentials.service`. Assert no unit anywhere in the rendered configuration carries a `Requires=` or `Requisite=` on the Forge unit. Paired sabotages: dropping the `requires` definition loses the first needle; adding the Forge unit to sshd's `requires` must fail the second.
15. **Neither materializer script has an optional branch.** Write both scripts to store files and assert `missing or empty, skipping` is absent from each, that each declared credential name appears in a validation command in its own unit's script, and that the write idiom is a rename — the needle is `mv -T`, because `install` alone truncates in place and would leave a window where a consumer reads a partial key. Paired positive control: the `missing or empty, skipping` needle must still be *present* in `machineConfigurations.allod-dev`'s `system.activationScripts.netrc.text`, which is what the existing `netrc-activation` check reads, so the two checks together prove the branch was removed from one path and kept on the other.
16. **The rendered `sshd_config`.** Read `config.environment.etc."ssh/sshd_config".source` for a microvm fixture. Assert exactly one `HostKey` line and that it points under the credential root. Paired sabotage: a fixture whose `extraConfig` is forced empty must have zero `HostKey` lines. This is the *only* guard against the compiled-in-default fallback: pinned `sshd -G -T` exits 0 with no host key configured, so nixpkgs' own build-time config check cannot catch it.
17. **`sshd-keygen` is masked** for a microvm fixture — its rendered unit is a symlink to `/dev/null` — and is a real unit with a non-empty script for `allod-dev`. This is the anti-regeneration proof; the option read `hostKeys == []` is not, because an empty `hostKeys` also produces an unloadable unit rather than a masked one, and the two are indistinguishable from the option value.
18. **The rendered `nix.conf`: append, do not force.** Read the built `/etc/nix/nix.conf` source. Assert the **last** `netrc-file =` occurrence in the file is the runtime path under the credential root.

    Not `mkForce`, and not "exactly one line". `nix.extraOptions` is `types.lines`, `allod/vm` `modules/guest-base.nix` defines `nix.extraOptions = "netrc-file = /etc/nix/netrc"` in it, and `mkForce` on a `lines` option discards **every** other definition. The original draft argued that counting `netrc-file =` lines would catch a future `allod/vm` change adding an unrelated `extraOptions` setting that `mkForce` swallowed — it cannot, because anything `mkForce` discarded is simply absent from the artifact the check reads. The count is over the wrong file.

    So append with `lib.mkAfter` and assert last-wins, which is how Nix reads a repeated setting anyway. The preservation control is what makes this real: inject one unrelated `extraOptions` line into the fixture (`lib.mkAfter` on a second definition, any setting that is not `netrc-file`), and assert it survives into the rendered `nix.conf` alongside the runtime `netrc-file` line. Deleting the inherited definitions is then a failing test rather than an invisible one.

    `nix.settings.netrc-file` is **not** the simpler route, and the reason is mechanical: `pkgs.formats.nixConf`'s generator emits every `settings` key first and interpolates `extraOptions` last (pinned nixpkgs `pkgs/pkgs-lib/formats.nix`, the `text` block of `nixConf.generate`). A `settings`-supplied runtime path would therefore be *overridden* by the inherited legacy `extraOptions` line, silently. No cross-repo change is needed and none is proposed.
19. **Both rendered Git config files.** Contract 4 sets out the design; this test reads both artifacts in their own rendering.
    - `/etc/gitconfig`: read `config.environment.etc."gitconfig".text`. Assert the inherited `helper = store` is still present (nothing was forced away), that a bare `helper =` reset appears after it, and that the *last* helper line is `helper = store --file=<root>/git-credentials`, unquoted.
    - The user's file: read the microvm fixture's `home-manager.users.<user>.programs.git.settings.credential.helper` and the rendered `git/config` text. Assert the rendered form is `helper = ""` followed by `helper = "store --file=<root>/git-credentials"` — quoted, because `lib.generators.toGitINI`'s `mkValueString` quotes every string, unlike the raw `types.lines` above.
    - In both files: no helper line after the final reset other than the runtime one, so no bare `store` remains effective. Assert on the *ordered* list of helper values, not on presence.

    Paired sabotages: a fixture that drops the reset from either file must fail; a fixture that reverses the order within a file must fail; a fixture that fixes only `/etc/gitconfig` and leaves the user's `git/config` at bare `store` must fail, which is the failure this test was added for.
20. **The generated Home Manager result.** Read the microvm fixture's `home-manager.users.<user>.programs.ssh` merged config and assert the Forge host's `identityFile` is `<root>/forge-ssh-key`. Paired sabotage: dropping the `mkForce` must let `allod/profiles`' `~/.ssh/<key>` win, proving the override is what produces the value and not an accident of merge order.
21. **`FORGE_TOKEN_FILE`.** Assert the rendered `environment.sessionVariables` names `<root>/forge-token` for microvm and is unset for libvirt.
22. **Closure scan, with a control per detector.** Build the closure of a microvm dev fixture's `system.build.toplevel` and assert no path under it is or contains a `.age` file, an age/rage identity, or an OpenSSH private key.

    Three detectors need three controls, and the libvirt closure supplies only one. `machineConfigurations.allod-dev`'s closure must still contain the two `.age` ciphertexts — that proves traversal happens and that the filename predicate fires, and it proves nothing about the other two, which could both be deleted with this check still green. So also run the age-identity predicate over a store file containing the literal `AGE-SECRET-KEY-1` prefix and the private-key predicate over a store file containing an `-----BEGIN OPENSSH PRIVATE KEY-----` header, and require each to reject its own canary. Synthetic strings written by the check; no real key material, no generated key, and no second NixOS fixture.
23. **No consumer path under a persistent mount, read off generated artifacts.** Scan both materializer scripts, the rendered `sshd_config`, `nix.conf`, both Git config files and the Home Manager result for any literal equal to or under a declared `microvm.volumes` mount point. This is the scanner contract 14 asks for, and it is what catches a copy command as well as a destination.

### Existing checks that must stay true

24. `netrc-activation` keeps reading `allod-dev` and keeps requiring the skip branch. Unchanged, and test 15 above turns it into half of a paired proof.
25. `dev-forge-opt-out` keeps its literal `expectedWithAccess` projection for the libvirt `allod-dev`. Its `withTokens` helper gains the `forgeKey` argument test 2 needs; its own two cases are unchanged. Add the microvm cases as test 2 above rather than widening this check, so the libvirt contract stays pinned to a literal.
26. `credential-profiles` hard-codes `/root/.git-credentials`, `/etc/nix/netrc`, `/root/.netrc`, `/home/<u>/.netrc` and `netrc-file = /etc/nix/netrc` in its `localRefreshErrorsForRef` helper. It reads `machineConfigurations`, all of which are libvirt, so it passes unchanged. Make that scoping *explicit* — a comment and, if it costs one line, a filter — so the first microvm machine in a private inventory does not silently fail a check whose expectations were never runtime-aware.

## Rollback Plan

Revert the PR. Every declaration is additive and gated on a runtime no public machine selects, so the revert restores the exact prior derivation for every libvirt machine and returns a microvm machine to composing the agenix path it composed before.

The two edits that are not new declarations are still no-ops to revert. `modules/home-shared.nix` gains a defaulted `credentialRoot ? null` argument whose null branch is today's `credential.helper = "store"`, and `mkDevVm` gains a defaulted `forgeKey ? machines.${name}.forge_key`; every existing call site omits both, which is what the four unchanged derivation paths prove.

No rollback step touches key material, a ciphertext, a registry, a host path, or a disk image. Nothing in this change creates, moves, rotates or deletes a credential; it changes only which paths a generated configuration names.

A partial state is not reachable: the change is one commit in one repo with no migration, no activation-time side effect on any machine that exists today, and no host counterpart. A machine that had already been enabled on the microvm runtime — which the Risk Assessment gate forbids — would return to failing evaluation on the missing agenix identity, which is loud.
