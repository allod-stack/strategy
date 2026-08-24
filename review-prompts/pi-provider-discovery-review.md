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

1. **Plaintext lifetime across staging.** A new credential's bearer now survives from the prompt, through discovery, staging, and `nix eval` preflight, and only then reaches Age. Does anything in that window put it into a child environment, a shell trace, an error string, or the tmpfs staging tree, and is one paste guaranteed to be both the discovery bearer and the encrypted `default`, with no second prompt that could diverge?
2. **Single-route proof under stdin contention.** curl takes the bearer from a pipe while the hidden prompt and the removal `yes` read the operator's stdin. Can the ordering be implemented without one consuming the other, and can a PATH-shimmed client fixture prove no route but the exact `/models` path is ever requested, including on the second snapshot and the refresh path?
3. **Shape failure versus warning.** Discovery fails on a wrong-shaped allowlisted field but only warns on an absent one, so a single junk `context_length` blocks a whole catalogue. Is fail-closed right here, or should a malformed optional field be dropped under the same incomplete-metadata warning? Are the remaining limit/capability spellings grounded in a catalogue anyone has actually seen, or is the next SIMPLIFY cutting more of them?
4. **Confirmation placement in the transaction.** Refusal leaves no journal, but the plan never pins the `yes` gate relative to the impact summary, tmpfs staging, and derived-output preflight. Where does it sit, does refresh assert both repositories clean before the second snapshot, and do refusal, EOF, signal, rollback, and `recover` hold at each boundary?
5. **Witnesses for this pass's refusals.** Fixtures must execute the production path for the impact summary preceding any prompt or decryption, a stored `baseUrl` that fails the URL rule, a stored catalogue carrying non-allowlisted fields, and an oversized chunked body with no `Content-Length`. Are these reachable at the existing PATH-shim seam, or does one of them need plan text no fixture can execute?

## Pass History

| Pass | Model | Effort | Reviewed | New findings (origin) | Fixes | Stability |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `claude-opus-5` | `high` | `cf8acaa`, full plan | 0 BLOCKER, 5 GAP, 1 SIMPLIFY, all original-plan | `0e075f6`, `a648e1d`, `194284a`, `68898ef` | superseded |
| 2 | `gpt-5.6-terra` | `high` | `cf8acaa..6f15216`, full plan, Nexus command/tests, Pi schema | 0 BLOCKER, 3 GAP, 0 SIMPLIFY, all original-plan | `694104f` | terminal |

Terminal pass findings:

1. [GAP] A bearer header sent through curl is not confined to the planned request while curl still reads a user curlrc or proxy environment. The plan now requires initial `--disable`, `--noproxy '*'`, stdin-only curl config, and an executable client witness for those arguments and `/models`.
2. [GAP] An existing ciphertext can decrypt to text that was never accepted at a prompt; without validating it, it can violate the curl-config transport boundary. The plan now validates decrypted `b64token` text, resolves once, suppresses xtrace, removes inherited export, and requires malformed-default, one-paste, and xtrace leak witnesses.
3. [GAP] The removal `yes` gate had no fixed relationship to preflight or a late source-tree cleanliness check. The plan now takes the agreeing snapshots before tmpfs staging/preflight, asks only after their result is known, and checks both repositories before the second read and immediately before journaling; dirt/refusal/signal witnesses cover the no-journal boundary.

SIMPLIFY sweep considered removing direct-client options, the late clean checks, and malformed-default fallback; each closes an authenticated-boundary or source-loss hole. The remaining catalogue spelling and optional-field questions are implementation witnesses under the existing fail-closed translation contract, not grounds for a third prose pass. The Nexus implementation PR alone still closes `allod/nexus#32`; execute its named synthetic witnesses and stop this strategy review here.

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
