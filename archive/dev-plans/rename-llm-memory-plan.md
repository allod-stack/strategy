# Rename llm-memory to agent-memory

## Tracking Issue

TBD — create before implementation.

## Goal

Rename the `llm-memory` repo to `agent-memory` across Forgejo, local checkouts, and all references.

## Scope

### In scope

**Forgejo:**
- Rename repo `vnprc/llm-memory` -> `vnprc/agent-memory` via Forge UI/API

**Local checkout:**
- `mv ~/work/llm-memory ~/work/agent-memory`
- Update git remote URL

**Self-references inside agent-memory (8 files):**
- `memory.md` — absolute path
- `dev-plans.md` — template path reference
- `git-workflow.md` — repo name in push policy
- `allod.md` — repo inventory table
- `templates/plan-review-prompt.md` — dev-plans.md reference (2 occurrences)
- `adapters/claude/CLAUDE.md` — absolute path to memory.md
- `adapters/codex/AGENTS.md` — absolute path to memory.md

**allod repos (live references that affect runtime or tooling):**
- `profiles/modules/ai-agents.nix` — symlink paths (3 occurrences)
- `inventory/scripts/repositories.json` — remote and checkout fields
- `nexus/tests/registry-resolver.sh` — test fixture and assertion
- `secrets/git/protected-branches` — comment
- `vm/docs/adding-a-new-vm-type.md` — example JSON
- `strategy/dev-plans/agent-isolation-dev-plan.md` — many references
- `strategy/brainstorm/licensing.md` — repo list

**notes repo (live docs only):**
- `notes/allod/open-source-plan.md`
- `notes/allod/audit-and-sanitize-plan.md`
- `notes/allod/audit-and-sanitize-plan-review.md`
- `notes/allod/open-source-plan-review.md`
- `notes/allod/bitcoin-node-vm.md`
- `notes/allod/docs/vm-json-files.md`

**Claude Code config:**
- `~/.claude/projects/-home-vnprc-work/CLAUDE.md` — if it references llm-memory
- Symlinks created by ai-agents.nix will update automatically after nix rebuild

### Out of scope

- Archived docs (`notes/archive/`, `allod/strategy/archive/`)
- Renaming other repos (nvim-config, forgejo-config, etc.)
- Changing the repo's internal structure
- agent-isolation plan implementation (it already uses both names; just update references)

## Interface Contracts

None. This is a rename with no API or function signature changes.

## Agent Gates

1. **Forgejo repo rename** — human must rename via Forge web UI or API. Agent cannot do this.
2. **NixOS rebuild** — after `ai-agents.nix` is updated and pushed, human must pull on host and run rebuild to update symlinks.
3. **Local checkout rename** — human must `mv ~/work/llm-memory ~/work/agent-memory` and update remote URL on each machine where the repo is cloned.

## Steps

### Phase 1: Human — Forgejo rename
Human renames `vnprc/llm-memory` to `vnprc/agent-memory` on Forge. Forgejo keeps a redirect from the old URL.

### Phase 2: Human — Local checkout
```
mv ~/work/llm-memory ~/work/agent-memory
cd ~/work/agent-memory
git remote set-url origin ssh://git@forge.anarch.diy:2222/vnprc/agent-memory.git
```

### Phase 3: Agent — Update self-references
Update all 8 files inside the renamed repo. Commit and push directly to master.

### Phase 4: Agent — Update allod repos
Update live references in profiles, inventory, nexus, secrets, vm, and strategy. Each sub-repo gets its own commit. Push to master for unprotected repos; branch + PR for protected repos.

### Phase 5: Agent — Update notes
Bulk find-and-replace `llm-memory` -> `agent-memory` across notes. Single commit, push to master.

### Phase 6: Human — NixOS rebuild
Pull updated profiles on host, rebuild to pick up new symlink paths in ai-agents.nix.

## Acceptance Tests

```bash
# Repo cloned at new path
test -d ~/work/agent-memory/.git

# Remote URL updated
cd ~/work/agent-memory && git remote get-url origin | grep agent-memory

# No stale references in live files (exclude archive)
grep -r 'llm-memory' ~/work/agent-memory/ ~/work/allod/{profiles,inventory,nexus,secrets,vm,strategy/dev-plans}/ && echo "FAIL" || echo "OK"

# Symlinks work after rebuild
readlink ~/.claude/CLAUDE.md | grep agent-memory
readlink ~/.claude/projects/-home-*-work/memory | grep agent-memory

# Nix module references new path
grep 'agent-memory' ~/work/allod/profiles/modules/ai-agents.nix
! grep 'llm-memory' ~/work/allod/profiles/modules/ai-agents.nix
```

## Rollback Plan

1. Rename repo back to `llm-memory` on Forge.
2. `mv ~/work/agent-memory ~/work/llm-memory && cd ~/work/llm-memory && git remote set-url origin ssh://git@forge.anarch.diy:2222/vnprc/llm-memory.git`
3. Revert the reference-update commits in each repo (`git revert`).
4. Rebuild NixOS to restore old symlinks.

Forgejo's redirect from old->new URL means nothing breaks between phases 1 and 4, so there's no rush to complete all phases atomically.
