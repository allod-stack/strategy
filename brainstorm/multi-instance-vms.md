# Multi-Instance VM Architecture

## Goal

Allow multiple independently provisioned VM instances to share one reusable VM
type without duplicating type configuration.

## Problem

Today, every VM has a unique name that equals its type (`rust-dev`, `svelte-dev`,
`nix-dev`). The VM name is a single key that threads through every layer of the
system:

```
hosts/dev/<name>/          <- NixOS config directory
secrets/<name>-ssh.age     <- SSH host keypair
secrets/forge-https-token-<name>.age
vm-keys.json               <- agenix recipient list
vm-specs.json              <- IP, MAC, memory, repos

networking.hostName = name
services.openssh.hostKeys = [{ path = "/etc/ssh/${name}"; }]
age.identityPaths = [ "/etc/ssh/${name}" ]
age.secrets.forge-https-token.file = ./secrets/forge-https-token-${name}.age
```

Because name = type = instance, you cannot have two `rust-dev` VMs. Adding a
second one requires duplicating all config files under a new name (e.g.
`rust-dev-2`) with no mechanism to share the common parts.

---

## What Is Instance-Specific vs Type-Shared

**Instance-specific** (must be unique per running VM):
- Hostname
- Static IP + MAC (DHCP reservation in libvirt)
- SSH host keypair (`/etc/ssh/<instance>`, pre-generated, injected by nixos-anywhere)
- Forge HTTPS token (agenix secret, encrypted to host key + VM host key)
- Forge SSH keypair (stored through the secrets boundary)
- Entry in `vm-keys.json` (agenix recipient)
- Entry in `vm-specs.json`
- SSH `known_hosts` entry on the host

**Type-shared** (identical across all instances of a type):
- `configuration.nix` -- system packages and services
- `home.nix` -- user environment
- `disk.nix` -- disk layout (already identical across all QEMU VMs)
- `meta.nix` -- username
- Default resource sizing (memory, vcpus, disk) -- could be overridden per instance

---

## Design Options

### Option A: `type` field in vm-specs.json

Add a `"type"` key to each vm-specs.json entry. Instance names stay unique; the
type field points to the config directory.

```json
"rust-dev-1": { "type": "rust-dev", "ip": "192.168.122.XX", "mac": "...", "forge_key": "dev_key_1", ... },
"rust-dev-2": { "type": "rust-dev", "ip": "192.168.122.XX", "mac": "...", "forge_key": "dev_key_2", ... }
```

`flake.nix` changes: `mkDevVm` reads the type from vm-specs.json (or a new
`type.nix` in the instance dir) to locate the config directory.

```nix
mkDevVm = name:
  let
    spec = vmSpecs.${name};
    type = spec.type or name;   # fall back to name for backward compat
    meta = import ./hosts/dev/${type}/meta.nix;
  in nixpkgs.lib.nixosSystem {
    modules = (sharedModules name) ++ [
      ./hosts/dev/${type}/configuration.nix
      ...
    ];
  };
```

**Pros:** Minimal disruption. Backward compatible (existing VMs have
`type = name`). Config files stay in one place per type. Instance directory not
needed at all for identical instances.

**Cons:** vm-specs.json becomes the source of truth for the type mapping, but it
lives alongside inventory data rather than in the Nix profile layer. Flake eval
now depends on a JSON file it currently does not read.

### Option B: Two-level directory structure

```
hosts/dev/<type>/<instance>/
hosts/dev/<type>/configuration.nix   <- shared
hosts/dev/<type>/home.nix            <- shared
hosts/dev/<type>/disk.nix            <- shared
hosts/dev/<type>/meta.nix            <- shared (username)
hosts/dev/<type>/<instance>/meta.nix <- instance overrides (optional)
```

The flake scans one level deeper. An instance with no per-instance overrides
needs only an empty directory (or a single `{}` meta.nix).

