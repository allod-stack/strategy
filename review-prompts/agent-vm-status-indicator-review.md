# Agent VM Status Indicator Review Prompt

You are the terminal-harness operator people call when "just add a footer" turns
into a Home Manager activation failure, a broken TypeScript extension, and a JSON
settings file that somebody's CLI rewrote behind your back. NixOS module
arguments, Home Manager activation DAGs, generated store-path artifacts, Claude
Code settings semantics, Pi extension loading, and Codex TUI config limits are
all in your wheelhouse. This plan wires persistent VM identity into long-running
agent sessions; the sharp edge is not the text `VM: allod-dev`, it is proving the
text is generated from the right source, installed without clobbering operator
state, and fails loudly only where the operator can repair it. Do not hold back.

## Your Task

Review the [Persistent VM Hostname Status Indicator dev plan](../dev-plans/agent-vm-status-indicator.md)
for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag
anything that will block implementation, create unnecessary work, or leave a
landmine for future changes.

Read the actual codebase to ground the review in reality. Do not review the plan
in isolation. The relevant behavior is in `allod/profiles` today, plus the local
installed harness versions on the dev VM. The plan is single-repo implementation
work, but it crosses generated Home Manager activation, mutable user settings,
and runtime-loaded extension code; source review alone is not enough.

## Project Context

**Allod** is a self-sovereign NixOS VM stack for agentic coding, with dev VMs
assembled from flake repos on a self-hosted Forgejo instance.

Key repos in play:
- `allod/profiles` - the implementation repo. Owns
  `modules/ai-agents.nix`, `hosts/dev/allod-dev/home.nix`, and the `flake.nix`
  checks. The one implementation PR carries `Closes allod/profiles#7`.
- `allod/strategy` - owns this dev plan and review prompt.
- `allod/memory` - workspace policy, especially `dev-plans.md` risk levels and
  the generated-lifecycle review lens.

Current state (verify against the tree before trusting it; line numbers and file
layout drift):
- `modules/ai-agents.nix` currently has the shape
  `{ identity, memoryCheckouts ? [ "allod/memory" ] }: { lib, pkgs, ... }:`.
  It has one activation entry, `home.activation.llmMemoryLinks`, which writes
  `~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`, and
  `~/.pi/agent/AGENTS.md` atomically. It does not touch
  `~/.claude/settings.json` or `~/.pi/agent/extensions/`.
- `flake.nix` imports `ai-agents.nix` only in `mkDevVm`, under
  `home-manager.users.${identity.username}.imports`. Privacy and hypervisor
  builders do not import it. `mkDevVm` passes only `identity` and
  `memoryCheckouts` to the outer function today; `home-manager.extraSpecialArgs`
  includes `pkgs-unstable`, `allod-tools`, `identity`, and `runtimeIdentity`.
  If the implementation relies on a Home Manager-provided `osConfig` module
  argument, verify that it is actually present in this NixOS integration.
- `hosts/dev/allod-dev/home.nix` installs
  `pkgs-unstable.claude-code`, `pkgs-unstable.codex`, and
  `pkgs-unstable.pi-coding-agent`. On the current dev VM those commands report
  `claude` 2.1.204, `codex` 0.142.5, and `pi` 0.80.3.
- Existing checks live in `checks.<system>` inside `flake.nix`.
  `pi-integration` gets
  `machineConfigurations.allod-dev.config.home-manager.users.${username}`,
  writes `home.activation.llmMemoryLinks.data` through `pkgs.writeText`, and
  greps the generated activation script. The planned `agent-vm-status` check is
  a sibling of that pattern, but it must do more than grep source text: it must
  execute the generated Claude scripts and validate the Pi extension artifact.
- `nix eval --raw .#nixosConfigurations.allod-dev.config.networking.hostName`
  currently returns `allod-dev`, and `nix eval --raw .#vmFacts.allod-dev.username`
  returns `allod`. The plan's one-source-of-truth claim depends on baking the
  former into both harness artifacts.
