## Implementation Plan - Decompose home-shared.nix

### Goal

Split `profiles/modules/home-shared.nix` into a framework module (stays in profiles) and a personal preferences module (moves to secrets), so profiles contains only parameterized framework code suitable for open-source publication.

### Scope

**In scope:**
- `profiles/modules/home-shared.nix` — trim to framework-only
- `profiles/flake.nix` — remove `nvim-config` and `nur` inputs, import preferences from secrets
- `secrets/modules/preferences.nix` — new module with personal desktop opinions
- `secrets/flake.nix` — add `nvim-config` and `nur` inputs, export `homeModules.preferences`

**Out of scope:**
- Other modules moved in PR #60 (already pure framework or infrastructure plumbing)
- `profiles/hosts/dev/home-shared.nix` (workspace tool wiring, already clean)
- Privacy VM configuration (does not import home-shared.nix)
- Audit and sanitize work (step 9)
- `nixpkgs.config.allowUnfree` placement (verify during implementation whether anything outside preferences needs it; move it with preferences if not)

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
      core.editor = "nvim";
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
  nixpkgs.config.allowUnfree = true;

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
nvim-config = { url = "git+ssh://..."; flake = false; };
nur = { url = "github:nix-community/NUR"; inputs.nixpkgs.follows = "nixpkgs"; };
```

**secrets/flake.nix** adds a `homeModules` output:
```nix
homeModules.preferences = import ./modules/preferences.nix {
  inherit nvim-config nur;
};
```

**profiles/flake.nix** removes `nvim-config` and `nur` from its inputs.

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

secrets currently gets nixpkgs transitively. Verify that `nur.inputs.nixpkgs.follows` resolves correctly inside secrets' flake. If secrets does not have a top-level nixpkgs input, add one that follows the same source (the `vm` flake's nixpkgs) or adjust the follows chain.

### Agent Gates

**Human must update secrets flake inputs.** Adding `nvim-config` as an SSH-fetched flake input to secrets may require the same netrc/SSH wiring that profiles already has. Verify that the dev VM can fetch `nvim-config` from forge.anarch.diy via the secrets flake before proceeding.

**Human must rebuild and test on at least one dev VM and the hypervisor** after merging both PRs to confirm the effective Home Manager configuration is unchanged.

### Acceptance Tests

All tests run in the dev VM.

**Nix evaluation (automated):**
```bash
# In secrets repo:
nix flake check           # existing checks still pass
nix eval .#homeModules    # preferences module is exported

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
# Before the split, capture the effective home-manager config:
nix eval .#nixosConfigurations.<dev-vm-name>.config.home-manager.users.<username> --json > /tmp/before.json

# After the split, capture again and diff:
nix eval .#nixosConfigurations.<dev-vm-name>.config.home-manager.users.<username> --json > /tmp/after.json
diff /tmp/before.json /tmp/after.json
# Must produce no differences
```

**Input verification:**
- `profiles/flake.lock` no longer contains `nvim-config` or `nur` entries
- `secrets/flake.lock` contains both `nvim-config` and `nur` entries

**Negative tests:**
- `profiles/modules/home-shared.nix` contains no references to `nvim-config`, `nur`, `programs.bash`, `programs.neovim`, `programs.firefox`, or `xdg.configFile`
- `secrets/modules/preferences.nix` contains no references to `identity`, `programs.git`, `programs.ssh`, `home.username`, or `home.stateVersion`

### Rollback Plan

Two PRs (one per repo), both must land together:
1. **secrets PR** — adds `modules/preferences.nix`, new flake inputs, `homeModules` output
2. **profiles PR** — trims `home-shared.nix`, removes inputs, imports preferences from secrets

If verification fails: revert the profiles PR first (restores the old home-shared.nix and inputs), then revert the secrets PR. Order matters because profiles depends on secrets' new export. Reverting profiles first breaks the dependency cleanly; reverting secrets first would leave profiles referencing a nonexistent `homeModules.preferences`.

Flake lock files will need `nix flake update` after revert to drop stale entries.
