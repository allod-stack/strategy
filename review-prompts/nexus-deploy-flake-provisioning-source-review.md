# Nexus Deploy-Flake Provisioning Source Review Prompt

You read flake.locks the way a sommelier reads wine labels: one glance and you
know the vintage, the provenance, and that somebody swapped the grapes. You are
the NixOS provisioning expert other experts quietly consult — fluent in fetcher
semantics down to which refs a `git+file` rev is actually reachable from, in
bash pipelines where one stripped trailing newline strands a machine before its
first boot, and in supply-chain pinning where "it resolved from somewhere" is
the opening line of an incident report. You have refused more TOFU than a vegan
steakhouse. This plan rewires which repository, at which revision, decides where
`nixos-anywhere` lands and which SSH host key it trusts — the kind of change
that looks like plumbing and is actually the security boundary. That is why you
were summoned. Do not hold back.

## Your Task

Review the [deploy-flake provisioning source dev plan](../dev-plans/nexus-deploy-flake-provisioning-source.md)
for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag
anything that will block implementation, create unnecessary work, or leave a
landmine for future changes.

Read the actual codebase to ground the review in reality. Do not review the plan
in isolation. The behavior of `rebuild-vm-from-host`, `provision-vm-from-host`,
`rotation-common.sh`, and the fixture harness as they exist today — not the
prose describing them — is the contract this plan rewires. The load-bearing
property is the fail-closed host-key pin on the newly derived path; a review
that does not trace that property through the real die-vs-proceed code has not
done its job.

## Project Context

**Allod** is a self-sovereign NixOS VM stack for agentic coding, with its
tooling and configuration spread across flake repos on a self-hosted Forgejo
instance.

Key repos in play:
- `allod/nexus` — host provisioning and rotation scripts plus their fixture
  harness. Owns everything in scope: `scripts/rebuild-vm-from-host`,
  `scripts/provision-vm-from-host`, `scripts/lib/rotation-common.sh`, and
  `tests/`. The implementation PR carries `Closes allod/nexus#5`.
- `allod/profiles` — the public deploy flake and the `DEPLOY_FLAKE` default.
  Its `flake.lock` pins `inventory` and `secrets` as direct root git inputs by
  **remote forge URL** (`https://forge.anarch.diy/...`), which is why checkout
  locations cannot come from the lock.
- `allod/inventory` — machine facts; `scripts/vm-specs.json` carries per-VM
  `ip`, `forge_key`, `repos`, `self_rebuild`.
- `allod/secrets` — identity. `flake.nix` exports `lib.vmUsernames` and
  `lib.identity`; `machine-host-keys.json` is the host-key pin registry;
  `secrets/vm-host-keys/<vm>-ssh.age` holds the encrypted VM host keys. The
  public repo is a synthetic template; real values live in private forks.
- `allod/memory` — workspace policy: `dev-plans.md` (risk levels, standing
  lenses), `architecture.md` (principles 3, 7, 8, 11 are the ones this plan
  leans on), `vm-provisioning.md` (the never-command-substitute-the-host-key
  warning).

Current state (verify against the tree before trusting it; line numbers and
file layout drift):
- Both in-scope scripts resolve `MACHINE_PROFILES` / `IDENTITY_CONFIG` today
  through `lib/resolve-repos.sh` registry indirection over
  `inventory/scripts/repositories.json` (`resolve_checkout` with an
  `allod/<name>` fallback). The plan drops the registry for these two scripts
  and replaces it with hardcoded convention defaults — the default *resolution
  path* changes, not just the variable names.
- `rebuild-vm-from-host` today: `cd "$MACHINE_PROFILES"; git pull`, an inline
  unknown-VM `jq` check against working-tree `$SPECS`, `resolve_target_ip`
  (global `$SPECS`), `resolve_target_user` (`nix eval --raw
  "path:${IDENTITY_CONFIG}#lib.vmUsernames.<vm>"` — working tree, untracked
  files visible to `path:`), keyscan, then
  `assert_any_vm_host_key_material_pinned <vm> <materials> "$MACHINE_PROFILES"
  "$IDENTITY_CONFIG"`, then `nixos-rebuild switch --flake ".#${VM_NAME}"`.
