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
  to run bootstrap and verify. That late `forge_key` read is a `$SPECS`
  consumer the plan's environment-surface replacement never mentions.
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
  interpolates the lock path into its message: `"<flake_lock> pins
  secrets@<rev>. Update the secrets input in profiles, commit, push, then
  retry."` The new `(vm, material, secrets_checkout, secrets_rev)` signature
  receives no lock path — reconcile that with "message preserved verbatim".
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
  exercise real `nix eval git+file://...?rev=` semantics — the stub is forced,
  and real-eval confirmation lands on the human Agent Gate. The plan's
  acceptance loop runs only five suites; the other five (`nexus-host-key.sh`,
  `vm-ssh-host-key.sh`, `forge-ssh-key.sh`, `bootstrap-orchestration.sh`,
  `registry-resolver.sh`) also exercise code that sources the changed lib and
  will run in `nix flake check` regardless.
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
PRs (closing keyword on the second) — if the split is real, the plan must
assign residual risk per PR; it currently scores one R3 for the whole. Verify
the rollback plan's revert order (wiring PR before lib PR) matches the split it
describes.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have
problems. These are lenses, not checklists — follow the thread wherever it
leads.

The six standing lenses in `dev-plans.md` (internal consistency, operational
sequencing, risk calibration, acceptance-test coverage, rollback fidelity,
generated lifecycle behavior) apply as defaults on top of the plan-specific
areas below.

1. **The scope boundary: two parallel resolver families in one lib.** The plan
   adds `_at_pin` siblings for IP and username while leaving
   `resolve_target_ip` / `resolve_target_user` / `resolve_forge_connection`
   signature-stable for three out-of-scope rotation scripts. Two code paths
   answering "what is this VM's IP/user" invite drift, and the boundary must
   actually hold. Concretely: (a) `provision-vm-from-host` reads `.forge_key`
   from `$SPECS` after `nixos-anywhere` to decide bootstrap/verify — the plan
   deletes `SPECS` from the environment surface without saying where
   `forge_key` now comes from; pinned read, retained working-tree read, or
   oversight? (b) provision's children (`bootstrap-vm-from-host.sh`,
   `verify-vm-from-host`) re-derive IP and username from the registry and the
   working tree — one provisioning run can install to the pinned IP and then
   bootstrap against a drifted working-tree IP. Is that seam acknowledged?
   (c) Is there a real semantic reason the global family survives (rotation
   tools mutate live working-tree state; provisioning tools consume pinned
   state), and does the plan say so, or does it only argue blast radius? Ask
   directly: after this change, does any in-scope code path still read a
   working-tree fact the plan claims is pinned?

2. **`nix eval` at a local git rev is not `git show`.** `resolve_target_user_at_pin`
   runs `nix eval --raw "git+file://${secrets_checkout}?rev=${secrets_rev}#lib.vmUsernames.${vm}"`.
   Nix's git fetcher resolves revs through the refs it fetches, not the
   checkout's object database: after `ensure_git_commit_available` fetches, the
   pinned rev may exist only via a remote-tracking ref while the local branch
   is behind — `git show` reads it fine, but `git+file?rev=` can die with
   "cannot find Git revision" absent `?ref=`/`allRefs` depending on nix
   version. Dirty-tree semantics also differ from the current `path:` eval.
   And evaluating the secrets flake at any rev forces its own locked inputs
   (nixpkgs from GitHub, inventory from the forge) to be fetched or cached —
   the "local pinned read" has network dependencies and real weight. None of
   this is exercisable in fixtures: the sandbox forces a stub keyed on the
   `?rev=` string, so the only validation of the real incantation is the human
   gate. Does the exact incantation work when the rev is not reachable from
   the checked-out branch? What is the failure mode offline? Should the plan
   pin the flake-ref shape (`?ref=`, `allRefs`) or document the constraint —
   and is the heaviest, least-testable mechanism in the plan even warranted
   for a fact whose worst failure is availability, not a pin bypass (see the
   SIMPLIFY sweep)?

3. **Convention checkout locations versus the private deploy flake.** The
   issue's whole point is the operator's private deploy flake, yet
   `INVENTORY_CHECKOUT`/`SECRETS_CHECKOUT` default to the public
   `~/work/allod/{inventory,secrets}`. A private deploy flake pins private
   forks; those revs will not exist in the public checkouts, and
   `ensure_git_commit_available`'s single `git fetch` hits the *public* origin
   and dies. So the flagship private invocation needs `DEPLOY_FLAKE` plus two
   checkout overrides — three env vars where the Goal advertises zero — and
   the plan simultaneously deletes the registry indirection that exists to map
   repo aliases to real checkout paths on the host. Is the failure loud and
   correctly attributed ("git fetch failed" does not tell the operator the fix
   is pointing `SECRETS_CHECKOUT` at their private checkout)? Should checkout
   defaults be validated against the lock's pinned URL (cheap: compare
   `origin` to `locked.url` and die with a pointed message), resolved via the
   registry, or is convention-plus-env genuinely the simplest thing that
   works? Where does the operator learn the private invocation?

4. **`DEPLOY_FLAKE` default and the build-target swap.** Old: rebuild `cd`s
   into the registry-resolved `MACHINE_PROFILES` and builds `.#vm`; provision
   builds `${MACHINE_PROFILES}#vm`. New: `--flake "${DEPLOY_FLAKE}#${VM_NAME}"`
   with `DEPLOY_FLAKE` hardcoded to `$HOME/work/allod/profiles`. The flake-ref
   shape is equivalent, but the default changed from registry-resolved to
   convention — on a host whose `repositories.json` maps `profiles` elsewhere,
   the same command now silently builds a different flake than yesterday. Does
   the profiles flake define a `nixosConfigurations` entry for every name in
   `vm-specs.json` so `${DEPLOY_FLAKE}#<vm>` resolves for all of them? For
   each machine the operator can name, does the default produce the same build
   as the old path — and if not, is the difference loud?

