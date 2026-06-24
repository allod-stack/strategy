## Implementation Plan - Decompose home-shared.nix

### Goal

Split `profiles/modules/home-shared.nix` into a framework module (stays in profiles) and a personal preferences module (moves to secrets), so profiles contains only parameterized framework code suitable for open-source publication.

### Scope

**In scope:**
- `profiles/modules/home-shared.nix` — trim to framework-only
- `profiles/flake.nix` — remove `nvim-config` and `nur` inputs, import preferences from secrets
- `profiles/README.md` — update the flake-input diagram so `nvim-config` and `nur` are no longer shown as direct profiles inputs
- `secrets/modules/preferences.nix` — new module with personal desktop opinions
- `secrets/flake.nix` — add `nvim-config` and `nur` inputs, export `homeModules.preferences`
- `secrets/README.md` — update the repo-purpose and architecture text because secrets will own one private Home Manager preferences module after this split

**Out of scope:**
- Other modules moved in PR #60 (already pure framework or infrastructure plumbing)
- `profiles/hosts/dev/home-shared.nix` (workspace tool wiring, already clean)
- Privacy VM configuration (does not import home-shared.nix)
- Audit and sanitize work (step 9)
- `nixpkgs.config.allowUnfree` placement, except removing the duplicate setting from `profiles/modules/home-shared.nix`. Existing explicit `allowUnfree` sites in `profiles/flake.nix` and host configuration stay where they are because dev, privacy, and hypervisor configs use unfree packages outside the preferences module.

### Interface Contracts

#### After split: `profiles/modules/home-shared.nix`

Signature changes from `{ identity, nvim-config, nur }` to `{ identity }`:

```nix
{ identity }: { lib, ... }:
{
  home.username = identity.username;
  home.homeDirectory = "/home/${identity.username}";

  programs.git = {
    enable = true;
    settings = {
      user.name = identity.username;
      user.email = identity.email;
      credential.helper = "store";
    };
  };

  programs.ssh = {
    enable = true;
    enableDefaultConfig = false;
    matchBlocks = builtins.mapAttrs (name: hostCfg:
      if name == identity.forgeHost
      then hostCfg // { identityFile = lib.mkDefault hostCfg.identityFile; }
      else hostCfg
    ) identity.sshHosts;
  };

  home.stateVersion = "25.11";
}
```

#### New module: `secrets/modules/preferences.nix`

Takes `nvim-config` and `nur` as closure parameters. Contains personal desktop preferences moved intact from the current `profiles/modules/home-shared.nix`, without publishing their exact values in this public plan.

The implementation contract is structural:
- Apply the NUR overlay needed by private browser/addon preferences.
- Move the existing private shell preference block to secrets.
- Move the existing private editor enablement, package list, and config-file wiring to secrets.
- Move the existing private browser profile, addon, and settings block to secrets.
- Move editor-specific Git preferences to secrets; keep identity-bearing Git settings in profiles.
- Do not put identity values, SSH host maps, usernames, email addresses, or Home Manager state version in the preferences module.
- Do not set `nixpkgs.config.allowUnfree` in the preferences module; existing explicit allowUnfree sites outside `home-shared.nix` remain responsible for unfree package evaluation.

Do not use the preferences module as a new framework API. It is intentionally private user configuration exported through `homeModules.preferences` only so profiles can compose it without owning the values.

```nix
{ nvim-config, nur }: { pkgs, ... }:
{
  nixpkgs.overlays = [ nur.overlays.default ];

  # Private preference blocks moved from profiles/modules/home-shared.nix:
  # - shell behavior
  # - editor packages and config-file wiring through nvim-config
  # - browser profile/addons/settings through NUR where needed
  # - editor-specific Git defaults
}
```

#### Flake input changes

**secrets/flake.nix** adds two inputs:
```nix
nvim-config = {
  url = "git+https://forge.anarch.diy/vnprc/nvim-config.git";
  flake = false;
};
nur = { url = "github:nix-community/NUR"; inputs.nixpkgs.follows = "nixpkgs"; };
```

**secrets/flake.nix** binds the new inputs in its `outputs` argument list and adds a `homeModules` output:
```nix
outputs = { self, nixpkgs, nvim-config, nur, ... }:

homeModules.preferences = import ./modules/preferences.nix {
  inherit nvim-config nur;
};
```

**profiles/flake.nix** removes `nvim-config` and `nur` from both its `inputs` set and its `outputs` argument list.

**profiles/flake.nix** import sites change from:
```nix
(import ./modules/home-shared.nix {
  inherit nvim-config nur;
  identity = secrets.lib.identity;
})
```
to:
```nix
(import ./modules/home-shared.nix {
  identity = secrets.lib.identity;
})
secrets.homeModules.preferences
```

This applies to both `mkDevVm` and `mkHypervisor` builders. Privacy VMs are unaffected (they do not import home-shared.nix).

#### nixpkgs follows chain

`secrets` already has a top-level `nixpkgs` input. Keep it. Add `nur.inputs.nixpkgs.follows = "nixpkgs"` in `secrets`; `profiles` already has `secrets.inputs.nixpkgs.follows = "nixpkgs"`, so when profiles consumes secrets, NUR follows the same nixpkgs source as profiles through that existing edge.

