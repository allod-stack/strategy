# Split Agent Memory into Public and Private

## Tracking Issue

TBD — create before implementation and update this section with the Forge issue URL/number. Every implementation PR references this issue; the final `vnprc/profiles` PR carries `Closes`, earlier PRs use `Refs`.

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
- `inventory/flake.nix` — add the `allod/memory` repo alias to existing dev VM repos lists (`nix-dev`, `rust-dev`, `svelte-dev`)
- `inventory/scripts/vm-specs.json` — regenerate from `inventory/flake.nix`
- `inventory/scripts/repositories.json` — add an `allod/memory` entry with `remote = "Allod/memory"` and `checkout = "allod/memory"`

**Profiles:**
- `profiles/modules/ai-agents.nix` — default `memoryCheckouts` becomes public-only (`[ "allod/memory" ]`)
- `profiles/flake.nix` — existing vnprc dev VMs explicitly compose `memoryCheckouts = [ "allod/memory" "agent-memory" ]`; private checkout stays last because `ai-agents.nix` links Claude project memory to `lib.last memoryCheckouts`

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

The `ai-agents.nix` module does not concatenate adapter contents. At activation time it writes pointer files:

- `~/.claude/CLAUDE.md` contains one `Read shared instructions from .../adapters/claude/CLAUDE.md` line per checkout.
- `~/.codex/AGENTS.md` contains one `Read shared instructions from .../adapters/codex/AGENTS.md` line per checkout.
- `~/.claude/projects/-home-${username}-work/memory` symlinks to `$HOME/work/${lib.last memoryCheckouts}`.

Order is significant. For existing vnprc dev VMs, use `[ "allod/memory" "agent-memory" ]` so the public Allod workflow is read first and private user/project memory is last and remains the Claude project memory symlink target.

### User memory repos

Users can optionally supply their own memory repo for private knowledge (personal preferences, non-allod projects, etc). They add it to the `memoryCheckouts` list:

```nix
memoryCheckouts = [ "allod/memory" "my-agent-memory" ]
```

No cross-repo configuration inside the memory repo is needed — the Nix module handles pointer-file composition. Users without a private repo use the module default (`[ "allod/memory" ]`) and get allod's adapter alone.

### Allod adapter (`allod/memory/adapters/claude/CLAUDE.md`)

```markdown
# Claude Adapter
Read `../../memory.md` relative to this adapter file before relying on Allod workflow memory.

## Claude-Specific Policy
- Never add AI attribution anywhere
```

`allod/memory/adapters/codex/AGENTS.md` follows the same identity-neutral pattern:

```markdown
# Codex Adapter

Read `../../memory.md` relative to this adapter file before relying on Allod workflow memory.
```

Do not hardcode `/home/allod` or `/home/vnprc` in public adapters; the same repo is consumed by current vnprc dev VMs and future allod-dev VMs.

## Agent Gates

1. **Create `Allod/memory` repo** — human must create via Forge web UI or API
2. **Existing VM workspace repair** — after inventory lands, human runs host-side `verify-vm-from-host --repair <vm>` or equivalent for each dev VM so `~/work/allod/memory` exists before agent startup depends on it
3. **NixOS rebuild** — human must rebuild VMs after profiles changes land so pointer files and symlinks are regenerated

## PR Sequence

1. **allod/memory** — initial content (public workflow files + adapters)
   - `Refs` tracking issue
2. **vnprc/agent-memory** — slim down memory.md, remove migrated files
   - `Refs` tracking issue
3. **vnprc/inventory** — add `allod/memory` to dev VM repos lists, `scripts/repositories.json`, and regenerated `scripts/vm-specs.json`
   - `Refs` tracking issue
4. **vnprc/profiles** — change `memoryCheckouts` defaults/explicit values so existing dev VMs compose public and private memory
   - `Closes` tracking issue

