## Implementation Plan - Allod-dev VM and Agent Isolation

### Prerequisites

These must be completed before this plan:
- `dev-plans/archive/parameterize-ai-agents.md` — ai-agents.nix accepts `memoryCheckouts`
- `archive/dev-plans/per-vm-checkout-uniqueness.md` — inventory check relaxed to per-VM
- `archive/dev-plans/split-agent-memory.md` — `allod/memory` exists with public workflow content
- `archive/dev-plans/rename-llm-memory-plan.md` — repo rename complete

See `dev-plans/agent-isolation-roadmap.md` for the full sequence.

### Tracking Issue

TBD — create before implementation starts.

### Goal

Prevent agents from reading or leaking private identity and secret data by isolating agent work to a dedicated VM with a restricted Forgejo identity that can only access public allod org repos, backed by public template repos containing real structure but synthetic values.

### Scope

**In scope:**

Phase 1 — Forge bot user (human-only):
- Forgejo admin: create `allod-agent` user and configure org membership

Phase 2a — Public secrets template (`allod/secrets`):
- `allod/secrets/flake.nix` — public flake mirroring `vnprc/secrets` interface
- `allod/secrets/identity.nix` — synthetic identity with allod-agent values
- `allod/secrets/credentials.nix` — credential inventory for allod-dev only
- `allod/secrets/secrets.nix` — agenix secret declarations for allod-dev
- `allod/secrets/machine-host-keys.json` — allod-dev host key only
- `allod/secrets/modules/preferences.nix` — copy of user preferences module (no secrets)
- `allod/secrets/keys/` — allod-agent SSH public keys
- `allod/secrets/secrets/` — age-encrypted dummy tokens
- `allod/secrets/git/` — git workflow config files (protected-branches, allowed-external-remotes, etc.)

Phase 2b — Public inventory template (`allod/inventory`):
- `allod/inventory/flake.nix` — public flake mirroring `vnprc/inventory` interface
- `allod/inventory/scripts/vm-specs.json` — derived from Nix attrset
- `allod/inventory/scripts/repositories.json` — allod-org-only repo registry

Phase 3 — allod-dev VM profile:
- `vnprc/inventory/flake.nix` — add `allod-dev` machine entry
- `vnprc/inventory/flake.nix` — allow isolated dev VMs that do not self-rebuild from a local profiles checkout
- `vnprc/inventory/scripts/vm-specs.json` — regenerate
- `vnprc/inventory/scripts/repositories.json` — add `allod/secrets` and `allod/inventory` aliases
- `vnprc/secrets/identity.nix` — add `allod-dev` to `devVMs` with the public runtime username and allod-agent forge user
- `vnprc/secrets/flake.nix` — allow per-VM username/forge-user overrides in `devIdentities`
- `vnprc/secrets/credentials.nix` — add `allod_vm` forge-git credential and allod-agent token records
- `vnprc/secrets/secrets.nix` — add allod-dev token paths without adding allod-dev to the shared `agent-pr-token.age` recipients
- `vnprc/secrets/machine-host-keys.json` — add allod-dev entry (after provisioning)
- `profiles/flake.nix` — add the public `allod/secrets` flake input and split dev VM secret identity from runtime identity
- `profiles/modules/home-shared.nix` — consume the runtime identity, not raw private identity
- `profiles/modules/ai-agents.nix` — consume the runtime identity and allod/memory checkout list
- `profiles/modules/agent-hooks.nix` — accept a git-policy source and disable private-profile helper sources for allod-dev
- `profiles/modules/agent-forgejo-token.nix` — accept a per-VM token file instead of hardcoding `agent-pr-token.age`
- `profiles/hosts/dev/allod-dev/configuration.nix` — VM system config
- `profiles/hosts/dev/allod-dev/home.nix` — Home Manager config
- `nexus/scripts/bootstrap-vm-from-host.sh` and `nexus/scripts/bootstrap-vm.sh` — support clone-only isolated dev VMs that skip on-VM `nixos-rebuild switch`

Phase 4 — Verification:
- Manual verification of access control boundaries

