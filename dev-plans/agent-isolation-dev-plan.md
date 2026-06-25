## Implementation Plan - Agent Isolation and Secrets Visibility

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

Phase 2c — Public agent memory (`allod/llm-memory`):
- `allod/llm-memory/memory.md` — root memory file with allod workflow sections
- `allod/llm-memory/allod.md` — allod overview, repo inventory, forge CLI
- `allod/llm-memory/git-workflow.md` — branching strategy, forge CLI usage
- `allod/llm-memory/dev-plans.md` — dev plan guidelines and review process
- `allod/llm-memory/security-practices.md` — token handling policy
- `allod/llm-memory/vm-tooling.md` — VM package policy
- `allod/llm-memory/examples/plan-review-prompt.md` — iterative review template
- `allod/llm-memory/adapters/claude/CLAUDE.md` — allod-dev Claude adapter
- `allod/llm-memory/adapters/codex/AGENTS.md` — allod-dev Codex adapter

Phase 4 — allod-dev VM profile:
- `vnprc/inventory/flake.nix` — add `allod-dev` machine entry
- `vnprc/inventory/scripts/vm-specs.json` — regenerate
- `vnprc/inventory/scripts/repositories.json` — add `allod-secrets` and `allod-inventory` aliases
- `vnprc/secrets/identity.nix` — add `allod-dev` to `devVMs`
- `vnprc/secrets/credentials.nix` — add `allod_vm` forge-git credential
- `vnprc/secrets/secrets.nix` — add allod-dev token paths
- `vnprc/secrets/machine-host-keys.json` — add allod-dev entry (after provisioning)
- `profiles/hosts/dev/allod-dev/configuration.nix` — VM system config
- `profiles/hosts/dev/allod-dev/home.nix` — Home Manager config

Phase 5 — Verification:
- Manual verification of access control boundaries

Phase 4 — allod-dev VM profile (continued):
- `profiles/modules/ai-agents.nix` — update to support allod-dev memory path
- `vnprc/llm-memory/memory.md` — add pointer to allod memory for non-allod VMs

**Out of scope:**
- Open-sourcing nexus or profiles (separate effort, depends on this work)
- Pre-commit content scanning hooks (VM isolation is sufficient)
- Changes to existing VMs (nix-dev, rust-dev, svelte-dev) beyond adding allod-memory to their repos lists
- Splitting the profiles flake to support both secrets sources (allod-dev is built from vnprc/secrets like all other VMs)

### Architecture

The isolation model has three layers:

1. **Forge identity**: The `allod-agent` Forgejo user belongs to the `allod` org but has zero access to `vnprc/*` repos. The allod-dev VM authenticates as this user.

2. **Filesystem isolation**: The allod-dev VM only clones allod org repos. Private repos (`vnprc/secrets`, `vnprc/inventory`, `vnprc/llm-memory`, `vnprc/nvim-config`) are never present on disk.

3. **Public repos**: `allod/secrets`, `allod/inventory`, and `allod/llm-memory` provide the module structure, workflow memory, and machine specs agents need for cross-repo development without exposing real identity data.

The allod-dev VM is built by the human through the existing profiles infrastructure (using `vnprc/secrets` for build-time identity). What the agent sees at runtime is controlled by the repos list and the forge credentials provisioned onto the VM.

