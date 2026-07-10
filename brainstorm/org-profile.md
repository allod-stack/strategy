# Allod org profile draft

Target: `allod/.profile/README.md` (rendered as Forgejo org landing page)

---

# Allod

**Own the machine your agents run on.**

Allod is a self-sovereign NixOS stack for agentic development: self-hosted Forgejo, isolated QEMU VMs, age-encrypted secrets, and focused CLI tools that make a multi-repo workspace feel like one system.

The code is the architecture. Fork it, audit it, host it on your own server, and rebuild it on your own hardware.

No GitHub dependency. No cloud IDE. No agents roaming around your host.

## Why

Coding agents are powerful because they touch everything: repos, shells, tokens, build systems, notes, and private context. Allod gives that power a boundary.

- Reproducible development VMs instead of fragile laptops and cloud workspaces.
- Self-hosted git, issues, pull requests, and runners instead of rented workflow control.
- Explicit secrets and identity data instead of credentials scattered across machines.
- Agent-friendly tooling without making proprietary AI providers the foundation.

## Architecture

Allod is declarative infrastructure split by ownership. Public framework repos describe how the system works. Consumer-owned inventory and secrets decide what actually exists and who can materialize it.

| Layer | What it owns |
|---|---|
| **Nexus / hypervisor** | The physical NixOS host, libvirt/QEMU, provisioning, rebuilds, key rotation, and VM lifecycle. |
| **Dev VMs** | Agent-ready coding cages with workspace clones, Forge access, build tools, and project-specific state. |
| **Privacy VMs** | Isolated environments for Tor and privacy-sensitive work, designed without Forge access or durable project state. |
| **Service VMs** | Planned homes for persistent self-hosted services such as Forgejo, runners, and LLM routers. |

## Trust model

Public repos describe machines. Private keys activate them.

A single host identity is the root of trust: it can SSH into guests and decrypt age secrets. VM SSH host keys are generated on the host and injected during provisioning, so agenix can decrypt VM secrets on first boot without manual post-install state.

## Repos at a glance

| Repo | Purpose |
|---|---|
| [`profiles`](https://forge.anarch.diy/allod/profiles) | Assembles hosts and VM archetypes from framework modules, inventory, and secrets. |
| [`vm`](https://forge.anarch.diy/allod/vm) | Shared QEMU guest, disk, NixOS, and Home Manager modules. |
| [`nexus`](https://forge.anarch.diy/allod/nexus) | Host framework and provisioning lifecycle tools. |
| [`inventory`](https://forge.anarch.diy/allod/inventory) | Machine specs, VM sizing, networking, platforms, and repo registry. |
| [`secrets`](https://forge.anarch.diy/allod/secrets) | Encrypted secrets, identities, credential inventory, and git policy data. |
| [`tools`](https://forge.anarch.diy/allod/tools) | Workspace sync, Forgejo, flake, and protected-git workflow CLIs. |
| [`memory`](https://forge.anarch.diy/allod/memory) | Version-controlled workflow memory for coding agents. |
| [`strategy`](https://forge.anarch.diy/allod/strategy) | Project direction, development plans, and review prompts. |

## Status

Allod is active infrastructure, not a polished appliance. The public repos are being split, licensed, and documented so the stack can be forked, audited, and adapted without inheriting private machine state.

Start with [`profiles`](https://forge.anarch.diy/allod/profiles) to see how the pieces assemble.
