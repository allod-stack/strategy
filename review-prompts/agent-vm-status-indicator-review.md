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

Pass 2 (claude-opus-4-8) refuted the pass-1 ordering fix and replaced it with a
non-fatal merge (commit ba7d476); pass 3 is primarily a scoped diff review of that
fix, plus the carried live/VM-side items below.

1. **[SCOPED DIFF - highest priority] Non-fatal merge fix (commit ba7d476).**
   Pass 2 proved the pass-1 ordering anchor (7ab6771) does NOT place
   `agentVmStatus` last: evaluating the real allod-dev DAG through home-manager's
   own `lib.hm.dag.topoSort` (home-environment.nix:753) sorts it 10th of 12, with
   `installPackages` and `syncPrBranchProtection` AFTER it - so under the shared
   `set -eu -o pipefail` activation a fatal merge would still skip a package
   install and the branch-protection sync. The fix invokes the merge non-fatally
   (`${claudeStatusMerge} || true`), drops the ordering anchor (bare
   `entryAfter ["writeBoundary"]`), rewrites Design Decisions 1/4 and the Risk
   Assessment, and adds an acceptance-test grep that the merge is invoked
   non-fatally. Verify: (a) does `|| true` truly satisfy principle 11 - is the
   merge script's stderr ERROR actually surfaced to the operator during
   `nixos-rebuild switch` and in `journalctl -u home-manager-<user>.service`, or
   should the entry also call home-manager's `warnEcho`/`errorEcho` so the failure
   is unmissable? (b) The Pi `mkdir`/`ln` steps stay FATAL - confirm that is
   acceptable (deterministic store ops, no operator input) or make the whole entry
   non-fatal. (c) Does the new grep assertion
   (`claude-vm-status-merge[^|]*\|\| *(true|:)`) actually fail if someone reverts
   to a fatal invocation? (d) With a merge refusal no longer failing the unit, is
   R2 (vs R1) still right - the mutation of operator-authored `settings.json` is
   the remaining justification.

2. **Claude behavior with a missing/GC'd command path (carry, live-TUI,
   human-gated).** The rollback story claims a `statusLine.command` pointing at a
   garbage-collected store path makes Claude silently blank the row. Verify that
   against Claude Code 2.1.204 - both `claude -p` and interactive - rather than
   erroring or refusing to start. This is the one rollback claim with no
   agent-runnable test.

3. **Codex exclusion (carry, one-glance confirm at implementation).** Pass 2 could
   not settle this from the installed binary: codex 0.142.5 does expose a
   `status_line` config and its bare item tokens (`spinner`, `model`, `project`,
   `token_usage`, `current_dir`, `rate_limit`, `approval_mode`, `context_window`)
   match the plan's documented predefined set, but the giant stripped Rust binary
   also contains generic `custom`/`command`/`text` strings that could NOT be
   confirmed as status-line item variants (they appear for countless unrelated
   reasons). Confirm against the codex 0.142.5 config reference/docs - NOT the
   binary - that `tui.status_line` has no custom-text or external-command item. Do
   not accept wrapper hacks, aliases, prompt text, or one-time startup banners as
   substitutes for a persistent in-harness status surface.

4. **SIMPLIFY sweep (standing).** The `claudeStatusMerge` / `claudeStatusLine`
   split stays justified: Test #1b exercises the merge behaviorally and the render
   test executes the statusline directly - do not inline unless that coverage is
   dropped. The `.statusLine.command==$claudesh` linkage assertion earns its keep
   (proves one-source-of-truth). Pass 2 deleted the ordering-anchor list (dead
   after the non-fatal fix). Hunt for any residual ceremony.

Do not re-open focus areas fully resolved unless the current plan contradicts
itself. RESOLVED (do not re-litigate):
- Pass 1: `osConfig` IS available in this home-manager-as-NixOS-module integration
  (home-manager passes `osConfig = <the NixOS config>` as a specialArg;
  `networking.hostName` = machine name in `sharedModules`), and `ai-agents.nix` is
  imported only by the dev-VM builder - one-source hostname and dev-only scoping
  are sound, no `mkDevVm` change needed.
- Pass 2, statusline extraction (2efe090): HELD UP. Built the real artifacts - the
  statusline path is absent from `agentVmStatus.data`, is realized+executable via
  the merge closure (Nix string context), prints `VM: allod-dev`, and
  `.statusLine.command==$claudesh` holds. The extraction fix is correct.
- Pass 2, Pi `--offline`+RPC schema (2efe090/ca4f9cb): CONFIRMED by a real run on
  allod-dev. `pi --mode rpc --offline -e <ext>` exits 0 and emits
  `{"type":"extension_ui_request",...,"method":"setStatus","statusKey":"vm-status","statusText":"VM: allod-dev"}`;
  the sabotage variant exits non-zero with a `ParseError` and no `setStatus`. The
  plan's Test 4/5 assertions match observed output.

Next pass: scoped diff review of commit ba7d476 (non-fatal merge + risk
recalibration + anchor removal). Per `dev-plans.md` rotation guidance, prefer a
different model (e.g. gpt-5 or claude-sonnet) for the verification pass. Fix
stability so far: 2efe090 (statusline extraction, claude-opus-4-8) held up under
pass-2 scrutiny; 7ab6771 (ordering, claude-opus-4-8) did NOT - refuted and
replaced in the next pass, so DAG-ordering fixes from any model warrant a
generated-artifact topoSort check, not prose.

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
