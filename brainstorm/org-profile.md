# Allod org profile -- draft README

Target: `allod/.profile/README.md` (rendered as Forgejo org landing page)

---

# Allod

*An allod is land held in absolute ownership -- no rent, no fealty, no master.*

Allod is a self-sovereign NixOS VM stack for agentic coding and privacy tasks. It treats infrastructure as a declarative architecture: the code is the architecture. Hosts, guests, disk layouts, identities, repo registries, credentials, and workflow tools live as versioned Nix flakes and data files you can fork, audit, and host on your own server.

Run your forge. Run your VMs. Encrypt secrets to your keys. Plug AI providers in as replaceable services, not foundations.

## Stack archetypes

**Nexus / hypervisor** is the physical machine: libvirt/QEMU, host-side provisioning, rebuilds, key rotation, and VM lifecycle.

**Dev VMs** are reproducible coding environments with workspace clones, Forge access, build tools, and agent harnesses. Each project type can have its own cage.

**Privacy VMs** are isolated environments for Tor and privacy-sensitive work, designed without Forge access or durable project state.

**Service VMs** are persistent self-hosted services: Forgejo, CI runners, LLM routers, and other infrastructure declared with the same inventory, secrets, modules, and rebuild policy. This archetype is the natural home for services that should outlive a dev session.

## Root of trust

Allod keeps plaintext authority out of the public repos. Inventory and profile code can be public; secrets are age-encrypted; a single host identity is the root of trust that can SSH into VMs and decrypt secrets. VM SSH host keys are generated on the host and injected during provisioning so agenix can decrypt on first boot without manual state.

The public code describes the system. The private keys decide who can materialize it.

## Principles

NixOS everywhere. No configuration drift, no snowflake machines, no hidden clickops. AI agents run as first-class citizens beside the human who owns the stack.

Nothing here requires GitHub, AWS, or any service you do not control.

## Repos

| Repo | Role |
|---|---|
| [`allod/profiles`](https://forge.anarch.diy/allod/profiles) | Archetype profiles that assemble framework repos with inventory and secrets. |
| [`allod/vm`](https://forge.anarch.diy/allod/vm) | Shared QEMU guest, disk layout, NixOS, and Home Manager modules. |
| [`allod/nexus`](https://forge.anarch.diy/allod/nexus) | Hypervisor framework, provisioning scripts, lifecycle tools, and host policy. |
| [`allod/inventory`](https://forge.anarch.diy/allod/inventory) | Machine specs, VM sizing, networking, platforms, and repo registry. |
| [`allod/secrets`](https://forge.anarch.diy/allod/secrets) | Encrypted secrets, identities, credential inventory, and git policy data. |
| [`allod/tools`](https://forge.anarch.diy/allod/tools) | Workspace, Forgejo, flake, and git-policy CLI tools. |
| [`allod/memory`](https://forge.anarch.diy/allod/memory) | Version-controlled workflow memory for coding agents. |
| [`allod/strategy`](https://forge.anarch.diy/allod/strategy) | Development plans, review prompts, and project direction. |
