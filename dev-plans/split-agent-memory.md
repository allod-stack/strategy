# Split Agent Memory into Public and Private

## Tracking Issue

TBD — create before implementation.

## Goal

Create a public `Allod/memory` repo with workflow knowledge that allod-dev agents need, while keeping private data in `vnprc/agent-memory`. VMs compose both via the `memoryCheckouts` list — no manual cross-references needed.

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
- `adapters/claude/CLAUDE.md` — allod Claude adapter
- `adapters/codex/AGENTS.md` — allod Codex adapter

**Private repo (`vnprc/agent-memory`) — changes:**
- `memory.md` — slim down to private root (user preferences, private projects)
- Remove migrated files (allod.md, git-workflow.md, dev-plans.md, security-practices.md, vm-tooling.md, vm-provisioning.md, nix.md, age.md, templates/)
- Keep: hashpool.md, adapters/ (private versions)

**Inventory:**
- Add `allod/memory` to repos lists for existing dev VMs (nix-dev, rust-dev, svelte-dev)
- Add `allod/memory` entry to `repositories.json`

**Profiles:**
- vnprc's VMs: `memoryCheckouts = [ "allod/memory" "agent-memory" ]` (prereq: parameterize-ai-agents is landed)

**Out of scope:**
- Creating allod/secrets or allod/inventory template repos (allod-dev VM plan)
- Creating the allod-dev VM profile
- hashpool.md — stays private (separate project)

## Interface Contracts

### `allod/memory` structure

| File | Public (`allod/memory`) | Private (`vnprc/agent-memory`) |
|------|---------------------------|----------------------------|
| `memory.md` | Allod workflow root | Private root (user preferences, private projects) |
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
| `adapters/` | Allod adapter | Private adapter |

### Memory composition

Each repo is self-contained — its adapter references only its own memory.md, its memory.md lists only its own topic files. No repo references any other repo.

The `ai-agents.nix` module concatenates all adapters at activation time. The agent sees a single CLAUDE.md containing instructions from all repos.

### User memory repos

Users can optionally supply their own memory repo for private knowledge (personal preferences, non-allod projects, etc). They add it to the `memoryCheckouts` list:

```nix
memoryCheckouts = [ "allod/memory" "my-agent-memory" ]
```

No configuration inside the memory repo is needed — the Nix module handles composition. Users without a private repo use the default (`[ "allod/memory" ]`) and get allod's adapter alone.

### Allod adapter (`allod/memory/adapters/claude/CLAUDE.md`)

```markdown
# Claude Adapter
Read shared memory from `/home/allod/work/allod/memory/memory.md`.

## Claude-Specific Policy
- Never add AI attribution anywhere
```

## Agent Gates

1. **Create `Allod/memory` repo** — human must create via Forge web UI or API
2. **NixOS rebuild** — human must rebuild VMs after inventory repos lists are updated

## PR Sequence

1. **allod/memory** — initial content (public workflow files + adapters)
   - `Refs` tracking issue
2. **vnprc/agent-memory** — slim down memory.md, remove migrated files
   - `Refs` tracking issue
3. **vnprc/inventory** — add `allod/memory` to dev VM repos lists and repositories.json
   - `Closes` tracking issue

PRs are independent — no ordering dependency since repos don't reference each other. PR 3 should land last so the repo exists before VMs try to clone it.

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
! test -f allod.md && echo "OK" || echo "FAIL: allod.md should have been removed"
! test -f nix.md && echo "OK" || echo "FAIL: nix.md should have been removed"
```

### Composed CLAUDE.md contains both adapters

```bash
grep allod/memory ~/.claude/CLAUDE.md && echo "OK" || echo "FAIL: missing allod adapter"
grep agent-memory ~/.claude/CLAUDE.md && echo "OK" || echo "FAIL: missing private adapter"
```

### VMs still build

```bash
cd ~/work/allod/profiles
nix build .#nixosConfigurations.nix-dev.config.system.build.toplevel --dry-run
```

## Rollback Plan

1. Delete `Allod/memory` repo from Forgejo.
2. Revert the agent-memory commits (restore migrated files).
3. Revert the inventory commit (remove allod/memory from repos lists).
4. Rebuild VMs.

Files are all in git history so nothing is lost.