- `provision-vm-from-host` today: two `git pull`s, working-tree IP from
  `$SPECS`, decrypts the working-tree age file with `age --decrypt -i` to a
  file, derives the public key, asserts the pin, runs `nixos-anywhere --flake
  "${MACHINE_PROFILES}#${VM_NAME}"` — and then, near the end, reads
  `FORGE_KEY=$(jq -r ".\"${VM_NAME}\".forge_key" "$SPECS")` to decide whether
  to run bootstrap and verify. Pass 1 rewired that late read to the pinned
  inventory rev (resolved up front next to the IP) — verify the rewiring, not
  its absence.
- `provision-vm-from-host` invokes `bootstrap-vm-from-host.sh` and
  `verify-vm-from-host` as child processes. Both re-derive `SPECS` and
  `IDENTITY_CONFIG` from their own registry defaults and read the **working
  tree** (`path:` evals, plain `jq` on `$SPECS`). They are out of scope by
  design, but they run inside an in-scope invocation.
- The pin machinery today: `resolve_flake_input_locked_rev` is already generic
  and fail-loud on lock shape (missing input, follows-array, non-git locked
  type, missing rev all die with distinct messages).
  `machine_host_key_materials_at_secrets_pin` derives the secrets rev from
  `profiles/flake.lock` itself; `ensure_git_commit_available` fetches once from
  the checkout's origin on a miss. `die_vm_host_key_material_not_pinned`
  interpolates the lock path into its message. Pass 1 replaced "preserved
  verbatim" with a reworded refusal and threaded `flake_lock` through the
  asserts as message-only context (Focus Area 2 below).
- The secrets flake has its own locked inputs (`nixpkgs` from GitHub,
  `inventory` from the forge URL). Evaluating it — at any rev — forces those
  inputs to be fetched or already cached. A "pinned local read" via `nix eval`
  is not a purely local, offline operation the way `git show` is.
- Out-of-scope scripts `nexus-host-key`, `vm-ssh-host-key`, `forge-ssh-key`
  share `resolve_target_ip` / `resolve_target_user` / `resolve_forge_connection`
  and print rotation runbooks whose step 2 is "in profiles: `nix flake update
  secrets`, commit, push" — guidance that under a private deploy flake bumps a
  lock the in-scope scripts no longer read.
- Harness reality: fixtures stub `git`/`nix`/`ssh-keyscan`/`nixos-rebuild`/
  `nixos-anywhere`/etc. as PATH shims. The `nix` stub in
  `tests/rebuild-vm-from-host.sh` is keyed on the literal string
  `lib.vmUsernames.alpha-dev`; the stale-pin tests grep for
  `"${FIXTURE_PROFILES}/flake.lock"` in refusal output; the provision fixture
  already models head/old/missing-entry pin states; `run_rebuild`/`run_provision`
  currently export `INVENTORY`/`MACHINE_PROFILES`/`IDENTITY_CONFIG` explicitly.
  A genuine no-env test must instead plant fixture repos under the fixture
  `$HOME/work/allod/{inventory,secrets}`.
- `nix flake check` in nexus runs **all ten** test suites inside the build
  sandbox (no network, no real nix daemon), so the harness structurally cannot
  exercise real remote-fetching `nix eval` semantics. The plan's acceptance
  gate is now `nix flake check` itself (pass 1); the five suites outside the
  focused loop (`nexus-host-key.sh`, `vm-ssh-host-key.sh`, `forge-ssh-key.sh`,
  `bootstrap-orchestration.sh`, `registry-resolver.sh`) exercise code that
  sources the changed lib and run in it regardless.
- `rotate-token` sources both `resolve-repos.sh` and `rotation-common.sh` and
  keeps registry usage; its suite staying green is a real signature-stability
  probe for the shared helpers, not ceremony.
- Execution constraint: agents run inside dev VMs; live rebuild/provision is
  Nexus-only and human-gated. The plan gates it correctly — do not treat the
  human step as a defect, but confirm every agent-runnable path is exercisable
  in fixtures without it.

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections
from `dev-plans.md`: Tracking Issue, Goal, Scope, Risk Assessment, Interface
Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Agent Gates may be
omitted only if no actions require a human (they apply here). The plan expects
a single PR carrying `Closes allod/nexus#5` with a contemplated split into two
PRs (closing keyword on the second). Pass 1 made the split additive-first
(PR1 adds helpers only, existing signatures untouched; PR2 carries the
signature changes and wiring) and assigned per-PR risk (PR1 R1, PR2 R3) —
verify the additive-first claim survives contact with the actual contract
table, and that the rollback plan's revert order (wiring PR before lib PR)
still matches.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have
problems. These are lenses, not checklists — follow the thread wherever it
leads.

