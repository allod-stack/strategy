# Nexus Deploy-Flake VM Facts Review Prompt

You read Nix evaluation the way a grandmaster reads a board: one glance and you
know which thunk forces which, which path coercion quietly copies a file into
the store, and exactly where somebody wrote `"${path}"` when they meant
`toString path` and shipped a second store object nobody ordered. You are the
reviewer other reviewers pretend they consulted — fluent in flake composition
down to which inputs a single `nix eval` actually fetches, in bash test
harnesses where a PATH shim silently decides what "proven" means, and in
supply-chain pinning where "the eval succeeded" and "the eval proved something"
are two different sentences. TOFU has never once gotten past you. This plan
re-sources the fail-closed host-key pin — the control that decides which
machine gets a root filesystem written onto it — from git-show plumbing to
flake outputs, across three repos, and deletes the old machinery on the way
out. Plumbing this security-shaped is exactly why you were summoned. Do not
hold back.

## Your Task

Review the [deploy-flake vm-facts dev plan](../dev-plans/nexus-deploy-flake-vm-facts.md)
for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag
anything that will block implementation, create unnecessary work, or leave a
landmine for future changes.

Read the actual codebase to ground the review in reality. Do not review the
plan in isolation. The behavior of `rebuild-vm-from-host`,
`provision-vm-from-host`, `rotation-common.sh`, the fixture harness, and the
three flakes as they exist today — not the prose describing them — is the
contract this plan rewires. The load-bearing property is the fail-closed
host-key pin re-sourced onto the `nix eval` facts path; a review that does not
trace that property through the real die-vs-proceed code, on both the old and
the planned flow, has not done its job.

## Project Context

**Allod** is a self-sovereign NixOS VM stack for agentic coding, with its
tooling and configuration spread across flake repos on a self-hosted Forgejo
instance.

Key repos in play:
- `allod/nexus` — host provisioning and rotation scripts plus their fixture
  harness. Owns PR3: `scripts/rebuild-vm-from-host`,
  `scripts/provision-vm-from-host`, `scripts/lib/rotation-common.sh`, and
  `tests/`. PR3 carries `Closes allod/nexus#6` and merges last.
- `allod/profiles` — the public deploy flake and the `DEPLOY_FLAKE` default;
  already the composition point of `inventory` + `secrets` (both direct root
  git inputs pinned by forge URL). Owns PR2: the `nix/vm-facts.nix` builder,
  the `vmFacts` output and `lib.mkVmFacts` export, the coherence/negative
  checks, and the `secrets` lock bump.
- `allod/secrets` — identity. Exposes `lib.vmUsernames` and `lib.identity`
  today; `machine-host-keys.json` is the host-key pin registry;
  `secrets/vm-host-keys/<vm>-ssh.age` holds the encrypted VM host keys. The
  public repo is a synthetic template; real values live in private forks. Owns
  PR1 (protected repo — goes through `allod change begin`).
- `allod/inventory` — machine facts. The flake exposes `machines.<vm>`
  (ip, forge_key, type, mac, platform, ...) and `lib.vmSpecsJson`. Explicitly
  no change in this plan; the builder consumes it as data.
- `allod/memory` — workspace policy: `dev-plans.md` (risk levels, standing
  lenses), `architecture.md` (principles 7, 8, and 11 do the heavy lifting
  here), `vm-provisioning.md`.

