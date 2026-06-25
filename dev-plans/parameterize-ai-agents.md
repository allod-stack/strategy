# Parameterize ai-agents.nix

## Tracking Issue

TBD — create before implementation.

## Goal

Make `ai-agents.nix` accept a configurable memory repo checkout path instead of hardcoding `agent-memory`, so different VMs can point at different memory repos.

## Scope

**In scope:**
- `profiles/modules/ai-agents.nix` — add `memoryCheckout` parameter with default `"agent-memory"`
- `profiles/flake.nix` — thread the parameter through `mkDevVm` (no callers change the default yet)

**Out of scope:**
- Changing the default for any existing VM (that happens in the memory split plan)
- Creating `allod/memory` or any new repos

## Interface Contracts

```nix
# ai-agents.nix becomes:
{ identity, memoryCheckout ? "agent-memory" }: { lib, pkgs, ... }:
{
  home.activation.llmMemoryLinks = lib.hm.dag.entryAfter ["writeBoundary"] ''
    mkdir -p "$HOME/.claude/projects/-home-${identity.username}-work"
    ln -sfn "$HOME/work/${memoryCheckout}" "$HOME/.claude/projects/-home-${identity.username}-work/memory"
    ln -sfn "$HOME/work/${memoryCheckout}/adapters/claude/CLAUDE.md" "$HOME/.claude/CLAUDE.md"

    mkdir -p "$HOME/.codex"
    ln -sfn "$HOME/work/${memoryCheckout}/adapters/codex/AGENTS.md" "$HOME/.codex/AGENTS.md"
  '';
}
```

`mkDevVm` in `flake.nix` accepts an optional `memoryCheckout` and passes it through. All existing callers omit it, getting the default.

## Agent Gates

1. **NixOS rebuild** — human must pull profiles on host and rebuild all VMs to pick up the refactored module.

## Acceptance Tests

```bash
# All existing VMs still build
cd ~/work/allod/profiles
nix build .#nixosConfigurations.nix-dev.config.system.build.toplevel --dry-run
nix build .#nixosConfigurations.nexus.config.system.build.toplevel --dry-run

# After rebuild, symlinks still point to agent-memory
readlink ~/.claude/CLAUDE.md | grep agent-memory
readlink ~/.claude/projects/-home-*-work/memory | grep agent-memory
```

## Rollback Plan

Revert the commit in profiles. Rebuild VMs. The module goes back to the hardcoded path.