**Out of scope:**
- Open-sourcing nexus, vm, or profiles (separate effort, depends on this work)
- Pre-commit content scanning hooks (VM isolation is sufficient)
- Changes to existing VMs (nix-dev, rust-dev, svelte-dev) beyond what prereqs already cover
- A general profiles split for public consumers; this plan adds only the allod-dev-specific public runtime input needed to keep private data out of the isolated VM
- ai-agents.nix parameterization (prereq, already landed)
- Checkout path collision fix (prereq, already landed)
- Memory repo split (prereq, already landed)
- Repo rename (prereq, already landed)

### Architecture

The isolation model has three layers:

1. **Forge identity**: The `allod-agent` Forgejo user belongs to the `allod` org but has zero access to `vnprc/*` repos. The allod-dev VM authenticates as this user.

2. **Filesystem isolation**: The allod-dev VM only clones allod org repos. Private repos (`vnprc/secrets`, `vnprc/inventory`, `vnprc/agent-memory`, `vnprc/nvim-config`) are never present on disk.

3. **Runtime closure isolation**: The allod-dev Home Manager configuration, git policy files, agent adapter files, and symlinked sources must come from public inputs or generated text. The VM closure must not contain the private `vnprc/secrets`, `vnprc/inventory`, `vnprc/profiles`, or `vnprc/nvim-config` source trees. The only private material allowed in the closure is individual encrypted `.age` payloads for allod-dev, copied as single-file store paths so the parent private repo source is not traversable.

4. **Public repos**: `allod/secrets`, `allod/inventory`, and `allod/memory` provide the module structure, workflow memory, and machine specs agents need for cross-repo development without exposing real identity data.

The allod-dev VM is built by the human through the existing profiles infrastructure. Host-side evaluation may read `vnprc/secrets` and `vnprc/inventory` to provision host keys, encrypted allod-dev credentials, and the actual VM IP. The allod-dev user environment is composed from the public `allod/secrets` runtime identity and public git policy files, so private build inputs are not readable from the booted VM.

