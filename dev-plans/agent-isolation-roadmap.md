# Agent Isolation Roadmap

## Overview

The original agent-isolation dev plan was a monolith. This roadmap breaks it
into independent prerequisites that each land separately, reducing the blast
radius of the final allod-dev VM work.

## Sequence

### 1. [Rename llm-memory to agent-memory](../archive/dev-plans/rename-llm-memory-plan.md) (complete)

Pure rename across Forgejo, local checkouts, and all references. No structural
changes. Clears the decks so later plans don't have to deal with the old name.

### 2. [Parameterize ai-agents.nix](archive/parameterize-ai-agents.md) (complete)

Refactor `ai-agents.nix` to accept a `memoryCheckout` parameter instead of
hardcoding the memory repo path. Pure refactor with zero behavior change for
existing VMs. Required so allod-dev can point at `allod/memory` later.

### 3. [Per-VM checkout path uniqueness](../archive/dev-plans/per-vm-checkout-uniqueness.md) (complete)

Relax the `repository-registry` flake check from global checkout path uniqueness
to per-VM uniqueness. Required because allod-dev's public `allod/secrets` and
`allod/inventory` repos use the same checkout paths as the private aliases on
other VMs.

### 4. [Split agent-memory into public and private](../archive/dev-plans/split-agent-memory.md) (complete)

Create `Allod/memory` with public workflow knowledge. Update `agent-memory` to
keep only private files and point to the public repo. Add `allod/memory` to
existing dev VM repos lists. Uses the parameterized `ai-agents.nix` from step 2.

### 5. [Allod-dev VM](agent-isolation-dev-plan.md)

With prereqs landed, this plan covers only: Forge bot user, public
secrets/inventory template repos, allod-dev VM profile, and access control
verification. The memory split, ai-agents.nix refactor, and checkout collision
fix are already done.

## Dependencies

```
1 (rename) ──> 2 (ai-agents.nix) ──> 4 (memory split) ──> 5 (allod-dev VM)
                                                       /
                          3 (checkout uniqueness) ─────/
```

Steps 1-3 have no dependencies on each other and can proceed in parallel.
Step 4 depends on steps 1 and 2. Step 5 depends on steps 3 and 4.
