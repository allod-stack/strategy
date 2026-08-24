# Make pi-provider follow the deployment repositories

`pi-provider` will use the host's configured inventory and the deployment checkouts it names, then prove their actual staged flake behavior in tmpfs before asking for a bearer. The command stops hardcoding a host name, recipient file, or check name; it still leaves commits, pin updates, rebuilds, live verification, and remote revocation to the owner.

## Tracking Issue

[allod/nexus#29](https://forge.anarch.diy/allod/nexus/issues/29). The plan and secrets implementation PRs use `Refs allod/nexus#29`; the final Nexus implementation PR carries `Closes allod/nexus#29`.

## Goal

Make the plain installed `pi-provider` command operate safely on redirected deployment data repositories, including deployments whose hypervisor key is separate from VM host-key data or whose hypervisor is not named `nexus`.

## Scope

Two ordered public implementation slices:

1. `allod/secrets` separates the hypervisor recipient source from VM host-key data. Its existing recipient derivation accepts an explicit ordered `nexusPublicKeys` list, uses `identity.hostPublicKeys` in the live flake and `secrets.nix`, and leaves `machine-host-keys.json` responsible only for targeted VM keys. The exported `mkPiCredentialContract` keeps a compatibility fallback to `machineHostKeys.${nexusName}` for its existing deploy consumer, but the secrets flake itself and all new fixtures exercise the split source.
2. `allod/nexus`, after the secrets slice is available:

- render the configured inventory checkout as both `INVENTORY` and the compatibility pointer `INVENTORY_CHECKOUT` in the host session;
- resolve profiles and secrets through that inventory's deployment aliases with strict diagnostics rather than a silent public fallback;
- replace the script's recipient reimplementation with the redirected secrets checkout's existing agenix recipient derivation;
- preflight staged copies of both redirected repositories in tmpfs before dry-run completion or bearer input, and validate the installed working trees against those results afterward;
- remove hardcoded, platform-specific post-install check attribute builds; and
- document and fixture the resolved-path and derived-recipient contract.

No profiles/inventory schema change, deploy-pin bump, private data change, host rebuild, VM rebuild, remote provider request, bearer revocation, or automatic commit is in scope. `pi-provider` does not parse or rewrite operator-owned Nix syntax: the secrets checkout remains responsible for deriving Pi entries in `secrets.nix` from `pi-credentials.json`.

## Risk Assessment

Residual risk is **R3 High** for each slice. The secrets slice changes a shared recipient interface and the Nexus slice selects which working trees receive provider metadata and Age ciphertext. A faulty result can leave two recoverable local data checkouts dirty or encrypt a new bearer to the wrong public keys, but cannot publish, deploy, revoke, or expose plaintext by itself. The standing journal restores ordinary failures, ciphertext is reconstructable from the remote provider, and rollback is reverting the relevant PR or retaining prior deployment pins. Human scrutiny is most useful on split-key ordering and duplicate rejection, compatibility for the deploy consumer, pre-prompt ordering, exact staged/live output equality, redirected-path resolution, and negative cases that leave both repositories untouched.

## Interface Contracts

`lib/pi-credential-recipients.nix` receives `nexusPublicKeys` separately from `machineHostKeys`. The list must be non-empty, contain only non-empty unique strings, and preserves its supplied active-then-staged order. Target VM recipients then follow registry target order, with each target's active key before its optional staged key. Duplicate source keys are rejected globally across the hypervisor list and every referenced target record; the derivation never silently deduplicates.

`lib.mkPiCredentialContract` accepts optional `nexusPublicKeys`. The live secrets flake and `secrets.nix` always pass `identity.hostPublicKeys`, so they never require the hypervisor in `machine-host-keys.json`. When an external caller omits the new argument, the constructor temporarily derives the old list from `machineHostKeys.${nexusName}`; this compatibility path preserves the current deploy canary and is covered as an existing consumer, not used as the new deployment contract. `lib.nexusIdentity.sshPublicKeys` continues to expose the same ordered identity list.

The host option `nexus.provisioning.inventoryCheckout`, when set, renders both `INVENTORY` and `INVENTORY_CHECKOUT` to the same absolute path. An unset option renders neither, preserving the public template fallback; malformed paths still fail Nix evaluation.

`pi-provider` reads repository aliases from `${INVENTORY}/scripts/repositories.json`. It accepts the deployment aliases `profiles` and `secrets` when present and the public `allod/profiles` and `allod/secrets` aliases otherwise, but it never falls back after a missing, malformed, or incomplete registry: the error names the registry, alias choices, and resolved purpose.

After ordinary source validation and clean-tree checks but before a dry run returns or a live command prompts, the script copies the exact profiles and secrets working trees into its existing tmpfs workdir, excluding no tracked source. It overlays the staged registries and creates empty mode-`0600` placeholder ciphertexts only inside the temporary secrets copy for newly planned token paths. Those placeholders contain no bearer and witness existence only.

The script evaluates the staged temporary profiles copy's `lib.piProviders` and the staged temporary secrets copy's `lib.piCredentialRecipients`. The latter is the exact recipient map the secrets flake and `secrets.nix` derive, so a split hypervisor registry, nonliteral hypervisor hostname, active/staged ordering, and the contract's global rejection of keys reused by distinct recipient sources come from the data owner rather than Nexus. The script requires every planned ciphertext path to have a non-empty, internally unique `publicKeys` list and uses that list's exact stored order as Age's recipient order; it never silently deduplicates. Repetition of the same valid hypervisor or target key across different ciphertext paths is expected and is not a duplicate-source error.

The preflight captures both staged outputs. After installing the journaled changes, the script evaluates the live `profiles.lib.piProviders` and `secrets.lib.piCredentialRecipients` and requires exact JSON equality with the captured results. A static or disconnected redirected fork therefore fails in preflight when its temporary output does not follow the overlaid registry; any later divergence fails post-install and triggers rollback. Repository-specific `checks.<system>.<name>` attributes are not part of the CLI contract.

## Agent Gates

The agent cannot rebuild the host or VMs, enter a real bearer, make a provider request, change private repositories, update private deploy pins, or revoke a remote token. After the public PR lands, the human must rebuild Nexus, run a real dry run and lifecycle operation against the redirected checkouts, review and commit the data changes, update deployment pins, rebuild affected VMs, verify a fresh request, and only then revoke an obsolete token. A private secrets fork with static Pi declarations must first adopt the public derived `secrets.nix` pattern; the staged-copy preflight will refuse it beforehand.

## Acceptance Tests

1. In `allod/secrets`, build `.#checks.x86_64-linux.pi-credential-registry` and `.#checks.x86_64-linux.credential-inventory`. Synthetic fixtures use a non-`nexus` hypervisor absent from `machineHostKeys`; prove hypervisor active/staged then target active/staged order, invalid or duplicate hypervisor keys, cross-source duplicate rejection, derived `secrets.nix` equality, and the compatibility fallback.
2. Against the secrets worktree, build the existing deploy `.#checks.x86_64-linux.composed-layer` with its `secrets` input overridden to the worktree. This measures the only known external `mkPiCredentialContract` consumer without requiring a deploy PR.
3. In `allod/nexus`, run `bash tests/pi-provider.sh`. Fixtures must prove configured inventory selects redirected `profiles`/`secrets` aliases; the public aliases remain a fallback only inside a valid registry; and missing, malformed, or incomplete registries fail with both repositories untouched.
4. The same witness must prove a staged temporary profiles output follows the overlaid catalog and a staged temporary secrets output follows the overlaid credential registry before dry-run completion and before bearer input. Static/disconnected output, missing recipient paths, empty or internally duplicated recipient lists, and secrets-output evaluation failures (including duplicate recipient sources) fail before a prompt and leave both repositories untouched.
5. Synthetic split-key fixtures must use a non-`nexus` hypervisor absent from `machine-host-keys.json`; its active/staged keys and the target active/staged keys can decrypt, while unrelated keys cannot. Recipient argv ordering must equal the staged secrets output exactly.
6. Post-install output drift or evaluation failure must restore both recorded pre-run states. Existing interruption/recovery and bearer leak witnesses remain green, including absence from output, regular files, argv, environment, and logs.
7. Build `.#checks.x86_64-linux.host-provisioning-env`: a configured inventory path renders identically as `INVENTORY` and `INVENTORY_CHECKOUT`, an unset option renders neither, and malformed paths fail evaluation.
8. Build `.#checks.x86_64-linux.provisioning-contract` so Bash syntax, ShellCheck, the full host-script fixture suite, and packaged executable witness pass. The fixture Nix shim asserts the exact staged/live evaluation argv and includes a deliberate sabotage proving each new comparator fails.

Host/VM rebuilds and a live provider request remain human-only measurements and are not claimed by the public PR.

## Rollback Plan

Before private adoption, leave deployment pins unchanged or revert the secrets and Nexus commits in reverse order. During an ordinary `pi-provider` failure, rely on its journaled rollback; after an interruption, run `pi-provider recover` before touching either checkout. After adoption, restore the prior Nexus and secrets pins, rebuild Nexus, and rerun the prior command. A bearer encrypted only during a failed local attempt can be replaced at the provider; remote revocation remains a deliberate human step after verification.
