# allod org profile -- draft README

Target: `allod/.profile/README.md` (rendered as Forgejo org landing page)

---

# allod

*An allod is land held in absolute ownership -- no rent, no fealty, no master.*

A self-hosted NixOS stack for people who want to own their infrastructure. Code on your forge. VMs on your hardware. Secrets encrypted to your keys.

## The stack

**Nexus** is the bare metal host -- home base and portal to everything else. From here you SSH into purpose-built VMs for different kinds of work: Rust development, Nix configuration, privacy-sensitive tasks. Each VM is reproducible, declarative, and replaceable in a single command.

**The Forge** is a self-hosted Forgejo instance. PRs, issues, CI runners -- yours, on your hardware.

**Tools** are the CLI scripts that make working across a multi-repo, multi-VM workspace feel like one coherent environment: `pull-all`, `work-diff`, `flake-status`, `forge`, `flake-update-cascade`.

## Principles

NixOS everywhere -- no configuration drift, no snowflake machines. Secrets age-encrypted in version control. AI agents run as first-class citizens alongside the human who owns the stack.

Nothing here requires GitHub, AWS, or any service you don't control.

## Repos

| | |
|---|---|
| [`allod/tools`](https://forge.example.org/allod/tools) | Workspace CLI tools |
| [`allod/profiles`](https://forge.example.org/allod/profiles) | NixOS configs for all VMs |
| [`allod/nexus`](https://forge.example.org/allod/nexus) | Nexus configuration |
| [`allod/vm`](https://forge.example.org/allod/vm) | VM bootstrap and install |
