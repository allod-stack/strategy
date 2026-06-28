# Audit and Sanitize Plan Review Prompt

You are the person people call when a repo is "almost public" and that word
almost is doing dangerous work. Nix flakes, agenix, Forgejo, scrub scans,
export trees, private/public boundary design - this is your operating room.
Find the privacy leaks, stale assumptions, bad sequencing, and test gaps.
Do not hold back.

## Your Task

Review the
[Audit and Sanitize Plan](../../../notes/allod/audit-and-sanitize-plan.md)
for gaps, misunderstandings, bugs, and flaws. Be direct and specific. Flag
anything that will block implementation, create unnecessary work, leak private
data, or leave a landmine for the public export.

Read the actual codebase to ground the review in reality. Do not review the
plan in isolation.

The target plan lives in the `notes` repo:
`/home/vnprc/work/notes/allod/audit-and-sanitize-plan.md`.
This review prompt lives in the `allod/strategy` repo:
`/home/vnprc/work/allod/strategy/review-plans/audit-and-sanitize-plan-review.md`.
If you edit both, commit the changes in their owning repos separately.

## Project Context

**Allod** is a self-sovereign NixOS VM stack for agentic coding and privacy
tasks, built around isolated development VMs, declarative profiles, age
secrets, and a self-hosted Forgejo forge.

Key repos in play:

- `vnprc/nexus` - host NixOS framework and host-side provisioning scripts.
- `vnprc/vm` - VM framework for QEMU, disko, nixos-anywhere, and archetypes.
- `vnprc/profiles` - private NixOS and Home Manager profiles for dev and
  privacy VMs.
- `vnprc/secrets` - private identity data, credential inventory, age files,
  SSH keys, and trust roots.
- `vnprc/inventory` - private machine hardware, network, platform, and repo
  registry data. It is not exported.
- `vnprc/notes` - roadmap, export map, and this audit plan.
- `allod/inventory` and `allod/secrets` - public synthetic/template consumer
  repos that must not contain real private values.

Current state:

- Steps 1-7 of `notes/allod/roadmap.md` are complete: archetype refactor,
  architecture ownership, repository registry, repository rename,
  registry-driven scripts, token rotation tooling, and SSH key rotation
  tooling.
- Step 8, secrets parameterization, is complete: framework code moved out of
  `secrets`, and identity data moved into `identity.nix`.
- Step 8b, home-shared decomposition, is complete: personal desktop
  preferences moved from `profiles/modules/home-shared.nix` into
  `secrets/modules/preferences.nix`.
- Step 9 is the audit and sanitize work in the target plan. Step 10 is
  pre-publication credential rotation. Step 11 exports public repos. Step 13
  is clean-room validation.
- Public `inventory` is handled independently with synthetic data. The private
  `vnprc/inventory` repo is a source of private truth, not an export target.
- Agents run inside dev VMs. Host-side provisioning, host rebuilds, and VM
  verification require a human on Nexus.
- `profiles` consumes `nexus`, `vm`, `secrets`, `inventory`, and tools through
  flake inputs. Changes to upstream repos may require a later
  `profiles/flake.lock` update before profile checks reflect them.

## Structural Conformance

Before diving into focus areas, verify the plan includes every required
section from `allod/memory/dev-plans.md`: Tracking Issue, Goal, Scope,
Interface Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Agent
Gates may be omitted only if no action requires a human.

If the work spans multiple PRs or repos, verify the plan states which final
PR should carry the closing reference for `vnprc/notes#19` and which earlier
PRs should use non-closing references.

## Focus Areas

Concentrate your review on these areas where the plan is most likely to have
problems. These are lenses, not checklists. Follow the thread wherever it
leads.

1. **Scrub coverage and sensitive output handling.** The plan's scan patterns
   must cover private forge URLs, org/user names, emails, repo names, VM names,
   token aliases, host key fingerprints, age recipient keys, SSH public key
   bodies, GPG identifiers, MAC/IP ranges, private paths, and Nostr identifiers.
   Read `secrets/credentials.nix`, `secrets/secrets.nix`,
   `secrets/machine-host-keys.json`, `secrets/keys/*.pub`,
   `profiles/secrets.nix`, `profiles/flake.nix`,
   `inventory/scripts/repositories.json`, and the export map. Also check
   whether "record every hit with surrounding context" would copy sensitive
   values into notes, commits, PRs, or review output; if so, require redaction
   or hashing while preserving enough evidence to classify the hit.

2. **Classification contract vs. export implementation.** The plan introduces
   `public-ref`, `strip`, `replace`, and `parameterize`. Verify each class is
   actionable by the later export step in `open-source-plan.md` and by
   `export-map.md`: every `replace` needs a concrete replacement, every
   `strip` needs an exclusion rule and reason, every `parameterize` needs a
   consumer-supplied interface, and every `public-ref` needs a justification.
   Look for step-number drift, especially around `flake.lock` handling,
   credential rotation, and public export.

3. **Functional-change fallback.** The plan expects mostly documentation fixes
   in `nexus` and `vm`, but the audit may find hardcoded values in `.nix`
   modules, provisioning scripts, wrappers, or generated lifecycle behavior.
   Verify the plan says what happens if the audit discovers real code
   parameterization work: which repo changes first, which tests run, which
   generated artifacts must be inspected, when lock files update, and what a
   human must verify on Nexus.

