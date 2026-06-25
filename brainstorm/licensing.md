# Licensing: Top 3 Options for Allod

## Context

Allod is a self-sovereign NixOS VM stack for agentic coding and privacy tasks. The public repos live on a self-hosted Forgejo instance (forge.anarch.diy/allod). The project includes CLI tools (forge, allod change), NixOS modules, VM configs, and workspace orchestration scripts. Some repos stay private (secrets, inventory, nvim-config, agent-memory); the licensing decision applies to the public repos.

Core values that inform the license choice: sovereignty, privacy, no central authority, self-hosting over platform dependence.

---

## Option 1: AGPL-3.0 (Affero GPL v3)

**What it does:** Strong copyleft. Everything GPL-3.0 does, plus: anyone who runs a modified version as a network service must offer the source to users of that service.

**Why it fits allod:**

- The AGPL's network clause is the only copyleft that actually bites in the SaaS era. If someone forks allod's tooling into a hosted "agentic dev platform," they must publish their modifications. GPL alone wouldn't require this — they'd only need to share source if they distributed binaries, and SaaS distribution dodges that.
- Allod's tools are designed to be run as services (agent harnesses, forge API interactions, VM orchestration). The network clause is directly relevant to how this code would be reused.
- Signals intent clearly: this project is meant to be sovereign infrastructure, not a free lunch for platform vendors.
- Used by: Nextcloud, Grafana, MongoDB (before SSPL), Mastodon.

**Downsides:**

- Some companies have blanket policies against AGPL dependencies. This limits corporate adoption — but that may not matter for allod's audience.
- NixOS ecosystem leans permissive (nixpkgs is MIT). AGPL NixOS modules are unusual and could create friction if someone wants to upstream anything.
- If allod ever wants to be a flake input consumed by other people's NixOS configs, AGPL's copyleft trigger on "conveying" could create ambiguity about whether importing a flake makes your config a derivative work. (In practice, Nix evaluation is build-time tooling, not linking — but the legal theory is untested.)

**Best if:** You want maximum protection against proprietary forks and SaaS extraction, and you're not concerned about corporate adoption friction.

---

## Option 2: GPL-3.0

**What it does:** Strong copyleft. Derivative works must be GPL-licensed. Includes patent grant and anti-tivoization provisions (vs GPLv2).

**Why it fits allod:**

- Battle-tested, universally understood. No ambiguity about what it means.
- Strong enough to prevent proprietary forks — anyone distributing modified allod tools must share source under GPL.
- The patent grant in v3 protects contributors and users from patent trolling.
- Anti-tivoization: if someone ships allod on locked-down hardware, users must still be able to install modified versions. Relevant if allod VM images ever get distributed as appliances.
- Natural fit for the Linux/NixOS ecosystem. The kernel is GPLv2; many system tools are GPLv3. Shell scripts and CLI tools sit comfortably here.

**Downsides:**

- No network/SaaS clause. Someone can take allod's agent harness tooling, modify it, run it as a hosted service, and never share a line of code. They only trigger copyleft if they distribute the software itself.
- For shell scripts specifically, the "distribution" trigger is straightforward (you ship the script), but the derivative work boundary for NixOS modules consumed via flake inputs is the same gray area as AGPL.

**Best if:** You want solid copyleft without the AGPL's extra friction, and the SaaS loophole doesn't concern you because allod's audience self-hosts by definition.

---

## Option 3: MPL-2.0 (Mozilla Public License 2.0)

**What it does:** Weak/file-level copyleft. Modified MPL files must stay MPL and source-available. But MPL code can be combined with proprietary code in a larger work, as long as the MPL files themselves remain open.

**Why it fits allod:**

- Pragmatic middle ground. Modifications to allod's actual files flow back; new files built on top don't have to.
- Much simpler compliance than GPL/AGPL. No "linking" ambiguity — copyleft is per-file, period. This matters for NixOS modules where the derivative-work boundary is fuzzy.
- Explicitly compatible with GPL (MPL 2.0 Section 3.3) — allod tools can be combined with GPL projects without license conflict.
- Lower friction for adoption. Companies that avoid GPL often accept MPL.
- Used by: Firefox, Terraform (before BSL switch), HashiCorp tools historically.

**Downsides:**

- Weaker protection. Someone can write a proprietary wrapper around allod's tools, only keeping the original files open. The "spirit" of the contribution stays open, but the valuable integration work doesn't.
- Doesn't signal the same sovereignty ethos as GPL/AGPL. MPL says "play fair with my files"; GPL says "the whole work stays free."
- No network clause — same SaaS gap as GPL.

**Best if:** You want copyleft protection for your actual code but don't want to impose license constraints on people who build larger systems that include allod components.

---

## Recommendation

The question is whether the SaaS loophole matters. Allod is sovereignty-oriented infrastructure — the people who care about it are self-hosters, not SaaS consumers. The realistic threat isn't "someone runs allod as a service" but "someone forks the tools into a proprietary product."

**GPL-3.0** covers that threat cleanly, is universally understood, and fits naturally in the Linux/NixOS ecosystem. It's the conservative-correct choice.

**AGPL-3.0** is worth choosing if you believe agentic dev tooling will increasingly be offered as hosted services and you want to preemptively close that door. The friction cost is real but small for allod's audience.

**MPL-2.0** is the right pick if you find yourself wanting to make allod modules easy to consume as flake inputs without copyleft anxiety — but it gives up the "whole work stays free" guarantee that aligns with allod's sovereignty values.

The choice between GPL and AGPL is the real decision. MPL is the escape hatch if copyleft boundary concerns around Nix flake composition turn out to be a real problem.
