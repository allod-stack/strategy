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

- **Pass 1** — `claude-opus-4-8` (Claude Code runner), reasoning `medium`, reviewed `a6db27f`. 3 original-plan findings, 0 review-introduced: 1 BLOCKER (ownership-manifest version transition would strand the first post-upgrade activation — the reconciler's strict `.version==1`/per-credential-link check would treat a valid prior-version manifest as corrupt and never converge), 1 GAP (token-directory lifecycle unspecified; `realpath -e` parent check, single-segment `safe_journal_path` allowlist, and file-only rollback/recover/retire leave empty untracked directories), 1 SIMPLIFY (drop dead `rotationStrategy` metadata). Fixed in `e04ccc0`. Verified against Pi 0.84.2 docs that `registerProvider` from `session_start`, `agent_settled`, and `ctx.hasPendingMessages()`/`isIdle()` exist and support the startup and queued-work pinning claims (Focus Areas 1 and 2 hold, no findings there). Fix stability: re-verified in pass 2, with two ambiguities tightened in `9897fcd`.
- **Pass 2** — `gpt-5.6` (Pi/nullsink runner), reviewed `e04ccc0` and performed a fresh sweep. 0 original-plan findings, 2 review-introduced findings: 0 BLOCKER, 2 GAP, 0 SIMPLIFY. GAP 1: the pass-1 manifest wording said to re-derive ownership from desired links but did not preserve the strictly validated old ownership proof needed to retire exact legacy credential-level links, risking either orphaned links or weakened corrupt-state refusal. GAP 2: the pass-1 claim that no untracked directory survives conflicted with preserving operator/concurrent contents and could imply unsafe recursive cleanup. Fixed both in `9897fcd` by requiring exact old-manifest/link validation solely for legacy retirement, desired-only derivation of new ownership, and empty-only `rmdir` pruning with a populated-directory race witness. The SIMPLIFY sweep reconsidered `rotationStrategy`, legacy fallback, startup flags, provider/token command separation, state helpers, deployment scope, and quota/failover machinery; no further removable scope was found. Fix stability: not yet re-verified. Stop rule: not met; this is the first consecutive pass both with review-introduced findings outnumbering originals and with neither a BLOCKER nor an original-plan GAP.
- **Pass 3** — `claude-sonnet-5` (Pi runner), reasoning `medium`, reviewed `9897fcd`. Scoped diff verification of `9897fcd` held: reading `archetypes/modules/pi-provider-reconcile.sh` confirmed the current single-token reconciler's exact old-manifest-shape check (`.version == 1`, `credentialLinks` path/target equality) matches the plan's "old strict validation" wording, and the `rmdir`-only pruning direction has no counter-evidence in the (not-yet-implemented) named-token code paths. 1 original-plan finding, 0 review-introduced: 0 BLOCKER, 1 GAP, 0 SIMPLIFY. GAP (present since `a6db27f`, missed by passes 1-2): the plan never pinned the per-token runtime credential link naming scheme. Because both credential and token names may contain hyphens, a flat `<credential>-<token>` link name cannot be split back into its parts for per-credential token listing, forcing nested `<credential>/<token>` links — but nesting means the legacy single-file per-credential symlink and the new per-credential token directory occupy the identical leaf path (confirmed against `path=$credential_dir/$credential` in the live reconciler), so the pass-1/2 manifest-upgrade fix needed an explicit retire-before-create ordering and same-path collision refusal or an implementation could `mkdir` over the still-live legacy symlink and fail the very upgrade path those passes fixed. Fixed in `31ae5c1` by pinning nested link naming, stating the retire-before-create order and same-path collision refusal, and adding a witness item for the same-path conversion. SIMPLIFY sweep: re-examined `rotationStrategy` against the live Nexus script rather than trusting pass 1's characterization — it is read in exactly three places (`stage_registries`'s add-time consistency check, the credential-registry schema's `exact([...])`/enum check, and `print_impact`'s `rotate` case, which prints an extra remote-invalidation warning only for `"in-place"`), and all three are reachable only through the provider-scoped `add`/`rotate` commands the plan already replaces with the token model, so pass 1's drop is confirmed lossless at the code level; the plan's generic follow-up guidance ("revoke the previous remote token only after verification, when the provider supports overlap") already carries the operator-facing intent of the dropped per-credential warning, so no replacement text was added. Also checked Pi 0.84.2 `docs/models.md` ("pi intentionally does not apply built-in TTL, stale reuse, or recovery logic" for `!command` values) confirming no hidden credential-command caching undermines the `/token` command-cache-separation acceptance test; no further removable scope found among legacy fallback, startup flags, command separation, state helpers, deployment scope, or quota/failover machinery. Fix stability: not yet re-verified. Stop rule: not met — pass 3 found an original-plan GAP, so neither disjunct holds (review-introduced 0 does not outnumber original-plan 1, and pass 3 is not a "neither BLOCKER nor original-plan GAP" pass); continue to a pass 4 scoped diff verification of `31ae5c1`.

