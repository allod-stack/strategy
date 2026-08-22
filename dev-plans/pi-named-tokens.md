# Managed Pi Named Tokens

## Tracking Issue

[allod/strategy#36](https://forge.anarch.diy/allod/strategy/issues/36). The prerequisite `secrets`, `nexus`, and `archetypes` PRs use `Refs allod/strategy#36`; the final public `deploy` composition PR uses `Closes allod/strategy#36`. Private rollout is separate and carries no closing-keyword responsibility.

## Goal

A logical Pi credential can contain multiple encrypted named bearer tokens. Pi chooses one predictably at startup and can switch it safely with `/token`, while provider/model selection remains in `/model` and no bearer escapes the existing secret pipe.

## Scope

In scope:

- `allod/secrets`: named-token/default schema, per-token ciphertext paths, recipients, inventory, and per-VM projections.
- `allod/nexus`: migration and complete named-token lifecycle in `pi-provider`.
- `allod/archetypes`: volatile delivery, selector helper, persistent non-secret preference, generated Pi extension, reconciliation, and generated-behavior checks.
- `allod/deploy`: compatible pins and a composition canary for the changed projections.
- Public documentation for those interfaces.

Out of scope:

- Changes to the profiles provider catalogue; providers remain endpoint/API/model records selected by `/model`.
- Live endpoints, provider/model IDs, targets, recipient keys, bearer values, or private selections in public history.
- Private registry migration, rebuild, or live request; provider-side issuance/revocation; automatic quota/error failover; startup token flags; microVM delivery; and isolation from agents sharing Pi's Unix account.

Ordered slices are `secrets`, `nexus`, `archetypes`, then `deploy`. Nexus may be implemented against fixtures before the secrets PR merges, but deploy must pin one compatible graph.

## Risk Assessment

Residual risk is **R3 High**: this crosses four repositories, changes ciphertext layout and Age projection, installs a Pi extension, and performs live authentication changes. Synthetic tests can bound public behavior but cannot attest to a private token or service.

| Slice | Risk | Main scrutiny |
|---|---:|---|
| Secrets contract | R3 | Exact schema, per-token recipients/paths, no secret data in projections |
| Nexus lifecycle/migration | R3 | Recoverable multi-file moves, prompt-only plaintext, no accidental activation/removal |
| Archetypes runtime | R3 | startup ordering, queued-work pinning, concurrent state writes, owned removal, leak boundary |
| Deploy composition | R2 | compatible follows graph and empty/public generated behavior |

## Interface Contracts

### Credential data and migration

`pi-credentials.json` keeps provider and target ownership per logical credential and adds exactly `tokens` and `defaultToken` to each record. `tokens` is a non-empty unique list of opaque names matching `^[a-z][a-z0-9-]{0,62}$`; `defaultToken` is either one listed name or `null`. Credential/provider/target rules remain unchanged. Provider IDs and token names are different namespaces.

Each token derives `secrets/pi-credentials/<credential>/<token>.age`. Secrets derives the same Nexus-plus-target active/staged recipient set for every token, emits inventory consumers per ciphertext, and projects to each target:

```nix
credentials.<credential> = {
  providers = [ ... ];
  tokens.<token>.file = <ciphertext path>;
  defaultToken = null | "<token>";
};
```

No endpoint metadata or bearer appears in that projection. Invalid/duplicate/empty token sets, invalid defaults, missing ciphertexts, unknown fields, recipient drift, and unsupported targets fail evaluation. Empty public registries remain valid and generate nothing.

`pi-provider migrate` is the only legacy-layout reader. In one recoverable transaction it changes each singleton record to `tokens = ["default"]`, `defaultToken = "default"`, and moves `<credential>.age` to `<credential>/default.age` without decrypting or requesting the bearer. It refuses mixed legacy/new state, destination collisions, or dirty checkouts. Migration is forward-only: new schema and runtime code do not retain legacy fallback.

### Nexus CLI and lifecycle

The existing provider operations remain except that provider-scoped `rotate` is replaced by the unambiguous token operation:

```text
pi-provider [--dry-run] token list <credential>
pi-provider [--dry-run] token add <credential> <name>
pi-provider [--dry-run] token rotate <credential> <name>
pi-provider [--dry-run] token remove <credential> <name>
pi-provider [--dry-run] token default <credential> <name|none>
pi-provider [--dry-run] migrate
```

Adding a new provider with a new credential creates token `default` and makes it the deployment default. Adding a provider to an existing credential does not prompt or alter tokens/default. `token add` prompts once and never activates or changes the default. `token rotate` replaces only that ciphertext. `token remove` refuses the last token of a credential that still serves providers and refuses the current default until `token default` changes or clears it. `token default` and `token list` never read bearer input. Retirement of a credential's last provider removes its record and all named ciphertexts.

`retarget` re-encrypts every token to the replacement recipient set through one labeled hidden prompt per token and commits registry plus all ciphertexts atomically; no partial token set is allowed. Existing clean-tree, tmpfs staging, RFC 6750 validation, dry-run, lock, journal, rollback, and `recover` guarantees extend to every new path and migration move. Bearers are accepted only by hidden input and piped directly to Age: no value enters argv, environment, regular staging, logs, diagnostics, or command output. Dry-run/list output contains names, paths, targets, and impact only.

### Runtime selection and `/token`

Every named ciphertext becomes a user-owned mode-`0600` Age secret in volatile storage. Reconciliation creates owned per-token credential links, generated auth commands, a safe token catalogue, and an auto-discovered extension; it removes only paths recorded in its ownership manifest and preserves/refuses operator collisions as the current lifecycle does.

The command-backed selector has two modes: resolve a credential by startup precedence, or resolve one explicit provisioned token name. It validates exactly one RFC 6750 line and writes the bearer only to Pi's stdout credential pipe. Safe credential/token/catalogue identifiers may appear in its argv; the bearer may not. The startup command in `auth.json` keeps managed models discoverable, while the extension selects and re-registers provider authentication before the first request.

Startup precedence per credential is: valid remembered name, valid deployment default, sole delivered token, then interactive picker. Invalid remembered state is reported and ignored. If multiple choices remain without UI, selection fails clearly. Remembered choices live in `${XDG_STATE_HOME:-$HOME/.local/state}/allod/pi-token-selections.json`, containing only credential-to-token names. Creation/update uses a user-only directory, mode `0600`, locking, merge-on-write, validation, and atomic replacement so concurrent Pi processes cannot lose unrelated credential choices.

The generated extension exposes only `/token`, `/token <name>`, and `/token cancel`, scoped to the current model provider's credential. Status/picker identifies token names and active/default/remembered/pending roles without normally showing the technical credential ID. It refuses unmanaged providers and unknown names. Selection applies to every provider sharing the credential by supported `registerProvider()` re-registration with a token-specific command-backed key; it never accesses private Pi runtime objects or injects an Authorization header.

When idle, a selection activates immediately and is remembered. During active or queued work, one pending choice is retained: latest wins, cancel clears it, and activation waits for `agent_settled` with no pending messages. Thus the active request and already queued messages retain the old command/key. Selection state is per process after activation; another process changing remembered state does not mutate it. No code infers failure, quota, priority, or behavior from HTTP responses or token names.

Reconciliation is a whole-operation no-op when any desired Age token is absent. Rebuild, reboot, idempotence, partial token/provider removal, final empty state, extension/link collision, corrupt preference/ownership state, and rollback preserve unrelated Pi configuration. Generated auth, catalogue, extension, activation, runtime files, and closures contain no bearer.

## Agent Gates

- Public implementation starts only after this plan passes review and the owner says `go`.
- Public agents use synthetic names/tokens only and do not inspect, modify, decrypt, commit, migrate, rebuild, or test private provider data.
- The owner prepares a separate private rollout plan after all public PRs merge. It owns running migration, private commits/pins, rebuild, fresh and live-switch requests, and provider-side revocation.
- Before private migration, preserve a coordinated old-code/old-data revision. After migration, rollback is roll-forward repair or coordinated restoration of both; never run legacy code against the token-directory layout.
- Missing secrets and microVM targets fail closed; no fallback may copy plaintext into persistent storage.

## Acceptance Tests

```sh
cd <secrets-worktree> && nix build .#checks.x86_64-linux.credential-inventory .#checks.x86_64-linux.pi-credential-registry
cd <nexus-worktree> && nix build .#checks.x86_64-linux.provisioning-contract
cd <archetypes-worktree> && nix build .#checks.x86_64-linux.pi-provider-lifecycle
cd <deploy-worktree> && nix build .#checks.x86_64-linux.composed-layer
```

The secrets witness covers multiple tokens/default-null, exact projection and recipients, missing/duplicate/invalid tokens, bad defaults, missing ciphertexts, and schema sabotage. The Nexus witness covers migration without bearer input; independent add/rotate/remove/default/list; no activation on add; all-token retarget; shared-provider impact; last/default refusal; provider retirement; dry-run; interrupted migration/mutation and recovery; and child argv/environment/files/output leak scans using runtime-generated synthetic bearers.

The archetypes witness inspects generated activation and extension artifacts and uses synthetic providers plus a local server. It proves both tokens authenticate only with their expected bearer; startup precedence, stale state, interactive ambiguity, and non-interactive refusal; `/token` immediate switching; active-plus-queued deferral, latest-wins, and cancel; all shared providers switch together; separate credentials do not; simultaneous processes stay independently pinned; selection-file mode/concurrency; Pi command-cache separation; reconciliation/removal/absence/collision/reboot/GC behavior; and no bearer in persistent home, diagnostics, generated files, process argv/environment, observations, or store closures. Event/command tests must execute the generated extension through Pi's supported extension surface rather than testing a rewritten copy. Each witness includes a deliberate sabotage that its comparator rejects.

The deploy canary binds the compatible secrets/archetypes graph, exercises empty public data and a synthetic non-empty composition, checks the resulting activation/secret projections, and independently sabotages each relevant follows edge.

## Rollback Plan

Before merge, amend or close the affected PR. After individual public merges but before deploy composition, keep deploy pinned to the prior compatible graph or revert that slice and its dependents. Empty public registries cause no live credential change.

If `pi-provider` is interrupted, run `pi-provider recover` before touching either data checkout. Before private migration, rollback by retaining old pins and layout. After migration, do not downgrade one side: restore the pre-migration profiles/secrets commits together with old deploy pins, or fix forward with the new command and rebuild. Runtime activation failure leaves the previous owned Pi files intact; repair missing/colliding inputs and rebuild. Remote token revocation is human-owned and occurs only after fresh and live-switch verification in the private rollout.
