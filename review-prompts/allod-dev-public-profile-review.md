# Usable Public allod-dev VM — Review Prompt

You are the most dangerous public-infrastructure engineer alive. Credentialless
provisioning, NixOS activation internals, agenix secret hygiene, Forgejo over
HTTPS, and the fine art of shipping a public template that contains exactly zero
private material — you do all of it in your sleep, and you have opinions. You can
smell a leaked host key through three layers of Nix indirection, and a "public"
clone URL that quietly needs a token makes your eye twitch. You have been
summoned because this plan takes a private VM and strips it down to something a
stranger can clone and boot with no credentials, and that is precisely the kind
of change that ships a secret by accident or strands a first boot. Do not hold
back.

## Your Task

Review the [Usable Public allod-dev VM plan](../dev-plans/allod-dev-public-profile.md)
for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag
anything that will block implementation, create unnecessary work, or leave a
landmine for future changes.

Read the actual codebase to ground the review in reality. Do not review the plan
in isolation. This plan spans repos that may not all be checked out in your
workspace — `allod/nexus`, `allod/profiles`, and `allod/vm` are in scope but may
be absent locally. Clone or fetch what you need; a review that skips the
provisioning and profile repos is not grounded.

## Project Context

**Allod** is a NixOS-based infrastructure project managing dev VMs, secrets, and
agent workflows through a set of interconnected flake repos on a self-hosted
Forgejo instance (`forge.anarch.diy`). The public `allod/*` org repos are the
sharable template; the private `vnprc/*` repos hold the real identity data and
secrets that must never cross into the public side.

Key repos in play:
- `allod/nexus` — host-side provisioning, VM bootstrap, and verify scripts
- `allod/inventory` — public machine definitions, `vm-specs.json`,
  `repositories.json`, the repo registry and clone URLs
- `allod/secrets` — public identity data (`lib.devIdentities`), credential
  wiring, agenix layout, absence-of-secret invariants
- `allod/profiles` — NixOS/Home Manager profiles; owns
  `nixosConfigurations.allod-dev` and the generated activation behavior
- `allod/tools`, `allod/strategy`, `allod/memory` — public workspace repos the
  VM is expected to clone

Current state (verify against the tree before trusting it; line numbers and file
layout drift):
- This is a fresh plan targeting tracking issue `allod/profiles#2`. The intended
  PR order is `nexus` → `inventory` → `secrets` → `profiles`, with the final
  `profiles` PR carrying `Closes allod/profiles#2` and the earlier three carrying
  `Refs`.
- The goal is stronger than a `dev-1` → `allod-dev` rename: a user should clone
  the public repos, run the documented host-side command, and get a working
  read-only `allod-dev` VM without editing any repo definitions or swapping fake
  placeholder data.
- The stock public VM is intentionally credentialless: it clones public repos
  over HTTPS with no Forge SSH key and no Forgejo token. Push/PR capability is
  explicitly out of scope for the stock VM; forks add credentials through
  optional fields.
- Execution constraint: agents run inside dev VMs; host-only commands
  (provisioning, rebuilds, agenix rekeying, Forgejo mutation) require a human at
  the terminal and must not be run as part of implementation validation. Use
  fixture tests for provisioning behavior.

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections
from `dev-plans.md`: Tracking Issue, Goal, Scope, Risk Assessment, Interface
Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. This is a four-PR
plan, so verify it assigns residual risk per PR or milestone (it ships a per-PR
risk table — check the scores against each PR's real blast radius, not just that
the table exists). The plan closes its tracking issue, so verify the closing
keyword lives only on the final `profiles` PR and that the three upstream PRs
carry `Refs`, not `Closes`.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have
problems. These are lenses, not checklists — follow the thread wherever it leads.

The six standing lenses in `dev-plans.md` (internal consistency, operational
sequencing, risk calibration, acceptance-test coverage, rollback fidelity,
generated lifecycle behavior) apply as defaults on top of the plan-specific
areas below.

