# Allod org profile draft

Target: `allod/.profile/README.md` (rendered as Forgejo org landing page)

---

# Allod

> An allod is land held in absolute ownership — no rent, no fealty, no master.

Allod is a self-sovereign NixOS stack: one human owner, a bare-metal host that provisions disposable VMs, a self-hosted forge, and AI agents working as first-class citizens inside those VMs.

The key architectural choice is the VM isolation model. Each project type gets its own VM that serves as a declarative development environment and a cage to contain your agents. VMs are accessed using ssh keys encrypted to the root of trust: a single key on the hypervisor, stored outside of the code repository. Version controlled agent memory, and focused CLI tools make a multi-repo workspace feel like one system.

The code is the architecture. Fork it, audit it, run it on your own hardware, and choose your forge, agents, and model routers.

## Why

Coding agents are powerful because they touch everything: repos, shells, tokens, build systems, notes, and private context. Unlike container-based approaches, Allod treats the VM as the primary security boundary while keeping performance high with direct KVM/QEMU hardware acceleration. This design removes the host kernel from the agent's attack surface, eliminating entire classes of security exploits.

- Reproducible development environments
- Project-scoped agent cages
- Secrets and identity encrypted and stored in your own repo fork
- Git-tracked memory and planning docs form the basis of the software factory
- Interchangeable integrations: GitHub or Forgejo, local or remote models, one agent harness or another.

## Architecture

Allod is declarative infrastructure split by ownership. Public framework repos describe how the system works. Consumer-owned inventory, identities, and secrets decide what exists.

| Layer | What it owns |
|---|---|
| **Nexus / hypervisor** | The physical NixOS host, libvirt/QEMU, provisioning, rebuilds, key rotation, and VM lifecycle. |
| **Dev VMs** | Agent-ready coding cages with workspace clones, build tools, agent harnesses, and project-specific state. |
| **Privacy VMs** | Isolated environments for Tor and privacy-sensitive work, designed without Forge access or durable project state. |
| **Service VMs** | Planned homes for persistent self-hosted services such as Forgejo, runners, and LLM routers. |

## Integrations

Allod does not require Forgejo. You can use GitHub, a hosted forge, or a self-hosted Forgejo instance. Self-hosted Forgejo is highly recommended for sovereignty, but bootstrapable Forgejo service VMs are planned work, not something this repo set ships today.

Forge hosts are integrations, not foundations. So are coding agents and model routers.

## Agent context

Allod keeps agent work inspectable and portable. The agent harness runs inside the VM boundary; durable instructions live in the git-tracked [`memory`](https://forge.anarch.diy/allod/memory) repo; plans, review prompts, and project direction live in [`strategy`](https://forge.anarch.diy/allod/strategy).

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