- Pi 0.80.3's installed example
  `pi-monorepo/examples/extensions/status-line.ts` uses
  `ctx.ui.setStatus(...)` on `session_start`. Its package manager auto-discovers
  user extensions from `~/.pi/agent/extensions`, follows symlinked files, and
  skips dangling symlinks after `stat` fails. Do not generalize this beyond the
  installed version without checking.
- Claude Code's `statusLine` command behavior is external CLI behavior, not a
  Nix contract. The plan says the command receives session JSON on stdin and
  that no hostname field exists there; verify against the installed docs or the
  official docs if needed.
- Codex 0.142.5 is explicitly out of scope because the plan says its
  `tui.status_line` only accepts predefined items. Confirm that against the
  installed version before blessing the exclusion; if the hook exists now, the
  scope decision needs to be revisited.
- Execution constraint: agents run inside dev VMs. A real `nixos-rebuild` or
  live TUI eyeball after rebuild is human-only; generated flake checks, `nix
  eval`, local fixture scripts, and headless Pi/Claude probes are agent-runnable.

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections
from `dev-plans.md`: Tracking Issue, Goal, Scope, Risk Assessment, Interface
Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. This is intended as
a single `allod/profiles` PR, so verify the closing-keyword story says that PR
carries `Closes allod/profiles#7`. Agent Gates are required because live rebuild
and live TUI verification are human-only. Verify the plan's residual risk score
is calibrated to generated Home Manager activation plus mutation of
operator-authored `~/.claude/settings.json`, not just to the small amount of
Nix code.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have
problems. These are lenses, not checklists - follow the thread wherever it
leads.

The six standing lenses in `dev-plans.md` (internal consistency, operational
sequencing, risk calibration, acceptance-test coverage, rollback fidelity,
generated lifecycle behavior) apply as defaults on top of the plan-specific
areas below.

1. **Home Manager module boundary and hostname source.** The plan adds
   `osConfig` to the inner `ai-agents.nix` module and reads
   `osConfig.networking.hostName`. Verify this argument is available in the
   current `home-manager.nixosModules.home-manager` integration, or that the
   implementation explicitly passes a hostname from `mkDevVm`. A missing module
   arg is a build-time blocker; a fallback to runtime `hostname`, identity data,
   or duplicated profile data violates the plan's one-source-of-truth premise.
   Also confirm the module still imports only for dev VMs and does not leak this
   behavior into privacy or hypervisor profiles.

2. **Activation failure semantics and mutable Claude settings.** The plan relies
   on an activation-time `jq` merge that owns only `.statusLine`, rejects empty,
   scalar, array, multi-document, and malformed settings, and leaves the file
   byte-unchanged on refusal. Trace that through the generated Home Manager
   activation script, not only the Nix source. Does a refusal fail the right unit
   loudly, with a repairable error, and without half-writing a temp file? Does
   `entryAfter ["writeBoundary"]` interact correctly with existing activation
   entries? Challenge the R2 risk claim here: blocking Home Manager activation
   on a user-authored settings file may be R3 unless the tests and rollback story
   really contain the blast radius.

3. **Generated artifact closure and check mechanics.** The new flake check plans
   to extract store paths from `home.activation.agentVmStatus.data` and execute
   them in the build sandbox. Verify the activation text preserves Nix string
   context so `piStatusExtension`, `claudeStatusLine`, and `claudeStatusMerge`
   are realized in the check closure, and that the grep/regex extraction cannot
   silently grab the wrong path. The check must execute the actual generated
   merge/status scripts with `jq` available, prove idempotence and refusal
   behavior, and include sabotage cases that can fail for the intended reason.

4. **Pi runtime proof, discovery, and sabotage.** The plan's sharpest runtime
   risk is a malformed Pi extension preventing `pi` from starting. Verify the
   implementation tests the real generated `.ts` artifact under the installed
   Pi runtime, ideally `pi --mode rpc --offline -e "$piext"` with an isolated
   `HOME`, and asserts a `setStatus` event carrying the baked hostname. Also
   verify the broken-extension sabotage genuinely fails before emitting
   `setStatus`, does not depend on an API key, and would catch a load/parse
   regression. If the build sandbox cannot run Pi, the VM-side step must remain
   explicit and non-optional.

