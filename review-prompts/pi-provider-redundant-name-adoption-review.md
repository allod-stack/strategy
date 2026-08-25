# Pi Provider Redundant-Name Adoption Review

You are the metadata-boundary bouncer: exact equality gets in, hand-authored meaning stays out. Review this narrowly and do not hold back.

## Your Task

Review the [redundant-name adoption plan](../dev-plans/pi-provider-redundant-name-adoption.md) against current `allod/nexus` `scripts/pi-provider`, `tests/pi-provider.sh`, and `docs/credentials.md`. Find only implementation-blocking, security-boundary, witness, rollback, or unnecessary-scope defects.

## Project Context

Allod Nexus owns a Bash transaction that authenticates to one fixed `/models` route and replaces only discovery-compatible model fields. `discover-refresh` currently rejects every stored `name` before bearer access. Manual `add` refuses existing providers, so it is not valid remediation. The requested exception is only `name == id`; generated synthetic fixtures must prove both acceptance and refusal.

## Structural Conformance

Verify Tracking Issue, Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Confirm the strategy PR references and the Nexus PR closes allod/nexus#34.

## Focus Areas

1. **Exact accepted set.** Does the contract accept absent names and exact string equality only, across every model, while refusing custom/wrong-shaped names and every unrelated field before bearer access?
2. **Normalization witness.** Will comparison against the original redundant field force a real refresh rather than an unchanged no-op, and does the production translation necessarily omit `name`?
3. **Executable remediation.** Does every refusal point to an action that can update an existing provider without falsely recommending create-only `add`?
4. **Scope and rollback.** Can implementation remain a small validation/docs/test change without touching HTTP, credentials, transactions, or manual-add schema?

## Review Guidelines

- Forward momentum is king; report real defects, not style.
- Security matters, ceremony does not.
- Prefer the existing direct jq validation over a new abstraction.
- A property executable in the black-box witness belongs in that witness, not more prose.
- Run an explicit SIMPLIFY sweep.

## Severity Rubric

Use `[BLOCKER]` for likely bearer-boundary violation, accepted metadata loss, nonfunctional implementation, or missing human authority. Use `[GAP]` for material ambiguity, contradictory contracts, stale remediation, or witness/rollback blind spots. Use `[SIMPLIFY]` for removable scope or abstraction. Use `[QUESTION]` only when repository evidence cannot answer it.

## Deliverable

Return a numbered finding list with tags, or explicitly state no findings. Name sound decisions briefly so implementation does not undo them. Do not edit files; the driver records the result and applies any justified fixes.