The six standing lenses in `dev-plans.md` (internal consistency, operational
sequencing, risk calibration, acceptance-test coverage, rollback fidelity,
generated lifecycle behavior) apply as defaults on top of the plan-specific
areas below.

Pass 3 is a scoped diff review of the pass-2 plan commits
`7b3ed2a..01766be` (two commits on `dev-plans/nexus-deploy-flake-provisioning-source.md`:
a refusal-wording correctness fix and a recorded design decision), not a full
re-review. Pass 2 found no blockers and no original-plan defects — every
pass-1 structural fix (`c36b9d2..9213eda`) held up against the real scripts
except one wording collision, now fixed. The plan text has substantively
converged; the remaining targets are a confirming diff review plus two seams
better resolved at implementation than in more plan prose.

1. **The pass-2 refusal-wording fix (7b3ed2a).** Pass 2 corrected the
   Interface Contracts claim that provision keeps a `pin_mode=old` "stale-pin
   case": because provision reads and decrypts the age secret at the pinned
   rev *before* the host-key assert, a `pin_mode=old` provision fixture trips
   the "No SSH host key secret" age-absent guard first, so provision's
   reworded-refusal coverage rides the head-pin derived-key mismatch instead.
   Confirm the edit is internally consistent with the provision test matrix
   (which already lists only the mismatch case) and that no other plan text
   still implies a `pin_mode=old` provision refusal. Do not reintroduce a
   `pin_mode=old` provision case.

2. **Provision age-at-pin fixture precondition (hand to implementation
   review).** The whole age-at-pin redesign presumes `setup_fixture` commits
   the age blob into the secrets repo at the pinned rev — but today's
   `setup_fixture` writes `alpha-dev-ssh.age` to the working tree *after* all
   commits, uncommitted, because the current script reads it from the working
   tree. The plan's matrix implies the commit (absent-at-pin "working-tree
   copy present"; ciphertext-skew "the pin holds the good ciphertext"), so the
   plan need not carry fixture mechanics. Verify at implementation that the
   success/skew/garbage cases actually commit the blob at the pin; flag only if
   the harness cannot express it.

3. **The forge_key decide/execute seam (hand to implementation review).**
   Provision's bootstrap gate now reads `forge_key` at the pinned inventory
   rev, but the real `bootstrap-vm-from-host.sh` child re-reads `forge_key`
   from the working tree and refuses on a null. If the pin carries a non-null
   forge_key while the working tree nulls it (IP unchanged, so the DHCP
   preflight does not catch it), provision decides "bootstrap" but the child
   exits 1 after `nixos-anywhere` has installed — a loud, recoverable failure,
   not a strand. The plan explicitly scopes the child's working-tree reads
   (forge key state, username, known_hosts) to the follow-up migration. Judge
   whether the plan's "one provisioning run coherent" framing should name this
   decide/execute gap, or whether the follow-up scoping already covers it.