- **Pass 4** — `gpt-5.6-terra` (Pi runner), reasoning `ultra`, reviewed `31ae5c1`. The same-leaf order is enforceable in one reconciliation, but not crash-consistent as written: unlink then `mkdir` necessarily has an absent-name interval, while the current reconciler has only trap rollback and strict old-manifest validation would refuse after `SIGKILL`/reboot. 1 review-introduced BLOCKER fixed in `c7a5caa`: require a synced, bearer-free conversion journal that recovery consumes before manifest validation, exact-link-only rollback with empty-only directory removal, directory syncs, and kill/reboot witnesses. The pass-3 naming/order fix therefore remains unverified and now requires a scoped diff pass by a different model. Fresh sweep found 1 original-plan GAP fixed in `c7a5caa`: defer preference-file persistence until a pending `/token` selection actually activates, so cancel cannot leave a canceled remembered token. SIMPLIFY considered conversion-journal scope versus rename-only/atomic-dir alternatives, preference-state helpers, legacy fallback, startup flags, selector/provider split, deployment work, and quota/failover machinery; the journal is necessary for the type-changing crash window and no removable scope remains. Stop rule: not met — review-introduced findings (1) do not outnumber original-plan findings (1), and this pass has both a BLOCKER and an original-plan GAP.

This is review pass 5 (scoped diff verification of `c7a5caa`, then a fresh sweep). Use a different eligible model than `gpt-5.6-terra`, because this is a blocker-level fix. Work only in public checkouts under /home/vnprc/work/allod and read-only Pi package docs/store paths; do not inspect private checkouts. Commit fixes and prompt metadata directly to master. A push may be rejected by the environment boundary; still leave complete local commits and a clean tree. Read the full applicable allod memory files first.

## Focus Areas

The six standing lenses from `dev-plans.md` apply, plus:

1. **Startup before first request.** Verify Pi actually loads/authenticates models and extensions in an order where the generic selector plus `session_start` can resolve remembered/default/interactive choice before any request. Flag any UI/non-interactive path the proposed APIs cannot implement.

2. **Queued-work pinning.** Inspect event ordering and provider re-registration. Can `agent_settled` plus pending-message state truly keep the active request and every already queued message on the old token without delaying forever or switching one provider early?

3. **Transactional directory migration (verify pass-2 fix).** Pass 2 clarified the pass-1 directory fix: journal the directory's pre-run absence, but prune only through empty-directory `rmdir` semantics after managed files are gone. Verify implementation and recovery planning can represent the directory object without broadening path ownership, and that a populated-directory race preserves and reports unrelated contents.

4. **Crash-consistent same-leaf conversion (verify pass-4 BLOCKER fix).** A symlink-to-directory type change cannot be one filesystem-atomic rename. Verify `c7a5caa`'s durable conversion journal records only validated ownership, is synced before unlink and cleared only after all conversion/JSON/manifest commits, and is consumed before ordinary strict-manifest validation. After injected kills/reboots at every mutation, recovery must either restore the exact legacy symlink then converge, or preserve/report a foreign or populated occupant; it must never recursively delete it or strand a missing leaf (`archetypes/modules/pi-provider-reconcile.sh`, `archetypes/checks/pi-provider-lifecycle.nix`).

5. **Generated ownership and preference concurrency.** Check token links, extension installation/removal, preference locking/atomic writes, pending selection persistence only after successful activation, cancel behavior, absent secrets, corrupt state, rebuild/reboot, final empty state, and operator collisions. Demand executable generated-artifact witnesses rather than source-only assertions.

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