5. **Claude `statusLine` contract and settings overwrite policy.** Confirm the
   generated settings shape is exactly what Claude Code 2.1.204 expects:
   camel-case `statusLine`, `{ "type": "command", "command": "<store path>" }`,
   command stdout rendering, stdin ignored safely, and workspace-trust caveats
   correctly classified as live-TUI verification notes rather than defects.
   The merge overwrites any existing `.statusLine`; decide whether that is the
   right ownership boundary and make sure the plan says so plainly. Verify
   rollback behavior when the configured command points to a missing store path,
   because `claude -p` and interactive Claude can differ in how they handle bad
   settings.

6. **Codex exclusion and scope discipline.** Codex is out of scope only if the
   installed `codex` really cannot render custom text or an external command in
   the persistent TUI footer. Verify the full local config surface rather than
   trusting a partial identifier list. If Codex still lacks the hook, keep the
   exclusion and do not accept wrapper hacks, prompt text, shell aliases, or
   one-time startup banners as substitutes for a persistent in-harness status
   surface.

7. **Rollback residue and partial states.** The rollback plan says a reverted PR
   plus human rebuild leaves Claude blank and Pi with an inert dangling symlink.
   Test or inspect the actual failure modes: status script missing, merge script
   never ran, Pi symlink created but Claude merge failed, Claude merge succeeded
   but Pi link failed, and store path garbage-collected later. Rollback should
   preserve non-`statusLine` Claude settings, remove only the generated Pi
   symlink when requested, and never require host-side commands beyond the human
   rebuild.

8. **SIMPLIFY sweep.** Reconsider whether every planned artifact earns its
   keep. A factored Claude merge script is justified only if the check executes
   it behaviorally; otherwise it is ceremony. Likewise, do not let the Pi tests
   grow into a package-manager harness if a direct `pi --mode rpc -e` probe proves
   the contract. Delete optional extras, alternative status text, or future
   harness hooks that do not move `VM: <hostname>` into Pi and Claude now.

Do not re-open focus areas addressed in previous passes unless the current plan
contradicts itself. This is the first pass; the areas above are the seed.

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves.
  Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any
  interface, but do not silently clobber operator-authored settings or broaden
  agent authority.
- **Do not overengineer.** If the plan introduces abstraction that is not needed
  yet, call it out. Three explicit generated artifacts beat a generic harness
  integration layer. Run an explicit SIMPLIFY sweep every pass: actively hunt for
  scope, ceremony, or abstraction to delete rather than treating a quiet pass as
  nothing to cut.
- **Solo project, one human.** No team coordination overhead. No release
  process. No migration guides for other consumers.
- **Security matters, ceremony does not.** No secrets are involved, but the
  privacy boundary and host authority boundary still matter: agents must not run
  host rebuilds or invent runtime sources of truth.
- **Solve problems as they come.** If the plan front-loads optional status
  fields, generic harness registries, or speculative Codex workarounds, flag it.
- **Think operationally.** Consider a fresh dev VM with no existing Claude
  settings, an operator-customized settings file, a malformed settings file, a
  dangling Pi symlink after rollback, and a human rebuilding after the PR lands.
- **Calibrate residual risk.** The risk score is for human triage after
  validation passes. Generated activation plus mutable local user state may
  deserve more scrutiny than the small code diff suggests; challenge both
  under- and over-statements using the actual tests and rollback fidelity.
- **Inspect generated lifecycle artifacts.** For this plan, the generated Home
  Manager activation script, Nix store shell scripts, symlink target, merged JSON,
  and runtime extension load are the contract. Source evaluation alone does not
  count.

The person implementing this is technically sharp. They do not need
hand-holding; they need the sharp edges they missed.

## Severity Rubric

Use `[BLOCKER]` only when following the plan literally is likely to:

- perform a destructive or unsafe operation;
- fail before implementation can complete;
- leave the resulting system nonfunctional;
- break first boot, activation, provisioning, rebuild, rotation, or rollback
  lifecycle behavior;
