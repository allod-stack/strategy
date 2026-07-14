# Nexus External Backup SSH Rotation Gates Review Prompt

You are the engineer other engineers page at 3 a.m. when a key rotation has
half-happened and nobody can SSH anywhere. Bash state machines that must be
correct on the unhappy path, SSH client-auth semantics down to
`IdentitiesOnly`/`BatchMode`/`StrictHostKeyChecking`, age recipient graphs and
trust roots, NixOS/flake identity data that is public template on one side and
private fork on the other, and the exact difference between "the network is down"
and "the host no longer trusts me" — this is the ground you own. You can tell a
fail-open gate from a fail-closed one by reading the die-vs-continue disposition,
not the comment above it, and you know that a backup channel that auto-accepts an
unknown host key is not a backup channel, it is a MITM waiting for a reason. You
have been summoned because this plan wires new gates into the one script whose job
is to *not* strand the operator's only path to their offsite backups. Do not hold
back.

## Your Task

Review the [Nexus External Backup SSH Rotation Gates dev plan](../dev-plans/nexus-external-backup-ssh-rotation-gates.md)
for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag
anything that will block implementation, create unnecessary work, or leave a
landmine for future changes.

Read the actual codebase to ground the review in reality. Do not review the plan
in isolation. The behavior of `nexus-host-key`, `rotation-common.sh`, and the
fixture harness as they exist today — not the prose describing them — is the
contract this plan extends. The whole point of the plan is a safety property; a
review that does not trace that property through the real die-vs-continue paths
has not done its job.

## Project Context

**Allod** is a self-sovereign NixOS VM stack for agentic coding, with its tooling
and configuration spread across flake repos on a self-hosted Forgejo instance.

Key repos in play:
- `allod/nexus` — host NixOS config and provisioning. Owns
  `scripts/nexus-host-key` (the stage/activate/retire rotation state machine),
  `scripts/lib/rotation-common.sh` (shared resolvers: `resolve_target_ip`,
  `resolve_target_user`, `resolve_forge_connection`), `tests/nexus-host-key.sh`
  (the fixture harness), and `README.md` (the rotation runbook). This PR carries
  `Closes allod/nexus#4`.
- `allod/secrets` — the identity-bearing repo. Owns `identity.nix` (the
  `sshHosts` registry and everything under `lib.identity`, exposed by
  `flake.nix`). The new `externalSshTrustTargets` attrset lands here. Public
  template ships synthetic values only; real targets live in private forks. This
  PR carries `Refs allod/nexus#4`.
- `allod/profiles` — Home Manager packaging for dev VMs. `modules/home-shared.nix`
  is the sole consumer of `identity.sshHosts` — relevant to whether adding
  synthetic registry entries is safe.
- `allod/memory` — workspace policy (`dev-plans.md` risk levels and standing
  review lenses).

Current state (verify against the tree before trusting it; line numbers and file
layout drift):
- `nexus-host-key` is a ~600-line bash tool with three phases. `do_activate`
  swaps `~/.ssh/host` at the `install -m 0600 "${tmpdir}/key" "$AGE_IDENTITY"`
  line, *after* the staged-key decrypt / VM SSH / Forgejo checks. `do_retire`
  crosses its point of no return at `promote_staged_json` + `reencrypt_declared_secrets`.
  Any new external gate must sit *before* those exact points; the plan says so —
  confirm the insertion points named in Interface Contracts are the real ones.
- `vm_ssh_unreachable` classifies an ssh failure by matching connectivity strings
  in ssh's stderr (`No route to host`, `Connection timed out`, `Connection
  refused`, `Could not resolve hostname`, …); everything else — including auth
  rejection and host-key mismatch — falls through to fatal. In the VM loop,
  "unreachable" is the *skippable* outcome and auth-reject is fatal. The plan
  reuses this predicate for external gates but *inverts the disposition*: for an
  external gate, unreachable is a HARD STOP. The predicate is shared; the
  die-vs-continue decision around it is not. Check that the plan asks for the
  disposition flip and not a predicate change, and that a reused predicate cannot
  silently import the VM loop's skip semantics.
- Two different probe shapes already exist and they are not interchangeable. The
  VM loop probes with `-o UserKnownHostsFile=${KNOWN_HOSTS_VMS}` (a dedicated VM
  known-hosts file) and does *not* pass `-F /dev/null`. `host_git_ls_remote_with_key`
  probes with `-F /dev/null` and default known-hosts. The plan's external probe
  wants `-F /dev/null` + the operator's *default* `~/.ssh/known_hosts` (no
  `UserKnownHostsFile` override). These are three distinct trust roots; the
  reviewer must not let the plan conflate them.
- `resolve_target_user` / `resolve_forge_connection` resolve identity fields via
  `nix eval --raw path:${IDENTITY_CONFIG}#lib.identity.<...>` and `die` on
  failure. The new external-target resolver must match this pattern and its
  fail-closed disposition. `resolve_target_user` dies on an unresolved alias
  today — the model for "declared alias missing from `sshHosts` ⇒ die."
