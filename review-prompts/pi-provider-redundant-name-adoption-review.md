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

## Pass History

| Pass | Model | Effort | Reviewed | Findings | Fixes | Next focus |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `gpt-5.6-terra` | `high` | `42e3103`, full plan and Nexus implementation baseline | 0 BLOCKER, 2 GAP, 0 SIMPLIFY, 1 linkage QUESTION | require a mixed-sibling universal-predicate sabotage and commit-before-retry guidance | implementation verification |

The pass confirmed the direct jq equality approach, original-model comparison, translation omission, narrow scope, R3 calibration, and straight-revert rollback. It found that the refusal witness needed a sibling accepted model to reject an accidental existential predicate, and that remediation must tell the operator to edit and commit the existing catalogue before retrying rather than merely “normalize/remove” metadata. PR linkage is verified when both PRs are opened. The SIMPLIFY sweep found no abstraction or scope to remove.

## Implementation Review

| Pass | Model | Effort | Reviewed | Findings | Fixes | Stability |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `gpt-5.5` | `high` | uncommitted Nexus implementation and final plan | 1 BLOCKER, 1 GAP, 0 SIMPLIFY | detect unsupported-key array length; add non-string and empty-key refusals | superseded |
| 2 | `gpt-5.6-terra` | `high` | scoped unsupported-key and non-string-name fixes | 0 BLOCKER, 0 GAP, 0 SIMPLIFY | none | terminal |

The implementation review found that joining unsupported keys into diagnostic text made an empty-string key indistinguishable from no unsupported keys, allowing bearer access and metadata loss. The implementation now retains the key array for the decision and renders it separately, with an empty-key production-command sabotage. It also adds the missing non-string-name refusal witness. A different-model scoped pass verified both fixes, ordinary unsupported-field refusal, redundant-name normalization, safely escaped diagnostics, and the direct implementation; it found no remaining issue or simplification.

## Deliverable

Return a numbered finding list with tags, or explicitly state no findings. Name sound decisions briefly so implementation does not undo them. Do not edit files; the driver records the result and applies any justified fixes.
