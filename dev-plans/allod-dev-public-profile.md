# Usable Public allod-dev VM

## Tracking Issue

https://forge.anarch.diy/allod/profiles/issues/2

This work is broader than a mechanical `dev-1` rename. The implementation should
make the public `allod-dev` example usable without editing repo definitions:
clone the public Allod repos, run the documented host-side command, and get an
`allod-dev` VM with the public Allod workspace checked out.

Expected PR sequence:

- `allod/nexus`: `Refs allod/profiles#2`
- `allod/inventory`: `Refs allod/profiles#2`
- `allod/secrets`: `Refs allod/profiles#2`
- `allod/profiles`: `Closes allod/profiles#2`

The final `profiles` PR carries the closing keyword because it consumes the
merged public `nexus`, `inventory`, and `secrets` contracts.

## Goal

Ship a public `allod-dev` VM definition that a user can clone and provision for
read-only Allod development without replacing fake placeholder data in the repos.

## Scope

In scope:

- `allod/nexus`: support credentialless public dev VM provisioning and workspace
  bootstrap, including HTTPS clones and verification when no Forge SSH key or
  Forgejo token is configured, runtime VM host-key generation with host-side
  `known_hosts` pinning when no committed host-key state exists, injection of
  the operator's `~/.ssh/host.pub` into the VM user's `authorized_keys` via
  `--extra-files`, and the documented installer-image path that authorizes
  the same runtime key.
- `allod/inventory`: replace `dev-1` with `allod-dev`, declare the real public
  Allod workspace repo set, and use public HTTPS clone URLs for repos that need
  no credentials.
- `allod/secrets`: remove fake credential assumptions from the public example;
  expose real public defaults for the Allod org, forge host, VM username, and
  optional credential fields without shipping working private keys or tokens;
  empty the machine host-key state (`machine-host-keys.json`) so the stock
  template pins no VM host keys.
- `allod/profiles`: expose `nixosConfigurations.allod-dev`, convert the
  `nexus` and `inventory` flake inputs from `git+ssh://` to public
  `git+https://` URLs, make dev-profile credential modules inert when
  token/key fields are null, make the `credential-profiles` host-key entry
  requirement conditional on committed host-key state, and document the
  no-edit public provisioning path plus optional credential wiring for forks.
- Tests and docs that prove `allod-dev` builds, provisions through fixture paths,
  bootstraps public repo checkouts, and has no stale `dev-1` placeholders.

Out of scope:

- Shipping any working private SSH key, Forgejo token, GPG key, or user-specific
  host key in public repos.
- Granting Forgejo permissions or configuring the `allod-agent` account.
- Re-encrypting real secrets, rotating credentials, or changing private
  `vnprc/*` repos.
- Guaranteeing push/PR capability in the stock public VM. Stock public
  `allod-dev` is read-only for public repo clone/build workflows; forks can add
  credentials through the documented optional fields.
- Host-side VM creation outside existing `nexus` provisioning commands, except
  for local runtime inputs those commands collect without modifying repo files.

## Risk Assessment

Residual risk: R3 High

Why:

- The change spans four repos and changes cross-repo contracts between
  inventory, identity data, profile generation, and provisioning scripts.
- The goal intentionally removes fake credential material from active runtime
  paths. That improves public usability but touches generated NixOS activation,
  workspace bootstrap, health checks, and optional auth behavior.
- Validation can cover Nix flake checks, generated activation text, no-auth
  bootstrap fixtures, public repo registry resolution, and stale-placeholder
  absence.
- This is not R4 because the public example must not mutate deployed
  infrastructure or remote Forgejo state, must not ship real secrets, and can be
  rolled back by reverting public repo PRs in reverse dependency order.

Human scrutiny:

- Confirm the public VM is usable for read-only work without repo edits and
  without fake placeholders.
- Confirm no private material, personal host addresses, private repo URLs, or
  bundled private keys/tokens were introduced.
- Review the no-auth provisioning/bootstrap path and generated activation
  behavior before merge.

| PR or milestone | Risk | Reason | Human scrutiny |
|---|---|---|---|
| PR 1: `allod/nexus` no-auth bootstrap | R3 High | Changes provisioning, bootstrap, and verification scripts so dev VMs can have public repos without Forge SSH credentials. | Verify no-auth branches are fixture-tested and authenticated behavior remains intact. |
| PR 2: `allod/inventory` public allod-dev data | R2 Medium | Renames the public VM and records the real public Allod repo workspace using public clone URLs. | Verify repo registry paths, checkout paths, and `vm-specs.json` generation. |
| PR 3: `allod/secrets` public identity cleanup | R3 High | Removes fake credential assumptions and exposes null/optional auth fields consumed by profiles. | Verify checks prove absence of fake keys/tokens and preserve optional credential contracts. |
| PR 4: `allod/profiles` integration | R3 High | Consumes the updated upstreams and changes generated profile behavior from placeholder `dev-1` to usable `allod-dev`. | Verify locks point at merged upstreams, generated config has no required fake secrets, and docs match the command path. |

