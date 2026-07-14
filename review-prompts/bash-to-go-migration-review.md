# Bash-to-Go Migration Dev Plan Review Prompt

You are the most dangerous kind of engineer: the one who has written a 2,500-line
bash tool, hated every line of it, and then rewritten it in Go without breaking a
single caller. Stdlib-only Go CLIs, Forgejo REST internals, `buildGoModule` with
`vendorHash = null`, PATH-shim test harnesses, and the exact ways a "drop-in
rewrite" silently drifts its exit codes — this is your home turf. You can smell a
broken parity claim through three layers of Nix wrappers. You have been summoned
because this plan swaps the tools agents use every day, and it needs someone who
will not accept "bug-for-bug parity" on faith. Do not hold back.

## Your Task

Review the [Bash-to-Go Migration dev plan](../dev-plans/bash-to-go-migration.md)
for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag
anything that will block implementation, create unnecessary work, or leave a
landmine for future changes.

Read the actual codebase to ground the review in reality. Do not review the plan
in isolation. The tools being migrated live in `allod/tools`; their current
behavior — not the docs describing it — is the contract this plan freezes.

## Project Context

**Allod** is a self-sovereign NixOS VM stack for agentic coding, with its tooling
and configuration spread across flake repos on a self-hosted Forgejo instance.

Key repos in play:
- `allod/tools` — the four bash tools being migrated (`forge`, `allod` at the
  repo root; `flake/flake-update-cascade`, `flake/flake-status`), their contract
  docs (`docs/forge.md`, `docs/allod-patch.md`), and the bash test suites under
  `tests/`. This is the source of truth for parity.
- `allod/profiles` — Home Manager packaging for dev VMs; the plan switches each
  tool from `writeShellApplication` to `buildGoModule` here per phase.
- `allod/nexus` — host NixOS config and provisioning; same packaging switch for
  the host.
- `allod/memory` — workspace policy (`vm-tooling.md` toolchain line, `allod.md`
  tool notes).

Current state (verify against the tree before trusting it; line numbers and file
layout drift):
- All four tools are bash today. There is no Go in the repo yet — no `go.mod`,
  `cmd/`, or `internal/` — and no Go toolchain on the VMs. Phase 0 introduces all
  of that, which is why nothing downstream is testable on-VM until a human
  rebuilds.
- The bash test harness mocks by putting fake executables on `PATH`. `forge`'s
  suite mocks **`curl`** as a PATH shim (`tests/forge/testlib.sh`, ~line 256:
  `export PATH="$MOCK_BIN:$PATH"`); a Go `net/http` client never execs `curl`, so
  that apparatus does not transfer — Phase 1 is a test rewrite, not a reuse. The
  `allod`/cascade suites mock `git`/`forge` as PATH shims **and launch the tool
  under test via `bash "$ALLOD" …`** (`tests/allod-change.sh`, ~line 469); a Go
  binary is not `bash`-invocable, so the driver must change the invocation, not
  only the path, before "run unchanged" means anything.
- `allod` shells out to `forge` (`change submit`), and `flake-update-cascade`
  shells out to `forge pr find-by-head`. Composition across mixed bash/Go phases
  and partial rollbacks is load-bearing.
- Agents run inside dev VMs. Host-only actions and VM rebuilds are human-gated;
  the plan's on-VM acceptance tests depend on a human having rebuilt first. This
  is real and correctly gated — do not treat the human steps as a plan defect,
  but do check that every agent-runnable test can actually run without them.
