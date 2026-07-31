# Runtime-Selected Guest Module Review Prompt

You have spent years inside Nix flake evaluation semantics, NixOS module composition, and the exact moment a lazily-forced assertion decides whether a machine boots or bricks. You can read a `flake.lock` graph the way most people read a sentence, and you have a specific, personal grudge against configuration that *looks* load-bearing and isn't. This plan is about exactly that: a label that decides nothing. Go find what it still gets wrong. Do not be polite about it.

## Your Task

Review [runtime-guest-module-selection.md](../dev-plans/runtime-guest-module-selection.md) for gaps, wrong premises, and landmines. Be direct and specific. Flag anything that blocks implementation, creates unnecessary work, or leaves a trap for the milestone that follows.

Read the actual code. The plan makes several empirical claims — derivation-path equality, what forces which assertion, what fails without a volume. Verify them against the tree rather than trusting the prose. If a claim is wrong, that is the highest-value finding you can return.

## Project Context

**Allod** is a self-sovereign NixOS VM stack for agentic coding and privacy tasks.

Repos in play (local checkouts under `/home/allod/work/allod/`, all at origin tip):

- `archetypes` — the VM framework. `flake.nix` builds every machine; `sharedModules` at `flake.nix:60` is what this plan changes. **This is the repo being changed.**
- `vm` — guest and host NixOS modules. Owns `nixosModules.qemuGuest` and `nixosModules.microvmGuest` and their contract checks. Not changed here.
- `inventory` — machine data, including the `runtime` fact. Public template with synthetic example machines. Not changed here.
- `strategy` — where this plan and its parent live.

Current state, verify before trusting:

- `archetypes/flake.nix:61` hard-codes `vm.nixosModules.qemuGuest` into every dev and privacy VM. `mkHypervisor` inlines its own module list and does not use `sharedModules`.
- The fleet is `allod-dev` (dev), `privacy-1` (privacy), `nexus` (hypervisor). Both guests are `runtime = "libvirt"`. No machine anywhere declares `microvm`.
- `archetypes/flake.lock` pins `inventory` and `vm` at revisions predating the runtime fact and the microvm guest module respectively.
- Agents run in dev VMs. `nix flake check` in archetypes takes ~20s offline. No host access, no real credentials.

The parent plan is [microvm-framework-adoption.md](../dev-plans/microvm-framework-adoption.md) — a large, heavily-reviewed arc. This plan implements its **contract 2** only. Do not re-review the parent's settled contracts; do check that this plan is consistent with them and does not quietly violate contract 1, 1a, 13, or 21.

## Structural Conformance

Verify the plan includes every section required by `dev-plans.md`: Tracking Issue, Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates, Acceptance Tests, Rollback Plan. Agent Gates may be omitted only if no actions require a human — this plan claims none apply; check that. Verify the closing-keyword assignment is right: this PR closes `allod/archetypes#22` but must not close `allod/strategy#20`.

## Focus Areas

The six standing lenses in `dev-plans.md` apply as defaults. Concentrate here:

1. **The no-op proof's attribution.** The plan claims the input bump is byte-identical for all three machines and therefore requires the derivation-path comparison be taken across the *selection commit*, with the bump already committed. Is that sequencing actually sufficient to attribute a difference to the selection? Is there a path by which the selection changes the derivation for a libvirt machine that this test would miss — or, conversely, a way it produces a false pass? Re-measure the three paths yourself.

2. **Whether the selection can be written without a fallback.** Contract 1 forbids any default branch. Given `machines.${name}.runtime` must be read inside `sharedModules`, and `checkedMachines` returns machines unfiltered including the runtime-less `nexus`, is a total, fallback-free selection actually expressible cleanly here? What does the natural Nix idiom for it look like, and does contract 3's hypervisor guard belong as an assertion, a filter, or nothing at all? Push back if the plan is asking for ceremony.

3. **Whether the check can pass by construction.** Contract 4 says the check must compare two independently derived facts rather than restate the builder's conditional. Look hard at whether the five assertions in the Acceptance Tests actually achieve that, particularly fixture 2 — it overrides a runtime on a synthetic machine, and the fixture's runtime and the builder's input may be the same value threaded twice. Fixture 5 is meant to be the guard against this; is it?

4. **The microvm fixture's volume placeholder.** Contract 5 has the check fixture declare a `microvm.volumes` entry the builder does not. Is that a legitimate seam or is it hiding the fact that the selection is untestable at this scope? Consider whether it creates a fixture that diverges from what a real microvm machine will eventually compose, and whether that divergence will silently invalidate this check when contract 13's volumes land.

5. **Scope honesty.** The plan defers `vmFacts.<name>.runtime` (parent contract 1) to a later milestone. Is that defensible, or does it leave the runtime fact half-wired in a way that makes the next milestone harder than doing it now? Run an explicit SIMPLIFY sweep: what in this plan can be deleted outright?

6. **Risk calibration.** The plan scores R2 and explicitly invites challenge, arguing the parent's R3 row covers a much larger scope. Is R2 right? The change alters guest composition for every machine and bumps two cross-repo inputs — both R3 signals — against a measured fleet-wide no-op and a microvm branch unreachable from real data. Decide, and say which premise would have to be wrong for your answer to flip.

## Review Guidelines

- **Forward momentum is king.** No style nitpicks, no nice-to-haves. Only what will actually cause problems.
- **No backwards compatibility required.** Pre-alpha. Any interface can break.
- **Do not overengineer.** Three similar lines beat a premature helper. Hunt for scope and ceremony to delete every pass.
- **Solo project, one human.** No team coordination, no release process, no migration guides.
- **Security matters, ceremony does not.**
- **Think operationally.** What happens when someone executes this cold, or in the wrong order?
- **Inspect generated lifecycle artifacts.** Do not stop at source evaluation — this is guest composition, so look at what the built system actually reports.

## Review Evidence

| Pass | Model | Effort | Reviewed commit | Findings | Fixing commit | Survived later passes |
|---|---|---|---|---|---|---|
| 1 | `gpt-5.6-sol` | `high` | (this pass) | | | |

Plan authored by `claude-opus-5`. Pass 1 is a cross-vendor rotation, the strongest available.
