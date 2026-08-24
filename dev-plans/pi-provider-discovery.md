# Pi Provider Model Discovery

This change lets a Nexus operator create and refresh an OpenAI-compatible Pi provider from its authenticated model catalogue, while keeping inference and every deployment action outside the command.

## Tracking Issue

[allod/nexus#32](https://forge.anarch.diy/allod/nexus/issues/32). The strategy plan PR uses `Refs allod/nexus#32`; the Nexus implementation PR uses `Closes allod/nexus#32`.

## Goal

An operator can discover every advertised model during provider creation and safely synchronize that catalogue later without hand-writing model JSON or exposing a bearer.

## Scope

In scope:

- `allod/nexus/scripts/pi-provider`: `discover-add` and `discover-refresh`, authenticated metadata reads, translation, comparison, removal confirmation, and reuse of the existing transaction/recovery path.
- `allod/nexus/tests/pi-provider.sh`: synthetic HTTP-client, Age, lifecycle, interruption, and leak witnesses that execute the production command.
- `allod/nexus/docs/credentials.md` and command help: cold-operator workflow, warnings, recovery, dry-run network behavior, and command boundaries.

Out of scope:

- Real providers, tokens, identities, inventory, ciphertext, private integration, commits, pushes, rebuilds, deployment, inference, model preference, health monitoring, or provider-side token lifecycle.
- Changing manual `add`, its open model schema, named-token operations, or `retarget`.
- Waiting for or implementing the proposed Go rewrite in allod/nexus#28.

## Risk Assessment

Residual risk: **R3 High**.

Why:

- The command decrypts or prompts for a bearer and performs authenticated network I/O before a two-repository mutation.
- A false removal or secret leak is consequential, but bounded synthetic fixtures exercise the HTTP boundary, double-snapshot removal gate, transaction rollback/recovery, and child-process/persistent-file leak surfaces.
- Rollback before commit is the existing journal restore; after commit it is a normal source revert and rerun.

Human scrutiny should focus first on bearer flow into Age/curl stdin, the fixed `/models` route, removal confirmation ordering, and whether discovery mutations truly stay inside the existing journal.

## Interface Contracts

The CLI adds:

```text
pi-provider [--dry-run] discover-add <provider> --url <https-base> [--credential <credential>] [--target <dev-vm>[,<dev-vm>...]] [--api <adapter>] [--prompt-token]
pi-provider [--dry-run] discover-refresh <provider> [--prompt-token]
```

`discover-add` defaults credential ID to provider ID and, when no target is supplied, selects every inventory machine with `type == "dev"` and `runtime == "libvirt"`. Explicit repeated/comma-separated targets retain current validation. An existing credential must have exactly the requested/default target set; the operator uses explicit `--target` or `retarget` rather than discovery changing it. Manual `add` remains unchanged.

The base URL is a control-character-free HTTPS URL without userinfo, query, or fragment. `discover-refresh` applies that same rule to the provider's stored `baseUrl`, which open-world manual `add` never constrained beyond the `https://` prefix, and refuses before decrypting or prompting when it fails. Discovery performs one authenticated `GET <base-without-trailing-slash>/models`, never follows redirects, never invokes another route, and applies connection/overall timeouts. The 1 MiB body bound is enforced on bytes actually received rather than on an advertised `Content-Length`, so a chunked or mislabeled response cannot exceed it. Non-2xx responses fail before mutation; output may include status and at most 512 normalized characters of provider error text after exact bearer redaction.

Before any prompt or decryption, both commands print the provider, the credential and whether it is new, the resolved target set, and the exact request URL, so the all-dev-VM blast radius is visible before the operator hands over a bearer. Discovery reads a bearer even when the credential already exists; the current promise that an existing credential prompts for nothing survives for manual `add` only, and help and docs must say so.

Bearer resolution is: force a hidden prompt with `--prompt-token`; otherwise, for an existing credential, decrypt only its non-null `defaultToken` ciphertext with the Nexus hypervisor Age identity (`AGE_IDENTITY`, default `$HOME/.ssh/host`, as in `rotate-token`) into a shell variable, never a file; on null default, missing ciphertext, missing or non-recipient identity, decryption failure, or a decrypted value that is empty or not an RFC 6750 `b64token`, explain and use the same hidden prompt. A fallback bearer is request-only and changes no token state. A new credential prompts once, uses that value for discovery, and encrypts it as `default` only after discovery and all staging preflights pass. Resolve a bearer once per command: retain a new credential's one pasted value only through its fetch, preflight, and Age encryption; retain a refresh bearer only through its first and, if required, second snapshot; otherwise clear it as soon as its request completes. The dry-run new-credential value clears after its fetch because no Age write follows. Before resolving it, explicitly disable xtrace and do not restore it while the variable exists; explicitly remove any inherited export attribute. Plaintext is held only in that non-exported shell variable and enters Age/curl through builtin-printf pipes—not argv, environment, logs, diagnostics, shell tracing, regular files, or journals. Dry-run still performs this authenticated read and may decrypt/prompt, but never encrypts or mutates.

The fixed curl invocation starts with `--disable` (before every other curl option, so no user curlrc is read), uses `--noproxy '*'`, and receives the bearer header only as curl-config text on stdin while its response body goes to a tmpfs file. Thus neither user curl configuration nor proxy environment can add a request or redirect bearer-bearing traffic; the fixture may replace the client but must assert those fixed arguments as well as the exact `/models` route.

A response must be an object with a non-empty `data` array of model objects carrying unique, non-empty, control-character-free string IDs. Translation sorts by ID and copies only this allowlist:

- `context_window`, then `context_length`, then `max_model_len` supplies positive-integer `contextWindow`.
- `max_output_tokens`, then `max_completion_tokens` supplies positive-integer `maxTokens`.
- `input_modalities`, `architecture.input_modalities`, `supports_image_input`, or `capabilities.image_input` may add `image` to the always-present `text` input.
- `reasoning`, `supports_reasoning`, `capabilities.reasoning`, then `capabilities.thinking` may set `reasoning`; there is no name-based inference.
- `supported_endpoints` may identify Chat Completions or Responses. Provider-level `api` is never discovered: it stays `openai-completions` unless `--api` names another adapter. A model that explicitly supports Responses and not Chat Completions carries model-level `api: openai-responses`, which Pi honors over the provider value; every other model carries none.

Absent limits/capabilities are omitted where Pi has a default, not invented. Wrong allowlisted shapes or non-positive selected limits fail. Stable warnings report incomplete metadata, unknown/unhealthy route or reliability, unverified tool use, deprecation, and `maxTokens <= 1024`; operational metadata is never persisted. Manual `--models-file` remains pass-through.

`discover-refresh` resolves the provider's current credential and URL, fetches/normalizes a catalogue, and prints sorted `Added`, `Removed`, and `Changed` ID groups. It refuses a provider whose stored models carry fields outside the translation allowlist: refresh rewrites the whole catalogue and cannot regenerate hand-authored JSON, so such a provider stays on manual `add` and `--models-file`. It changes only that provider's translated `models`, including their model-level `api`; the provider-level `api`, credential targets, named tokens, default selection, URL, and unrelated providers remain byte-semantically unchanged. Equal normalized catalogues return without staging, preserving a no-diff result.

If removals appear, refresh fetches a second complete snapshot and compares normalized persistence-affecting data. Empty/invalid or differing snapshots abort before mutation. For a live removal, both repositories must still be clean immediately before the second read; after the agreeing read clears the bearer, the command stages and preflights the exact update in tmpfs, prints the removal diff, and accepts only an explicit `yes`. It reasserts that both repositories are clean immediately after that confirmation and immediately before creating the journal. Refusal, EOF, a signal before the journal, or either cleanliness refusal leaves no journal or repository change; a signal after journal creation uses the existing recovery path. Dry-run performs both reads and reports that confirmation would be required without prompting. New/changed metadata needs no confirmation.

## Agent Gates

- The agent uses only generated synthetic tokens, URLs, identities, inventories, and responses. It cannot test a private provider, run host deployment/rebuild commands, commit private data, or verify inference.
- After public merge, the human performs private discovery, reviews and commits the two data repositories, updates deploy pins, rebuilds affected VMs, and verifies a fresh real request. Those steps are not evidence for this public PR and block any provider-side retirement decision.

## Acceptance Tests

```sh
cd <nexus-worktree>
bash -n scripts/pi-provider tests/pi-provider.sh
shellcheck -x --source-path=SCRIPTDIR scripts/pi-provider tests/pi-provider.sh
bash tests/pi-provider.sh
nix build .#checks.x86_64-linux.provisioning-contract
```

The black-box witness retains the existing lifecycle suite and adds rich/minimal translation; default/all and explicit targets; model-level adapter overrides and capability translation; warning-only metadata; malformed, empty, duplicate, non-2xx, and bounded-error responses; an oversized chunked body that advertises no `Content-Length`; a stored `baseUrl` that fails the base-URL rule, refused before any bearer read; new/shared credentials; encrypted-default reads, malformed-default fallback, missing-identity fallback, and forced prompts; one new-credential paste reused for its request and encrypted default; the impact summary preceding every prompt or decryption; xtrace/argv/environment/output/file leak scans; stable add/change/remove groups and ordering; a hand-authored stored catalogue refused before any bearer read; agreeing/disagreeing removal snapshots; injected repository dirt before the second snapshot and before journal creation; yes/refusal/EOF/signal behavior; no-diff refresh; rollback/recover at refresh mutation boundaries; and a sabotaged client that requires initial `--disable`, `--noproxy '*'`, and no route except the exact `/models` endpoint. Fixtures replace only HTTP/Age inputs at existing executable seams and run the production generator/transaction path.

## Rollback Plan

Before merge, amend or close the implementation PR. An ordinary command failure restores its journaled pre-run paths; after an untrapped interruption at mutation time, run `pi-provider recover` before touching either checkout. Discovery, snapshot disagreement, refusal, EOF, and pre-mutation interruption require no recovery because no journal exists. After a public merge, revert the Nexus commit; after a private data commit, restore the prior profiles/secrets commits together and rebuild. Provider-side tokens remain human-owned and are never revoked by rollback.