- `identity.sshHosts` has exactly one consumer: `home-shared.nix` maps it with
  `builtins.mapAttrs` into `programs.ssh.matchBlocks`, so every entry becomes a
  literal SSH `Host` block on every dev VM. Since a gate's `sshHost` must resolve
  to an existing `sshHosts` alias, shipping synthetic gate entries in the public
  template forces synthetic `sshHosts` entries too — which become real match
  blocks. This is the crux of the public/private split, not a footnote.
- The fixture harness (`tests/nexus-host-key.sh`) mocks `ssh`/`nix`/`virsh`/`git`/
  `age`/`agenix` as PATH shims. The fake `ssh` stub today *hard-requires*
  `UserKnownHostsFile=${fixture}/known_hosts_vms` and exits 69 if it is absent,
  and asserts the identity is a rotation key (`-i .../nexus-host-key.*` or
  `-i .../.ssh/host`) but does **not** distinguish targets. Reachability and
  auth-failure are global toggle files (`ssh-unreachable`, `ssh-fail`), not
  per-target. The plan's external probe deliberately omits `UserKnownHostsFile`,
  so the *current* stub would reject every external probe with exit 69. Teaching
  the stub per-target accept/reject/unreachable and the new probe shape is real,
  load-bearing work, not a copy-paste.
- `nix flake check` in nexus runs the bash suite under the *fake* `nix`, so it
  never evaluates the real `secrets` flake; the secrets-side `credential-inventory`
  check does not reference `sshHosts` or `externalSshTrustTargets` today. Neither
  side automatically validates the new attrset's shape unless the plan adds a
  check. Confirm whether that is a gap or an accepted limit.
- Execution constraint: agents run inside dev VMs. The real service-backup VPS and
  the offsite backup destination are human-only infrastructure the agent cannot
  reach; proving the real gate against them is an operator step, correctly gated.
  Do not treat the human step as a defect — but do confirm every agent-runnable
  path is exercisable in fixtures without it.

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections
from `dev-plans.md`: Tracking Issue, Goal, Scope, Risk Assessment, Interface
Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. This is multi-PR
work, so verify it assigns residual risk per PR or milestone. Verify the
closing-keyword story: the `allod/secrets` PR carries `Refs allod/nexus#4` and
only the `allod/nexus` PR carries `Closes allod/nexus#4`. The plan states landing
order is not load-bearing because the consumer no-ops on an absent registry —
pressure-test that claim against the rollback plan (revert nexus-before-secrets)
and the fail-closed contract (an *eval error* must die, an *absent attribute*
must be empty); a no-op-on-absent claim is only safe if "absent" and "errored" are
genuinely distinguishable at the resolver.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have
problems. These are lenses, not checklists — follow the thread wherever it leads.

The six standing lenses in `dev-plans.md` (internal consistency, operational
sequencing, risk calibration, acceptance-test coverage, rollback fidelity,
generated lifecycle behavior) apply as defaults on top of the plan-specific areas
below.

1. **Scoped diff review of the pass-1 plan commits (`b5a9569..55f8c3f`).** Pass 1
   cut two schema fields (`requiredFor`, `verify`), pinned the resolver to a
   single guarded `nix eval --json --apply` join, added a third probe-failure
   class (host-key verification failure), moved registry resolution to
   phase-start before any mutation, pinned the retire gate before the
   `OLD_BACKUP` handling, and rewrote the stub contract and the secrets R2 row.
   Structural fixes are where new blockers enter; verify each against the tree
   the way pass 1 verified the original. Sharpest seams: (a) does the pinned
   `--apply` expression carry everything the gate needs (names, port-as-number
   vs `-p` string, the `identityFile` input for the premise warning — which the
   join as sketched does *not* emit)? (b) does running the retire gate before
   the cheap `OLD_BACKUP` state checks trade fail-fast on local corruption for
   prompt ordering, and is that the right trade? (c) is the sorted-name gate
   order + stdin confirmation sequence implementable with the
   `confirm_missing_old_key_retire` pattern when `--accept-missing-old-key`
   and multiple overrides combine in one run — is there a combined-stdin test
   case, and should there be? (d) did dropping `requiredFor`/`verify` lose
   anything the issue's registry design note actually needs?

2. **The client-key premise (carried; needs a human answer).** The plan now
   states the premise (Risk Assessment), gates it on per-target operator
   confirmation (Agent Gates), and warns when a gate alias declares an
   `identityFile` other than the rotated identity. Still open until the
   operator confirms, per real target: (a) the backup jobs authenticate with
   `~/.ssh/host`, and (b) `ssh <user>@<host> true` is a valid no-op there — a
   restricted storage-box shell fails the probe against a healthy key. If (b)
   fails for a real target, a second verifier kind must be designed before
   that target can be gated, which reshapes Interface Contract 3 and revisits
   the `verify`-field cut. Do not close this from repo context; it is
   operator-only.

