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

## Pass History

- **Pass 1** — `claude-opus-4-8` (Claude Code runner), reasoning `medium`, reviewed `a6db27f`. 3 original-plan findings, 0 review-introduced: 1 BLOCKER (ownership-manifest version transition would strand the first post-upgrade activation — the reconciler's strict `.version==1`/per-credential-link check would treat a valid prior-version manifest as corrupt and never converge), 1 GAP (token-directory lifecycle unspecified; `realpath -e` parent check, single-segment `safe_journal_path` allowlist, and file-only rollback/recover/retire leave empty untracked directories), 1 SIMPLIFY (drop dead `rotationStrategy` metadata). Fixed in `e04ccc0`. Verified against Pi 0.84.2 docs that `registerProvider` from `session_start`, `agent_settled`, and `ctx.hasPendingMessages()`/`isIdle()` exist and support the startup and queued-work pinning claims (Focus Areas 1 and 2 hold, no findings there). Fix stability: not yet re-verified. Next pass should be a scoped diff review of `e04ccc0` by a different model (prefer a cross-vendor `gpt-5.6`-family runner) because pass 1 landed a BLOCKER fix.

This is review pass 2 (scoped diff verification of `e04ccc0` by a different model, then a fresh sweep). Work only in public checkouts under /home/vnprc/work/allod and read-only Pi package docs/store paths; do not inspect private checkouts. Commit fixes and prompt metadata directly to master. A push may be rejected by the environment boundary; still leave complete local commits and a clean tree. Read the full applicable allod memory files first.

## Focus Areas

The six standing lenses from `dev-plans.md` apply, plus:

1. **Startup before first request.** Verify Pi actually loads/authenticates models and extensions in an order where the generic selector plus `session_start` can resolve remembered/default/interactive choice before any request. Flag any UI/non-interactive path the proposed APIs cannot implement.

2. **Queued-work pinning.** Inspect event ordering and provider re-registration. Can `agent_settled` plus pending-message state truly keep the active request and every already queued message on the old token without delaying forever or switching one provider early?

3. **Transactional directory migration (verify pass-1 fix).** Pass 1 made the token directory a journaled object (create-under-lock, prune on rollback/recover/last-token-remove/retire) and widened the journal allowlist and recoverable-plan checks to the two-segment path. Verify the fix is coherent against `nexus/scripts/pi-provider`: does the recovery journal record the directory's pre-run absence, and does pruning refuse to remove a directory an operator or concurrent writer populated?

4. **Ownership-manifest version transition (verify pass-1 fix).** Pass 1's BLOCKER fix requires the reconciler to upgrade a valid prior-version manifest in place rather than refusing it as corrupt. Verify the plan distinguishes prior-version from genuinely corrupt state without weakening the corrupt-state refusal, and that the upgrade re-derives owned links from the desired set rather than trusting stale link records (`archetypes/modules/pi-provider-reconcile.sh`, `archetypes/checks/pi-provider-lifecycle.nix`).

5. **Generated ownership and state concurrency.** Check token links, extension installation/removal, preference locking/atomic writes, absent secrets, corrupt state, rebuild/reboot, final empty state, and operator collisions. Demand executable generated-artifact witnesses rather than source-only assertions.

6. **SIMPLIFY.** Look for commands, metadata, compatibility paths, state helpers, or deployment work that can be deleted while preserving the settled issue contract. Do not add generic quota/failover machinery.

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
