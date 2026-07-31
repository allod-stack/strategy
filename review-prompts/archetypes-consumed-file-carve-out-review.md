# Archetypes Consumed-File Carve-Out Review Prompt

You are the absolute best at what you do. You read Nix store closures the way other people read a shopping list, you have personally watched a `builtins.path` call quietly change what a machine carries, and you have never once been fooled by a check that passes because it matches the wrong suffix. Nix evaluation semantics, agenix activation, home-manager file placement, and the difference between exposure and access are all your home turf. You are exactly the person who should be looking at this. Do not hold back.

## Your Task

Review [the consumed-file carve-out plan](../dev-plans/archetypes-consumed-file-carve-out.md) for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag anything that will block implementation, create unnecessary work, or leave a landmine for future changes.

Read the actual codebase to ground the review in reality. Do not review the plan in isolation. Several of this plan's claims are empirical and cheap to verify — verify them rather than reasoning about them.

## Project Context

**Allod** is a self-sovereign NixOS stack: one human owner, a bare-metal host provisioning disposable VMs, a self-hosted Forgejo, and AI agents working inside those VMs.

Key repos in play:

- `allod/archetypes` — the VM framework: archetype builders (`mkDevVm`, `mkPrivacyVm`, `mkHypervisor`), shared modules, `vmFacts`, and the checks. This is the only repo the plan changes.
- `allod/secrets` — the identity template: encrypted blobs, the recipient graph, and the `lib.devIdentities` attrset that hands machines their token file paths. Public example data; the real one is private.
- `allod/tools` — workspace CLI tools and the tracked git hooks a dev machine installs.
- `allod/deploy` — the composition root that pins the above and exposes `nixosConfigurations`.

Current state (verify against the tree before trusting it; line numbers drift):

- Interpolating a flake input and appending a path yields a string naming a file *inside* the input's store directory, so the derivation depends on the whole directory. This is the defect under repair.
- `modules/agent-hooks.nix` and `modules/github-credentials.nix` are imported only from `mkDevVm`. Privacy VMs and the hypervisor do not use them.
- `nix flake check` over the composition root peaks near 7 GiB and is OOM-killed on an 8 GiB VM. Use per-configuration `nix eval` instead.
- Agents run in dev VMs and cannot merge, rebuild a machine, provision, or touch keys.

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections from `dev-plans.md`: Tracking Issue, Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Agent Gates may be omitted only if no actions require a human. If the work spans multiple PRs, verify it assigns residual risk per PR or milestone. If the plan closes its tracking issue, verify which PR carries the closing keyword.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have problems. These are lenses, not checklists — follow the thread wherever it leads.

The six standing lenses in `dev-plans.md` (internal consistency, operational sequencing, risk calibration, acceptance-test coverage, rollback fidelity, generated lifecycle behavior) apply as defaults on top of the plan-specific areas below.

1. **The token-file claim is the plan's biggest bet — test it.** The plan asserts that `secrets/flake.nix:49-56`'s `./secrets + "/<name>.age"` resolves to a path *inside* the secrets flake's store directory, and therefore that `agent-forgejo-token.nix:5` and `flake.nix:199` must also be carved or the closure test can never pass. The tracking issue does not name these sites at all. If this claim is wrong, the plan's scope is inflated and one of its acceptance tests is unreachable. Evaluate it against the real tree and say which way it falls.

2. **Is the enumeration of sites complete?** The plan lists eight. Sweep `mkDevVm`'s full module list — `home-shared.nix`, `ai-agents.nix`, `dev-home-shared.nix`, `preferences`, `vm.homeModules.base`, and anything in `sharedModules` — for any other construction that interpolates an input root and appends a path. A site missed here is a site the closure test will catch late and expensively, or worse, one that makes the closure test's premise unreachable and sends someone hunting.

3. **Does loosening the GitHub matcher weaken a real guard?** The plan adds a `-<basename>` suffix alternative to `githubTargetMatchesGeneratedAgeSecret` (`flake.nix:494,499`) so a carved path binds, mirroring the Forgejo matcher. A basename suffix is weaker than a full relative-path suffix. Can two distinct declared targets now both match the same generated secret, or a target match a secret it should not? If the loosening is unsafe, what binds a carved secret to its target instead?

4. **Is the closure query the right instrument?** The plan uses `nix-store -q --requisites <drvPath>` to prove the source directory is gone without building the system. Does that actually reveal source paths referenced by an activation script, or does it report build-time inputs in a way that either misses the reference or reports it even after a correct carve? If it is the wrong instrument, name the right one — this test carries all of the plan's confidence, and a test that cannot distinguish success from failure makes the whole plan hollow.

5. **SIMPLIFY: does the `allod-tools` carve belong here at all?** `agent-hooks.nix:11` carries no ciphertext and no exposure argument; its justification is purely that commits to `allod/tools` move every dev machine's derivation. It rides along because it is the same edit in the same file. Argue it either way, but argue it: if it belongs in a separate change, say so and cut it.

