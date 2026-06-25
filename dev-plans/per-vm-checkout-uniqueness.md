# Per-VM Checkout Path Uniqueness

## Tracking Issue

TBD — create before implementation.

## Goal

Relax the `repository-registry` flake check to validate checkout path uniqueness per-VM instead of globally, so different VMs can use different repos that share the same checkout path.

## Scope

**In scope:**
- `vnprc/inventory/flake.nix` — update the `repository-registry` check

**Out of scope:**
- Adding new repository entries (that happens in the allod-dev VM plan)
- Changing any existing VM repos lists

## Context

The private `secrets` and `inventory` aliases use checkout paths `allod/secrets` and `allod/inventory`. The planned public aliases for allod-dev will use the same paths. They never coexist on the same VM, but the current global uniqueness check rejects this.

## Interface Contracts

The `repository-registry` check validates:
- Each repo entry has required fields (`source`, `remote`, `checkout`)
- Within each VM's repos list, no two entries resolve to the same checkout path
- No longer: global checkout path uniqueness across all repos

## Acceptance Tests

```bash
cd ~/work/allod/inventory  # or vnprc/inventory
nix flake check

# Verify the check still catches actual collisions (two repos in the same VM's list
# mapping to the same checkout). This requires a test fixture or manual verification
# that existing vm-specs with duplicate checkouts would still fail.
```

## Rollback Plan

Revert the commit in inventory. `nix flake check` goes back to the global uniqueness constraint.
