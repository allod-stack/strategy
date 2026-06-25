# Split Agent Memory into Public and Private

## Tracking Issue

TBD — create before implementation.

## Goal

Create a public `Allod/memory` repo with workflow knowledge that allod-dev agents need, while keeping private data in `vnprc/agent-memory`. Non-allod VMs clone both and read them seamlessly.

## Scope

**In scope:**

**Human gates (Forgejo):**
- Create `Allod/memory` repo on forge.anarch.diy

**Public repo (`allod/memory`) — new files:**
- `memory.md` — allod workflow root (PR workflow, git workflow, execution architecture, subagent usage, dev plans)
- `allod.md` — allod overview, repo inventory, forge CLI
- `git-workflow.md` — branching strategy, forge CLI usage
- `dev-plans.md` — dev plan guidelines and review process
- `security-practices.md` — token handling policy
- `vm-tooling.md` — VM package policy
- `vm-provisioning.md` — provisioning stack, source of truth, gotchas
- `nix.md` — NixOS gotchas (nix.conf, netrc, disko, SSH key newlines)
- `age.md` — age/agenix workflows (safe secret input, recipient keys)
- `templates/plan-review-prompt.md` — iterative review template
- `adapters/claude/CLAUDE.md` — allod-dev Claude adapter (points to allod memory path)
- `adapters/codex/AGENTS.md` — allod-dev Codex adapter (points to allod memory path)

**Private repo (`vnprc/agent-memory`) — changes:**
- `memory.md` — slim down to private root; add pointer to `allod/memory`
- Remove migrated files (allod.md, git-workflow.md, dev-plans.md, security-practices.md, vm-tooling.md, vm-provisioning.md, nix.md, age.md, templates/)
- Keep: hashpool.md, adapters/ (private versions pointing to agent-memory path)

**Inventory:**
- Add `allod/memory` to repos lists for existing dev VMs (nix-dev, rust-dev, svelte-dev)
- Add `allod/memory` entry to `repositories.json`

**Profiles:**
- Pass `memoryCheckout = "allod/memory"` for allod-dev in `mkDevVm` (prereq: parameterize-ai-agents is landed)

**Out of scope:**
- Creating allod/secrets or allod/inventory template repos (allod-dev VM plan)
- Creating the allod-dev VM profile
- hashpool.md — stays private (separate project)

## Interface Contracts

### `allod/memory` structure

| File | Public (`allod/memory`) | Private (`vnprc/agent-memory`) |
|------|---------------------------|----------------------------|
| `memory.md` | Allod workflow root | Private root + pointer to allod/memory |
| `allod.md` | Yes | Remove after migration |
| `git-workflow.md` | Yes | Remove after migration |
| `dev-plans.md` | Yes | Remove after migration |
| `security-practices.md` | Yes | Remove after migration |
| `vm-tooling.md` | Yes | Remove after migration |
| `vm-provisioning.md` | Yes | Remove after migration |
| `nix.md` | Yes | Remove after migration |
| `age.md` | Yes | Remove after migration |
| `templates/` | Yes | Remove after migration |
| `hashpool.md` | No | Yes (separate project) |
| `adapters/` | Allod-dev versions (point to allod memory) | Private versions (point to agent-memory) |

### Private memory.md pointer

```markdown
## Allod Workflow
Read allod-specific workflow notes from `~/work/allod/memory/memory.md`.
Topic files: allod.md, git-workflow.md, dev-plans.md, security-practices.md, vm-tooling.md, vm-provisioning.md, nix.md, age.md
```

### Allod-dev adapter (`allod/memory/adapters/claude/CLAUDE.md`)

```markdown
# Claude Adapter
Read shared memory from `/home/allod/work/allod/memory/memory.md`.

## Claude-Specific Policy
- Never add AI attribution anywhere
```

### Non-allod VMs

Both repos are cloned. The private `agent-memory/memory.md` references `allod/memory` by path. Agents read both seamlessly. The `ai-agents.nix` symlink still points to `agent-memory` (the default).

## Agent Gates

1. **Create `Allod/memory` repo** — human must create via Forge web UI or API
2. **NixOS rebuild** — human must rebuild VMs after inventory repos lists are updated

## PR Sequence

1. **allod/memory** — initial content (public workflow files + adapters)
   - `Refs` tracking issue
2. **vnprc/agent-memory** — slim down memory.md, remove migrated files, add pointer
   - `Refs` tracking issue
3. **vnprc/inventory** — add `allod/memory` to dev VM repos lists and repositories.json
   - `Closes` tracking issue

PR 2 depends on PR 1 being merged (so the pointer target exists). PR 3 depends on PR 2.

## Acceptance Tests

### Public memory validation

```bash
cd ~/work/allod/memory
test -f memory.md && echo "OK" || echo "FAIL: missing memory.md"
test -f allod.md && echo "OK" || echo "FAIL: missing allod.md"
test -f git-workflow.md && echo "OK" || echo "FAIL: missing git-workflow.md"
test -f dev-plans.md && echo "OK" || echo "FAIL: missing dev-plans.md"
test -f vm-provisioning.md && echo "OK" || echo "FAIL: missing vm-provisioning.md"
test -f nix.md && echo "OK" || echo "FAIL: missing nix.md"
test -f age.md && echo "OK" || echo "FAIL: missing age.md"
test -f adapters/claude/CLAUDE.md && echo "OK" || echo "FAIL: missing CLAUDE.md adapter"
```

### No private data in public repo

```bash
cd ~/work/allod/memory
grep -r 'hashpool' --include='*.md' && echo "FAIL: private project reference" || echo "OK"
grep -rE '62\.76\.229\.|80\.71\.235\.' --include='*.md' && echo "FAIL: real VPS IP" || echo "OK"
grep -r 'protonmail' --include='*.md' && echo "FAIL: real email" || echo "OK"
```

### Private repo slimmed correctly

```bash
cd ~/work/agent-memory
test -f hashpool.md && echo "OK" || echo "FAIL: hashpool.md missing"
test -f adapters/claude/CLAUDE.md && echo "OK" || echo "FAIL: private adapter missing"
grep 'allod/memory/memory.md' memory.md && echo "OK" || echo "FAIL: pointer to allod memory missing"
! test -f allod.md && echo "OK" || echo "FAIL: allod.md should have been removed"
! test -f nix.md && echo "OK" || echo "FAIL: nix.md should have been removed"
```

### Non-allod VMs still work

```bash
cd ~/work/allod/profiles
nix build .#nixosConfigurations.nix-dev.config.system.build.toplevel --dry-run
```

## Rollback Plan

1. Delete `Allod/memory` repo from Forgejo.
2. Revert the agent-memory commits (restore migrated files, remove pointer).
3. Revert the inventory commit (remove allod/memory from repos lists).
4. Rebuild VMs.

Files are all in git history so nothing is lost.
