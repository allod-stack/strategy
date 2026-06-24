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

Takes `nvim-config` and `nur` as closure parameters. Contains everything personal:

```nix
{ nvim-config, nur }: { pkgs, ... }:
{
  nixpkgs.overlays = [ nur.overlays.default ];

  programs.bash = {
    enable = true;
    shellAliases = {
      claude = "mkdir -p ~/work && cd ~/work && command claude";
      codex = "mkdir -p ~/work && cd ~/work && command codex";
    };
    sessionVariables = { GIT_TERMINAL_PROMPT = "1"; };
    initExtra = ''
      unset SSH_ASKPASS
    '';
  };

  programs.git.settings.core.editor = "nvim";

  programs.neovim = {
    enable = true;
    defaultEditor = true;
    viAlias = true;
    vimAlias = true;
    plugins = with pkgs.vimPlugins; [
      tokyonight-nvim fzf-lua nvim-web-devicons gitsigns-nvim
      nvim-lspconfig oil-nvim nvim-treesitter.withAllGrammars blink-cmp
    ];
    extraPackages = with pkgs; [ gcc fzf ripgrep fd nixd rust-analyzer ];
  };

  xdg.configFile."nvim/init.lua".source = "${nvim-config}/init.lua";
  xdg.configFile."nvim/lua".source = "${nvim-config}/lua";
  xdg.configFile."nvim/CHEATSHEET.md".source = "${nvim-config}/CHEATSHEET.md";

  programs.firefox = {
    enable = true;
    profiles.default = {
      isDefault = true;
      search = { default = "ddg"; force = true; };
      extensions.packages = with pkgs.nur.repos.rycee.firefox-addons; [
        multi-account-containers ublock-origin
      ];
      settings = { /* all existing privacy/telemetry settings */ };
    };
  };
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

**Diff verification (manual):**
```bash
home_snapshot='u: {
  home = { inherit (u.home) username homeDirectory stateVersion; };
  git = {
    userName = u.programs.git.settings.user.name;
    userEmail = u.programs.git.settings.user.email;
    coreEditor = u.programs.git.settings.core.editor;
    credentialHelper = u.programs.git.settings.credential.helper;
    signingKey = u.programs.git.signing.key or null;
  };
  sshHosts = builtins.attrNames u.programs.ssh.matchBlocks;
  bash = { inherit (u.programs.bash) shellAliases sessionVariables initExtra; };
  neovim = {
    inherit (u.programs.neovim) enable defaultEditor viAlias vimAlias;
    plugins = map (p: p.pname or (builtins.parseDrvName p.name).name) u.programs.neovim.plugins;
    extraPackages = map (p: p.pname or (builtins.parseDrvName p.name).name) u.programs.neovim.extraPackages;
  };
  nvimConfig = {
    initLua = toString u.xdg.configFile."nvim/init.lua".source;
    lua = toString u.xdg.configFile."nvim/lua".source;
    cheatsheet = toString u.xdg.configFile."nvim/CHEATSHEET.md".source;
  };
  firefox = {
    inherit (u.programs.firefox) enable;
    search = { inherit (u.programs.firefox.profiles.default.search) default force; };
    extensions = map (p: p.pname or (builtins.parseDrvName p.name).name) u.programs.firefox.profiles.default.extensions.packages;
    settings = u.programs.firefox.profiles.default.settings;
  };
}'

# Before the split, capture only concrete options that should be preserved:
nix eval --no-write-lock-file --json \
  .#nixosConfigurations.<dev-vm-name>.config.home-manager.users.<username> \
  --apply "$home_snapshot" > /tmp/before.json

# After the split, capture again and diff:
nix eval --no-write-lock-file --json \
  .#nixosConfigurations.<dev-vm-name>.config.home-manager.users.<username> \
  --apply "$home_snapshot" > /tmp/after.json
diff /tmp/before.json /tmp/after.json
# Must produce no differences
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
- `profiles/modules/home-shared.nix` contains no references to `nvim-config`, `nur`, `programs.bash`, `programs.neovim`, `programs.firefox`, `xdg.configFile`, or `core.editor = "nvim"`
- `secrets/modules/preferences.nix` contains no references to `identity`, `programs.ssh`, `home.username`, `home.stateVersion`, `programs.git.settings.user`, or `programs.git.settings.credential`
- `secrets/modules/preferences.nix` does not set `nixpkgs.config.allowUnfree`

### Rollback Plan

Two PRs (one per repo), landed in this order:
1. **secrets PR** — adds `modules/preferences.nix`, new flake inputs, `homeModules` output
2. **profiles PR** — trims `home-shared.nix`, removes inputs, imports preferences from secrets, and updates `profiles/flake.lock` to the landed secrets commit

If verification fails: revert the profiles PR first (restores the old home-shared.nix and inputs), then revert the secrets PR. Order matters because profiles depends on secrets' new export. Reverting profiles first breaks the dependency cleanly; reverting secrets first would leave profiles referencing a nonexistent `homeModules.preferences`.

Flake lock files will need `nix flake update` after revert to drop stale entries.
