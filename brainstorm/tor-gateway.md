# Whonix-Style Tor Gateway

**Roadmap position:** post-release capability after public profiles and the
multi-instance VM model are stable.

## Goal

Enforce Tor-only egress for privacy VMs through network topology rather than
trusting application-level proxy configuration.

## Scope

In scope:

- Add a dual-homed Tor gateway service VM.
- Add an isolated libvirt network for privacy workloads.
- Transparently proxy TCP and DNS through Tor.
- Migrate selected privacy VMs to the isolated network.

Out of scope:

- General-purpose VPN routing.
- Clearnet fallback.
- Application-managed Tor processes after migration.

## Interface Contracts

- Privacy VMs have no direct route to the external network.
- Only the Tor service identity may egress from the gateway.
- DNS and TCP from the isolated network are transparently redirected.
- Gateway failure must fail closed.

## Context

The original privacy-wallet design used a system Tor daemon as a nftables
gatekeeper: only UID 35 (`tor`) was allowed outbound. The assumption was that Wasabi
would use the system Tor's SOCKS5 at `:9050`.

This assumption was wrong. Wasabi v2 always launches its own embedded Tor binary,
running as the `wasabi` user (UID 1000). The embedded Tor was silently dropped by the
firewall for 9+ hours -- 391 failed connection attempts -- while the UI showed "Syncing."

### Short-term fix (applied to the privacy wallet VM)

- Remove `services.tor` -- system Tor is unused, wastes ~160 MB RAM
- Change nftables output rule from `skuid 35` to `skuid 1000`
- Wasabi's embedded Tor now reaches the internet; all traffic still goes through Tor
- Limitation: the OS trusts Wasabi's application-level Tor enforcement. If `UseTor`
  is set to `"Disabled"` in Wasabi's Config.json, the firewall does not catch it.

### Why the long-term fix matters

The right security model for a privacy VM is: **the application cannot make clearnet
connections regardless of its own configuration**. This requires enforcement at the
network topology level, not inside the guest OS.

---

## Target Architecture

```
[Host]
  |
  +-- libvirt external network (NAT) -- [tor-gateway VM]
  |                                           |
  |                                     UID 35 (tor) only
  |                                     allowed outbound
  |                                           |
  |                                      [Internet via Tor]
  |
  +-- libvirt isolated network --------- [tor-gateway VM] -- [privacy VM]
      (no route to host or internet)      TransPort :9040     Wasabi (example)
                                          DNSPort   :5353     UseTor: Disabled
                                          SocksPort :9050     default GW = gateway
```

Key properties:
- The workstation VM has **no route to the external network** -- enforced by libvirt
  network topology, not guest OS firewall rules
- All TCP from the workstation is transparently redirected to Tor's `TransPort`
- All DNS from the workstation is redirected to Tor's `DNSPort`
- Even a fully compromised workstation kernel cannot make clearnet connections
- Wasabi needs no Tor configuration -- `UseTor: Disabled`, transparent proxy handles it
- One Tor process (gateway), no embedded Tor, no double-Tor

---

## New VM: `tor-gateway`

VM archetype: `service` (always-on, no identity, no display)

### `hosts/service/tor-gateway/meta.nix`

```nix
{ username = "gateway"; }
```

### `hosts/service/tor-gateway/configuration.nix`

```nix
{ config, lib, ... }:
let
  internalIp  = "192.168.200.1";   # gateway's IP on the isolated network
  internalNet = "192.168.200.0/24";
  torTransPort = 9040;
  torDnsPort   = 5353;
in
{
  # -- Tor ---------------------------------------------------------------

  services.tor = {
    enable = true;
    client.enable = true;
    settings = {
      TransPort             = "${internalIp}:${toString torTransPort}";
      DNSPort               = "${internalIp}:${toString torDnsPort}";
      VirtualAddrNetworkIPv4 = "10.192.0.0/10";   # for .onion transparent proxy
      AutomapHostsOnResolve  = 1;
    };
  };

  # -- nftables: tor-only egress, transparent proxy for internal net ------

  networking.firewall.enable  = false;
  networking.nftables.enable  = true;
  networking.nftables.ruleset = ''
    table inet nat {
      chain prerouting {
        type nat hook prerouting priority -100;
        # Redirect workstation TCP to Tor TransPort
        iifname "internal" ip protocol tcp redirect to :${toString torTransPort};
        # Redirect workstation DNS to Tor DNSPort
        iifname "internal" udp dport 53 redirect to :${toString torDnsPort};
      }
    }

    table inet filter {
      chain input {
        type filter hook input priority 0; policy drop;
        iif lo accept;
        iifname "internal" accept;        # workstation traffic in
        ct state established,related accept;
        tcp dport 22 accept;              # SSH from host for management
      }
      chain forward {
        type filter hook forward priority 0; policy drop;
        # No forwarding -- transparent proxy, not a router
      }
      chain output {
        type filter hook output priority 0; policy drop;
        oif lo accept;
        ct state established,related accept;
        meta skuid ${toString config.ids.uids.tor} accept;
        drop;
      }
    }
  '';

  # -- IP forwarding not needed (transparent proxy, not routing) ----------

  boot.kernel.sysctl."net.ipv4.ip_forward" = 0;

  # -- No clearnet DNS or NTP --------------------------------------------

  services.resolved.enable   = false;
  networking.nameservers     = [];
  services.timesyncd.enable  = false;

  # -- Hardening ----------------------------------------------------------

  swapDevices = [];
  boot.tmp.cleanOnBoot = true;
  services.journald.extraConfig = ''
    Storage=volatile
    MaxRetentionSec=0
  '';
}
```