## Interface Contracts

- The public dev VM machine/system name is `allod-dev`.
- The stock public `allod-dev` VM is a read-only-capable public development VM:
  it can clone and rebuild from public Allod repos without a Forge SSH key or
  Forgejo token.
- Stock public repos must not include working private keys, GPG private keys,
  Forgejo tokens, or user-specific host private keys.
- Any user-specific SSH public key or host provisioning input must be collected
  by host-side commands or environment variables at runtime, not committed into
  public repo definitions.
- Stock public repos ship no `allod-dev` host-key state: no
  `secrets/vm-host-keys/allod-dev-ssh.age` and no `machine-host-keys.json`
  entry. When a VM has no pinned host-key state, `provision-vm-from-host`
  generates an ed25519 host key on the host at runtime, injects it through the
  existing `--extra-files` tree, and records the public half host-side outside
  the repos (for example `~/.ssh/vm-host-keys/<vm>.pub`). Do not resolve this
  by letting the VM generate its own key at first boot: the host could then
  never pin `known_hosts` before connecting with
  `StrictHostKeyChecking=yes`, and fork agenix decryption expects the
  identity present before first boot. Refusal on missing, undecryptable, or
  mismatched ciphertext remains fail-fast for VMs that do declare pinned
  host-key state.
- `bootstrap-vm-from-host.sh` and `verify-vm-from-host` must pin
  `KNOWN_HOSTS_VMS` from the runtime-recorded public key when
  `machine-host-keys.json` has no entry for the VM, keeping
  `StrictHostKeyChecking=yes`. Dying on a missing JSON entry remains only for
  VMs with committed host-key state.
- The `credential-profiles` check in `allod/profiles` requires a
  `credentials."<vm>-host"` entry for every dev/privacy VM, and
  `allod/secrets` generates those entries from `machine-host-keys.json`. That
  requirement must become conditional on committed host-key state so
  `allod-dev` passes with no entry; renaming the `dev-1` entry to `allod-dev`
  with the old synthetic key is not an option, because bootstrap would pin
  `known_hosts` to a key the runtime-generated VM can never present.
- The operator's runtime SSH identity is the existing `~/.ssh/host` /
  `~/.ssh/host.pub` keypair (the `AGE_IDENTITY` / `KEY` defaults the `nexus`
  scripts already use). Provisioning must authorize `~/.ssh/host.pub` on the
  installed system by writing the VM user's `~/.ssh/authorized_keys` through
  the existing nixos-anywhere `--extra-files` tree. NixOS's default
  `authorizedKeysFiles` already includes `%h/.ssh/authorized_keys`, and an
  on-disk file survives stock `self_rebuild` runs — unlike anything derived
  from the build-time `hostPublicKeys` in `secrets`, which activation
  regenerates from the synthetic committed value on every stock rebuild.
  The baked `hostPublicKeys` stay as the fork seam; they are not the stock
  access path.
- The install media must trust the same runtime key. `profiles#installer`
  bakes the synthetic `secrets.lib.identity.hostPublicKeys` into root's
  authorized keys, so an ISO built purely from stock inputs rejects a real
  operator and nixos-anywhere never connects. The documented host-side path
  must produce install media whose root login accepts `~/.ssh/host.pub`
  (for example building `profiles#installer` with a runtime secrets
  override); per the Agent Gates below, the exact command ships in the
  `nexus` PR documentation before downstream repos rely on it.
- Optional credential fields use `null` to mean "not configured"; modules and
  scripts must treat null as inert instead of trying to deploy or decrypt fake
  files.
- `allod/inventory` must expose `machines.allod-dev` with:
  - `platform = "x86_64-linux"`
  - `type = "dev"`
  - `memory_mb = 8192`
  - `vcpus = 6`
  - `disk_gb = 50`
  - `ip = "192.168.122.15"`
  - `mac = "52:54:00:ab:cd:15"`
  - `forge_key = null`
  - `self_rebuild = true`
  - `repos` containing the public Allod workspace: `profiles`, `vm`, `nexus`,
    `workspace-tools`, `strategy`, `allod/secrets`, `allod/inventory`, and
    `allod/memory`