- Wrapper mechanism (checked out here — inspect it directly):
  `profiles/hosts/dev/home-shared.nix` and `nexus/nix/home.nix` both wrap each
  tool with `pkgs.writeShellApplication`, reading sources from the `allod-tools`
  input, which `profiles/flake.nix` consumes as `flake = false`. `forge` is
  wrapped plainly with `runtimeInputs = [ jq curl ]` — both dropped by the Go
  rewrite. `allod`, `flake-status`, and `flake-update-cascade` are wrapped via a
  `workspaceTool` helper that **concatenates `lib/workspace.sh` into the script
  text** and pins `runtimeInputs` of `git` (plus `jq`, and `util-linux` for
  cascade's `flock`). Note `nix` and `forge` are **not** in cascade's
  `runtimeInputs`; the bash cascade finds them via the inherited login-shell PATH,
  because `writeShellApplication` *prepends* `runtimeInputs` rather than replacing
  PATH — so the plan's "on PATH exactly as runtimeInputs pins them today" phrasing
  papers over a real distinction.
- `lib/workspace.sh` is consumed by three of the four *migrated* tools, not only
  the staying ones: `allod` (`workspace_is_repo_root`,
  `workspace_repo_default_branch`), and `flake-update-cascade` + `flake-status`
  (`workspace_collect_repos`, `workspace_repo_default_branch`, `WORK_DIR`). Each
  `source`s it at runtime *and* gets it concatenated by `workspaceTool`; `forge`
  uses none of it. `buildGoModule` drops the concatenation, so the Go rewrites
  must reimplement that repo-discovery and default-branch logic (see Focus Areas).

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections
from `dev-plans.md`: Tracking Issue, Goal, Scope, Risk Assessment, Interface
Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. (Language Decision
and Sequencing are extra, not required — do not flag their presence; judge
whether they earn their space.) This is multi-PR work, so verify it assigns
residual risk per PR or milestone. Verify the closing-keyword story: earlier PRs
carry `Refs allod/tools#98` and only the final Phase 4 `allod/tools` PR carries
`Closes allod/tools#98`.

## Focus Areas

Concentrate your review on these areas where this plan is most likely to have
problems. These are lenses, not checklists — follow the thread wherever it leads.

The six standing lenses in `dev-plans.md` (internal consistency, operational
sequencing, risk calibration, acceptance-test coverage, rollback fidelity,
generated lifecycle behavior) apply as defaults on top of the plan-specific areas
below.

1. **Contract capture vs. contract trust.** The entire migration is "bug-for-bug
   parity" against contracts "frozen at current behavior," but the plan sources
   those contracts from `docs/forge.md`, `forge --help`, and `docs/allod-patch.md`.
   Docs drift from code. For each rule in Interface Contracts (exit codes, the
   bare-number `find-by-head` output, `--body-file -` trailing-newline handling,
   the `/tmp/allod-patch.<10 alnum>` remote pattern, manifest schema), ask: is
   there an *executable* capture of the current bash behavior that the Go version
   is diffed against, or is it trust-me-the-doc-is-right? Where a rule has no test
   pinning it, "frozen" is only as strong as the prose. Which rules are pinned,
   and which are load-bearing but unverified?

2. **`workspace.sh` is an unenumerated contract the migration inherits.** The plan
   files `lib/workspace.sh` under "bash — thin glue; lib still needed by *staying*
   tools," but three of the four *migrated* tools source it: `allod` uses
   `workspace_is_repo_root` and `workspace_repo_default_branch`;
   `flake-update-cascade` and `flake-status` use `workspace_collect_repos`,
   `workspace_repo_default_branch`, and `WORK_DIR`. The Go rewrites lose the
   `writeShellApplication` concatenation and must reimplement all of it. The sharp
   edge: `workspace_collect_repos`'s discovery order and filtering is load-bearing
   yet absent from Interface Contracts — it sets cascade's multi-repo mutation
   order and flake-status's row order. A Go reimplementation that walks `WORK_DIR`
   in a different order (readdir vs. sorted, different filtering) is exactly the
   "silent contract drift" the Risk Assessment fears, with nothing pinning it.
   Should the plan freeze repo-collection behavior and add a fixture for the order?

3. **Parity-harness reuse is overclaimed per tool.** The plan says the `allod` and
   `flake-update-cascade` suites "run unchanged against the Go binaries." The PATH-
   shim mocks (`git`, `forge`) *will* intercept a Go child's exec — that part is
   fine — but the driver launches the tool as `bash "$SCRIPT" …`, which a Go binary
   cannot be. So the parameterization must swap the *invocation shape*, not just an
   `ALLOD_UNDER_TEST` path, and any mock that is a shell function rather than a PATH
   executable will not intercept a Go child at all. Separately, `forge`'s suite
   mocks `curl`, which the Go binary never execs, so it is a rewrite, not a
   reuse. Verify the plan's per-tool language (`parameterize` vs. `rewrite`)
   matches what each harness actually requires. Getting this wrong turns a claimed
   parity gate into a green suite that never exercised the Go binary.

4. **`forge` token-safety static check vs. its own surface.** The acceptance test
   asserts `! grep -rn '"os/exec"' cmd/forge/ internal/forgeapi/` to prove the
   token path spawns no children. But `-R/--repo` inference from `origin` is part
   of `forge`'s frozen surface. If that inference execs `git remote get-url`, then
   `os/exec` appears legitimately — and whether the grep fails depends entirely on
   which package the inference lives in, which the plan never pins. Is the check
   coherent with the tool actually needing `git` for repo inference (read
   `.git/config` directly? confine exec to a package outside the grep scope?), or
   will it force an awkward layout or a false failure? A static check that fights
   the tool's real requirements is worse than no check.

5. **`forge` live read-only diff as the parity backstop.** Phase 1 carries the
   largest test-migration burden and leans on `diff <(forge pr list)
   <(result/bin/forge pr list)` plus `issue list` as its live gate. Two read-only
   commands against mutable, networked Forge state: is the output even stable
   enough to diff (ordering, timestamps, pagination, tty/color, count)? Does two
   commands' worth of surface constitute evidence for a 2,505-line client, or is
   it reassurance theater? What is the *real* backstop — the httptest fixtures —
   and do those fixtures derive from **recorded real Forgejo responses** or from
   hand-written guesses at the response shape? Fixtures invented from the client's
   own assumptions cannot catch the client's wrong assumptions.

6. **Mixed-phase and rollback composition of `allod → forge` and `cascade →
   forge`.** `allod change submit` and `flake-update-cascade` both shell out to
   `forge`. Walk composition through the *intermediate* states, not just the all-Go
   end state: forge-Go + allod-bash (Phase 1 merged, Phase 2 not); a partial
   rollback that pins forge back to bash while allod stays Go; the window where one
   is deployed and the other is only in-repo. In each, which `forge` is on PATH,
   and does the wrapper's `runtimeInputs` pin the intended one? Note it does *not*
   today: cascade's `runtimeInputs` are `jq git util-linux`, not `forge`, so the
   bash tool resolves `forge` from the inherited login-shell PATH. A Go wrapper that
   switches to a strict/exclusive PATH (`makeWrapper --set PATH`) would change which
   `forge` is found; one that keeps prepending would not — which does the plan
   intend? The plan calls the bash-source fallback (`~/work/allod/tools/forge`) an
   operator workaround — does
   that fallback's own `forge`/`jq`/`curl` dependency still resolve in every
   rollback state it is offered for?

7. **Cascade rollback and Phase 4 deletion guards — the scary-runtime and the
   irreversible-step.** `flake-update-cascade` is the highest-consequence runtime:
   multi-repo mutation with force-with-lease and rollback. "Preflight/rollback
   parity in fixtures" is only meaningful if the fixtures exercise *partial-failure*
   rollback (repo N fails after repo N-1 was already pushed), not just the happy
   path — confirm which the existing fixtures cover. Then Phase 4: deletion
   ordering is correctly reasoned, but the guard is `! grep -rn "tools/forge\b"
   ../profiles ../nexus` and `! test -e forge.bak`. Does that grep catch every
   reference form, given the input is consumed as `flake = false` and a consumer
   may name the tool through the input attr rather than the literal `tools/forge`?
   And where does `forge.bak` ever get created — if nothing in the plan produces
   it, that check guards a phantom and gives false confidence.

Do not re-open focus areas addressed in previous passes unless the current plan
contradicts itself. (This is the first pass; the areas above are the seed.)

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves.
  Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any
  interface — except the ones this plan deliberately freezes, where the whole
  point is that agents already script against them.
- **Do not overengineer.** If the plan introduces abstraction that is not needed
  yet, call it out. Three similar lines beat a premature helper; three
  `if err != nil` beat a clever wrapper — the plan's own "Simplicity" ethos.
  Run an explicit SIMPLIFY sweep every pass: actively hunt for scope, ceremony,
  premature Go structure (`internal/` packages that could be one file), or
  deferred-tool hedging to delete rather than treating a quiet pass as nothing to
  cut.
- **Solo project, one human.** No team coordination overhead. No release process.
  No migration guides for other consumers.
- **Security matters, ceremony does not.** The token-handling and
  privacy/security boundaries must be airtight. Everything else can be pragmatic.
- **Solve problems as they come.** If the plan front-loads work for hypothetical
  future needs (e.g. abstractions for tools it explicitly defers, like the
  rotation suite), flag it.
- **Think operationally.** Consider what happens when someone executes this plan
  cold, with incomplete context, or runs the phases out of order.
- **Calibrate residual risk.** The risk score is for human triage after
  validation passes. Challenge understated or overstated risk using blast radius,
  rollback fidelity, and validation evidence. R3 for the cutovers is plausible;
  test it against the mixed-phase composition and the parity evidence, not the
  command names.
- **Inspect generated lifecycle artifacts.** For the `buildGoModule` wrappers,
  `vendorHash = null` enforcement, PATH pinning via `runtimeInputs`, the dev-only
  `flake.nix` that must not change how `profiles` reads a `flake = false` input,
  and the Go-version-≤-nixpkgs-Go pin — do not stop at "it builds." Reason about
  the generated wrapper, the negative paths (a third-party import must *fail* the
  build), and what a nixpkgs Go bump does to the pin.

The person implementing this is technically sharp. They do not need hand-holding;
they need the sharp edges they missed.

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
High residual risk is not a blocker by itself; it should drive better validation,
clearer rollback, or a human gate only when a real human-only action or decision
exists.

Use `[SIMPLIFY]` for unnecessary scope, ceremony, or abstraction. Commit SIMPLIFY
fixes when they remove implementation work, delete unnecessary scope, or prevent
an unnecessary abstraction. Do not create plan commits for wording-only
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

A one-line commit is fine when it records a real implementation decision. Fold or
skip commits that only rephrase already-correct guidance.

## Output Format

Give your review as a numbered list of findings, each tagged as one of:
`[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, `[QUESTION]`. Start with blockers, end with
questions. Be blunt.

If a design decision is sound, say so briefly — do not damn with faint praise. If
something is right, name it and explain why so a later pass does not undo it (the
language decision and the split rule, the bash-source-stays-executable rollback,
and the deletion-ordering reasoning are candidates — pressure-test them, then
bless the ones that hold). QUESTIONs must be resolved in the plan, not left as
open items. If the answer is clear from the codebase, update the plan and commit.
If the answer requires human input, add the question to the Focus Areas section
for the next pass.

## After Each Pass

As a final commit, update this prompt's Focus Areas section:

1. Remove focus areas that were fully addressed.
2. Refine any focus areas that were partially addressed.
3. Add new focus areas discovered during the review.

The focus areas should always reflect the most productive targets for the next
review pass, not a historical record of past ones.

Include a plain-text findings summary in this commit's message:

- Count only findings new to this pass, by tag. Carried-over unresolved items are
  not findings; move them to Focus Areas rather than re-counting them, so the
  per-pass counts track a real severity trend instead of inflating it.
- Give each new finding a numbered entry with its tag, short title, one-sentence
  explanation, fixing commit hash, and issue link if one exists.
- Classify each finding's origin: an original-plan defect, or introduced by an
  earlier review pass (name the commit that introduced it). Origin is what makes
  the convergence heuristic below and any trend read possible.
- State what the SIMPLIFY sweep considered for deletion this pass, even when
  nothing was cut. Two or more consecutive passes with zero SIMPLIFY on a growing
  plan is a smell to call out, not a clean bill: pure accretion is what breeds
  internal contradictions.
- Put a final `Model: <exact model>` footer at the bottom (e.g.,
  `Model: claude-opus-4-8`, `Model: gpt-5.5`). Use the exact model identifier, not
  the agent framework or product name. "Codex", "Claude Code", etc. are agent
  software, not models. This is review-pass metadata, not authorship attribution.

When a pass commits a structural or design change (a blocker-level fix), the next
pass should be a scoped diff review of that change, not a full re-review, run by a
different model than the one that wrote the fix — structural fixes are where new
blockers enter, and the author model tends not to catch its own gaps. Passes are
launched manually one at a time, so you cannot pick or start the next model
yourself; make the handoff explicit instead. In the Focus Areas update, add a
`Next pass:` line that names the commit(s) to review, says whether it is a scoped
diff or a full pass, and recommends a model other than the fix's author
(preferably the most fix-stable one on record; see Agent Rotation in
`dev-plans.md`). Whoever starts the next pass reads that line and the previous
`Model:` footer and picks accordingly. Record in the same update how the fix held
up so its stability stays traceable.

Stop the plan-text review when either condition holds:

- Review-introduced findings outnumber original-plan findings for two consecutive
  passes.
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
