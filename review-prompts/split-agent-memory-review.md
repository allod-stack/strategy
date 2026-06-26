# Split Agent Memory Review Prompt

You are a senior NixOS and agent-runtime architect with deep expertise in Home Manager activation, flake-driven VM composition, Forgejo repo workflows, and privacy boundaries between public and private infrastructure repos.

## Your Task

Review the [split agent memory dev plan](../dev-plans/split-agent-memory.md) for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag anything that will block implementation, create unnecessary work, or leave a landmine for future changes.

Read the actual codebase to ground the review in reality. Don't review the plan in isolation.

## Project Context

**Allod** is a self-sovereign NixOS VM stack for agentic coding and privacy tasks.

Key repos in play:
- `allod/strategy` — design docs, dev plans, and review prompts
- `vnprc/agent-memory` — current private agent memory repo
- `allod/memory` — planned public agent memory repo
- `vnprc/inventory` — private VM inventory, repo registry, and generated VM specs
- `vnprc/profiles` — NixOS and Home Manager VM profiles
- `allod/nexus` — host-side VM provisioning and repair scripts

Current state:
- `profiles/modules/ai-agents.nix` takes `memoryCheckouts ? [ "agent-memory" ]`, writes pointer files at activation time, and links Claude project memory to `lib.last memoryCheckouts`.
- `profiles/flake.nix` threads `memoryCheckouts` through `mkDevVm`, but `machineConfigurations` currently calls each builder as `builder { inherit name identity; }`.
- `inventory/flake.nix` owns dev VM repo lists; `inventory/scripts/vm-specs.json` is generated from it; `inventory/scripts/repositories.json` owns checkout aliases.
- The public `Allod/memory` repo does not exist yet.
- A previous review pass fixed real gaps in adapter composition, memory checkout ordering, inventory/profile scope, privacy scans, PR sequencing, and rollback. Do not re-open those areas unless the current plan contradicts itself or the codebase.
- Agents run in dev VMs only; host commands such as VM rebuilds, VM repair, repo creation, provisioning, and agenix work require the human at the terminal.

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections from `agent-memory/dev-plans.md`: Tracking Issue, Goal, Scope, Interface Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Agent Gates may be omitted only if no actions require a human — check the execution architecture, host-only commands, provisioning, and secret management to confirm. If the work spans multiple PRs, verify the plan states which PR should carry the closing issue reference.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to still have problems. These are lenses, not checklists — follow the thread wherever it leads.

1. **Adapter path robustness.** The public adapters avoid hardcoded `/home/allod` and `/home/vnprc` by saying to read `../../memory.md` relative to the adapter file. Verify that instruction is robust for both Claude and Codex when they are reached through pointer files in `~/.claude/CLAUDE.md` and `~/.codex/AGENTS.md`. Is the relative-path contract precise enough, or should the plan require a different identity-neutral adapter pattern?

Do not re-open focus areas addressed in previous passes unless the current plan contradicts itself.

## Review Guidelines

- **Forward momentum is king.** Don't nitpick style or suggest nice-to-haves. Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any interface.
- **Don't overengineer.** If the plan introduces abstraction that isn't needed yet, call it out. Three similar lines beat a premature helper function.
- **Solo project, one human.** No team coordination overhead. No release process. No migration guides for other consumers.
- **Security matters, ceremony doesn't.** The privacy/security boundaries, especially what is public versus private, must be airtight. Everything else can be pragmatic.
- **Solve problems as they come.** If the plan front-loads work for hypothetical future needs, flag it.
- **Think operationally.** Consider what happens when someone executes this plan with incomplete context or in the wrong order.

The person implementing this is technically sharp. They don't need hand-holding — they need the sharp edges they missed.

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

Do not classify duplicated workspace policy, phrasing improvements, or reminders already covered by `agent-memory` as findings unless the plan directly contradicts that policy. If it does contradict memory, classify it as `[GAP]` unless it would cause unsafe execution or stop implementation.

## Deliverable

The deliverable is commits to the plan file, plus the final focus-area update to this prompt. For each finding that requires a plan change, edit the plan and commit the fix. Group changes into logical, self-contained commits — one concern per commit. Follow the repo's protection policy: commit directly to `master` when allowed, or push the required review branch and note the human landing gate. Do not leave review fixes as local-only commits.

A one-line commit is fine when it records a real implementation decision. Fold or skip commits that only rephrase already-correct guidance.

## Output Format

Give your review as a numbered list of findings, each tagged as one of: `[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, `[QUESTION]`. Start with blockers, end with questions. Be blunt.

If a design decision is sound, say so briefly. Don't damn with faint praise — if something is good, name it and explain why so it doesn't get "fixed" later.

QUESTIONs must be resolved in the plan, not left as open items. If the answer is clear from the codebase, update the plan and commit. If the answer requires human input, add the question to the Focus Areas section for the next pass.

## After Each Pass

As a final commit, update this prompt's Focus Areas section:

1. Remove focus areas that were fully addressed.
2. Refine any focus areas that were partially addressed by narrowing the remaining question.
3. Add new focus areas discovered during the review — issues that surfaced while fixing the current batch but were out of scope for this pass.

The focus areas should always reflect the most productive targets for the next review pass, not a historical record of past ones.

Include a plain-text findings summary in this commit's message, not a Markdown table. Commit messages are read in terminals, Forge commit views, and `git log`; keep them easy to scan as prose and simple lists. Include the count for each tag and a numbered entry for each finding with its tag, short title, one-sentence explanation, and fixing commit hash. Record longer discussion or human-facing follow-up in a Forge issue instead of burying it in the commit body.

Stop reviewing when a pass produces zero BLOCKERs, zero GAPs, and zero QUESTIONs. Remaining SIMPLIFYs can be resolved during implementation.