Run an explicit SIMPLIFY sweep every pass. Pass 1 cut the username `_at_pin`
resolver; pass 2 recorded the two-checkout-knobs decision (kept, not
collapsed: `inventory` and `secrets` are independently forkable and the vars
mirror the lock's two root inputs). No standing SIMPLIFY candidate remains on
the plan text — the DHCP preflight is one comparison, the fetch-miss
enrichment is two message lines, the pinned `forge_key` read is one
git-show-pipe, and the inline username eval avoids resurrecting the deleted
`IDENTITY_CONFIG` global. If a pass finds nothing to cut, say so; do not
manufacture a deletion. (Two-plus consecutive zero-SIMPLIFY passes on a growing
plan would be a smell — but pass 1 cut and pass 2 recorded, so the plan is not
in pure accretion.)

Fix stability: pass 1 (claude-fable-5) commits `c36b9d2..9213eda` held up well
under pass-2 scrutiny — nine of ten fixes were clean; only the refusal-wording
sentence (652f4fa) collided with the age-absent guard (bfe07f7) and needed the
7b3ed2a correction. Pass 2 (claude-opus-4-8) commits `7b3ed2a..01766be` have
not yet been reviewed by a later pass; record how they hold up.

Next pass: scoped diff review of `7b3ed2a..01766be`, not a full re-review, on a
model other than the pass-2 author (claude-opus-4-8). claude-fable-5 is the
most fix-stable on record and would be reviewing a different model's edit to
its own 652f4fa area — a good check; any non-Opus reviewer is fine. If that
pass produces no BLOCKER and no original-plan GAP, the two-consecutive-clean
stop condition is met (pass 2 already qualifies) — declare the plan text
converged and hand focus areas 2 and 3 to implementation review.

Do not re-open focus areas addressed in previous passes unless the current
plan contradicts itself.

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves.
  Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any
  interface — including the two scripts' env-var surface and the exact refusal
  wording, if better wording serves the deploy-flake model (Focus Area 7).
- **Do not overengineer.** If the plan introduces abstraction that is not
  needed yet, call it out. Three similar lines beat a premature helper; a
  working-tree read beats a pinned read plus fetch machinery when the fact is
  not security-load-bearing. Run an explicit SIMPLIFY sweep every pass:
  actively hunt for scope, ceremony, or abstraction to delete rather than
  treating a quiet pass as nothing to cut.
- **Solo project, one human.** No team coordination overhead. No release
  process. No migration guides for other consumers.
- **Security matters, ceremony does not.** The host-key pin must stay
  fail-closed on the derived path (principle 7: pinned, never TOFU), and the
  single-source-of-truth convergence (principle 8) is the win being purchased
  — make sure the price is not a silently weakened preflight. Everything else
  can be pragmatic.
- **Solve problems as they come.** If the plan front-loads work for
  hypothetical future needs (extra env knobs, speculative flake outputs, the
  rejected read-only-outputs alternative sneaking back in), flag it.
- **Think operationally.** Consider the operator running these cold: against a
  private deploy flake with default checkouts; mid-rotation with staged keys;
  right after `vm-ssh-host-key init` of a brand-new machine; with a
  deploy-flake checkout three commits behind; offline.
- **Calibrate residual risk.** The plan claims R3: fixtures cover the pin
  path, rollback is a straight revert, the one irreversible act is
  human-gated. Pressure-test the load-bearing clause — a pin-path regression
  that fails open on the derived path would be a security-boundary violation
  (R4 territory); the claim rests on the regression tests in Focus Area 8
  actually asserting refusal-plus-no-build. Read the new test assertions, not
  the suite exit code.
- **Inspect generated lifecycle artifacts.** This is wrapper-script territory:
  inspect the command lines the stubs record (`nixos-rebuild` /
  `nixos-anywhere` args, SSH options), the negative paths (absent entry at
  pin, absent age file at pin, missing lock input, fetch miss), and the one
  seam fixtures cannot reach — the real `nix eval git+file` call (Focus Area
  2). Source evaluation alone does not count.

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
test blind spots, stale docs, or implementation ambiguity, but where a
competent agent with workspace memory could still proceed safely.

Use `[GAP]` when the Risk Assessment is missing, materially understated,
materially overstated, or unsupported by the acceptance tests and rollback
plan. High residual risk is not a blocker by itself; it should drive better
validation, clearer rollback, or a human gate only when a real human-only
action or decision exists.

Use `[SIMPLIFY]` for unnecessary scope, ceremony, or abstraction. Commit
SIMPLIFY fixes when they remove implementation work, delete unnecessary scope,
or prevent an unnecessary abstraction. Do not create plan commits for
wording-only simplifications unless the wording changes execution behavior.

Use `[QUESTION]` only when the plan cannot be corrected from repo context
(host-side `nix` behavior on the real Nexus box may be one — the fixture
harness structurally cannot answer it). If the answer is inferable, resolve it
as `[GAP]` or `[SIMPLIFY]`.

Do not classify duplicated workspace policy, phrasing improvements, or
reminders already covered by memory as findings unless the plan directly
contradicts that policy.

## Deliverable

The deliverable is not a report. Every review pass ends with:

1. Plan-file commits for findings that require plan changes, or an explicit
   no-findings result.
2. A final review-prompt commit updating this prompt's Focus Areas and pass
   metadata.
3. A push to the remote.

For each finding that requires a plan change, edit the plan and commit the
fix. Group changes into logical, self-contained commits.

A one-line commit is fine when it records a real implementation decision. Fold
or skip commits that only rephrase already-correct guidance.

## Output Format

Give your review as a numbered list of findings, each tagged as one of:
`[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, `[QUESTION]`. Start with blockers, end
with questions. Be blunt.

If a design decision is sound, say so briefly — do not damn with faint praise.
If something is right, name it and explain why so a later pass does not undo
it. Strong candidates to bless (pressure-test first, then protect): reading
content at the pin and never the working tree; resolving the rev once in the
script and passing it to the assert (decoupling the security check from lock
parsing); piping the age blob instead of command substitution; reusing
`ensure_git_commit_available` for the inventory checkout; and treating the
deploy-flake lock as authoritative instead of auto-advancing it. QUESTIONs
must be resolved in the plan, not left as open items. If the answer is clear
from the codebase, update the plan and commit. If the answer requires human
input (e.g. real-host `nix eval` behavior), add the question to the Focus
Areas section for the next pass.

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
- Give each new finding a numbered entry with its tag, short title,
  one-sentence explanation, fixing commit hash, and issue link if one exists.
- Classify each finding's origin: an original-plan defect, or introduced by an
  earlier review pass (name the commit that introduced it). Origin is what
  makes the convergence heuristic below and any trend read possible.
- State what the SIMPLIFY sweep considered for deletion this pass, even when
  nothing was cut. Two or more consecutive passes with zero SIMPLIFY on a
  growing plan is a smell to call out, not a clean bill: pure accretion is
  what breeds internal contradictions.
- Put a final `Model: <exact model>` footer at the bottom (e.g.,
  `Model: claude-opus-4-8`, `Model: gpt-5.5`). Use the exact model identifier,
  not the agent framework or product name. "Codex", "Claude Code", etc. are
  agent software, not models. This is review-pass metadata, not authorship
  attribution.

When a pass commits a structural or design change (a blocker-level fix), the
next pass should be a scoped diff review of that change, not a full re-review,
run by a different model than the one that wrote the fix — structural fixes
are where new blockers enter, and the author model tends not to catch its own
gaps. Passes are launched manually one at a time, so you cannot pick or start
the next model yourself; make the handoff explicit instead. In the Focus Areas
update, add a `Next pass:` line that names the commit(s) to review, says
whether it is a scoped diff or a full pass, and recommends a model other than
the fix's author (preferably the most fix-stable one on record; see Agent
Rotation in `dev-plans.md`). Whoever starts the next pass reads that line and
the previous `Model:` footer and picks accordingly. Record in the same update
how the fix held up so its stability stays traceable.

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