- `allod/inventory/scripts/repositories.json` must resolve the stock public
  workspace repos to public URLs and stable checkout paths:
  - `profiles` -> `https://forge.anarch.diy/allod/profiles.git` ->
    `allod/profiles`
  - `vm` -> `https://forge.anarch.diy/allod/vm.git` -> `allod/vm`
  - `nexus` -> `https://forge.anarch.diy/allod/nexus.git` -> `allod/nexus`
  - `workspace-tools` -> `https://forge.anarch.diy/allod/tools.git` ->
    `allod/tools`
  - `strategy` -> `https://forge.anarch.diy/allod/strategy.git` ->
    `allod/strategy`
  - `allod/secrets` -> `https://forge.anarch.diy/allod/secrets.git` ->
    `allod/secrets`
  - `allod/inventory` -> `https://forge.anarch.diy/allod/inventory.git` ->
    `allod/inventory`
  - `allod/memory` -> `https://forge.anarch.diy/allod/memory.git` ->
    `allod/memory`
- `allod/secrets` must expose `lib.devIdentities.allod-dev` with username
  `allod`, public Forge host `forge.anarch.diy`, and null optional credential
  fields for token/GPG/Forge SSH material in the stock public state.
- Public `allod/secrets` checks must allow the stock no-credential profile while
  still validating any optional credential records a fork supplies.
- `allod/profiles` flake inputs must be fetchable without credentials. The
  `nexus` and `inventory` inputs currently use
  `git+ssh://git@forge.anarch.diy:2222/...`, which fails for a credentialless
  user at first `nix build` and fails again inside the VM when
  `self_rebuild = true` re-fetches inputs during `nixos-rebuild switch`. All
  forge-hosted inputs must use public `git+https://` URLs and the regenerated
  `flake.lock` must contain no `ssh://` URLs.
- `allod/profiles` must expose `nixosConfigurations.allod-dev` and must not
  expose `nixosConfigurations.dev-1`.
- `allod/profiles` credential modules must declare no `age.secrets` entries when
  their token/key inputs are null.
- `allod/profiles` netrc activation must continue to skip cleanly when
  `/root/.git-credentials` is absent.
- `allod/nexus` bootstrap and verify scripts must operate on repo lists even
  when `forge_key` is null, as long as every selected repo can be cloned through
  unauthenticated public URLs.
- Authenticated Forge SSH clone behavior must keep working for forks or private
  deployments that provide a Forge SSH key.

## Agent Gates

- Agents with private `vnprc` secret access must not push public Allod repos.
  A human or public-only agent must push public implementation branches and open
  PRs if the active agent has that private access.
- Do not run agenix rekeying, token rotation, Forgejo account mutation, or
  host-only VM provisioning/rebuild commands as part of implementation
  validation. Use fixture tests for provisioning behavior.
- Do not solve public usability by committing private keys, shared demo private
  keys, working tokens, or personal host-specific data.
- If runtime user SSH input requires a new command-line interface, document the
  contract in the plan or PR before relying on it in downstream repos.

## Acceptance Tests