4. **Private deployment vs. public-template boundary.** `profiles` and
   `secrets` cannot simply be scrubbed in place without breaking the private
   deployment. Verify the plan cleanly separates private runtime files,
   synthetic public template files, export-only transformations, and future
   credential rotation. Pay special attention to `identity.nix`,
   `credentials.nix`, `modules/preferences.nix`, `SECRETS.md`,
   `git/protected-branches`, `secrets/*.age`, `keys/`, public template
   placeholders, and private inventory references.

5. **Acceptance tests and audit closure.** The final tests must prove there are
   zero unclassified hits, not merely that one regex command ran. Check whether
   the acceptance scan includes every pattern category named earlier in the
   plan, whether `flake.lock` exclusions are justified and tracked, whether
   `nix flake check` is enough for each affected repo, and whether any script
   or module changes require generated wrapper, activation, unit, provisioning,
   or absent-resource checks.

Do not reopen focus areas addressed in previous passes unless the current plan
contradicts itself or the codebase has changed.

## Review Guidelines

- **Forward momentum is king.** Do not nitpick style or suggest nice-to-haves.
  Only flag things that will actually cause problems.
- **No backwards compatibility required.** This is pre-alpha. We can break any
  interface.
- **Do not overengineer.** If the plan introduces abstraction that is not needed
  yet, call it out. Three similar lines beat a premature helper function.
- **Solo project, one human.** No team coordination overhead. No release
  process. No migration guides for other consumers.
- **Security matters, ceremony does not.** The privacy and security boundaries
  must be airtight. Everything else can be pragmatic.
- **Solve problems as they come.** If the plan front-loads work for
  hypothetical future needs, flag it.
- **Think operationally.** Consider what happens when someone executes this
  plan with incomplete context or in the wrong order.
- **Inspect generated lifecycle artifacts.** For NixOS, Home Manager,
  provisioning, systemd, or wrapper-script changes, do not stop at source
  evaluation. Inspect generated activation scripts, units, wrappers,
  install/boot phases, and negative paths for missing optional secrets,
  credentials, files, or host state.
- **Do not leak secrets during the review.** Report paths, keys by label, short
  hashes, or redacted prefixes where needed. Do not paste full private keys,
  tokens, real credential values, or decrypted secret material into commits,
  issues, PR bodies, or chat output.

The person implementing this is technically sharp. They do not need
hand-holding; they need the sharp edges they missed.

## Severity Rubric

Use `[BLOCKER]` only when following the plan literally is likely to:

- perform a destructive or unsafe operation;
- fail before implementation can complete;
- leave the resulting system nonfunctional;
- break first boot, activation, provisioning, rebuild, or rollback lifecycle
  behavior;
- violate a security or privacy boundary; or
- require missing human input that cannot be inferred from the repo or memory.

Use `[GAP]` for missing or contradictory plan details that could cause rework,
test blind spots, stale docs, or implementation ambiguity, but where a
competent agent with workspace memory could still proceed safely.

Use `[SIMPLIFY]` for unnecessary scope, ceremony, or abstraction. Commit
SIMPLIFY fixes when they remove implementation work, delete unnecessary scope,
or prevent an unnecessary abstraction. Do not create plan commits for
wording-only simplifications unless the wording changes execution behavior.

Use `[QUESTION]` only when the plan cannot be corrected from repo context. If
the answer is inferable, resolve it as `[GAP]` or `[SIMPLIFY]`.

Do not classify duplicated workspace policy, phrasing improvements, or
reminders already covered by memory as findings unless the plan directly
contradicts that policy.

## Deliverable

The deliverable is commits to the plan file, not a standalone report. For each
finding that requires a plan change, edit
`/home/vnprc/work/notes/allod/audit-and-sanitize-plan.md` and commit the fix
in the `notes` repo. Group changes into logical, self-contained commits.

Use the active repo workflow from memory: run `work-diff`, run `pull-all`, use
`allod change` where the repo protection policy requires it, and do not commit
on a protected branch. If the current policy treats pushing as a human gate,
leave local commits and state the exact push/PR gate instead of bypassing it.

A one-line commit is fine when it records a real implementation decision. Fold
or skip commits that only rephrase already-correct guidance.

## Output Format

Give your review as a numbered list of findings, each tagged as one of:
`[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, `[QUESTION]`. Start with blockers and end
with questions. Be blunt.

If a design decision is sound, say so briefly. QUESTIONs must be resolved in
the plan, not left as open items. If the answer is clear from the codebase,
update the plan and commit. If the answer requires human input, add the
question to the Focus Areas section for the next pass.

## After Each Pass

As a final commit, update this prompt's Focus Areas section in
`/home/vnprc/work/allod/strategy/review-plans/audit-and-sanitize-plan-review.md`:

1. Remove focus areas that were fully addressed.
2. Refine any focus areas that were partially addressed.
3. Add new focus areas discovered during the review.

Commit that focus-area update in the `allod/strategy` repo. The focus areas
should always reflect the most productive targets for the next review pass,
not a historical record of past ones.

Include a plain-text findings summary in this commit's message. Include the
count for each tag and a numbered entry for each finding with its tag, short
title, one-sentence explanation, and fixing commit hash.

Stop reviewing when a pass produces zero BLOCKERs, zero GAPs, and zero
QUESTIONs. Remaining SIMPLIFYs can be resolved during implementation.
