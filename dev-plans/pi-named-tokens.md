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

`pi-credentials.json` keeps provider and target ownership per logical credential, adds exactly `tokens` and `defaultToken` to each record, and drops the now-vestigial `rotationStrategy` field: with provider-scoped `rotate` removed no code branches on it, so the record becomes exactly `providers`, `targets`, `tokens`, `defaultToken`. `tokens` is a non-empty unique list of opaque names matching `^[a-z][a-z0-9-]{0,62}$`; `defaultToken` is either one listed name or `null`. Credential/provider/target rules are otherwise unchanged. Provider IDs and token names are different namespaces.

Each token derives `secrets/pi-credentials/<credential>/<token>.age`. Secrets derives the same Nexus-plus-target active/staged recipient set for every token, emits inventory consumers per ciphertext, and projects to each target:

```nix
credentials.<credential> = {
  providers = [ ... ];
  tokens.<token>.file = <ciphertext path>;
  defaultToken = null | "<token>";
};
```

No endpoint metadata or bearer appears in that projection. Invalid/duplicate/empty token sets, invalid defaults, missing ciphertexts, unknown fields, recipient drift, and unsupported targets fail evaluation. Empty public registries remain valid and generate nothing.

`pi-provider migrate` is the only legacy-layout reader. In one recoverable transaction it changes each singleton record to `tokens = ["default"]`, `defaultToken = "default"`, drops `rotationStrategy`, and moves `<credential>.age` to `<credential>/default.age` without decrypting or requesting the bearer. It refuses mixed legacy/new state, destination collisions, or dirty checkouts. Migration is forward-only: new schema and runtime code do not retain legacy fallback.

The token directory is a first-class transaction object, not an implicit `mkdir -p` side effect. Creating `<credential>/` is a journaled step whose pre-run absence is recorded so rollback and `recover` can undo that creation, while last-token `remove` and full `retire` also attempt to prune the directory. Every prune uses empty-directory-only `rmdir` semantics after the managed files are gone: an operator or concurrent writer that populated the directory is preserved and reported, never recursively removed or treated as owned merely because the operation created the directory. Thus an empty managed token directory does not survive a completed or rolled-back operation, while unrelated contents remain safe. Journal path safety, the recover allowlist, and the recoverable-plan checks recognize both the directory object and the two-segment `secrets/pi-credentials/<credential>/<token>.age` layout, and creating a not-yet-existent token directory under the operation lock (verifying it is a real directory, not a symlink) replaces the current parent-must-already-exist assumption.

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

`retarget` re-encrypts every token to the replacement recipient set through one labeled hidden prompt per token and commits registry plus all ciphertexts atomically; no partial token set is allowed. Existing clean-tree, tmpfs staging, RFC 6750 validation, dry-run, lock, journal, rollback, and `recover` guarantees extend to every new path, token directory, and migration move per the directory-lifecycle rules above. Bearers are accepted only by hidden input and piped directly to Age: no value enters argv, environment, regular staging, logs, diagnostics, or command output. Dry-run/list output contains names, paths, targets, and impact only.

### Runtime selection and `/token`

Every named ciphertext becomes a user-owned mode-`0600` Age secret in volatile storage. Per-token credential links live under a new managed token-link root that is deliberately distinct from the legacy per-credential link path, at `<token-link-root>/<credential>/<token>`, nested to mirror the ciphertext layout: both credential and token names may contain hyphens, so a flat `<credential>-<token>` link name cannot be split back into its parts and cannot list one credential's tokens unambiguously. Choosing a root that never contains a legacy leaf means the upgrade never performs a symlink-to-directory type change at one path; the link path is an internal reconciler contract (the generated `auth.json` `!command` is emitted against the same root in the same operation), so nothing external pins the legacy location. Reconciliation creates these owned per-token credential links, generated auth commands, a safe token catalogue, and an auto-discovered extension; it removes only paths recorded in its ownership manifest and preserves/refuses operator collisions as the current lifecycle does. Path shape or coincidence with a computed target is never ownership proof by itself. The reconciler also owns the token-link root and each `<token-link-root>/<credential>/` subdirectory it creates: once a credential's last managed token link is retired (partial credential removal or full retirement), it prunes the emptied subdirectory, and the root at final empty state, with empty-directory-only `rmdir` after the managed links are gone — never recursively, and preserving and reporting any operator or concurrent contents exactly as the legacy-directory and Nexus token-directory rules do. An empty managed nested directory therefore does not survive a completed reconciliation, while unrelated contents remain safe; a leftover empty directory from an interrupted run is pruned idempotently by the next reconciliation.