3. **Stub implementability of the new contracts.** The plan now demands
   per-class probe-shape assertions keyed by `<user>@<host>` and a fake-`nix`
   arm for the single-eval registry query with per-fixture JSON and an
   eval-error toggle. The stubs pattern-match substrings of `$*`: confirm the
   external probe (no `UserKnownHostsFile`, `-F /dev/null`, optional `-p`) and
   the `--apply` query are cleanly distinguishable without loosening the
   VM/Forgejo assertions, and that the empty-registry fixture flows through
   the same arm as populated ones (not a bespoke bypass that would let the
   real query shape drift untested).

Run the template's SIMPLIFY sweep every pass. Standing candidates after the
pass-1 cuts: the `recovery` enum (kept — three genuinely different remediation
actions, all demanded by the issue; re-cut only if the printed variants
converge in implementation) and the `identityFile` premise warning (cut it if
tilde-vs-`$HOME` normalization makes it fiddly out of proportion to its
signal). The activate-time override was considered and kept with its rationale
now recorded in the plan; do not re-litigate without new evidence.

Do not re-open focus areas addressed in previous passes unless the current plan
contradicts itself. Pass 1 resolved the original seven seed areas (fail-closed
resolution mechanics, probe classification, known-hosts posture, override
binding, template blast radius, fixture realism) into plan text; the areas
above are what remains productive.

Fix-stability record: pass-1 fixes authored by claude-fable-5; stability
unknown until a later pass verifies them.

Next pass: scoped diff review of `b5a9569..55f8c3f` (the nine pass-1 plan
commits), not a full re-review, by a model other than claude-fable-5 — no
fix-stability record exists for this prompt yet, so any capable non-author
model qualifies. Read the findings summary in the pass-1 prompt commit for the
finding-to-commit map, and record how the pass-1 fixes held up.

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves.
  Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any
  interface — including the rotation flow's current "external hosts are a printed
  follow-up reminder" behavior, which this plan deliberately replaces with a gate.
- **Do not overengineer.** If the plan introduces abstraction that is not needed
  yet, call it out. Three similar lines beat a premature helper; a hardcoded probe
  beats a one-value `verify` enum. Run an explicit SIMPLIFY sweep every pass:
  actively hunt for scope, ceremony, or premature enum/registry structure to
  delete rather than treating a quiet pass as nothing to cut.
- **Solo project, one human.** No team coordination overhead. No release process.
  No migration guides for other consumers.
- **Security matters, ceremony does not.** The privacy boundary (no real external
  targets in the public template) and the fail-closed gate (never strand offsite
  backup access) must be airtight. Everything else can be pragmatic.
- **Solve problems as they come.** If the plan front-loads work for hypothetical
  future needs (a second `verify` kind, a fourth `recovery` mode, non-rotation
  `requiredFor` consumers), flag it.
- **Think operationally.** Consider what happens when the operator runs
  `stage`/`activate`/`retire` cold, mid-rotation, with a target that is genuinely
  down, with a fork that trimmed the example hosts, or with the two PRs landed in
  either order.
- **Calibrate residual risk.** The risk score is for human triage after validation
  passes. The plan claims R3 for the nexus PR because the operation is local and
  explicit, every path but the real-host proof is exercised in fixtures, rollback
  is a straight `git revert`, and a bug fails toward *more* gating. Pressure-test
  that last clause hard — Focus Area 1 is where R3 becomes R4 if a required gate
  can silently vanish. R2 for the additive secrets schema is plausible; test it
  against the `sshHosts`-becomes-matchBlocks blast radius, not just "it's additive
  data."
- **Inspect generated lifecycle artifacts.** This plan is mostly a bash state
  machine, but it touches Home Manager (`sshHosts` → `programs.ssh.matchBlocks`)
  and flake evaluation (`externalSshTrustTargets` under `lib.identity`,
  `nix flake check` on both repos). Reason about the generated dev-VM SSH config,
  the negative paths (fork declares a gate but the alias is absent → does
  evaluation die or produce a bad block?), and whether either `nix flake check`
  actually exercises the new attrset or merely tolerates it.

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
an unnecessary abstraction (the `verify`/`recovery` enums and the activate-time
override are the standing candidates). Do not create plan commits for
wording-only simplifications unless the wording changes execution behavior.

Use `[QUESTION]` only when the plan cannot be corrected from repo context (the
"which client key authenticates to the real backup targets" premise in Focus
Area 2 may be one of these — it depends on infra the agent cannot see). If the
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
something is right, name it and explain why so a later pass does not undo it. The
strong candidates to bless (pressure-test first, then protect): running the proof
*before* the `~/.ssh/host` swap and before `promote_staged_json`; failing closed
on an eval error while treating an absent attribute as empty; using the rotation
key (not the operator's `identityFile`) for the probe; and failing closed on an
unknown/changed external host key for a backup channel. QUESTIONs must be resolved
in the plan, not left as open items. If the answer is clear from the codebase,
update the plan and commit. If the answer requires human input (e.g. the client-key
premise), add the question to the Focus Areas section for the next pass.

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