PRs are not fully independent. PR 1 must land before inventory points VMs at the new repo. PR 3 must land before PR 4 is deployed, otherwise rebuilt VMs can generate pointers to a checkout that provisioning does not know how to repair. Merge PR 4 last because it is the integration PR that changes runtime composition.

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
test -f adapters/codex/AGENTS.md && echo "OK" || echo "FAIL: missing AGENTS.md adapter"
grep -RInE '/home/(allod|vnprc)/work' adapters && echo "FAIL: public adapter hardcodes a VM user" || echo "OK"
grep -RIn 'memory.md' adapters/claude/CLAUDE.md adapters/codex/AGENTS.md >/dev/null && echo "OK" || echo "FAIL: adapters do not direct agents to memory.md"
```

### No private data in public repo

```bash
cd ~/work/allod/memory
grep -RInE --exclude-dir=.git 'hashpool|agent-memory|vnprc/|/home/vnprc|protonmail' . && echo "FAIL: private project or identity reference" || echo "OK"
grep -RInE --exclude-dir=.git '62\.76\.229\.|80\.71\.235\.|192\.168\.122\.|52:54:00:ab:cd' . && echo "FAIL: real host IP or MAC data" || echo "OK"
grep -RInE --exclude-dir=.git 'AGE-SECRET-KEY-|-----BEGIN (OPENSSH|RSA|EC|DSA) PRIVATE KEY-----|ghp_[A-Za-z0-9_]{20,}|github_pat_[A-Za-z0-9_]+|glpat-[A-Za-z0-9_-]{20,}|sk-[A-Za-z0-9_-]{20,}|xox[baprs]-[A-Za-z0-9-]{20,}' . && echo "FAIL: token or private key material" || echo "OK"
```

### Private repo slimmed correctly

```bash
cd ~/work/agent-memory
test -f hashpool.md && echo "OK" || echo "FAIL: hashpool.md missing"
test -f adapters/claude/CLAUDE.md && echo "OK" || echo "FAIL: private adapter missing"
! test -f allod.md && echo "OK" || echo "FAIL: allod.md should have been removed"
! test -f nix.md && echo "OK" || echo "FAIL: nix.md should have been removed"
! test -d templates && echo "OK" || echo "FAIL: templates should have been removed"
grep -RInE 'allod.md|git-workflow.md|dev-plans.md|security-practices.md|vm-tooling.md|vm-provisioning.md|nix.md|age.md|templates/plan-review-prompt.md' memory.md adapters && echo "FAIL: stale migrated-file reference" || echo "OK"
```

### Inventory validation

```bash
cd ~/work/allod/inventory
nix eval .#lib.vmSpecsJson --raw | jq -S . > /tmp/vm-specs.expected.json
jq -S . scripts/vm-specs.json > /tmp/vm-specs.actual.json
diff -u /tmp/vm-specs.expected.json /tmp/vm-specs.actual.json
jq -e '.repositories["allod/memory"].remote == "Allod/memory" and .repositories["allod/memory"].checkout == "allod/memory"' scripts/repositories.json
jq -e '."nix-dev".repos | index("allod/memory")' scripts/vm-specs.json
jq -e '."rust-dev".repos | index("allod/memory")' scripts/vm-specs.json
jq -e '."svelte-dev".repos | index("allod/memory")' scripts/vm-specs.json
nix build .#checks.x86_64-linux.vm-specs-json
nix build .#checks.x86_64-linux.repository-registry
```

### Runtime pointer files contain both adapters

```bash
grep -F "/work/allod/memory/adapters/claude/CLAUDE.md" ~/.claude/CLAUDE.md && echo "OK" || echo "FAIL: missing allod Claude adapter pointer"
grep -F "/work/agent-memory/adapters/claude/CLAUDE.md" ~/.claude/CLAUDE.md && echo "OK" || echo "FAIL: missing private Claude adapter pointer"
grep -F "/work/allod/memory/adapters/codex/AGENTS.md" ~/.codex/AGENTS.md && echo "OK" || echo "FAIL: missing allod Codex adapter pointer"
grep -F "/work/agent-memory/adapters/codex/AGENTS.md" ~/.codex/AGENTS.md && echo "OK" || echo "FAIL: missing private Codex adapter pointer"
readlink ~/.claude/projects/-home-"$(whoami)"-work/memory | grep -F "/work/agent-memory" && echo "OK" || echo "FAIL: Claude project memory should point at private checkout"
```

### Profiles composition validation

```bash
cd ~/work/allod/profiles
grep -F 'memoryCheckouts ? [ "allod/memory" ]' modules/ai-agents.nix && echo "OK" || echo "FAIL: ai-agents default should be public-only"
grep -F '[ "allod/memory" "agent-memory" ]' flake.nix && echo "OK" || echo "FAIL: existing vnprc dev VMs should compose public then private memory"
```

### VMs still build

```bash
cd ~/work/allod/profiles
for vm in nix-dev rust-dev svelte-dev; do
  nix build ".#nixosConfigurations.${vm}.config.system.build.toplevel" --dry-run
done
```

## Rollback Plan

1. Revert the `vnprc/profiles` change so `memoryCheckouts` no longer points at `allod/memory`; rebuild affected VMs to regenerate `~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`, and the Claude project memory symlink.
2. Revert the `vnprc/inventory` change: remove `allod/memory` from `inventory/flake.nix`, regenerate `inventory/scripts/vm-specs.json`, and remove the `allod/memory` registry entry from `inventory/scripts/repositories.json`.
3. Revert the `vnprc/agent-memory` commits to restore migrated files and private `memory.md` references.
4. Delete or archive the `Allod/memory` repo from Forgejo only after no VM profile or inventory file references it.
5. Optionally remove stale local `~/work/allod/memory` checkouts after rebuild/repair confirms no runtime pointers target them.

Files are all in git history so nothing is lost.
