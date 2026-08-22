# Managed Pi Named Tokens Review Prompt

You are the authentication-lifecycle boss: part Nix graph surgeon, part crash-consistency gremlin, part Pi extension whisperer. Be ruthless; this plan handles bearer credentials and live runtime switching, so polite hand-waving is the enemy.

## Your Task

Review the [Managed Pi Named Tokens plan](../dev-plans/pi-named-tokens.md) for gaps, misunderstandings, bugs, and unnecessary work. Read the actual repositories and Pi 0.84.2 docs/runtime before judging it. Make plan fixes as commits rather than merely reporting them.

## Project Context

Allod composes public Nix repositories into private development machines. This change extends the existing managed Pi provider lifecycle without publishing private integration facts.

Key repositories:

- `allod/secrets` — credential schema, ciphertext/recipient derivation, and per-VM projections.
- `allod/nexus` — the host-side `pi-provider` transactional lifecycle command.
- `allod/archetypes` — Age delivery, reconciliation, helper, generated extension, and lifecycle witness.
- `allod/deploy` — compatible public input graph and composition canary.

Current facts to verify against the tree:

- One credential can serve several providers, but each credential currently has one `<credential>.age` ciphertext and one volatile file/link.
- `pi-provider` journals exact touched paths across profiles and secrets and currently re-prompts for retargeting rather than decrypting ciphertext.
- The helper validates one RFC 6750 line and emits it only to Pi's command-backed credential stdout.
- The reconciler atomically merges managed `auth.json`, `models.json`, links, and ownership state while preserving unrelated/operator-owned data.
- Pi 0.84.2 has supported provider re-registration and non-persistent runtime key overlays; extension commands execute during streaming, and pending-message/settled APIs exist.
- Public history may contain identifiers and synthetic tokens only, never private endpoints, provider/model IDs, targets, recipient keys, selections, or bearers.

## Structural Conformance

Verify all mandatory sections from `dev-plans.md` exist: Tracking Issue, Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Check per-slice residual risk, generated lifecycle witnesses, and that only the final deploy PR closes issue #36.

## Focus Areas

The six standing lenses from `dev-plans.md` apply, plus:

1. **Startup before first request.** Verify Pi actually loads/authenticates models and extensions in an order where the generic selector plus `session_start` can resolve remembered/default/interactive choice before any request. Flag any UI/non-interactive path the proposed APIs cannot implement.

2. **Queued-work pinning.** Inspect event ordering and provider re-registration. Can `agent_settled` plus pending-message state truly keep the active request and every already queued message on the old token without delaying forever or switching one provider early?

3. **Transactional directory migration.** Follow `pi-provider` journal/path safety literally. Can `<credential>.age` become `<credential>/default.age`, and can token-directory add/remove/recovery work without untracked directories, symlink traversal, plaintext staging, or incomplete rollback?

4. **Generated ownership and state concurrency.** Check token links, extension installation/removal, preference locking/atomic writes, absent secrets, corrupt state, rebuild/reboot, final empty state, and operator collisions. Demand executable generated-artifact witnesses rather than source-only assertions.

5. **SIMPLIFY.** Look for commands, metadata, compatibility paths, state helpers, or deployment work that can be deleted while preserving the settled issue contract. Do not add generic quota/failover machinery.

## Review Guidelines

- Forward momentum is king; flag implementation blockers and real rework, not style.
- This is pre-alpha and migration is forward-only; do not demand legacy fallback.
- Security boundaries are strict, ceremony is not.
- One human operates this system; avoid team/release abstractions.
- Inspect generated Nix/Home Manager artifacts and negative lifecycle paths.
- Do not re-open settled provider-versus-token semantics or add startup override flags without concrete code evidence.

## Severity Rubric

Use `[BLOCKER]` for unsafe/destructive behavior, nonfunctional startup/activation/migration/rollback, a privacy breach, or missing human input. Use `[GAP]` for material ambiguity or test/rollback holes. Use `[SIMPLIFY]` for removable scope or abstraction. Use `[QUESTION]` only when the answer cannot be inferred from code or documented decisions.

## Deliverable

Every pass ends with plan-file commits for real findings (or explicit no findings), then a review-prompt commit updating Focus Areas and pass metadata, and a push. Number findings, tag them, put blockers first and questions last. Resolve inferable questions directly in the plan.

The review-prompt commit message must count new findings by tag, classify each as original-plan or review-introduced, include fixing hashes, state what the SIMPLIFY sweep considered, and end with `Model: <exact model>`.

Stop when review-introduced findings outnumber original-plan findings for two consecutive passes, or two consecutive passes produce neither a BLOCKER nor an original-plan GAP. A blocker-level fix requires a scoped diff pass by a different model next.

## Before Final Response

- Plan fixes are committed or no-findings is explicit.
- This prompt is updated and committed with findings metadata.
- The repository is pushed and clean.