Do not re-open focus areas addressed in previous passes unless the current plan contradicts itself.

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves. Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any interface.
- **Do not overengineer.** If the plan introduces abstraction that is not needed yet, call it out. Three similar lines beat a premature helper. Run an explicit SIMPLIFY sweep every pass: actively hunt for scope, ceremony, or abstraction to delete rather than treating a quiet pass as nothing to cut.
- **Solo project, one human.** No team coordination overhead, no release process, no migration guides for other consumers.
- **Security matters, ceremony does not.** The privacy and security boundaries must be airtight. Everything else can be pragmatic.
- **Solve problems as they come.** If the plan front-loads work for hypothetical future needs, flag it.
- **Think operationally.** Consider what happens when someone executes this plan with incomplete context or in the wrong order.
- **Calibrate residual risk.** The plan claims R3 and argues explicitly against R4 on the grounds that a mistake fails loud or leaves today's exposure, never silently exposing more. Attack that argument. Construct the mistake that exposes more, if one exists.
- **Inspect generated lifecycle artifacts.** Do not stop at source evaluation. Consider the agenix activation script, the home-manager file placements, and the negative paths — a machine with `forgeAccess = false` and therefore a null token file, a credential target with no matching consumer, an absent optional secret.

The person implementing this is technically sharp. They do not need hand-holding; they need the sharp edges they missed.

## Severity Rubric

Use `[BLOCKER]` only when following the plan literally is likely to: perform a destructive or unsafe operation; fail before implementation can complete; leave the resulting system nonfunctional; break first boot, activation, provisioning, rebuild, rotation, or rollback lifecycle behavior; violate a security or privacy boundary; or require missing human input that cannot be inferred from the repo or memory.

Use `[GAP]` for missing or contradictory plan details that could cause rework, test blind spots, stale docs, or implementation ambiguity, but where a competent agent with workspace memory could still proceed safely. Also use `[GAP]` when the Risk Assessment is missing, materially understated, materially overstated, or unsupported by the acceptance tests and rollback plan.

Use `[SIMPLIFY]` for unnecessary scope, ceremony, or abstraction. Commit SIMPLIFY fixes when they remove implementation work, delete unnecessary scope, or prevent an unnecessary abstraction.

Use `[QUESTION]` only when the plan cannot be corrected from repo context. If the answer is inferable, resolve it as `[GAP]` or `[SIMPLIFY]`.

Do not classify duplicated workspace policy, phrasing improvements, or reminders already covered by memory as findings unless the plan directly contradicts that policy.

## Deliverable

The deliverable is not a report. Every review pass ends with:

1. Plan-file commits for findings that require plan changes, or an explicit no-findings result.
2. A final review-prompt commit updating this prompt's Focus Areas and pass metadata.
3. A push to the remote.

For each finding that requires a plan change, edit the plan and commit the fix. Group changes into logical, self-contained commits. A one-line commit is fine when it records a real implementation decision. Fold or skip commits that only rephrase already-correct guidance.

Use `allod change record -m <message> -f <file>...` to commit; name your files, because `record` stages `git add -u` and skips untracked files. Commit messages are plain text, carry no agent attribution trailer, and end with an exact `Model: <slug>` footer.

## Output Format

Give your review as a numbered list of findings, each tagged `[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, or `[QUESTION]`. Start with blockers, end with questions. Be blunt.

If a design decision is sound, say so briefly — do not damn with faint praise. If something is right, name it and explain why so a later pass does not undo it. QUESTIONs must be resolved in the plan, not left as open items.

## After Each Pass

As a final commit, update this prompt's Focus Areas section: remove areas fully addressed, refine partially addressed ones, add new areas discovered. The focus areas should always reflect the most productive targets for the next pass, not a historical record.

Include a plain-text findings summary in this commit's message:

- Count only findings new to this pass, by tag. Carried-over unresolved items move to Focus Areas rather than being re-counted.
- Give each new finding a numbered entry with its tag, short title, one-sentence explanation, fixing commit hash, and issue link if one exists.
- Classify each finding's origin: an original-plan defect, or introduced by an earlier review pass (name the commit).
- State what the SIMPLIFY sweep considered for deletion, even when nothing was cut.
- End with a `Model: <exact model>` footer — the model identifier, not the agent software.

Add a `Next pass:` line naming the commit(s) to review, whether it is a scoped diff or a full pass, and a recommended model other than the fix's author.

Stop the plan-text review when either condition holds:

- Review-introduced findings outnumber original-plan findings for two consecutive passes.
- Two consecutive passes produce no BLOCKER and no original-plan GAP.

## Before Final Response

- Plan fixes are committed, or the pass explicitly found no plan changes.
- This prompt's Focus Areas are updated and committed.
- The final commit message includes the findings summary and `Model:` footer.
- The repo is pushed. `git status` is clean.

## Pass History

Pass 0 — plan authored by `claude-opus-5`. No review yet.
