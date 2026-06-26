# Agent Isolation Dev Plan Review Prompt

You are a senior infrastructure engineer with deep expertise in NixOS module composition, VM provisioning pipelines, Forgejo administration, and security boundary design for multi-machine deployments with secret management (agenix/age).

## Your Task

Review the [Agent Isolation Dev Plan](agent-isolation-dev-plan.md) for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag anything that will block implementation, create unnecessary work, or leave a landmine for future changes.

Read the actual codebase to ground the review in reality. Do not review the plan in isolation.

## Project Context

**Allod** is a NixOS-based infrastructure project managing dev VMs, secrets, and agent workflows through a set of interconnected flake repos on a self-hosted Forgejo instance.

Key repos in play:
- `vnprc/secrets` — private identity data, credentials, age-encrypted secrets, git workflow config
- `vnprc/inventory` — private machine definitions, vm-specs.json, repositories.json
- `vnprc/profiles` — NixOS/Home Manager profiles for all VMs
- `allod/secrets` — new public template mirroring vnprc/secrets interface with synthetic values
- `allod/inventory` — new public template mirroring vnprc/inventory interface
- `allod/memory` — public agent workflow memory (already split from agent-memory)

Current state:
- Four prerequisites are complete: repo rename, ai-agents.nix parameterization, per-VM checkout uniqueness, and agent-memory split (see `agent-isolation-roadmap.md`)
- Agents currently run on nix-dev, rust-dev, and svelte-dev VMs with full access to vnprc/* repos
- The allod-dev VM does not exist yet; this plan creates it
- `allod/secrets` and `allod/inventory` repos do not exist yet on the forge

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections from `dev-plans.md`: Tracking Issue, Goal, Scope, Interface Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Agent Gates may be omitted only if no actions require a human. If the work spans multiple PRs, verify the plan states which PR should carry the closing issue reference.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have problems. These are lenses, not checklists — follow the thread wherever it leads.

1. **Build-time vs runtime data leakage.** The plan states allod-dev is "built from vnprc/secrets like all other VMs" but claims the agent only sees public repos at runtime. Verify that the NixOS build pipeline (mkDevVm, home.nix, configuration.nix) does not embed private identity values into the VM image, environment variables, or generated config files that the agent can read at runtime. If private data is baked into the image, the filesystem isolation layer is broken regardless of which repos are cloned.

2. **Identity.nix IP inconsistency.** The synthetic identity in `allod/secrets/identity.nix` uses `192.168.122.10` for dev-1, but the `allod-dev` machine entry in `vnprc/inventory` uses `192.168.122.15`. These are in different repos with different purposes, but verify whether any flake evaluation or cross-repo reference would surface a mismatch. Also check whether `sshHosts` entries in the public identity need to reference the actual allod-dev IP or whether these values are truly isolated.

3. **Null GPG key propagation.** The plan sets `gpgSigningKey = null` for allod-agent. Trace this value through downstream consumers — git config generation, commit signing hooks, the `preferences.nix` Home Manager module, and any flake checks that validate identity fields. A null that causes a build failure or runtime error in any of these paths is a blocker.

4. **Cross-repo flake input cycle.** PR 5 (profiles) depends on PRs 1-4 being merged, and profiles must reference `allod/secrets` and `allod/inventory` as flake inputs. But allod-dev is built from `vnprc/secrets` for identity. Verify there is no circular flake input dependency. Clarify whether profiles takes allod/secrets as a direct input or whether the allod-dev profile only references vnprc/secrets (with the agent seeing allod/* repos as workspace checkouts, not flake inputs).

5. **Acceptance test blind spots.** The grep-based leak checks hardcode specific private values (username, email, GPG key, IPs). These become stale if new private data is added to vnprc/secrets later. Evaluate whether a structural approach (e.g., diffing exported attr names between public and private repos) would catch leaks the grep approach misses. Also note: there is no acceptance test for allod/memory functioning correctly on allod-dev.

6. **Rollback plan numbering and multi-repo coordination.** The rollback plan references "Phase 5" but the plan only defines Phases 1-4. Verify the rollback steps actually correspond to the phases defined. Also assess whether the multi-repo revert sequence (secrets, inventory, profiles) has ordering constraints that aren't stated — e.g., reverting profiles first to avoid a broken flake input reference.

Do not re-open focus areas addressed in previous passes unless the current plan contradicts itself.

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves. Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any interface.
- **Do not overengineer.** If the plan introduces abstraction that is not needed yet, call it out. Three similar lines beat a premature helper function.
- **Solo project, one human.** No team coordination overhead. No release process. No migration guides for other consumers.
- **Security matters, ceremony does not.** The privacy and security boundaries must be airtight. Everything else can be pragmatic.
- **Solve problems as they come.** If the plan front-loads work for hypothetical future needs, flag it.
- **Think operationally.** Consider what happens when someone executes this plan with incomplete context or in the wrong order.

The person implementing this is technically sharp. They do not need hand-holding; they need the sharp edges they missed.

## Severity Rubric

Use `[BLOCKER]` only when following the plan literally is likely to:

- perform a destructive or unsafe operation;
- fail before implementation can complete;
- leave the resulting system nonfunctional;
- violate a security or privacy boundary; or
- require missing human input that cannot be inferred from the repo or memory.

Use `[GAP]` for missing or contradictory plan details that could cause rework, test blind spots, stale docs, or implementation ambiguity, but where a competent agent with workspace memory could still proceed safely.

Use `[SIMPLIFY]` for unnecessary scope, ceremony, or abstraction. Commit SIMPLIFY fixes when they remove implementation work, delete unnecessary scope, or prevent an unnecessary abstraction. Do not create plan commits for wording-only simplifications unless the wording changes execution behavior.

Use `[QUESTION]` only when the plan cannot be corrected from repo context. If the answer is inferable, resolve it as `[GAP]` or `[SIMPLIFY]`.

Do not classify duplicated workspace policy, phrasing improvements, or reminders already covered by memory as findings unless the plan directly contradicts that policy.

## Deliverable

The deliverable is commits to the plan file, not a report. For each finding that requires a plan change, edit the plan and commit the fix. Group changes into logical, self-contained commits.

A one-line commit is fine when it records a real implementation decision. Fold or skip commits that only rephrase already-correct guidance.

## Output Format

Give your review as a numbered list of findings, each tagged as one of: `[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, `[QUESTION]`. Start with blockers, end with questions. Be blunt.

If a design decision is sound, say so briefly. QUESTIONs must be resolved in the plan, not left as open items. If the answer is clear from the codebase, update the plan and commit. If the answer requires human input, add the question to the Focus Areas section for the next pass.

## After Each Pass

As a final commit, update this prompt's Focus Areas section:

1. Remove focus areas that were fully addressed.
2. Refine any focus areas that were partially addressed.
3. Add new focus areas discovered during the review.

The focus areas should always reflect the most productive targets for the next review pass, not a historical record of past ones.

Include a plain-text findings summary in this commit's message. Include the count for each tag and a numbered entry for each finding with its tag, short title, one-sentence explanation, and fixing commit hash.

Stop reviewing when a pass produces zero BLOCKERs, zero GAPs, and zero QUESTIONs. Remaining SIMPLIFYs can be resolved during implementation.