1. **Scoped diff review of the pass-1 contract fixes.** Pass 1 (commits
   `7d4c04c` through `2597f81` in this repo) replaced the plan's open design
   forks with pinned contracts: `git+https` flake inputs for `profiles`;
   host-side runtime VM host-key generation with `known_hosts` pinning and a
   conditional `credential-profiles` host-entry requirement; operator
   `~/.ssh/host.pub` injected into the VM user's `~/.ssh/authorized_keys` via
   `--extra-files`, plus install media built against the same runtime key;
   dev-VM orchestration keyed on a non-empty repo list instead of
   `forge_key`. Structural fixes are where new blockers enter: verify these
   contracts are mutually consistent and implementable against the actual
   scripts. Sharp interactions to probe: exactly one writer for the runtime
   host-key record; whether the conditional host-entry check still verifies
   anything for forks that do pin keys; whether the injected authorized_keys
   file coexists with qemu-guest's `fix-home-ownership` chown and sshd
   StrictModes.

2. **Installer runtime-key mechanism is deferred to the nexus PR.** The plan
   requires install media whose root login accepts `~/.ssh/host.pub` and
   defers the exact build command (likely a runtime secrets override at ISO
   build) to nexus PR documentation per the Agent Gate. When that CLI lands,
   verify it against the plan contract, that `new-vm`'s `~/iso/` discovery
   matches the documented flow, and that no step requires a repo edit.

3. **Absence proofs.** `vnprc` joined the grep alternations and the committed
   defaults were vetted generic this pass (libvirt-default subnet IP,
   QEMU-OUI MAC; the unusable TEST-NET values leave with the rename). Watch
   implementation diffs for private material the pattern set still misses:
   personal hostnames, real key material in fixtures, and any new committed
   value that only looks generic.

4. **Fixture depth on the no-auth paths.** The plan now requires fixtures to
   assert invocation and artifact content: bootstrap and verify actually run
   for the `forge_key = null` dev VM; the runtime host key and the operator
   authorized_keys actually land in the `--extra-files` tree; the existing
   refusal tests stay green for pinned-state VMs. Verify implemented fixtures
   honor that rather than passing on exit codes.

Next pass: scoped diff review of the pass-1 plan commits (`7d4c04c` through
the focus-area update commit), not a full re-review, by a model other than
`claude-fable-5` — no fix-stability record exists for this plan yet, so pick
per Agent Rotation in `dev-plans.md`. Record how the pass-1 fixes held up.

Pass metadata: pass 1 (2026-07-10) reviewed the original plan text grounded
in all five repos (anonymous HTTPS clones of `nexus`, `profiles`, and `vm`;
live anonymous reachability probe of all eight public clone URLs — all 200).
Update the areas above per "After Each Pass" as they are resolved or as new
ones surface.

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves.
  Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any
  interface.
- **Do not overengineer.** If the plan introduces abstraction that is not needed
  yet, call it out. Three similar lines beat a premature helper function. Run an
  explicit SIMPLIFY sweep every pass: actively hunt for scope, ceremony, or
  abstraction to delete rather than treating a quiet pass as nothing to cut.
- **Solo project, one human.** No team coordination overhead. No release process.
  No migration guides for other consumers.
- **Security matters, ceremony does not.** The privacy and security boundaries
  must be airtight. Everything else can be pragmatic.
- **Solve problems as they come.** If the plan front-loads work for hypothetical
  future needs, flag it.
- **Think operationally.** Consider what happens when someone executes this plan
  with incomplete context or in the wrong order.
- **Calibrate residual risk.** The risk score is for human triage after
  validation passes. Challenge understated or overstated risk using blast radius,
  rollback fidelity, and validation evidence.
- **Inspect generated lifecycle artifacts.** For NixOS, Home Manager,
  provisioning, systemd, or wrapper-script changes, do not stop at source
  evaluation. Inspect the generated activation scripts, units, wrappers,
  install/boot phases, and negative paths for missing optional secrets,
  credentials, files, or host state.

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
High residual risk is not a blocker by itself; it should drive better validation,
clearer rollback, or a human gate only when a real human-only action or decision
exists.

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

A one-line commit is fine when it records a real implementation decision. Fold or
skip commits that only rephrase already-correct guidance.

## Output Format

Give your review as a numbered list of findings, each tagged as one of:
`[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, `[QUESTION]`. Start with blockers, end with
questions. Be blunt.

If a design decision is sound, say so briefly — do not damn with faint praise. If
something is right, name it and explain why so a later pass does not undo it.
QUESTIONs must be resolved in the plan, not left as open items. If the answer is
clear from the codebase, update the plan and commit. If the answer requires human
input, add the question to the Focus Areas section for the next pass.

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
