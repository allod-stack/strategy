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

The R3 plan-text budget is complete. No third prose pass is scheduled. Implementation review inherits these executable witnesses:

1. **Revision graph.** Inspect the exported four-source observation and require independent sabotage of archetypes profiles/secrets/inventory plus secrets-inventory follows redirects.

2. **Host workspace.** Inspect the raw-machine registry check and require mutations proving every Nexus hypervisor has the three data checkouts while the public fixture added only profiles.

3. **Generated lifecycle.** Execute the crash-boundary, Pi-compatible locking, managed ownership, missing-secret no-op, empty-desired cleanup, shared credential, and stale-process witnesses named by the plan.

4. **Simplify.** Keep extension management out. Resolve any further executable simplification during implementation instead of reopening the plan text.

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

Pass 2 reviewed `734a9fe` and its immediate public-code consistency with `gpt-5.5` at high effort.

- Findings: 0 BLOCKER, 2 GAP, 0 SIMPLIFY, 0 QUESTION. Both gaps were introduced by the pass-1 fix: the three-input revision claim lacked the required public observation APIs, and the inventory witness read a guest-only projection that excludes Nexus.
- Fix: `9d639c9` specifies the secrets/archetypes source-observation interface, all four follows sabotages, raw-machine hypervisor validation, and the exact public Nexus workspace delta.
- Stability: global provider uniqueness, retarget, crash recovery, Pi locking/cache handling, managed ownership, missing-secret behavior, and extension-scope removal survived one independent pass unchanged. The revision and inventory witnesses needed immediate refinement.
- SIMPLIFY sweep: extension management remains deleted; no additional whole mechanism could be removed without losing the requested lifecycle.
- Stop: this is the second and final R3 plan-text pass. Remaining uncertainty is executable and moves to implementation review and the named witnesses.

Model: gpt-5.5
