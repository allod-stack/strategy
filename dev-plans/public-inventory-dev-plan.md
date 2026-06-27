# Create Public allod/inventory Repository

## Tracking Issue

[notes#19](https://forge.anarch.diy/vnprc/notes/issues/19) — parallel
foundation, not part of the core sequence.

## Goal

Create a public `allod/inventory` repository under the Allod org with
synthetic machine inventory data that matches the structure of the private
`vnprc/inventory` repository. Users fork this repo to define their own
machines.

This follows the same pattern as the already-created `allod/secrets`
repository: matching structure, synthetic contents.

## Scope

In scope:

- `flake.nix` with synthetic machine definitions matching the private
  inventory's attribute structure (platform, type, memory_mb, vcpus,
  disk_gb, ip, mac, forge_key, repos, self_rebuild).
- `scripts/vm-specs.json` generated from the synthetic machines.
- `scripts/repositories.json` containing only public allod repo URLs
  (no private `vnprc/` repos, no `nvim-config`, `forgejo-config`,
  `agent-memory`, or `notes`).
- `hosts/` directory with a placeholder `hardware.nix`.
- `README.md` with setup instructions.
- Flake checks (vm-specs-json-check and repository-registry) must pass.

Out of scope:

- Real machine inventory data from `vnprc/inventory`.
- Nexus hardware configuration (consumer-owned).
- Modifying the private `vnprc/inventory` repository.

## Interface Contracts

### Synthetic Machine Set

The public inventory must include at least one machine per type used by
public profiles:

| Machine | Type | Purpose |
|---|---|---|
| `nexus` | hypervisor | Host framework; `hardware` points to `./hosts/nexus/hardware.nix` |
| `nix-dev` | dev | General NixOS development VM |
| `rust-dev` | dev | Rust development VM |

Each VM entry must include all fields that `vmSpecsJson` serializes:
`memory_mb`, `vcpus`, `disk_gb`, `ip`, `mac`, `forge_key`, `repos`.

Synthetic values must be visibly fake:

- IPs: use `192.168.122.0/24` range with different addresses than private
  inventory (e.g., `192.168.122.100`, `192.168.122.101`).
- MACs: use `52:54:00:00:00:xx` pattern.
- Forge keys: use descriptive names like `dev_vm_1`.
- Repos: reference only public allod aliases that exist in the registry.

### Repository Registry

`scripts/repositories.json` must contain entries for the public allod
repos only:

- `profiles` → `allod/profiles`
- `vm` → `allod/vm`
- `nexus` → `allod/nexus`
- `secrets` → `allod/secrets`
- `inventory` → `allod/inventory`
- `workspace-tools` → `allod/tools`
- `allod/memory` → `Allod/memory`
- `strategy` → `Allod/strategy`

No private repo aliases (`nvim-config`, `forgejo-config`, `agent-memory`,
`notes`, `nowhere`, `cdk-ehash`, `hashpool`, `hashpool-website`,
`cdk-upstream`).

### Hardware Placeholder

`hosts/nexus/hardware.nix` ships as a stub that consumers replace with
their own hardware scan output (`nixos-generate-config --show-hardware-config`).

### Flake Checks

Both existing checks from `vnprc/inventory` must pass:

- `vm-specs-json-check`: `scripts/vm-specs.json` matches Nix-derived specs.
- `repository-registry`: `scripts/repositories.json` validates (all
  entries have source/remote/checkout, VM repo aliases resolve, checkout
  paths unique per VM, dev VMs with self_rebuild have profiles).

## Agent Gates

- **Repository creation on Forgejo.** A human must create `Allod/inventory`
  on the Forge and push the initial commit.
- **Pushing commits.** The agent prepares the content; the human reviews
  and pushes.

## Acceptance Tests

```sh
cd <public-inventory-dir>

# Flake evaluation
nix flake check

# No private data
rg -n 'forge\.anarch\.diy|vnprc|protonmail\.com|192\.168\.122\.1[0-5]|agent-memory|nvim-config|forgejo-config|notes' \
  --hidden -g '!.git/**' . && echo "FAIL: private data found" || echo "PASS"

# Registry has no private repos
jq -e '.repositories | keys | map(select(. == "nvim-config" or . == "forgejo-config" or . == "agent-memory" or . == "notes" or . == "nowhere")) | length == 0' \
  scripts/repositories.json

# Synthetic values are visibly fake (not matching private inventory)
! grep -q '192\.168\.122\.10"' flake.nix
! grep -q '192\.168\.122\.11"' flake.nix
! grep -q '192\.168\.122\.12"' flake.nix
```

## Rollback Plan

Delete the `Allod/inventory` repository on Forgejo. No other repos depend
on it yet.