- violate a security or privacy boundary; or
- require missing human input that cannot be inferred from the repo or memory.

Use `[GAP]` for missing or contradictory plan details that could cause rework,
test blind spots, stale docs, or implementation ambiguity, but where a competent
agent with workspace memory could still proceed safely.

Use `[GAP]` when the Risk Assessment is missing, materially understated,
materially overstated, or unsupported by the acceptance tests and rollback plan.
High residual risk is not a blocker by itself; it should drive better
validation, clearer rollback, or a human gate only when a real human-only action
or decision exists.

Use `[SIMPLIFY]` for unnecessary scope, ceremony, or abstraction. Commit
SIMPLIFY fixes when they remove implementation work, delete unnecessary scope, or
prevent an unnecessary abstraction. Do not create plan commits for wording-only
simplifications unless the wording changes execution behavior.

Use `[QUESTION]` only when the plan cannot be corrected from repo context. If the
answer is inferable, resolve it as `[GAP]` or `[SIMPLIFY]`.

Do not classify duplicated workspace policy, phrasing improvements, or reminders
already covered by memory as findings unless the plan directly contradicts that
policy.

## Deliverable

The deliverable is not a report. Every review pass ends with:

1. Plan-file commits for findings that require plan changes, or an explicit
   no-findings result.
2. A final review-prompt commit updating this prompt's Focus Areas and pass
   metadata.
3. A push to the remote.

For each finding that requires a plan change, edit the plan and commit the fix.
Group changes into logical, self-contained commits.

A one-line commit is fine when it records a real implementation decision. Fold
or skip commits that only rephrase already-correct guidance.

## Output Format

Give your review as a numbered list of findings, each tagged as one of:
`[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, `[QUESTION]`. Start with blockers, end with
questions. Be blunt.

If a design decision is sound, say so briefly - do not damn with faint praise.
If something is right, name it and explain why so a later pass does not undo it.
QUESTIONs must be resolved in the plan, not left as open items. If the answer is
clear from the codebase, update the plan and commit. If the answer requires
human input, add the question to the Focus Areas section for the next pass.

## After Each Pass

As a final commit, update this prompt's Focus Areas section:

1. Remove focus areas that were fully addressed.
2. Refine any focus areas that were partially addressed.
3. Add new focus areas discovered during the review.

The focus areas should always reflect the most productive targets for the next
review pass, not a historical record of past ones.

Include a plain-text findings summary in this commit's message:

- Count only findings new to this pass, by tag. Carried-over unresolved items
  are not findings; move them to Focus Areas rather than re-counting them, so
  the per-pass counts track a real severity trend instead of inflating it.
- Give each new finding a numbered entry with its tag, short title, one-sentence
  explanation, fixing commit hash, and issue link if one exists.
- Classify each finding's origin: an original-plan defect, or introduced by an
  earlier review pass (name the commit that introduced it).
- State what the SIMPLIFY sweep considered for deletion this pass, even when
  nothing was cut.
- Put a final `Model: <exact model>` footer at the bottom (for example,
  `Model: claude-opus-4-8`, `Model: gpt-5`). Use the exact model identifier,
  not the agent framework or product name.

When a pass commits a structural or design change, the next pass should be a
scoped diff review of that change, run by a different model than the one that
wrote the fix. In the Focus Areas update, add a `Next pass:` line that names the
commit(s) to review, says whether it is a scoped diff or a full pass, and
recommends a model other than the fix's author. Record in the same update how
the fix held up so its stability stays traceable.

Stop the plan-text review when either condition holds:

- Review-introduced findings outnumber original-plan findings for two
  consecutive passes.
- Two consecutive passes produce no BLOCKER and no original-plan GAP.

At that point the plan text has converged; hand any remaining focus areas to
implementation review, and resolve remaining SIMPLIFYs during implementation.

## Before Final Response

- Plan fixes are committed, or the pass explicitly found no plan changes.
- This review prompt's Focus Areas are updated and committed.
- The final review-prompt commit message includes the findings summary and
  `Model:` footer.
- The repo is pushed to the remote.
- `git status` is clean.
