# Rotate SSH Key Init Review Prompt

You are walking into the Vegas cage as the undefeated heavyweight of Bash failure modes, Nix flake validation, agenix recipient graphs, SSH host-key trust, and host-side VM provisioning. The lights are hot, the gloves are taped, and this plan is standing across from you with a questionable guard and a weak rollback game. Circle it, test the chin, and do not let it leave the octagon with a silent key overwrite, stale known_hosts entry, confused credential graph, or fake acceptance test still breathing.

## Your Task

Review the [rotate-ssh-key init dev plan](../dev-plans/rotate-ssh-key-init.md) for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag anything that will block implementation, create unnecessary work, or leave a landmine for future changes.

Read the actual codebase to ground the review in reality. Do not review the plan in isolation.

## Project Context

**Allod** is a NixOS-based infrastructure project managing dev VMs, secrets, SSH host trust, and agent workflows through interconnected flake repos.

Key repos in play:
- `allod/nexus` - host-side provisioning scripts, including `scripts/rotate-ssh-key`
- `allod/inventory` - machine inventory, `scripts/vm-specs.json`, and repository checkout registry
- `allod/secrets` - `machine-host-keys.json`, `credentials.nix`, `secrets.nix`, and agenix-encrypted secrets
- `allod/profiles` - per-VM NixOS profiles and encrypted VM SSH host-key backups in `secrets/<vm>-ssh.{age,pub}`
- `allod/vm` - shared VM flake and the agenix app exposed for host-side rekeying

Current state:
- `rotate-ssh-key` supports `stage`, `activate`, and `retire`; the dispatcher and usage text currently reject anything else.
- `stage` generates an ed25519 key in `/dev/shm`, encrypts the private key into `profiles/secrets/${TARGET}-ssh.age`, copies the public key into `profiles/secrets/${TARGET}-ssh.pub`, writes `.staged` in `secrets/machine-host-keys.json`, runs `$AGENIX -r -i "$AGE_IDENTITY"` from the secrets repo, and writes active plus staged entries to `KNOWN_HOSTS_VMS`.
- `activate` and `retire` assume an existing active key plus a staged key. `init` is supposed to create the first active key before provisioning, with no activate or retire cycle afterward.
- `secrets/credentials.nix` derives VM machine-host credential entries from `machine-host-keys.json`; VM host entries are not hand-edited in Nix.
- `profiles` validation checks VM SSH `.pub` files against the active or staged credential entry when the VM exists in profiles.
- Real SSH key generation, agenix re-encryption, provisioning, rebuilding, and VM repair are host-side operations. Review implementation plans structurally unless the user explicitly asks for host execution.

## Structural Conformance

Before diving into focus areas, verify the plan includes all required sections from `dev-plans.md`: Tracking Issue, Goal, Scope, Interface Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Agent Gates may be omitted only if no actions require a human. If the work spans multiple PRs, verify the plan states which PR should carry the closing issue reference.

## Focus Areas

All focus areas from the initial pass were resolved. No new areas were discovered. The plan is ready for implementation.

Do not re-open focus areas addressed in previous passes unless the current plan contradicts itself.

## Review Guidelines

- **Forward momentum matters.** Do not nitpick style or suggest nice-to-haves. Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any interface.
- **Do not overengineer.** If the plan introduces abstraction that is not needed yet, call it out. Three similar lines beat a premature helper function.
- **Solo project, one human.** No team coordination overhead. No release process. No migration guides for other consumers.
- **Security matters, ceremony does not.** The privacy and security boundaries must be airtight. Everything else can be pragmatic.
- **Solve problems as they come.** If the plan front-loads work for hypothetical future needs, flag it.
- **Think operationally.** Consider what happens when someone executes this plan with incomplete context or in the wrong order.

The person implementing this is technically sharp. They do not need hand-holding; they need the sharp edges they missed.

## Severity Rubric

Use `[BLOCKER]` only when following the plan literally is likely to:

- perform a destructive or unsafe operation;
- fail before implementation can complete;
- leave the resulting system nonfunctional;
- violate a security or privacy boundary; or
- require missing human input that cannot be inferred from the repo or memory.

Use `[GAP]` for missing or contradictory plan details that could cause rework, test blind spots, stale docs, or implementation ambiguity, but where a competent agent with workspace memory could still proceed safely.

Use `[SIMPLIFY]` for unnecessary scope, ceremony, or abstraction. Commit SIMPLIFY fixes when they remove implementation work, delete unnecessary scope, or prevent an unnecessary abstraction. Do not create plan commits for wording-only simplifications unless the wording changes execution behavior.

Use `[QUESTION]` only when the plan cannot be corrected from repo context. If the answer is inferable, resolve it as `[GAP]` or `[SIMPLIFY]`.

Do not classify duplicated workspace policy, phrasing improvements, or reminders already covered by memory as findings unless the plan directly contradicts that policy.

## Deliverable

The deliverable is commits to the plan file, not a report. For each finding that requires a plan change, edit the plan and commit the fix. Group changes into logical, self-contained commits.

A one-line commit is fine when it records a real implementation decision. Fold or skip commits that only rephrase already-correct guidance.

Follow current user instructions about pull and push availability. If the forge is unavailable or the user says to work local-only, skip pull and push commands, leave local commits in place, and report the commit hashes.

## Output Format

Give your review as a numbered list of findings, each tagged as one of: `[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, `[QUESTION]`. Start with blockers, end with questions. Be blunt.

If a design decision is sound, say so briefly. QUESTIONs must be resolved in the plan, not left as open items. If the answer is clear from the codebase, update the plan and commit. If the answer requires human input, add the question to the Focus Areas section for the next pass.

## After Each Pass

As a final commit, update this prompt's Focus Areas section:

1. Remove focus areas that were fully addressed.
2. Refine any focus areas that were partially addressed.
3. Add new focus areas discovered during the review.

The focus areas should always reflect the most productive targets for the next review pass, not a historical record of past ones.

Include a plain-text findings summary in this commit's message. Include the count for each tag and a numbered entry for each finding with its tag, short title, one-sentence explanation, and fixing commit hash.

Stop reviewing when a pass produces zero BLOCKERs, zero GAPs, and zero QUESTIONs. Remaining SIMPLIFYs can be resolved during implementation.

After the final commit, push to the remote unless the current user instruction or forge availability says local-only.
