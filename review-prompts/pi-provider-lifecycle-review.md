# Pi Provider Lifecycle Plan Review

You are the sharpest credential-lifecycle and Nix activation reviewer in the room. Treat cross-repository state, concurrent writers, and stale runtime processes as guilty until the generated evidence proves otherwise. Do not hold back.

## Task

Review [the Pi provider lifecycle plan](../dev-plans/pi-provider-credentials.md) for implementation-blocking gaps. Read the actual public repositories; do not review the prose in isolation. This is a public-only review: never request, infer, decrypt, or reproduce private provider data.

## Context

Allod is a self-sovereign NixOS VM stack. The public change spans:

- `profiles`: provider endpoint/protocol/model data;
- `secrets`: credential assignment, targets, recipients, and ciphertext paths;
- `inventory`: the Nexus workspace repository set;
- `archetypes`: generated Age and Pi lifecycle behavior;
- `nexus`: the owner-operated lifecycle command;
- `deploy`: compatible input composition.

Pi 0.84.2 accepts command-backed API keys, uses `proper-lockfile` for `auth.json`, and caches a resolved command result for the process lifetime. Nexus operations are host-only. Public templates contain no live provider entry.

## Structural Conformance

Verify Tracking Issue, Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Verify residual risk per slice and that public implementation PRs do not close the issue before private rollout.

## Focus Areas

1. **Scoped stability pass.** Review only the contract changes in `734a9fe` plus their immediate consistency with the rest of the plan. Do not re-open settled choices unless the fix contradicts another contract.

2. **Revision graph.** Confirm the profiles/secrets/inventory follow bindings and sabotage witness can establish one source revision at composition time.

3. **Credential cardinality.** Confirm global provider-to-credential uniqueness makes add, retarget, rotate, partial retire, and last-provider retire total and unambiguous.

4. **Crash recovery.** Walk SIGKILL or power loss across journal creation, each repository mutation, validation, rollback, and journal deletion. Confirm Git HEAD contains every required pre-run byte and recovery cannot erase unrelated work.

5. **Generated activation.** Check Pi-compatible auth locking, unowned-ID refusal, managed-ID replacement, missing-during-provisioning no-op, empty-desired cleanup, and stale-process verification against Pi's actual behavior.

6. **Simplify.** Confirm removing generic extension management did not leave dead contracts and look for another entire mechanism that can be deleted without losing the requested add/rotate/retire outcome.

## Review Rules

- Report only `[BLOCKER]`, `[GAP]`, `[SIMPLIFY]`, or `[QUESTION]` findings that change implementation or validation.
- Cite the plan contract and public code fact behind each finding.
- No style nits, compatibility ceremony, private deployment guesses, network use, file edits, commits, pushes, or Forge writes.
- End with decisions that are sound and should remain stable.

## Pass Evidence

Pass 1 reviewed `a32d545` with `gpt-5.6-sol` at high effort.

- Findings: 5 BLOCKER, 3 GAP, 1 SIMPLIFY, 1 QUESTION; all were original-plan defects.
- Fix: `734a9fe` binds the revision graph, makes provider resolution global, adds retarget and crash recovery, defines provisioning absence and Pi locking, gates revocation on a fresh process, declares managed-ID ownership, supplies the profiles checkout, and removes extension management.
- SIMPLIFY sweep: deleted the generic extension-link schema, state, cleanup, and garbage-collection scope.
- The earlier validation fixes in `be44135` remain stable through this pass: exact bearer grammar and command-backed auth validation were not reopened.

Next pass: perform the scoped stability review above on `734a9fe` with a model other than `gpt-5.6-sol`, preferably a cross-vendor R3-capable model; if unavailable use `gpt-5.5` at high or xhigh effort. This is the second and final plan-text pass; executable uncertainty moves to the named witnesses.

Model: gpt-5.6-sol