The command-backed selector has two modes: resolve a credential by startup precedence, or resolve one explicit provisioned token name. It validates exactly one RFC 6750 line and writes the bearer only to Pi's stdout credential pipe. Safe credential/token/catalogue identifiers may appear in its argv; the bearer may not. The startup command in `auth.json` keeps managed models discoverable, while the extension selects and re-registers provider authentication before the first request.

Startup precedence per credential is: valid remembered name, valid deployment default, sole delivered token, then interactive picker. Invalid remembered state is reported and ignored. If multiple choices remain without UI, selection fails clearly. Remembered choices live in `${XDG_STATE_HOME:-$HOME/.local/state}/allod/pi-token-selections.json`, containing only credential-to-token names. Creation/update uses a user-only directory, mode `0600`, locking, merge-on-write, validation, and atomic replacement so concurrent Pi processes cannot lose unrelated credential choices.

The generated extension exposes only `/token`, `/token <name>`, and `/token cancel`, scoped to the current model provider's credential. Status/picker identifies token names and active/default/remembered/pending roles without normally showing the technical credential ID. It refuses unmanaged providers and unknown names. Selection applies to every provider sharing the credential by supported `registerProvider()` re-registration with a token-specific command-backed key; it never accesses private Pi runtime objects or injects an Authorization header.

When idle, a selection activates immediately and is then remembered. During active or queued work, one pending choice is retained: latest wins, cancel clears it, and activation waits for `agent_settled` with no pending messages. A pending choice is process-local and is not written to the preference file until activation succeeds, so cancel leaves the remembered choice unchanged. Thus the active request and already queued messages retain the old command/key. Selection state is per process after activation; another process changing remembered state does not mutate it. No code infers failure, quota, priority, or behavior from HTTP responses or token names.

Reconciliation is a whole-operation no-op when any desired Age token is absent. The runtime ownership manifest gains a new version for per-token links and a bounded transitional `legacyCredentialLinks` map. The reconciler accepts a prior-version manifest only after its exact old shape, provider list, credential-link paths, and derived targets pass the old strict validation, and before mutation it requires every recorded legacy link still to be a symlink to that exact target. Missing, replaced, malformed, or path-divergent legacy state is corrupt and causes the usual whole-operation refusal. A valid old manifest supplies the only ownership proof for legacy retirement: the first new-version manifest copies its complete, strictly validated `credentialLinks` map into `legacyCredentialLinks`, including entries for credentials removed in the same activation. New per-token links and ordinary new-version ownership are derived solely from the current desired token set. New-version validation is equally strict about the transitional map's bounded shape and derived path/target values; it never augments that map by scanning the filesystem.

Because the new per-token links live under a root distinct from the legacy per-credential leaf, the upgrade is not a symlink-to-directory type change at one path and has no absent-name interval. The reconciler creates and verifies every new nested link first (an additive, idempotent step snapshotted for rollback), then publishes `auth.json`, `models.json`, and the new-version manifest carrying the transitional records in that order as separate atomic renames. Only after those records are durable does it process exactly their named legacy paths: an exact recorded symlink is removed, an absent path counts as already retired after a crash, and any other occupant is preserved and causes refusal with its transitional record retained for explicit repair. After all recorded paths are absent, a second atomic manifest replacement clears `legacyCredentialLinks`, and the emptied legacy directory is pruned with empty-only `rmdir`. New-version activations resume this bounded retirement before ordinary cleanup, independent of the current desired credential set, so a credential retired during the version bump remains discoverable without any directory-wide sweep.

