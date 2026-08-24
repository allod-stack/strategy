# Pi Provider Model Discovery Review Prompt

You are the bearer-boundary bouncer, HTTP failure gremlin, and crash-consistency boss this plan needs. Be ruthless: metadata reads are allowed, inference and token leaks are not.

## Your Task

Review the [Pi Provider Model Discovery plan](../dev-plans/pi-provider-discovery.md) for gaps, contradictions, unsafe bearer handling, and unnecessary machinery. Ground every finding in the current `allod/nexus` `pi-provider` command, its black-box witness, and Pi's custom-model schema. Make plan fixes as commits rather than only reporting them.

## Project Context

Allod is a self-sovereign NixOS VM stack. Nexus owns host-side provider catalogue and Age-ciphertext transactions; profiles own provider metadata; secrets own credential targeting and ciphertext.

Current facts to verify:

- `scripts/pi-provider` is Bash and already resolves both repositories, inventory classifications, credential/provider ownership, tmpfs staging, derived-output preflight, clean-tree locking, a durable exact-path journal, rollback, and `recover`.
- A credential has named ciphertexts and at most one `defaultToken`; manual add passes model JSON through and must remain open-world.
- The production package includes curl, jq, and age. Pi accepts provider-level and model-level `api`; model defaults permit omitted limits/capabilities.
- Agents may use synthetic fixtures only and cannot run private discovery, rebuild, deploy, or inference.

## Structural Conformance

Verify Tracking Issue, Goal, Scope, Risk Assessment, Interface Contracts, Agent Gates, Acceptance Tests, and Rollback Plan. Confirm R3 matches the authenticated network and transaction boundary and the Nexus implementation PR alone closes allod/nexus#32.

## Focus Areas

1. **Bearer flow.** Can prompted/decrypted plaintext reach only curl/Age stdin, remain out of every child environment/argv and persistent path, and still survive long enough for new-credential discovery followed by encryption? Does every error path redact it?
2. **Remote boundary.** Is URL validation coherent, redirect behavior fail-closed, the size/error bound real, and `/models` the only reachable route even under malformed provider data?
3. **Translation and refresh.** Do provider-level versus model-level adapters match Pi, are precedence and wrong-shape behavior deterministic, and can unchanged, changed, or removed catalogues accidentally rewrite credential state or manual schema?
4. **Removal and transaction ordering.** Are the second snapshot and explicit confirmation completed before staging/journaling? Do refusal, EOF, signal, rollback, and `recover` preserve both repositories at each boundary?
5. **Witness quality and simplification.** Do fixtures exercise the production command seam and sabotage the validator/client? Delete abstractions or compatibility paths that do not buy an issue requirement.

## Review Guidelines

- Forward momentum is king; flag real safety, implementation, and test holes—not style.
- This is pre-alpha, single-operator software. Do not add team/release machinery.
- Manual add stays open-world; discovery translation is intentionally allowlisted.
- Security boundaries are strict, ceremony is not. Never weaken a fail-closed check to make a fixture pass.
- A behavior executable in the black-box witness should be tested, not specified more elaborately.

## Severity Rubric

Use `[BLOCKER]` for likely secret exposure, inference access, unsafe removal/mutation, nonfunctional implementation, or missing human authority. Use `[GAP]` for material ambiguity, contradictory contracts, or test/rollback blind spots. Use `[SIMPLIFY]` for removable scope or abstraction. Use `[QUESTION]` only when repository evidence cannot answer it.

## Deliverable

Every pass ends with plan-file commits for real findings (or explicit no findings), then a review-prompt commit recording model, effort, reviewed commit, finding counts/origins/fixes, SIMPLIFY sweep, and next focus. Push and leave the worktree clean. Stop according to `dev-plans.md`; a blocker-level fix requires a different-model scoped verification next.
