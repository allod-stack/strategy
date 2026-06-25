# Bitcoin Node Service VM

**Roadmap position:** post-release capability after the public profiles and
[multi-instance VM model](multi-instance-vms.md) are stable.

## Goal

Add a reproducible service VM running a verified Bitcoin Core binary with an
explicit network privacy mode and no development identity.

## Scope

In scope:

- Implement the service VM archetype.
- Package Bitcoin Core through `bitcoind-gunix`.
- Define persistent storage, firewall, and RPC access.
- Support an explicit clearnet, Tor-only, or dual network decision.

Out of scope:

- Wallet deployment.
- Public RPC exposure.
- Inbound hidden service support.
- Automatic migration of existing Bitcoin data.

## Interface Contracts

- The service VM has no Home Manager development user or forge credentials.
- RPC is unreachable from the LAN and available only through an approved
  host-only path or SSH tunnel.
- The selected `bitcoind-gunix` package matches the VM platform from the
  machine inventory.
- Network mode is explicit and testable.

A service VM running Bitcoin Core sourced from the
[bitcoind-gunix](https://github.com/0xB10C/bitcoind-gunix) flake -- every
binary sha256-verified to be byte-identical to the official GUIX release.
First instance of the `service` VM archetype.

---

## Design Principles

- bitcoind binary from bitcoind-gunix; the build gate IS the verification --
  a successful `nix build` proves byte-parity with the GUIX release
- Headless service; no GUI, no dev tools, no forge access
- Networking mode is configurable -- Tor-only for privacy, clearnet for
  latency-sensitive use cases (e.g. mining); see Open Questions
- RPC accessible from host only via SSH tunnel -- never bound to the VM's LAN IP
- Implements the `mkServiceVm` archetype

---

## Architecture

New host `bitcoin-node` added under `hosts/service/bitcoin-node/`.

The flake already auto-discovers `hosts/service/` via
`serviceVmNames = listDirsIf ./hosts/service` -- no discovery change needed.
Two things to add:

1. `bitcoind-gunix` flake input
2. `mkServiceVm` function

Service VMs differ from the existing archetypes:

- No secrets import (no git identity, no agent hooks)
- No forge token (no forge access)
- No home-manager user session -- systemd service account only
- No egress restrictions (bitcoind needs unrestricted outbound for P2P + Tor bootstrap)

---

## bitcoind-gunix Integration

### Flake input

```nix
bitcoind-gunix = {
  url = "github:0xB10C/bitcoind-gunix";
  # bitcoind-gunix pins nixos-25.11; follow the project's nixpkgs to
  # avoid pulling a second copy.  If nixpkgs versions diverge, remove this
  # line and let bitcoind-gunix use its own -- its strict toolchain pinning
  # means mismatched nixpkgs will break the build anyway.
  inputs.nixpkgs.follows = "nixpkgs";
};
```

### mkServiceVm

```nix
mkServiceVm = name:
  nixpkgs.lib.nixosSystem {
    system = "x86_64-linux";
    specialArgs = { inherit bitcoind-gunix hostPublicKey; };
    modules = (sharedModules name) ++ [
      ./hosts/service/${name}/configuration.nix
    ];
  };
```

No home-manager, no forge token, no secrets input.

### configuration.nix usage

```nix
{ pkgs, bitcoind-gunix, ... }:
{
  services.bitcoind."mainnet" = {
    enable   = true;
    package  = bitcoind-gunix.packages.x86_64-linux.bitcoind;
    dataDir  = "/var/lib/bitcoind";
    extraConfig = ''
      # Pruned node -- keeps full UTXO set, drops old block data
      prune=550
      # Onion-only P2P for peer privacy
      proxy=127.0.0.1:9050
      onlynet=onion
      # RPC on loopback only -- access via SSH port-forward
      rpcbind=127.0.0.1
      rpcallowip=127.0.0.1
    '';
  };
}
```

The `services.bitcoind` NixOS module creates a `bitcoin` system user and
group automatically; no manual user declaration needed.

---

## Network / Firewall

nftables rules (no NixOS firewall -- same pattern as privacy VMs):

```
inbound:  SSH (22) from host; drop everything else
outbound: Tor process (uid tor) for bootstrap; bitcoin daemon for P2P over
          SOCKS5 to Tor; drop all other new outbound
```

Unlike privacy VMs, no transparent proxy is needed -- bitcoind's built-in
`proxy=` option routes P2P over Tor's SOCKS5 port directly. System Tor
runs only as a SOCKS5 proxy (no TransPort/DNSPort).

P2P inbound is disabled by default (`nolisten=1` unless a hidden service
is configured). If a Tor hidden service for inbound P2P is added later,
open port 8333 on loopback and register the onion address in bitcoind
config (`externalip=<onion>`).

---

## Pruned vs Full Node

Start pruned:

- `prune=550` -- minimum target (~1 GB retained); full UTXO set preserved
- Fast initial block download, minimal disk footprint
- Sufficient for feeding wallets (Wasabi, Lightning) via RPC

Upgrade path to full node:

- Set `prune=0`, wipe dataDir, re-sync -- or provision a fresh VM with
  a larger disk entry in `vm-specs.json`
- Full IBD with Tor is slow (~days); consider clearnet IBD then switch
  to onion-only once synced

---

## VM Specs

| Field  | Value              |
|--------|--------------------|
| Memory | 4096 MB            |
| vCPUs  | 2                  |
| Disk   | 20 GB (pruned)     |
| repos  | []                 |

IP and MAC are assigned at provisioning time from the host's DHCP range.

---

## Open Questions

- **Networking mode -- clearnet vs Tor-only**: The plan above defaults to
  Tor-only (`onlynet=onion`), but this is not the right default for all use
  cases.

  - **Mining**: pool communication and block submission are extremely
    latency-sensitive. A stale share costs real money. Tor adds 100-300 ms
    of round-trip latency and makes pool connections unreliable. A mining
    node MUST run clearnet (or at most a dual-stack config with clearnet
    preferred).
  - **Privacy-first node**: for a wallet backend (Wasabi, Lightning) where
    the goal is to not leak transaction origin or UTXO set queries to peers,
    Tor-only is the right choice. Clearnet P2P exposes your IP to every peer
    you connect to.
  - **Decision**: networking mode should be a first-class configuration
    option, not a hardcoded default. Consider parameterizing via a NixOS
    option or separate `meta.nix` field (e.g. `network = "clearnet" |
    "tor-only" | "dual"`). The `configuration.nix` and nftables rules
    diverge significantly between modes. May warrant two separate host
    profiles (`bitcoin-node-clearnet`, `bitcoin-node-private`) rather than
    one parameterized config.

- **Clearnet IBD**: Even if the final target is Tor-only, IBD over Tor is
  slow (~days to weeks). Options: (a) accept slow Tor IBD -- VM runs
  unattended, simplest; (b) allow clearnet during IBD then switch to
  onion-only after sync; (c) provision clearnet initially, resync after
  switching. Simplest approach: just set `onlynet=onion` from the start
  and wait.

- **Inbound P2P hidden service**: adds network health benefit (reachable
  node) at cost of extra Tor config. Defer until node is synced.

- **Wallet RPC**: Wasabi and Lightning clients need RPC. SSH tunnel from
  each VM is viable but tedious. Alternative: expose RPC on a host-only
  libvirt network interface (not the default NAT bridge). Defer.

- **nixpkgs version mismatch**: bitcoind-gunix pins `nixos-25.11`. If
  the project's nixpkgs lags behind, the `follows` override will break
  the reproducible build (wrong gcc/glibc version). Check with
  `flake-status` before first build; drop `follows` if versions diverge.

## Rollback Plan

Stop and undefine the service VM, remove its inventory entry and flake input,
and revert the service archetype commit. Preserve any Bitcoin data disk
separately if it contains a completed initial block download.