These files are not one filesystem transaction: the ordering must ensure that any published auth entry already has its new link, and rollback/retry must handle a kill between every link creation, file rename, and recorded legacy unlink. A crash or reboot before the first new-version manifest leaves the prior-version manifest and all legacy links intact; additive new links are recreated idempotently on retry. After that manifest lands, the new layout is usable and the transitional map is durable. A crash after a recorded unlink leaves that path absent and therefore safely clearable on retry; a crash after the final manifest replacement leaves no owned legacy path to revisit. No state ever infers ownership from an unrecorded exact-target symlink, so an operator-created legacy-root entry survives even when its name and target coincide with the old convention. No durable conversion journal, pre-unlink sync, restore-exact-symlink recovery, same-path collision handling, directory-wide legacy sweep, or desired-set-keyed retirement is needed. A legacy or new path occupied by an unowned or conflicting object is preserved and reported, never recursively removed. Rebuild, reboot, idempotence, partial token/provider removal, final empty state, extension/link collision, genuinely corrupt (not merely prior-version) preference/ownership state, and rollback preserve unrelated Pi configuration. Generated auth, catalogue, extension, activation, runtime files, and closures contain no bearer.

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

The secrets witness covers multiple tokens/default-null, exact projection and recipients, missing/duplicate/invalid tokens, bad defaults, missing ciphertexts, absence of any `rotationStrategy` field, and schema sabotage. The Nexus witness covers migration without bearer input; independent add/rotate/remove/default/list; no activation on add; all-token retarget; shared-provider impact; last/default refusal; provider retirement; token-directory creation and empty-only pruning across migrate, rollback/recover, last-token remove, and retire, including a populated-directory race proving unrelated contents are preserved rather than recursively removed; dry-run; interrupted migration/mutation and recovery; and child argv/environment/files/output leak scans using runtime-generated synthetic bearers.

The archetypes witness inspects generated activation and extension artifacts and uses synthetic providers plus a local server. It proves both tokens authenticate only with their expected bearer; startup precedence, stale state, interactive ambiguity, and non-interactive refusal; `/token` immediate switching; active-plus-queued deferral, latest-wins, cancel without changing the remembered choice, and persistence only after activation; all shared providers switch together; separate credentials do not; simultaneous processes stay independently pinned; selection-file mode/concurrency; Pi command-cache separation; a valid prior-version ownership manifest upgrades to distinct-root per-token links, carries the complete validated old link map into bounded transitional ownership, idempotently retires only those recorded legacy links, clears the transitional map, and prunes the emptied directory; a credential fully retired in the same operation as the version bump still has its recorded legacy link removed even though it is absent from every subsequent desired-credential set; partial token removal and full credential retirement prune the emptied `<token-link-root>/<credential>/` subdirectory and, at final empty state, the token-link root with empty-only `rmdir`, while a populated nested directory is preserved and reported rather than recursively removed; an unrecorded operator-created symlink in the legacy root survives even when its valid name and exact computed target mimic the old convention; malformed transitional records and malformed, missing, replaced, or path-divergent prior-version links refuse safely. Kill/reboot injection after every link creation, each separate auth/models/initial-manifest rename, every recorded legacy unlink, and the final manifest rename converges idempotently with no conversion journal, never publishes auth that points at an absent new link, strands no owned link, never leaves both layouts absent, and preserves/reports a foreign occupant of any legacy or new path; new-link collisions refuse without mutation. The witness also covers reconciliation/removal/absence/collision/reboot/GC behavior and proves no bearer enters persistent home, diagnostics, generated files, process argv/environment, observations, or store closures. Event/command tests must execute the generated extension through Pi's supported extension surface rather than testing a rewritten copy. Each witness includes a deliberate sabotage that its comparator rejects.

The deploy canary binds the compatible secrets/archetypes graph, exercises empty public data and a synthetic non-empty composition, checks the resulting activation/secret projections, and independently sabotages each relevant follows edge.

## Rollback Plan

Before merge, amend or close the affected PR. After individual public merges but before deploy composition, keep deploy pinned to the prior compatible graph or revert that slice and its dependents. Empty public registries cause no live credential change.

If `pi-provider` is interrupted, run `pi-provider recover` before touching either data checkout. Before private migration, rollback by retaining old pins and layout. After migration, do not downgrade one side: restore the pre-migration profiles/secrets commits together with old deploy pins, or fix forward with the new command and rebuild. A runtime activation failure rolls back its ordinary files; an interrupted legacy-link retirement is completed idempotently by the next activation, because the new per-token links already live at a distinct root and the bounded transitional manifest still records every owned legacy path and target. Repair missing/colliding inputs and rebuild. Remote token revocation is human-owned and occurs only after fresh and live-switch verification in the private rollout.