#### What the agent CAN do on allod-dev:
- Read and edit allod/* repos (tools, strategy, vm, profiles, nexus)
- Read and edit allod/secrets, allod/inventory, allod/llm-memory (public repos)
- Push to allod/* repos via the allod-agent forge identity
- Run `nix flake check` against the public template repos
- Read and write allod-specific agent memory

#### What the agent CANNOT do on allod-dev:
- Read vnprc/secrets, vnprc/inventory, or vnprc/llm-memory (not cloned)
- Push to vnprc/* repos (allod-agent has no access)
- See real IPs, hostnames, tokens, or personal identity data
- Read private memory files (vm-provisioning.md, nix.md, age.md, hashpool.md)

#### Cross-repo development workflow:
1. Agent implements structural changes in allod/secrets or allod/inventory
2. Agent's PR includes a `## Human steps` section listing what to copy to vnprc/*
3. Human reviews, merges the allod PR, then replicates structural changes to vnprc/* with real values
4. Human runs agenix re-encryption and VM rebuild as needed

### Interface Contracts

#### `allod/secrets` flake outputs

Must export the same `lib` and `homeModules` interface as `vnprc/secrets`:

```nix
lib.devIdentities     : { <vm-name> = { username, forgeHost, forgePort, gpgSigningKey, sshKeyName, forgeTokenFile, gpgPublicKeyFile }; }
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
lib.vmSpecsJson      : JSON string

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
    repos = [ "profiles" "workspace-tools" "strategy" ];
  };
};
```

#### `allod/llm-memory` structure

The public memory repo contains allod-generic workflow knowledge. No private identity data, no references to vnprc/* repo contents, no real IPs or hostnames.

File classification:

| File | Public (`allod/llm-memory`) | Private (`vnprc/llm-memory`) |
|------|---------------------------|----------------------------|
| `memory.md` | Allod workflow root (PR workflow, git workflow, execution architecture, subagent usage, dev plans) | Private root (VM provisioning pointers, private repo references, personal preferences) |
| `allod.md` | Yes | Remove after migration (replaced by public copy) |
| `git-workflow.md` | Yes | Remove after migration |
| `dev-plans.md` | Yes | Remove after migration |
| `security-practices.md` | Yes | Remove after migration |
| `vm-tooling.md` | Yes | Remove after migration |
| `examples/` | Yes | Remove after migration |
| `vm-provisioning.md` | No | Yes (references private repo paths, source of truth for secrets) |
| `nix.md` | No | Yes (references private config paths) |
| `age.md` | No | Yes (references private key paths) |
| `hashpool.md` | No | Yes (separate project) |
| `adapters/` | Allod-dev versions (point to allod memory path) | Private versions (point to private memory path) |

The private `vnprc/llm-memory/memory.md` gets a pointer to the allod memory:

```markdown
## Allod Workflow
Read allod-specific workflow notes from `~/work/allod-memory/memory.md`.
Topic files: allod.md, git-workflow.md, dev-plans.md, security-practices.md, vm-tooling.md
```

On non-allod VMs (nix-dev, rust-dev, svelte-dev), both repos are cloned. The private memory.md references the allod memory by path. The agent reads both seamlessly.

On allod-dev, only `allod/llm-memory` is cloned. The adapter CLAUDE.md points to it directly:

```markdown
# Claude Adapter
Read shared memory from `/home/vnprc/work/allod-memory/memory.md`.

## Claude-Specific Policy
- Never add AI attribution anywhere
```

#### `ai-agents.nix` update

The module currently hardcodes the `llm-memory` path. Update to accept the memory repo path as a parameter:

```nix
{ identity, memoryCheckout ? "llm-memory" }: { lib, pkgs, ... }:
{
  home.activation.llmMemoryLinks = lib.hm.dag.entryAfter ["writeBoundary"] ''
    mkdir -p "$HOME/.claude/projects/-home-${identity.username}-work"
    ln -sfn "$HOME/work/${memoryCheckout}" "$HOME/.claude/projects/-home-${identity.username}-work/memory"
    ln -sfn "$HOME/work/${memoryCheckout}/adapters/claude/CLAUDE.md" "$HOME/.claude/CLAUDE.md"

    mkdir -p "$HOME/.codex"
    ln -sfn "$HOME/work/${memoryCheckout}/adapters/codex/AGENTS.md" "$HOME/.codex/AGENTS.md"
  '';
  # ... rest unchanged
}
```

In `profiles/flake.nix`, the `mkDevVm` builder passes the checkout path:

- For allod-dev: `memoryCheckout = "allod-memory"`
- For all other dev VMs: `memoryCheckout = "llm-memory"` (default, no change)

#### `allod-dev` identity in `vnprc/secrets`

New entry in `identity.devVMs`:

```nix
allod-dev = { sshKeyName = "allod_vm"; };
```

This triggers the existing `devIdentities` derivation logic to produce a full identity attrset for allod-dev with the `allod_vm` SSH key name.

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
  repos = [ "profiles" "vm" "nexus" "workspace-tools" "strategy" "allod-secrets" "allod-inventory" "allod-memory" ];
};
```

The `repos` list excludes all vnprc-private aliases: `secrets`, `inventory`, `nvim-config`, `agent-memory`, `notes`.

New entries in `repositories.json`:

```json
"allod-secrets": {
  "source": "forge",
  "remote": "Allod/secrets",
  "checkout": "allod/allod-secrets"
},
"allod-inventory": {
  "source": "forge",
  "remote": "Allod/inventory",
  "checkout": "allod/allod-inventory"
},
"allod-memory": {
  "source": "forge",
  "remote": "Allod/llm-memory",
  "checkout": "allod-memory"
}
```

#### `allod-dev` VM profile

`profiles/hosts/dev/allod-dev/configuration.nix`:
- Imports `agent-forgejo-token.nix` (same as nix-dev)
- Packages: `git`, `jq`, `age` — no nix linting tools unless needed
- No special system config beyond the dev VM baseline

`profiles/hosts/dev/allod-dev/home.nix`:
- Packages: `claude-code` (primary agent harness)
- SSH matchBlock for forge.anarch.diy using `identity.sshKeyName` (resolves to `allod_vm`)
- No GPG signing (allod-agent has no GPG key; `gpgSigningKey = null`)

The `ai-agents.nix` module is updated to accept `memoryCheckout` (see interface contract above). For allod-dev, `mkDevVm` passes `memoryCheckout = "allod-memory"`, so symlinks point to `~/work/allod-memory` instead of `~/work/llm-memory`.

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

New entries in `secrets.nix`:

```nix
"secrets/allod-dev-forge-key.age".publicKeys     = [ hostKey ] ++ vmKeys "allod-dev";
"secrets/forgejo-https-token-allod-dev.age".publicKeys = [ hostKey ] ++ vmKeys "allod-dev";
```

#### Git workflow config for allod-dev (`allod/secrets/git/`)

The public secrets repo needs git workflow config that restricts the allod-dev agent to allod org repos only:

`git/protected-branches`:
```
# allod-dev only works on allod org repos
work/allod/tools master
work/allod/strategy master
work/allod/vm master
work/allod/nexus master
work/allod/profiles master
work/allod/allod-secrets master
work/allod/allod-inventory master
work/allod-memory master
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
8. Generate a Forgejo HTTPS access token for `allod-agent` (for git clone over HTTPS with netrc)

**Phase 2 — Public repos** (human creates repos, agent writes content):
9. Create `Allod/secrets` repo on forge.anarch.diy (Forgejo web UI or API)
10. Create `Allod/inventory` repo on forge.anarch.diy
11. Create `Allod/llm-memory` repo on forge.anarch.diy
12. Generate throwaway SSH ed25519 keypair for the public repo's dummy identity: `ssh-keygen -t ed25519 -C host -f throwaway-host`
13. Generate throwaway age keypair for the dummy identity's age encryption
14. Create dummy age-encrypted secret files using the throwaway key

**Phase 4 — allod-dev provisioning** (human-only):
15. Encrypt allod-agent forge SSH private key with age: creates `secrets/allod-dev-forge-key.age`
16. Encrypt allod-agent HTTPS token with age: creates `secrets/forgejo-https-token-allod-dev.age`
17. Run `agenix -e` or re-encryption for new secret paths
18. Add allod-dev host key to `machine-host-keys.json` (after provisioning generates the host key)
19. Provision allod-dev VM: `provision-vm-from-host allod-dev`
20. Rebuild allod-dev: `rebuild-vm-from-host allod-dev`

**Blocks:**
- Phase 2 agent work (writing repo content) is blocked on gates 9-11 (repos must exist)
- Phase 4 agent work (profile config) is blocked on gates 6-8 (need the real public key values)
- Phase 4 provisioning (gates 18-20) is blocked on all agent work being merged
- Phase 5 verification is blocked on provisioning

### PR Sequence

Work spans five repos. Use `Refs` on earlier PRs and `Closes` only on the final PR.

1. **allod/secrets** — public template repo (initial content)
   - `Refs Allod/strategy#<issue>`
2. **allod/inventory** — public template repo (initial content)
   - `Refs Allod/strategy#<issue>`
3. **allod/llm-memory** — public agent memory (allod workflow content)
   - `Refs Allod/strategy#<issue>`
4. **vnprc/secrets** — add allod-dev identity and credentials
   - `Refs Allod/strategy#<issue>`
5. **vnprc/inventory** — add allod-dev machine entry, update vm-specs.json and repositories.json
   - `Refs Allod/strategy#<issue>`
6. **vnprc/llm-memory** — update memory.md to reference allod memory, remove migrated topic files
   - `Refs Allod/strategy#<issue>`
7. **vnprc/profiles** — add allod-dev VM profile, update ai-agents.nix, update flake.lock
   - `Closes Allod/strategy#<issue>`

PRs 1-3 can proceed in parallel. PRs 4-5 can proceed in parallel after gates 6-8. PR 6 depends on PR 3 being merged (so the allod memory path exists). PR 7 depends on PRs 1-6 being merged and flake inputs updated.

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
# Must NOT match any of these real values
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
grep -rE 'llm-memory|nvim-config|forgejo-config' --include='*.json' && echo "FAIL: private repo reference found" || echo "OK"
```

#### Public llm-memory validation

```bash
cd /path/to/allod/llm-memory
# Must contain the allod workflow root
test -f memory.md && echo "OK" || echo "FAIL: missing memory.md"
test -f allod.md && echo "OK" || echo "FAIL: missing allod.md"
test -f git-workflow.md && echo "OK" || echo "FAIL: missing git-workflow.md"
test -f dev-plans.md && echo "OK" || echo "FAIL: missing dev-plans.md"
test -f adapters/claude/CLAUDE.md && echo "OK" || echo "FAIL: missing CLAUDE.md adapter"
```

Verify no private data:

```bash
cd /path/to/allod/llm-memory
grep -r 'vm-provisioning' --include='*.md' && echo "FAIL: private topic reference" || echo "OK"
grep -r 'age\.md' --include='*.md' && echo "FAIL: private topic reference" || echo "OK"
grep -r 'hashpool' --include='*.md' && echo "FAIL: private project reference" || echo "OK"
grep -rE '62\.76\.229\.|80\.71\.235\.' --include='*.md' && echo "FAIL: real VPS IP" || echo "OK"
grep -r 'protonmail' --include='*.md' && echo "FAIL: real email" || echo "OK"
```

Verify the adapter points to the allod memory path, not the private one:

```bash
grep 'allod-memory/memory.md' /path/to/allod/llm-memory/adapters/claude/CLAUDE.md && echo "OK" || echo "FAIL: adapter points to wrong path"
grep -v 'llm-memory/memory.md' /path/to/allod/llm-memory/adapters/claude/CLAUDE.md > /dev/null && echo "OK" || echo "FAIL: adapter references private memory path"
```

#### allod-dev VM profile build

```bash
cd /home/vnprc/work/allod/profiles
nix build .#nixosConfigurations.allod-dev.config.system.build.toplevel --dry-run
```

#### allod-dev repos list verification

After provisioning, verify the allod-dev VM's repo list contains no private repos:

```bash
# From vnprc/inventory
nix eval .#machines.allod-dev.repos --json | jq -r '.[]' | while read repo; do
  case "$repo" in
    secrets|inventory|nvim-config|agent-memory|notes|forgejo-config|llm-memory)
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
git ls-remote ssh://git@forge.anarch.diy:2222/vnprc/llm-memory.git HEAD && echo "FAIL: private repo accessible" || echo "OK: private repo blocked"
```

#### Existing VM builds still pass

```bash
cd /home/vnprc/work/allod/profiles
nix build .#nixosConfigurations.nix-dev.config.system.build.toplevel --dry-run
nix build .#nixosConfigurations.nexus.config.system.build.toplevel --dry-run
```

### Rollback Plan

**Phase 1 (Forge bot user):** Delete the `allod-agent` user from Forgejo admin panel. Remove its SSH key from the Forgejo database. No other systems are affected.

**Phase 2 (Public repos):** Delete the `Allod/secrets`, `Allod/inventory`, and `Allod/llm-memory` repos from Forgejo. No downstream consumers exist yet.

**Phase 4-5 (vnprc/secrets and vnprc/inventory changes):** Revert the commits that added allod-dev entries to `identity.devVMs`, `credentials.nix`, `secrets.nix`, `machine-host-keys.json`, `machines`, `vm-specs.json`, and `repositories.json`. Re-run `agenix --rekey` to remove the orphaned secret paths. Regenerate vm-specs.json: `nix eval .#lib.vmSpecsJson --raw | jq -S . > scripts/vm-specs.json`.

**Phase 6 (vnprc/llm-memory changes):** Revert the memory.md pointer update. Restore any topic files that were removed during migration. These files are still in git history if needed.

**Phase 7 (profiles):** Delete the `profiles/hosts/dev/allod-dev/` directory. Revert the `ai-agents.nix` changes (remove `memoryCheckout` parameter). Revert the flake.lock to before the allod-dev-related input updates. Verify existing VMs still build with `nix build --dry-run`.

**allod-dev VM:** If provisioned, destroy with `virsh destroy allod-dev && virsh undefine allod-dev --remove-all-storage`. Remove the allod-dev host key from `machine-host-keys.json`.