```sh
set -euo pipefail
WORK=${WORK:-$HOME/work}

cd "$WORK/allod/nexus"
bash -n scripts/provision-vm-from-host scripts/bootstrap-vm-from-host.sh \
  scripts/bootstrap-vm.sh scripts/verify-vm-from-host scripts/lib/resolve-repos.sh
bash tests/bootstrap-orchestration.sh
bash tests/provision-vm-from-host.sh
bash tests/rebuild-vm-from-host.sh
bash tests/provisioning-contract.sh
bash tests/registry-resolver.sh
nix flake check

cd "$WORK/allod/inventory"
nix flake check
nix eval .#lib.vmSpecsJson --raw | jq -S . > /tmp/allod-dev-vm-specs.expected.json
jq -S . scripts/vm-specs.json > /tmp/allod-dev-vm-specs.actual.json
diff -u /tmp/allod-dev-vm-specs.expected.json /tmp/allod-dev-vm-specs.actual.json
nix eval --json .#machines --apply builtins.attrNames \
  | jq -e 'index("allod-dev") != null and index("dev-1") == null'
jq -e '."allod-dev".repos
       | index("profiles") and index("vm") and index("nexus")
         and index("workspace-tools") and index("strategy")
         and index("allod/secrets") and index("allod/inventory")
         and index("allod/memory")' scripts/vm-specs.json
jq -e '."allod-dev".ip == "192.168.122.15"
       and ."allod-dev".mac == "52:54:00:ab:cd:15"
       and ."allod-dev".forge_key == null
       and ."allod-dev".self_rebuild == true' scripts/vm-specs.json
jq -e '.repositories.profiles.source == "git"
       and .repositories.profiles.remote == "https://forge.anarch.diy/allod/profiles.git"
       and .repositories.vm.remote == "https://forge.anarch.diy/allod/vm.git"
       and .repositories.nexus.remote == "https://forge.anarch.diy/allod/nexus.git"' \
  scripts/repositories.json
! rg -n 'dev-1|dev_1|forgejo-https-token-dev-1|dev-1-forge-key' \
  flake.nix scripts/vm-specs.json scripts/repositories.json

cd "$WORK/allod/secrets"
nix flake check
nix eval --json .#lib.devIdentities --apply builtins.attrNames \
  | jq -e 'index("allod-dev") != null and index("dev-1") == null'
nix eval --json .#lib.devIdentities.allod-dev \
  | jq -e '.username == "allod"
           and .forgeHost == "forge.anarch.diy"
           and (.forgeTokenFile == null)
           and (.agentTokenFile == null)
           and (.gpgPublicKeyFile == null)
           and (.gpgSigningKey == null)'
! find secrets keys -type f | grep -E 'dev-1|dev_1|token|forge-key'
jq -e '. == {}' machine-host-keys.json
! rg -n 'dev-1|dev_1|forgejo-https-token-dev-1|dev-1-forge-key|PRIVATE KEY|BEGIN OPENSSH' \
  identity.nix credentials.nix forge-ssh-keys.json forgejo-token-groups.json \
  secrets.nix machine-host-keys.json git keys secrets

cd "$WORK/allod/profiles"
nix flake check
! rg -n 'git\+ssh|ssh://' flake.nix flake.lock
nix eval --json .#nixosConfigurations --apply builtins.attrNames \
  | jq -e 'index("allod-dev") != null and index("dev-1") == null'
nix build .#nixosConfigurations.allod-dev.config.system.build.toplevel --no-link
nix eval .#nixosConfigurations.allod-dev.config.system.activationScripts.netrc.text --raw \
  > /tmp/allod-dev-netrc-activation.sh
! grep -n '^[[:space:]]*exit 0$' /tmp/allod-dev-netrc-activation.sh
grep -q 'missing or empty, skipping' /tmp/allod-dev-netrc-activation.sh
nix eval --json .#nixosConfigurations.allod-dev.config.age.secrets \
  | jq -e 'has("forgejo-https-token") | not'
test -d hosts/dev/allod-dev
test ! -e hosts/dev/dev-1
! rg -n 'dev-1|dev_1|forgejo-https-token-dev-1|dev-1-forge-key' \
  flake.nix README.md hosts modules
```

Required fixture coverage:

- `allod/nexus` tests include a dev VM with repos and `forge_key = null`; it
  bootstraps and verifies public `git` source repos without calling
  `setup-vm-forge-key.sh` or requiring `~/.ssh/<forge-key>`.
- `allod/nexus` tests include a public no-auth provisioning fixture with no
  `secrets/vm-host-keys/<vm>-ssh.age` and no `machine-host-keys.json` entry;
  it must generate a runtime host key, place it in the `--extra-files` tree,
  record the public half host-side, pin `KNOWN_HOSTS_VMS` from that record,
  write the fixture operator's public key into the `--extra-files`
  authorized_keys path for the VM user, and reach the installer/build phase. The existing refusal tests in
  `tests/provision-vm-from-host.sh` (`missing_ciphertext`, `decrypt_failure`,
  `key_mismatch`, `stale_pin`) must keep passing for VMs that declare pinned
  host-key state — the fail-fast becomes conditional, not deleted.
- `allod/nexus` tests keep existing authenticated Forge SSH bootstrap behavior
  for repos that use `source = "forge"` and provide a Forge SSH key.
- `allod/profiles` checks inspect generated `allod-dev` activation text and age
  secret attrs, not only source files.

## Rollback Plan

- Before merge, abandon the affected branch or worktree in each public repo.
- After merge, revert in reverse dependency order: `allod/profiles`, then
  `allod/secrets`, then `allod/inventory`, then `allod/nexus`.
- If a downstream flake lock points at a reverted upstream state, update the
  lock to the restored upstream commit and rerun acceptance tests.
- If no-auth bootstrap regresses while authenticated bootstrap still works,
  revert only the no-auth `nexus` PR before merging downstream public
  `allod-dev` changes.
- If fake credential material remains after a partial implementation, stop the
  release path until the stale files or references are removed; do not paper
  over them with new placeholder secrets.