Current state (verify against the tree before trusting it; line numbers and
file layout drift):
- Post-#5 scripts: both resolve the pinned `inventory`/`secrets` revs from
  `${DEPLOY_FLAKE}/flake.lock` via `resolve_flake_input_locked_rev`, read the
  target IP with `git show` at the inventory pin, and read host-key material
  with `git show` at the secrets pin. `rebuild` reads the username from the
  secrets **working tree** (`nix eval path:${SECRETS_CHECKOUT}#lib.vmUsernames.<vm>`
  — a deliberate #5 decision this plan reverses). `provision` additionally
  reads `forge_key` at the inventory pin, decrypts the age blob at the secrets
  pin, guards `new-vm` with `assert_inventory_worktree_ip_matches_pin`
  (worktree IP vs pinned IP), and exports `INVENTORY="$INVENTORY_CHECKOUT"`.
- The four helpers slated for deletion (`resolve_flake_input_locked_rev`,
  `ensure_git_commit_available`, `machine_host_key_materials_at_secrets_pin`,
  `resolve_target_ip_at_pin`) are called only by the two in-scope scripts and
  unit-tested only in `tests/rotation-common.sh` — the plan's grep claim holds
  today; re-verify at review time.
- The asserts today take `(vm, materials, secrets_checkout, secrets_rev,
  flake_lock)`, and `machine_host_key_materials_at_secrets_pin` returns
  empty-and-success for a missing VM entry — the fail-closed die happens in
  the assert's match loop against empty pinned materials. The new model moves
  entry existence to eval time; check that no case falls between the two.
- `provision`'s children re-read the working tree: `new-vm` takes `INVENTORY`
  (default `~/work/allod/inventory`) for specs; `bootstrap-vm-from-host.sh`
  and `verify-vm-from-host` re-derive `INVENTORY`, the registry, and
  `IDENTITY_CONFIG` (via `resolve_checkout`) themselves and `nix eval path:`
  the secrets working tree. None of them reads `SECRETS_CHECKOUT`, so deleting
  it from `provision` cannot affect them.
- `secrets` flake today: `lib.vmUsernames` covers dev and privacy VMs plus the
  nexus hostname; the `credential-inventory` check re-parses
  `machine-host-keys.json` with its own `builtins.fromJSON` — the target of
  PR1's one-parse-one-owner switch. Both `machine-host-keys.json` and
  `secrets/vm-host-keys/` carry a `nexus` (hypervisor) entry, so the
  readDir-derived `lib.vmHostKeySecretFiles` and `lib.machineHostKeys` will
  contain a key `vmFacts` never consumes — attr-name coherence must be against
  the non-hypervisor machine set, not against those attrsets.
- `inventory`: `vmSpecsJson` filters `type != "hypervisor"` — the same filter
  `mkVmFacts` specifies — and the `vm-specs-json` check pins
  `scripts/vm-specs.json` to the Nix machine data. That existing check is what
  makes PR2's coherence comparison against the JSON file meaningful.
- Harness reality: nexus `nix flake check` runs **all ten** suites plus
  `bash -n` and `shellcheck -x` inside one sandboxed `runCommand` — no
  network, no real nix daemon, no `nix` on `PATH` beyond the suites' own
  stubs. The rebuild suite's `nix` stub keys on the literal string
  `lib.vmUsernames.alpha-dev`; the provision suite has **no** `nix` stub today
  because the script never calls nix — PR3 introduces its first. Fixture age
  decryption is real `age` against fixture-encrypted blobs. Refusal greps
  today pin `${FIXTURE_DEPLOY}/flake.lock` and `secrets@<rev>` strings — all
  re-pinned under the plan's reworded messages.
- Zero-env coverage: `run_rebuild`/`run_provision` unset `DEPLOY_FLAKE`,
  `INVENTORY_CHECKOUT`, and `SECRETS_CHECKOUT` and rely on fixture-`$HOME`
  conventional paths — planting fixture state under `$HOME/work/allod/...` is
  how default-path behavior is proven.
- Execution constraint: agents run in dev VMs where real `nix` **is**
  available outside the flake-check sandbox — the PR1/PR2 acceptance evals are
  real evals against the template flakes. Live rebuild/provision is Nexus-side
  and human-gated. Do not treat the human step as a defect; confirm every
  agent-runnable path is exercisable without it.

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections
from `dev-plans.md`: Tracking Issue, Goal, Scope, Risk Assessment, Interface
Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Agent Gates may
be omitted only if no actions require a human (they apply here: merge
sequencing, the protected-repo PR1 flow, and the human-only live run). The
work spans three PRs: verify residual risk is assigned per PR (the plan claims
R1/R2/R3), that PR1 and PR2 carry `Refs allod/nexus#6` and only PR3 carries
`Closes allod/nexus#6`, and that the rollback plan's revert order is the merge
order reversed, including the stated partial-land safety claims (PR1 alone and
PR1+PR2 change no script behavior).

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have
problems. These are lenses, not checklists — follow the thread wherever it
leads.

The six standing lenses in `dev-plans.md` (internal consistency, operational
sequencing, risk calibration, acceptance-test coverage, rollback fidelity,
generated lifecycle behavior) apply as defaults on top of the plan-specific
areas below.

Plan-text review has converged. Pass 2 and pass 3 both produced no BLOCKER and
no original-plan GAP, so the second stop condition is met. Do not run another
plan-text review unless the plan text changes; carry the remaining focus areas
to implementation review.

1. **Specified assertions vs implemented assertions (implementation-review
   handoff).** The plan names the load-bearing test mechanics: stub argv cases
   grep the full recorded command line (not a substring that passes on a wrong
   shape); negative checks force the field under test (an unforced lazy
   attrset `tryEval`s to success and proves nothing); every rebuild refusal
   path — probe and facts dies included — asserts
   `assert_nixos_rebuild_not_called`, and every provision refusal asserts
   `assert_new_vm_not_called`. Pass 2 confirmed the plan's *specification* of
   these mechanics is sound against nix 2.31; pass 3 re-read the current
   nexus/profiles/secrets/inventory tree and found no plan-text gap. The
   residual risk is purely that the *test bodies* match the spec once the PRs
   open. The R3-not-R4 case rests on these being real: when PR3 lands, read the
   test bodies against those clauses, not the green. Structurally unreachable
   from plan text - this is the implementation review's live edge.

2. **The human-gated residue (carry until the live run).** Real `nix eval`
   behavior against the private deploy flake — input fetching through the
   host's netrc/libgit2 path, eval latency on the host store, and whether the
   passed-through diagnostics are actually legible mid-incident — is
   structurally unreachable from the dev VM and the sandboxed harness. The
   plan's Agent Gate carries it: by-hand
   `nix eval <private-deploy>#vmFacts.<vm> --json` before the first live
   rebuild. Pass 2 and pass 3 confirmed it stays a named gate (Agent Gates
   section), not plan text - keep it that way.

Do not re-open focus areas closed in previous passes unless the current plan
contradicts itself. Pass 1 traced and closed: the die-case homing (every
today-death has exactly one home in the new flow, refusals precede mutation),
all mid-chain states (old scripts parse a PR2-bumped lock unchanged;
PR2-before-PR1 fails its own acceptance), the DHCP preflight operands,
`SECRETS_CHECKOUT` deletion invisible to bootstrap (registry-derived
`IDENTITY_CONFIG`, code-verified), coherence reflexivity with the hypervisor
asymmetry, and the three deliberate semantic changes — grounded in the tree
and in `nix eval` experiments on nix 2.31 (quoted attrpaths, `--apply`
laziness over broken siblings, tryEval's throw/assert-only scope, `toString`
vs interpolation vs `--json` store behavior).

Pass 2 traced and closed the four pass-1 contract fixes against the real tree
plus fresh nix 2.31 experiments (all sound; none re-opened):
- **d03f3ae** (stderr passthrough + probe-first): sound. The unknown-VM path
  runs no per-VM eval, so the clean message and passthrough genuinely coexist;
  the probe trichotomizes the failure space (output-absent → adoption message,
  VM-absent → clean unknown-VM, completeness → passed-through throw), which is
  why it is not ceremony to cut.
- **7ad8500** (throw-based guards + laziness boundary): sound and
  self-enforcing. Confirmed `builtins.attrNames` succeeds over a throwing
  sibling value (names probe forces only `.type`), forcing the specific attr
  dies, `x.y or D` does NOT catch a present-but-null `y` (so the null-ip guard
  must be an explicit throw — and the forced negative case makes CI fail on any
  naive `or (throw)` implementation), `or null` catches a missing attr
  (adoption wiring), and a bare missing-attr access is uncatchable by
  `tryEval`.
- **b5428fc** (refusal coverage): sound. Absent-from-registry now homes at the
  builder completeness throw; present-but-all-null homes at the assert's
  empty-materials refusal — two correct fail-closed homes. The unit-test
  "single home" wording describes the assert level (both arrive as empty
  `pinned_materials`) and is accurate there.
- **4474ad5** (exact invocation shapes): sound. Verified the quoted hyphenated
  attrpath `vmFacts."allod-dev"`, `--apply builtins.attrNames`, and the
  `jq -e '.ip and .username and .hostKeys.active and .hostKeySecretFile'` shape
  all parse and evaluate on nix 2.31. No zero-env collision: the fixture path
  is known to the test (`FIXTURE_DEPLOY`), so the exact-argv grep interpolates
  it.

Pass 3 did a full fresh read against the same current tree (not a scoped diff)
and found no BLOCKER, no GAP, no SIMPLIFY, and no QUESTION. It re-checked the
required dev-plan sections, PR linkage, current nexus die-vs-proceed behavior,
child-script checkout behavior, secrets/inventory/profiles flake boundaries,
Nix 2.31 CLI shapes, jq empty-output behavior, rollback order, and the SIMPLIFY
candidates. No plan-file commit was needed.

Stop condition: met. Pass 2 and pass 3 are consecutive clean passes with no
BLOCKER and no original-plan GAP. Plan-text review ends here. Next work is
implementation review of the PR bodies and test bodies, especially Focus Areas
1 and 2 above. Fix-stability: all four claude-fable-5 fixes are 2-pass-stable
(survived pass 2 and pass 3 unchanged).

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves.
  Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any
  interface — the env knobs are deleted, not deprecated, and the refusal
  wording changes by design; judge the new surface on its own merits.
- **Do not overengineer.** If the plan introduces abstraction that is not
  needed yet, call it out. Three similar lines beat a premature helper. Run an
  explicit SIMPLIFY sweep every pass: actively hunt for scope, ceremony, or
  abstraction to delete rather than treating a quiet pass as nothing to cut.
  Candidates worth weighing here: the two-eval probe ceremony, the separate
  `vmHostKeySecretFiles` attr versus deriving the path in the builder (secrets
  owning its own layout is the counter-argument — settle it, don't split it),
  the `lib.mkVmFacts` export no public consumer calls yet, and any matrix case
  whose code path became unreachable.
- **Solo project, one human.** No team coordination overhead. No release
  process. No migration guides for other consumers.
- **Security matters, ceremony does not.** The host-key pin must stay
  fail-closed on the facts path (principle 7: pinned, never TOFU). The plan
  concentrates fact authority and build authority in one deploy flake on
  purpose — that is principle 8 working as intended, and the price must not be
  a silently weakened preflight. Everything else can be pragmatic.
- **Solve problems as they come.** The children's migration and private-side
  adoption are explicitly deferred; flag any scope creep toward them, and any
  speculative output or knob added for a consumer that does not exist yet.
- **Think operationally.** Consider the operator running these cold: against a
  deploy flake whose data lacks the VM; on a cold store with no network; with
  a dirty or three-commits-behind deploy-flake checkout; mid-rotation with
  staged keys; right after `vm-ssh-host-key init` of a brand-new machine whose
  facts eval now decides everything.
- **Calibrate residual risk.** The plan claims R3 overall (R1/R2/R3 per PR)
  and argues why not R4. Pressure-test the load-bearing clause: the security
  property stays fixture-testable only if the refusal tests actually assert
  refusal-plus-no-mutation on the facts-fed path. A pin regression that fails
  open there is R4 territory. Read the new test assertions, not the green.
- **Inspect generated lifecycle artifacts.** This is wrapper-script territory:
  inspect the command lines the stubs record (`nixos-rebuild`/`nixos-anywhere`
  args, SSH options, the `--flake "${DEPLOY_FLAKE}#<vm>"` forms), the negative
  paths (missing output, unknown VM, malformed facts, absent/unreadable/
  garbage ciphertext), and the one seam fixtures cannot reach — the real
  `nix eval` CLI contract against a real flake. Source evaluation alone does
  not count.

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
(real-host `nix eval` behavior against the private deploy flake may be one —
the sandboxed harness structurally cannot answer it). If the answer is
inferable, resolve it as `[GAP]` or `[SIMPLIFY]`.

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
it. Strong candidates to bless (pressure-test first, then protect): the single
facts snapshot feeding IP, username, pin materials, and ciphertext path (kills
the #5 cross-read drift class at the root); pure-comparison asserts with
extraction hoisted into the script (the security check no longer knows locks
or checkouts exist); `toString` at construction instead of interpolated paths;
no checkout fallback — a missing output dies loud instead of resurrecting the
second read path; the builder living in `profiles`, the existing composition
point, instead of adding a third repo to the chain; and the retained DHCP
preflight guarding `new-vm`. QUESTIONs must be resolved in the plan, not left
as open items. If the answer is clear from the codebase, update the plan and
commit. If the answer requires human input (e.g. real-host behavior against
the private deploy flake), add the question to the Focus Areas section for the
next pass.

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
  `Model: claude-opus-4-6`, `Model: gpt-5.5`). Use the exact model identifier,
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
The old zero-BLOCKER/zero-GAP/zero-QUESTION rule could grind a token budget to
nothing without terminating, because each accretive fix tended to seed the
next finding.

## Before Final Response

- Plan fixes are committed, or the pass explicitly found no plan changes.
- This review prompt's Focus Areas are updated and committed.
- The final review-prompt commit message includes the findings summary and
  `Model:` footer.
- The repo is pushed to the remote.
- `git status` is clean.