#### What the agent CAN do on allod-dev:
- Read, edit, and push to allod/* repos via the allod-agent forge identity
- Run `nix flake check` against the public template repos

#### What the agent CANNOT do on allod-dev:
- Read vnprc/secrets, vnprc/inventory, or vnprc/agent-memory (not cloned)
- Read private source trees through `/nix/store` or generated Home Manager files
- Push to vnprc/* repos (allod-agent has no access)
- See private inventory IPs, hostnames, tokens, or personal identity data beyond the VM's own observable network address
- Read private memory files (hashpool.md)

#### Cross-repo development workflow:
1. Agent implements structural changes in allod/secrets or allod/inventory
2. Agent's PR includes a `## Human steps` section listing what to copy to vnprc/*
3. Human reviews, merges the allod PR, then replicates structural changes to vnprc/* with real values
4. Human runs agenix re-encryption and VM rebuild as needed

### Interface Contracts

#### `allod/secrets` flake outputs

Must export the same `lib` and `homeModules` interface as `vnprc/secrets`:

```nix
lib.devIdentities     : { <vm-name> = { username, forgeHost, forgePort, forgeUser, gpgSigningKey, sshKeyName, forgeTokenFile, agentTokenFile, gpgPublicKeyFile }; }
lib.privacyIdentities : { <vm-name> = { username }; }
lib.nexusIdentity     : { username, hostname, forgeHost, forgePort, sshPublicKey, forgeTokenFile }
lib.vmUsernames       : { <vm-name> = <username>; }
lib.credentials       : { <name> = { name, kind, owner, public_key, consumers, rotation_state }; }
lib.identity          : <raw identity.nix attrset>

homeModules.preferences : Home Manager module (programs.git, neovim, firefox, bash)

checks.<system>.credential-inventory : validates internal consistency
```

Values in the public repo use synthetic identity data:

```nix
# allod/secrets/identity.nix
rec {
  username = "allod";
  email = "allod@example.com";
  hostname = "hypervisor";
  hostPublicKey = "<generated-throwaway-ed25519-pubkey>";
  forgeHost = "forge.anarch.diy";
  forgePort = 2222;
  forgeUser = "allod-agent";
  gpgSigningKey = null;

  devVMs = {
    dev-1 = { sshKeyName = "dev_1"; };
  };

  privacyVMs = {};

  sshHosts = {
    dev-1 = {
      hostname = "192.168.122.10";
      user = "allod";
      identityFile = "~/.ssh/host";
      extraOptions = {
        UserKnownHostsFile = "~/.ssh/known_hosts_vms";
      };
    };
    "forge.anarch.diy" = {
      hostname = "forge.anarch.diy";
      user = "git";
      port = 2222;
      identityFile = "~/.ssh/host";
      identitiesOnly = true;
    };
  };
}
```

#### `allod/inventory` flake outputs

Must export the same interface as `vnprc/inventory`:

```nix
machines             : { <vm-name> = { platform, type, ... }; }
lib.machines         : same as machines
lib.supportedPlatforms : [ <nix-system-string> ]
lib.vmSpecsJson      : JSON string, including `self_rebuild` for dev VMs (`true` unless explicitly false)

checks.<system>.vm-specs-json         : validates scripts/vm-specs.json matches Nix
checks.<system>.repository-registry   : validates scripts/repositories.json
```

Values in the public repo use a single example dev VM:

```nix
machines = {
  dev-1 = {
    platform = "x86_64-linux";
    type = "dev";
    memory_mb = 8192;
    vcpus = 4;
    disk_gb = 50;
    ip = "192.168.122.10";
    mac = "52:54:00:00:00:10";
    forge_key = "dev_1";
    self_rebuild = false;
    repos = [ "workspace-tools" "strategy" "allod/secrets" "allod/inventory" "allod/memory" ];
  };
};
```

#### `allod-dev` identity in `vnprc/secrets`

New entry in `identity.devVMs`:

```nix
allod-dev = {
  username = "allod";
  forgeUser = "allod-agent";
  sshKeyName = "allod_vm";
};
```

Update the `devIdentities` derivation logic so per-VM `username` and `forgeUser` override the global defaults. Export a per-VM `agentTokenFile` path as `./secrets + "/agent-pr-token-${name}.age"` alongside the existing `forgeTokenFile`.

The public username is intentional: `qemuGuest`, Home Manager, netrc activation, bootstrap SSH, and agent adapter paths all consume `identity.username`. Leaving allod-dev on the global private username would leak personal identity into `/home`, git config, adapter files, and provisioning output.

#### `allod-dev` machine in `vnprc/inventory`

New entry in `machines`:

```nix
allod-dev = {
  platform = "x86_64-linux";
  type = "dev";
  memory_mb = 8192;
  vcpus = 6;
  disk_gb = 50;
  ip = "192.168.122.15";
  mac = "52:54:00:ab:cd:15";
  forge_key = "allod_vm";
  self_rebuild = false;
  repos = [ "workspace-tools" "strategy" "allod/secrets" "allod/inventory" "allod/memory" ];
};
```

The `repos` list excludes all vnprc-private aliases: `profiles`, `vm`, `nexus`, `secrets`, `inventory`, `nvim-config`, `agent-memory`, `notes`, and `forgejo-config`.

`self_rebuild = false` marks allod-dev as host-managed after installation. Update `vmSpecsJson`, the repository registry check, and bootstrap scripts so development VMs with `self_rebuild = false` are allowed to omit `profiles`; bootstrap should copy the forge key and clone/pull repositories, then skip the on-VM `sudo nixos-rebuild switch`. Existing development VMs default to `self_rebuild = true` and must still include `profiles`.

New entries in `repositories.json`:

```json
"allod/secrets": {
  "source": "forge",
  "remote": "Allod/secrets",
  "checkout": "allod/secrets"
},
"allod/inventory": {
  "source": "forge",
  "remote": "Allod/inventory",
  "checkout": "allod/inventory"
}
```

Do not rename the existing `workspace-tools` alias in this plan. It is already the public `allod/tools` checkout, and renaming it would touch existing dev VMs for no isolation gain.

#### `allod-dev` VM profile

`profiles/flake.nix`:
- Add `allod-secrets.url = "git+https://forge.anarch.diy/Allod/secrets.git"` as a direct public input.
- Do not add `allod/inventory` as a profiles flake input; public inventory is a runtime workspace checkout only.
- Keep `secrets` and `inventory` pointed at `vnprc/*` for host-side build/provisioning data.
- Extend `mkDevVm` so secret-bearing values (`forgeTokenFile`, `agentTokenFile`, `sshKeyName`) stay separate from runtime values (`username`, `email`, `sshHosts`, git policy source, preferences module, memory checkouts).
- For existing dev VMs, defaults preserve current behavior.
- For allod-dev, pass `runtimeIdentity = allod-secrets.lib.identity`, `gitPolicySource = allod-secrets`, `preferencesModule = allod-secrets.homeModules.preferences`, and `memoryCheckouts = [ "allod/memory" ]`.

`profiles/hosts/dev/allod-dev/configuration.nix`:
- Imports `agent-forgejo-token.nix` with `tokenFile = identity.agentTokenFile`; do not use the shared `agent-pr-token.age`
- Packages: `git`, `jq`, `age` — no nix linting tools unless needed
- No special system config beyond the dev VM baseline

`profiles/hosts/dev/allod-dev/home.nix`:
- Packages: `claude-code`, `codex` (same as other dev VMs)
- SSH matchBlock for forge.anarch.diy using the allod-dev forge key `identity.sshKeyName` (resolves to `allod_vm`) and the public runtime forge host/port
- No GPG signing (allod-agent has no GPG key; `gpgSigningKey = null`)

For allod-dev, `mkDevVm` passes `memoryCheckouts = [ "allod/memory" ]`, so symlinks point to `~/work/allod/memory` instead of `~/work/agent-memory`.

#### allod-agent credential in `vnprc/secrets`

New entry in `credentials.nix`:

```nix
allod_vm = {
  name           = "allod_vm";
  kind           = "forge-git";
  owner          = "allod-dev";
  public_key     = "<allod-agent-ssh-ed25519-pubkey>";
  consumers      = [
    { type = "forge-key-secret"; repo = "secrets"; secret = "secrets/allod-dev-forge-key.age"; }
    { type = "forgejo-ssh"; account = "allod-agent"; key = "allod_vm"; }
  ];
  rotation_state = "active";
};
```

Note: the `forgejo-ssh` consumer references account `allod-agent` (the bot user), not `vnprc`.

Add an allod-agent API token record for the Forge CLI. It must be a distinct secret path from the shared private agent token:

```nix
agent-pr-token-allod-dev = {
  name           = "agent-pr-token-allod-dev";
  kind           = "agent";
  owner          = "allod-agent";
  public_key     = null;
  consumers      = [
    { type = "agenix"; repo = "secrets"; secret = "secrets/agent-pr-token-allod-dev.age"; }
  ];
  rotation_state = "active";
};
```

The HTTPS credential-store token and Forge CLI token may contain the same allod-agent token material if the scopes are sufficient, but they are stored separately because `/root/.git-credentials` needs URL format while `forge` reads a raw token.

New entries in `secrets.nix`:

```nix
"secrets/allod-dev-forge-key.age".publicKeys     = [ hostKey ] ++ vmKeys "allod-dev";
"secrets/forgejo-https-token-allod-dev.age".publicKeys = [ hostKey ] ++ vmKeys "allod-dev";
"secrets/agent-pr-token-allod-dev.age".publicKeys = [ hostKey ] ++ vmKeys "allod-dev";
```

#### Git workflow config for allod-dev (`allod/secrets/git/`)

The public secrets repo needs git workflow config that restricts the allod-dev agent to allod org repos only:

`git/protected-branches`:
```
# allod-dev only works on allod org repos
work/allod/tools master
work/allod/strategy master
work/allod/secrets master
work/allod/inventory master
work/allod/memory master
```

`git/allowed-external-remotes`: empty (allod-dev should only push to forge.anarch.diy)

`git/active-pr-branches` and `git/signing-required-branches`: empty (allod-agent does not GPG-sign)

### Agent Gates

**Phase 1 — Forge bot user** (all human/admin):
1. Create `allod-agent` user on forge.anarch.diy (admin panel)
2. Create the `allod` org team for the bot user (if not using default org membership)
3. Grant `allod-agent` write access to all `allod/*` repos
4. Verify `allod-agent` has NO access to any `vnprc/*` repos
5. Verify all `vnprc/*` repos are set to private visibility
6. Generate SSH ed25519 keypair for allod-agent on nexus host: `ssh-keygen -t ed25519 -C allod_vm -f allod_vm`
7. Register the public key on `allod-agent`'s Forgejo SSH keys
8. Generate a Forgejo HTTPS access token for `allod-agent` (URL-formatted for git clone over HTTPS with netrc)
9. Generate or reuse an allod-agent Forgejo API token for the `forge` CLI (raw token format, allod org scope only)

**Phase 2 — Public repos** (human creates repos, agent writes content):
10. Create `Allod/secrets` repo on forge.anarch.diy (Forgejo web UI or API)
11. Create `Allod/inventory` repo on forge.anarch.diy
12. Generate throwaway SSH ed25519 keypair for the public repo's dummy identity: `ssh-keygen -t ed25519 -C host -f throwaway-host`
13. Generate throwaway age keypair for the dummy identity's age encryption
14. Create dummy age-encrypted secret files using the throwaway key

**Phase 3 — allod-dev provisioning** (human-only):
15. Encrypt allod-agent forge SSH private key with age: creates `secrets/allod-dev-forge-key.age`
16. Encrypt allod-agent HTTPS token with age: creates `secrets/forgejo-https-token-allod-dev.age`
17. Encrypt allod-agent raw API token with age: creates `secrets/agent-pr-token-allod-dev.age`
18. Run `agenix -e` or re-encryption for new secret paths
19. Add allod-dev host key to `machine-host-keys.json` (after provisioning generates the host key)
20. Provision allod-dev VM: `provision-vm-from-host allod-dev`
21. Rebuild allod-dev from the host when config changes: `rebuild-vm-from-host allod-dev`

**Blocks:**
- Phase 2 agent work (writing repo content) is blocked on gates 10-11 (repos must exist)
- Phase 3 agent work (profile config) is blocked on gates 6-9 (need the real public key values and token secret paths)
- Phase 3 provisioning (gates 19-21) is blocked on all agent work being merged
- Phase 4 verification is blocked on provisioning

### PR Sequence

Work spans six repos. Use `Refs` on earlier PRs and `Closes` only on the final PR.

1. **allod/secrets** — public template repo (initial content)
   - `Refs Allod/strategy#<issue>`
2. **allod/inventory** — public template repo (initial content)
   - `Refs Allod/strategy#<issue>`
3. **vnprc/secrets** — add allod-dev identity and credentials
   - `Refs Allod/strategy#<issue>`
4. **vnprc/inventory** — add allod-dev machine entry, update vm-specs.json and repositories.json
   - `Refs Allod/strategy#<issue>`
5. **vnprc/nexus** — teach bootstrap scripts to support clone-only isolated dev VMs
   - `Refs Allod/strategy#<issue>`
6. **vnprc/profiles** — add allod-dev VM profile, update flake.lock
   - `Closes Allod/strategy#<issue>`

PRs 1-2 can proceed in parallel. PRs 3-5 can proceed in parallel after gates 6-9. PR 6 depends on PRs 1, 3, 4, and 5 being merged and the `allod-secrets` flake input being locked. There is no profiles flake input for `allod/inventory`; that repo is only a runtime checkout.

### Acceptance Tests

#### Public secrets template validation

```bash
cd /path/to/allod/secrets
nix flake check
```

Validates:
- `credential-inventory` check passes (internal consistency of credentials.nix, secrets.nix, keys/, machine-host-keys.json)
- Flake evaluates without errors

Manual grep to verify no real personal data leaked:

```bash
cd /path/to/allod/secrets
grep -r 'vnprc' --include='*.nix' --include='*.json' && echo "FAIL: real username found" || echo "OK"
grep -r 'protonmail' --include='*.nix' && echo "FAIL: real email found" || echo "OK"
grep -r 'AD88A262' --include='*.nix' && echo "FAIL: real GPG key found" || echo "OK"
grep -r 'fm2932' --include='*.nix' && echo "FAIL: real rsync.net account found" || echo "OK"
grep -rE '62\.76\.229\.|80\.71\.235\.' --include='*.nix' --include='*.json' && echo "FAIL: real VPS IP found" || echo "OK"
```

#### Public inventory template validation

```bash
cd /path/to/allod/inventory
nix flake check
```

Validates:
- `vm-specs-json` check passes (scripts/vm-specs.json matches Nix attrset)
- `repository-registry` check passes (repositories.json is valid and consistent)

Manual grep:

```bash
cd /path/to/allod/inventory
grep -r 'vnprc' --include='*.nix' --include='*.json' && echo "FAIL: real username found" || echo "OK"
grep -rE 'agent-memory|nvim-config|forgejo-config' --include='*.json' && echo "FAIL: private repo reference found" || echo "OK"
```

#### allod-dev VM profile build

```bash
cd /home/vnprc/work/allod/profiles
nix build .#nixosConfigurations.allod-dev.config.system.build.toplevel --dry-run
```

#### allod-dev repos list verification

After provisioning, verify the allod-dev VM's repo list contains no private repos:

```bash
nix eval .#machines.allod-dev.repos --json | jq -r '.[]' | while read repo; do
  case "$repo" in
    secrets|inventory|nvim-config|agent-memory|notes|forgejo-config)
      echo "FAIL: private repo alias '$repo' in allod-dev repos list"
      exit 1
      ;;
  esac
done && echo "OK: no private repos in allod-dev"
```

#### Forge access control verification (manual, post-provisioning)

From the allod-dev VM, verify the allod-agent identity cannot access private repos:

```bash
# Should succeed (allod org repo)
git ls-remote ssh://git@forge.anarch.diy:2222/Allod/tools.git HEAD && echo "OK: allod repo accessible"

# Should fail with permission denied (private repo)
git ls-remote ssh://git@forge.anarch.diy:2222/vnprc/secrets.git HEAD && echo "FAIL: private repo accessible" || echo "OK: private repo blocked"
git ls-remote ssh://git@forge.anarch.diy:2222/vnprc/inventory.git HEAD && echo "FAIL: private repo accessible" || echo "OK: private repo blocked"
git ls-remote ssh://git@forge.anarch.diy:2222/vnprc/agent-memory.git HEAD && echo "FAIL: private repo accessible" || echo "OK: private repo blocked"
```

#### Existing VM builds still pass

```bash
cd /home/vnprc/work/allod/profiles
nix build .#nixosConfigurations.nix-dev.config.system.build.toplevel --dry-run
nix build .#nixosConfigurations.nexus.config.system.build.toplevel --dry-run
```

### Rollback Plan

**Phase 1 (Forge bot user):** Delete the `allod-agent` user from Forgejo admin panel. Remove its SSH key from the Forgejo database. No other systems are affected.

**Phase 2 (Public repos):** Delete the `Allod/secrets` and `Allod/inventory` repos from Forgejo. No downstream consumers exist yet.

**Phase 3 (vnprc/secrets and vnprc/inventory changes):** Revert the commits that added allod-dev entries to `identity.devVMs`, `credentials.nix`, `secrets.nix`, `machine-host-keys.json`, `machines`, `vm-specs.json`, and `repositories.json`. Re-run `agenix --rekey` to remove the orphaned secret paths. Regenerate vm-specs.json: `nix eval .#lib.vmSpecsJson --raw | jq -S . > scripts/vm-specs.json`.

**Phase 5 (profiles):** Delete the `profiles/hosts/dev/allod-dev/` directory. Revert the flake.lock to before the allod-dev-related input updates. Verify existing VMs still build with `nix build --dry-run`.

**allod-dev VM:** If provisioned, destroy with `virsh destroy allod-dev && virsh undefine allod-dev --remove-all-storage`. Remove the allod-dev host key from `machine-host-keys.json`.
