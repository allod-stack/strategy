# Parameterize ai-agents.nix

## Tracking Issue

https://forge.anarch.diy/vnprc/profiles/issues/68

## Goal

Make `ai-agents.nix` accept a list of memory repo checkout paths instead of hardcoding `agent-memory`, so VMs can compose knowledge from multiple memory repos.

## Scope

**In scope:**
- `profiles/modules/ai-agents.nix` — add `memoryCheckouts` list parameter with default `[ "agent-memory" ]`
- `profiles/flake.nix` — thread the parameter through `mkDevVm` (no callers change the default yet)

**Out of scope:**
- Changing the default for any existing VM (that happens in the memory split plan)
- Creating `allod/memory` or any new repos

## Interface Contracts

```nix
# ai-agents.nix — outer function takes a list of memory repos:
{ identity, memoryCheckouts ? [ "agent-memory" ] }: { lib, pkgs, ... }:
{
  home.activation.llmMemoryLinks = lib.hm.dag.entryAfter ["writeBoundary"] ''
    mkdir -p "$HOME/.claude/projects/-home-${identity.username}-work"
    ln -sfn "$HOME/work/${lib.last memoryCheckouts}" "$HOME/.claude/projects/-home-${identity.username}-work/memory"

    cat ${lib.concatMapStringsSep " " (repo: "\"$HOME/work/${repo}/adapters/claude/CLAUDE.md\"") memoryCheckouts} > "$HOME/.claude/CLAUDE.md"

    mkdir -p "$HOME/.codex"
    cat ${lib.concatMapStringsSep " " (repo: "\"$HOME/work/${repo}/adapters/codex/AGENTS.md\"") memoryCheckouts} > "$HOME/.codex/AGENTS.md"
  '';
}

# flake.nix — mkDevVm gains optional memoryCheckouts, threaded to import:
mkDevVm = { name, identity, memoryCheckouts ? [ "agent-memory" ] }:
  # ...
  (import ./modules/ai-agents.nix {
    identity = secrets.lib.identity;
    inherit memoryCheckouts;
  })
```

All existing `mkDevVm` callers omit `memoryCheckouts`, getting the default.

Auto-memory symlink points to the last repo in the list (most specific/private).

## Agent Gates

1. **NixOS rebuild** — human must pull profiles on host and rebuild all VMs to pick up the refactored module.

## Acceptance Tests

```bash
# All existing VMs still build
cd ~/work/allod/profiles
nix build .#nixosConfigurations.nix-dev.config.system.build.toplevel --dry-run
nix build .#nixosConfigurations.nexus.config.system.build.toplevel --dry-run

# After rebuild, CLAUDE.md contains agent-memory adapter content
grep agent-memory ~/.claude/CLAUDE.md
readlink ~/.claude/projects/-home-*-work/memory | grep agent-memory
```

## Rollback Plan

Revert the commit in profiles. Rebuild VMs. The module goes back to the hardcoded path.