**Pros:** Clean separation in the filesystem. Type-level files are obvious.
Instance directories serve as the instance registry (the flake discovers them by
scanning).

**Cons:** Bigger structural change. Existing single-instance types would need
their files moved. The flake's `listDirsIf` scanning logic becomes two-pass.

### Option C: Instance meta.nix with type pointer

Keep flat `hosts/dev/<instance>/` directories but add a `type` field to meta.nix:

```nix
# hosts/dev/rust-dev-2/meta.nix
{ username = "user"; type = "rust-dev"; }
```

`mkDevVm` reads `type` from meta.nix; if absent, falls back to the directory
name (full backward compat). The config files (`configuration.nix`, `home.nix`,
`disk.nix`) are symlinks or absent -- when absent, the builder falls back to
`hosts/dev/<type>/`.

**Pros:** All instance info in one place (meta.nix). No json parsing in the
flake. Discovery still works via `listDirsIf`.

**Cons:** Symlinks are fragile in Nix flakes (pure eval mode may reject them).
Fallback logic adds complexity to `mkDevVm`.

---

## Recommendation: Option A with a `type.nix` shim

Use Option A's approach but express the type mapping in Nix rather than JSON:

1. Add `hosts/dev/<instance>/type.nix` for instances that differ from their type
   name:
   ```nix
   # hosts/dev/rust-dev-2/type.nix
   "rust-dev"
   ```
   When absent, the instance name is the type name (backward compatible).

2. `mkDevVm` reads `type.nix` if it exists:
   ```nix
   mkDevVm = name:
     let
       typePath = ./hosts/dev/${name}/type.nix;
       type = if builtins.pathExists typePath then import typePath else name;
       meta = import ./hosts/dev/${type}/meta.nix;
     in ...
   ```

3. New instance provisioning creates only:
   - `hosts/dev/<instance>/type.nix` (one line)
   - `secrets/<instance>-ssh.age` + `.pub`
   - `secrets/forge-https-token-<instance>.age`
   - `vm-keys.json` entry
   - `vm-specs.json` entry

   No duplication of configuration.nix, home.nix, or disk.nix.

4. vm-specs.json keeps its flat structure; instance names stay the flake
   attribute names.

---

## Constraints

- `nixosConfigurations.<instance>` must evaluate independently -- nixos-rebuild,
  nixos-anywhere, and rebuild scripts all target a specific flake attribute.
- Every instance needs a unique SSH host key (agenix identity + known_hosts
  fingerprint).
- Host-side provisioning scripts read vm-specs.json by instance name; that
  interface must stay stable.
- DHCP reservations are per-IP/MAC; each instance needs a unique pair.
- No agent-run host commands -- all provisioning must remain scriptable
  host-side.
- Adding a new instance of an existing type should be significantly simpler than
  adding a new type today.

---

## Open Questions

- Should instances of the same type share a forge HTTPS token, or always get
  separate ones? Separate tokens give better audit trails and revocation
  granularity.
- Should `vm-specs.json` grow a `"type"` field alongside the `type.nix`
  approach, or is the Nix file sufficient?
- Does the "adding a new VM type" doc need a parallel "adding a new instance"
  section, or does it collapse into a simpler checklist?
- For privacy VMs (no forge access), is multi-instance ever needed? If not, the
  design can focus on dev VMs only.

## Acceptance Tests

```sh
nix flake check
nix build .#nixosConfigurations.rust-dev-1.config.system.build.toplevel --no-link
nix build .#nixosConfigurations.rust-dev-2.config.system.build.toplevel --no-link
```

Provision two disposable instances of one type and verify unique hostnames,
IP/MAC pairs, SSH host keys, and secret recipients while confirming both use the
same type modules.

## Rollback Plan

Remove the additional instance entries and restore the one-instance-per-type
mapping. Existing VM disks and secrets remain valid because instance names and
deployment targets do not change during the compatibility phase.
