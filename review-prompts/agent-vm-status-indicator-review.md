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
- Codex 0.142.5 is explicitly out of scope because pass 3 confirmed from the
  installed 0.142.5 source that `tui.status_line` parses only built-in
  `StatusLineItem` identifiers and warns/ignores unknown strings. Re-check only
  if the plan or installed source changes.
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
is calibrated to generated Home Manager activation plus mutable operator state
(`~/.claude/settings.json` and `~/.pi/agent/extensions/vm-status.ts`), not just
to the small amount of Nix code.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have
problems. These are lenses, not checklists - follow the thread wherever it
leads.

The six standing lenses in `dev-plans.md` (internal consistency, operational
sequencing, risk calibration, acceptance-test coverage, rollback fidelity,
generated lifecycle behavior) apply as defaults on top of the plan-specific
areas below.

Pass 3 (gpt-5.5) committed three fixes: `07a1b7c` protects the Pi extension
target from clobber (new `piStatusInstall` helper + collision fixtures),
`3aa8011` settles the Codex exclusion from the installed 0.142.5 source, and
`6cea6b0` replaces `|| true` with `|| warnEcho` plus corrected
operator-visibility wording. Pass 4 (claude-opus-4-8) adversarially verified all
three against generated artifacts, the realized codex source, the installed
home-manager generation, and a faithful `topoSort` replay; all three HELD UP and
pass 4 found no new plan issues (details under RESOLVED). Two light focus areas
remain.

1. **SIMPLIFY sweep (standing).** The `claudeStatusMerge` / `claudeStatusLine`
   split stays justified: Test #1b exercises the merge behaviorally and the render
   test executes the statusline directly - do not inline unless that coverage is
   dropped. The `.statusLine.command==$claudesh` linkage assertion earns its keep
   (proves one-source-of-truth). The `piStatusInstall` helper was weighed for
   over-build this pass and kept: its collision fixtures are load-bearing (they
   fail against the old inline `ln -sfn` on the regular-file, directory-footgun,
   and custom-symlink cases - principle 11), a bare `ln -sfnT` would still silently
   clobber an operator-authored regular file or foreign symlink at the target
   (violating the never-clobber charter), and factoring it into a store script is
   exactly what makes Test #1a runnable - the same justification as
   `claudeStatusMerge`. Do not drop or inline any of the three helpers unless its
   behavioral coverage is dropped first. The current artifact set is minimal; keep
   hunting for residual ceremony anyway.

2. **Claude behavior with a missing/GC'd command path (carry, live-TUI,
   human-gated).** The rollback story claims a `statusLine.command` pointing at a
   garbage-collected store path makes Claude silently blank the row. Verify that
   against Claude Code 2.1.204 - both `claude -p` and interactive - rather than
   erroring or refusing to start. This is the one rollback claim with no
   agent-runnable test; it stays a human-gated live-TUI carry.

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
- Pass 3, Pi installer clobber-protection (07a1b7c): VERIFIED pass 4. Materialized
  the helper and ran Test #1a: it creates the managed symlink, replaces an old
  (even dangling) `/nix/store/*-pi-vm-status.ts` symlink, and refuses a regular
  file / directory / foreign symlink byte-unchanged. `ln -sfnT` empirically
  refuses to overwrite a real directory (`cannot overwrite directory`), so it
  cannot write inside one; the fixtures fail against the old inline `ln -sfn`
  (principle 11 satisfied). A synthetic writeText/closure eval confirmed the
  pass-1 subtlety did NOT reappear: `.data` names only the two helpers, yet the
  full closure realizes all four artifacts and `piext`/`claudesh` extracted from
  the helper files exist on disk. Correct and proportionate (see SIMPLIFY #1).
- Pass 3, Codex exclusion (3aa8011): VERIFIED pass 4 against the realized codex
  0.142.5 source (from `codex-0.142.5.drv`). All 26 `StatusLineItem` variants in
  `bottom_pane/status_line_setup.rs` are built-in and `EnumString`-parsed - no
  `Custom(String)` / `Text` / `Command` / `Exec` / `Script` variant exists;
  `tui_status_line: Option<Vec<String>>` (`config/mod.rs`) is parsed by
  `parse_items_with_invalids`, and unknown ids are collected and warned once
  (`Ignored invalid status line ...`) in `chatwidget/status_surfaces.rs`. The
  plan's item list matches. No custom-text/external-command hook: exclusion holds.
- Pass 3, warnEcho non-fatal visibility (6cea6b0): VERIFIED pass 4. `warnEcho` is
  defined (a trivial `echo`) in the sourced `home-manager.sh`, and that source
  runs at the top of `activate` before any DAG entry, so the `agentVmStatus`
  snippet can call it. With real external helper scripts, `${helper} || warnEcho`
  under the shared `set -eu -o pipefail` swallows a refusal (exit 1 stays in the
  child), warns, and continues to later entries (parent exit 0). The check's
  `grep '...\|\| *warnEcho'` asserts non-fatal+warn and fails if regressed to
  fatal or to `|| true`. The corrected wording (a successful `nixos-rebuild
  switch` is not assumed to stream unit stdout; journal/live-TUI is the real
  check) is accurate.
- Non-fatal ordering rationale: RE-VERIFIED pass 4 against the CURRENT allod-dev
  DAG (now including `syncPrBranchProtection` from `agent-hooks.nix`). Replaying
  `lib.hm.dag.topoSort` faithfully, a bare `entryAfter ["writeBoundary"]`
  `agentVmStatus` sorts to position 3 (front of the post-`writeBoundary` group);
  both `installPackages` and `syncPrBranchProtection` sort AFTER it, so a fatal
  refusal would skip a package install and the branch-protection sync. Non-fatal
  containment is required, not ordering - the plan's claim is exact.

Next pass: pass 4 made no plan-file changes, so there is no new fix to scope-diff.
Pass 5 is a fresh full convergence-confirmation pass - prefer a model not yet in
this rotation (claude-sonnet; gpt-5.5 acceptable), not claude-opus-4-8. If pass 5
is also clean (no BLOCKER and no original-plan GAP), stop condition 2 is met (two
consecutive clean passes) and the plan text has converged - hand the remaining
SIMPLIFY and live-TUI carry to implementation review. Fix stability: gpt-5.5's
`07a1b7c`, `3aa8011`, and `6cea6b0` ALL held up under pass-4 adversarial
verification (3/3) - gpt-5.5 has the strongest fix-stability record on this plan.
claude-opus-4-8: `2efe090` held (2+ passes); `7ab6771` (ordering) was refuted and
replaced by pass 2; `ba7d476` non-fatal containment held but its `|| true`
visibility wording needed pass-3 repair. Pass 4 (claude-opus-4-8) introduced no
fixes.

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves.
  Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any
  interface, but do not silently clobber operator-authored settings or broaden
  agent authority.
- **Do not overengineer.** If the plan introduces abstraction that is not needed
  yet, call it out. A few explicit generated artifacts beat a generic harness
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