Do not add a second nixpkgs input or a Home Manager input to `secrets` for this split. The NUR overlay is applied through the Home Manager user's `nixpkgs.overlays` option, so `pkgs.nur.repos.rycee.firefox-addons` should resolve in the Home Manager module using the pkgs instance constructed by profiles.

#### PR sequencing

Implement and land the secrets change first. Until that commit is on the secrets branch consumed by profiles, profiles will evaluate against a secrets flake that does not export `homeModules.preferences`.

For local profiles testing before the secrets PR lands, add a temporary flake override to every profiles `nix` command:
```bash
--override-input secrets /home/vnprc/work/allod/secrets
```

Do not commit that override. After the secrets PR lands, update `profiles/flake.lock` so the committed profiles change points at the secrets commit that exports `homeModules.preferences`; then run the profiles checks without the override.

### Agent Gates

No human gate is needed for editing flake inputs or lock files; the agent can do that in the dev VM. If `nix flake check` in secrets cannot fetch `nvim-config` through the existing Forge HTTPS credentials path, stop and surface that as an implementation blocker.

**Human must rebuild and test on at least one dev VM and the hypervisor** after merging both PRs to confirm the effective Home Manager configuration is unchanged.

### Acceptance Tests

All tests run in the dev VM.

If testing profiles before the secrets PR lands, include the local override from **PR sequencing** in every profiles `nix` command. If any Nix command says it would modify a lock file, update and commit the relevant lock before declaring the plan complete.

**Nix evaluation (automated):**
```bash
# In secrets repo:
nix flake check           # existing checks still pass
nix eval --json .#homeModules --apply 'm: builtins.hasAttr "preferences" m'

# In profiles repo:
nix flake check           # existing checks still pass
nix flake show            # all VM configurations still present
```

**Build verification (automated):**
```bash
# Build a dev VM configuration to verify the split produces a valid system:
nix build .#nixosConfigurations.<dev-vm-name>.config.system.build.toplevel --dry-run
```

**Private preservation verification (manual):**

Use a local scratch Nix expression outside this repo to compare the moved Home Manager option surface before and after the split. Do not commit the expression or its JSON output; it will necessarily mention private option paths and values.

The scratch expression should select only concrete leaf values from the blocks moved out of `profiles/modules/home-shared.nix`, redact identity/host-sensitive values before diffing, and avoid serializing whole Home Manager option subtrees.

```bash
# Create /tmp/home-shared-preservation.nix locally. It must not be committed.

# Before the split, capture a redacted snapshot of the moved option surface:
nix eval --no-write-lock-file --json \
  .#nixosConfigurations.<dev-vm-name>.config.home-manager.users.<username> \
  --apply "$(cat /tmp/home-shared-preservation.nix)" > /tmp/before.json

# After the split, capture again and diff:
nix eval --no-write-lock-file --json \
  .#nixosConfigurations.<dev-vm-name>.config.home-manager.users.<username> \
  --apply "$(cat /tmp/home-shared-preservation.nix)" > /tmp/after.json
diff /tmp/before.json /tmp/after.json
# Any difference must be intentional and recorded in the implementation PR.
```

**Input verification:**
```bash
# In profiles repo: direct root inputs are gone, but transitive secrets inputs remain expected.
profiles_secrets_node=$(jq -r '.nodes.root.inputs.secrets' flake.lock)
jq -e '.nodes.root.inputs | (has("nvim-config") | not) and (has("nur") | not)' flake.lock
jq -e --arg node "$profiles_secrets_node" '.nodes[$node].inputs | has("nvim-config") and has("nur")' flake.lock

# In secrets repo:
jq -e '.nodes.root.inputs | has("nvim-config") and has("nur")' flake.lock
```

**Docs verification:**
```bash
# In profiles repo:
! rg 'profiles (→|->) (nvim-config|nur)' README.md

# In secrets repo:
rg 'modules/preferences.nix|homeModules.preferences' README.md
! rg 'pure data repo|does not.*Home Manager modules|does not.*NixOS or Home Manager modules' README.md
```

**Negative tests:**
- `profiles/modules/home-shared.nix` contains no references to `nvim-config`, `nur`, `programs.bash`, `programs.neovim`, `programs.firefox`, `xdg.configFile`, or editor-specific Git defaults
- `secrets/modules/preferences.nix` contains no references to `identity`, `programs.ssh`, `home.username`, `home.stateVersion`, `programs.git.settings.user`, or `programs.git.settings.credential`
- `secrets/modules/preferences.nix` does not set `nixpkgs.config.allowUnfree`

### Rollback Plan

Two PRs (one per repo), landed in this order:
1. **secrets PR** — adds `modules/preferences.nix`, new flake inputs, `homeModules` output
2. **profiles PR** — trims `home-shared.nix`, removes inputs, imports preferences from secrets, and updates `profiles/flake.lock` to the landed secrets commit

If verification fails: revert the profiles PR first (restores the old home-shared.nix and inputs), then revert the secrets PR. Order matters because profiles depends on secrets' new export. Reverting profiles first breaks the dependency cleanly; reverting secrets first would leave profiles referencing a nonexistent `homeModules.preferences`.

Flake lock files will need `nix flake update` after revert to drop stale entries.