5. **The pinned age-secret read.** `git -C "$SECRETS_CHECKOUT" show
   "${SECRETS_REV}:secrets/vm-host-keys/${VM_NAME}-ssh.age" | age --decrypt`.
   `git show` of a blob is byte-exact absent textconv/eol attributes — confirm
   nothing in the secrets repo's `.gitattributes` can munge `.age` blobs, and
   that the decrypted key still lands in a file via redirect (piping preserved
   per the memory warning; command substitution strips the trailing newline
   and bricks first boot). Sharp edges: with `pipefail`, a path absent at the
   pin fails on the git side — does the wiring preserve the "No SSH host key
   secret for <vm>" message class or dump a raw git error, and does the test
   matrix cover absent-at-pin (the current missing-ciphertext test deletes the
   working-tree file, which stops mattering)? Reading at the pin means a
   just-init'd machine whose key is committed but not yet locked fails loud —
   the plan calls that correct; check it against `vm-ssh-host-key init`'s
   printed runbook (commit secrets, bump *profiles* lock, provision), which
   under a private deploy flake bumps the wrong lock. Is the end-to-end
   new-machine workflow still coherent, and is ciphertext working-tree-vs-pin
   skew (pinned bytes win) actually asserted by a test?

6. **The dropped implicit `git pull`.** Rebuild pulls profiles; provision
   pulls profiles and secrets — today, before anything else. The plan deletes
   all three pulls as contradicting lock authority. But the workspace's host
   workflow is "agent edits, commits, pushes; human pulls and runs host
   commands" — the auto-pull was quietly doing the pull half for the two
   most-used host commands. After the change, a stale deploy-flake checkout
   silently builds an old config with old pins: self-consistent, but the
   operator who just merged a profiles PR from a VM gets yesterday's system
   with no warning. Is silent-stale acceptable under fail-loud (principle 11),
   or should the scripts warn when the deploy-flake checkout is behind its
   upstream? If deliberate staleness is the intended semantics (deploy exactly
   what is pinned, update deliberately), the plan should state the
   operator-visible consequence, not just the rationale.

7. **The refusal message now points at the wrong repo.** The plan preserves
   the stale-pin refusal "verbatim", including "Update the secrets input in
   profiles, commit, push, then retry" and the `flake.lock` context. Two
   problems. First, under the deploy-flake model the pin lives in
   `${DEPLOY_FLAKE}/flake.lock` — for a private deploy flake, "in profiles"
   sends the operator to a repo whose lock the script no longer reads; the
   real fix is `nix flake update secrets` in the deploy flake. A security
   refusal that misdirects remediation is a defect, not a nicety. Second,
   literally: `die_vm_host_key_material_not_pinned` interpolates
   `${flake_lock}` and re-derives the rev from `profiles_checkout`; the new
   `(vm, material, secrets_checkout, secrets_rev)` signature has no lock path
   to print, so "preserved verbatim" is impossible without passing the lock
   path through or rewording — and the existing tests grep for the profiles
   lock path in refusal output. What should the message say under the
   deploy-flake model, and which test pins the new wording?

8. **Does the test matrix pin the properties it claims?** Trace each claimed
   property to a concrete asserted test: (a) fail-closed on the derived path —
   stale/mismatched/absent-entry cases under the new signature must assert the
   refusal text *and* that no `nixos-rebuild`/`new-vm` ran (the harness's
   `assert_*_not_called` pattern), not merely a nonzero exit; (b) anti-drift —
   inventory working tree at a different commit than the pin still yields the
   pinned IP is real (git-backed), but the username "anti-drift" case is a
   stub keyed on the `?rev=` string and proves only that the script builds the
   reference, not that it evaluates (see Focus Area 2 — do not let a green
   stub masquerade as coverage); (c) the genuine no-env invocation — only
   `DEPLOY_FLAKE` set means the fixture must plant repos under the fixture
   `$HOME/work/allod/{inventory,secrets}`; a test that exports
   `INVENTORY_CHECKOUT`/`SECRETS_CHECKOUT` proves nothing about the zero-env
   Goal; (d) suite coverage — the plan's acceptance loop runs five of the ten
   test files, but `nix flake check` runs all ten and the changed lib is
   sourced by scripts covered in the other five; the acceptance command should
   be `nix flake check` or the full set, else a signature regression in an
   out-of-scope script's suite is discovered after "done"; (e)
   missing-`machine-host-keys.json`-at-pin stays an infra failure, never a
   "not pinned" refusal.

Run an explicit SIMPLIFY sweep every pass. Standing candidates to weigh
honestly rather than wave through: the parallel `_at_pin` username resolver —
the username is not the security boundary (a wrong user fails SSH login
against an already-pin-verified host), so evaluating the *working tree* of
`SECRETS_CHECKOUT` with the existing `resolve_target_user` would keep zero-env
and delete the plan's heaviest, least-testable mechanism; is rev-consistency
for a non-security fact worth a `git+file` eval that fixtures cannot exercise?
Also: whether `INVENTORY_CHECKOUT`/`SECRETS_CHECKOUT` need to be two knobs or
one convention root, and whether folding the inline existence checks into
`resolve_target_ip_at_pin` (good — deletes duplicated inline `jq` guards)
should also fold the null-IP check the same way in both scripts. Cutting the
username pin would be a real scope deletion; record the decision either way.

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
or prevent an unnecessary abstraction (the username `_at_pin` resolver is the
standing candidate). Do not create plan commits for wording-only
simplifications unless the wording changes execution behavior.

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