### Network interface naming

`"internal"` is a placeholder for the actual libvirt isolated network interface name.
In practice this will be something like `eth1` or `enp2s0`. Resolve at implementation
time by inspecting the running VM or using udev rules to assign a stable name.

---

## Modified VM: Privacy Wallet

### Changes from the current single-VM config

1. **Remove `services.tor`** -- no Tor runs on the workstation
2. **Remove nftables Tor-UID rule** -- replaced by network topology guarantee
3. **Single NIC** pointing to the isolated internal network only
4. **DNS** set to gateway's internal IP (Tor DNSPort handles resolution)
5. **Default gateway** set to gateway's internal IP
6. **Wasabi config**: set `"UseTor": "Disabled"` in `~/.walletwasabi/client/Config.json`
   (or seed it via a NixOS activation script)

### Simplified `configuration.nix` (networking section)

```nix
# No services.tor
# No complex nftables -- gateway enforces Tor-only

networking.defaultGateway = "192.168.200.1";
networking.nameservers    = [ "192.168.200.1" ];

# Optional: keep a minimal output firewall to block non-gateway egress
# (defense in depth -- workstation has no route out anyway)
networking.nftables.enable  = true;
networking.nftables.ruleset = ''
  table inet filter {
    chain output {
      type filter hook output priority 0; policy drop;
      oif lo accept;
      ct state established,related accept;
      ip daddr 192.168.200.1 accept;   # gateway only
      drop;
    }
  }
'';
```

### Seeding Wasabi's Tor config via activation script

To prevent Wasabi from re-enabling its embedded Tor on first run:

```nix
system.activationScripts.wasabiConfig = {
  text = ''
    cfg=/home/${username}/.walletwasabi/client/Config.json
    if [ -f "$cfg" ]; then
      ${pkgs.jq}/bin/jq '.UseTor = "Disabled"' "$cfg" > "$cfg.tmp" && mv "$cfg.tmp" "$cfg"
    fi
  '';
  deps = [ "users" ];
};
```

---

## libvirt Network Setup

A new isolated libvirt network (no DHCP, no NAT, no route to host):

```xml
<!-- tor-internal.xml -->
<network>
  <name>tor-internal</name>
  <bridge name="virbr-tor"/>
  <!-- No <forward> element = fully isolated -->
  <ip address="192.168.200.1" prefix="24"/>
</network>
```

Both `tor-gateway` and the privacy wallet VM get a NIC on this network.
`tor-gateway` additionally keeps its existing external NIC on the default libvirt network.

VM specs will need updating to reflect the second NIC on `tor-gateway`.

---

## Security Properties (vs. Current Single-VM Model)

| Property | Current (single VM) | With tor-gateway |
|---|---|---|
| Clearnet enforcement | nftables in guest OS | libvirt network topology |
| Bypassed by kernel exploit in workstation? | Yes | No |
| Bypassed by changing Wasabi `UseTor` config? | Yes | No |
| Trusts application-level Tor? | Yes | No |
| Works with any app (no Tor support needed) | No | Yes |
| Reusable for future privacy VMs | No | Yes (shared gateway) |
| Infrastructure complexity | Low | Medium (two VMs, isolated network) |

---

## Reuse: Shared Gateway for Multiple Privacy VMs

Once built, `tor-gateway` can serve multiple workstation VMs on the same isolated
network -- any privacy VM that needs Tor-only egress. All get transparent
Tor proxying with zero per-VM Tor configuration.

---

## Prerequisites

- `tor-gateway` VM added to VM specs with two NICs
- Service VM factory implemented in `flake.nix`
- Isolated libvirt network created on host before VMs are provisioned
- Privacy wallet VM rebuilt from updated `configuration.nix`

---

## Acceptance Tests

```bash
# On the privacy wallet VM -- confirm no direct internet
curl https://check.torproject.org/api/ip    # must fail (no route to external net)

# On the privacy wallet VM -- confirm Tor works via gateway
curl --resolve check.torproject.org:443:$(dig +short check.torproject.org @192.168.200.1) \
  https://check.torproject.org/api/ip        # should return Tor exit IP

# On tor-gateway -- confirm only Tor UID goes out
sudo nft list ruleset

# On tor-gateway -- confirm Tor is healthy
systemctl status tor

# Wasabi: launch, confirm sync proceeds without embedded Tor in ps output
ps aux | grep -i tor   # should show only system tor on gateway, nothing on workstation
```

## Rollback Plan

Restore each privacy VM to its previous libvirt network and application-level
Tor configuration, rebuild it, then remove the isolated network and gateway
VM. Do not remove the known-good single-VM firewall configuration until the
gateway acceptance tests pass.
