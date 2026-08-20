# Managed Pi Provider Lifecycle

This adds a Nexus-only command and public reconciliation mechanism for adding, rotating, and retiring custom Pi providers. It reuses Pi's command-backed API-key support and Age delivery instead of inventing a second credential path. Implementation starts only after the owner approves this revised plan.

## Tracking Issue

allod/strategy#34. Public implementation PRs use `Refs allod/strategy#34`; none closes it. A separate private rollout plan owns migration and live verification, after which the owner closes the issue.

## Goal

The owner can add an HTTPS OpenAI-compatible Pi provider, rotate its bearer credential, or retire it from Nexus without placing plaintext in Git, persistent files, argv, the environment, or command output, and a rebuild reconciles both additions and removals on each target dev VM.

## Scope

In scope:

- `allod/profiles`: an empty public provider catalog, schema validation, and per-provider endpoint/protocol/model data.
- `allod/secrets`: an empty public credential registry, derived ciphertext paths, recipient/inventory entries, and per-VM provider-to-credential projection.
- `allod/inventory`: declare the profiles checkout on Nexus so the lifecycle command works from a rebuilt host workspace.
- `allod/archetypes`: libvirt Age delivery, Pi's command-backed auth entries, atomic managed-entry reconciliation, ownership state, and stale credential-link cleanup.
- `allod/nexus`: `pi-provider add`, `retarget`, `rotate`, `retire`, `recover`, and `--dry-run`, with crash recovery and fixture tests.
- `allod/deploy`: compatible public pins and a synthetic composition canary.

Out of scope:

- Live endpoints, provider/model identifiers, bearer values, target selections, or recipient keys in public history.
- Provider-side token issuance or revocation; those APIs are service-specific and remain human operations.
- OAuth, arbitrary headers, non-bearer authentication, HTTP endpoints, automatic discovery, privacy/service VMs, or microVM delivery.
- Automatically migrating or deleting the current private integration. Its one-shot cutover belongs to the private rollout plan.
- Hiding a runtime credential from agents on its target VM. Pi and agents share the same Unix account, so either can invoke the helper or read the user-owned runtime file; a broker would be a separate security design.
- Installing or removing integration-specific Pi extensions.

Ordered public slices:

1. `profiles` exports the provider catalog contract.
2. `secrets` exports the credential/target contract and recipient derivation.
3. `inventory` gives Nexus the profiles checkout the command requires and validates the host workspace from raw machine data rather than the guest-only JSON projection.
4. `archetypes` joins both contracts and reconciles generated state.
5. `nexus` safely edits the two data registries and ciphertext.
6. `deploy` pins the compatible revisions and binds every shared input to one revision.

## Risk Assessment

Residual risk: R3 High. The work crosses six repositories, accepts authentication material, changes Age recipients and Home Manager activation, and removes persistent configuration during retirement. Synthetic checks can prove the public lifecycle, including crash recovery and generated artifacts, but cannot prove any private endpoint or provider-side revocation. Human review should focus on source-of-truth separation, shared-credential impact, removal ownership, secret transport, and the stage/activate/retire order.

| Slice | Risk | Main scrutiny |
|---|---|---|
| Profiles/secrets contracts | R3 | No duplicated target fact; complete recipient and reference validation |
| Inventory workspace | R2 | Every Nexus hypervisor receives required data checkouts; the public fixture adds only profiles |
| Archetypes consumer | R3 | Only owned entries removed; empty desired state still cleans stale state |
| Nexus command | R3 | No plaintext boundary breach; exact crash recovery across two repos |
| Deploy composition | R3 | One profiles/secrets/inventory revision graph and absent-resource behavior |

## Interface Contracts

### Declarative data and ownership

`pi-providers.json` lives in profiles and is keyed by provider ID matching `^[a-z0-9][a-z0-9-]*$`. Each provider has an HTTPS `baseUrl` with no userinfo, query, fragment, whitespace, or trailing slash; a default `api` of `openai-completions` or `openai-responses`; and a non-empty unique model array. Models contain `id` and may contain `name`, `api`, `reasoning`, and `maxTokens`; unknown fields fail validation.

`pi-credentials.json` lives in secrets and is keyed by credential ID using the same ID grammar. Each entry contains non-empty unique `targets`, non-empty unique `providers`, and `rotationStrategy = "overlap" | "in-place"`. The ciphertext path is derived as `secrets/pi-credentials/<credential>.age`; it is never repeated in the registry. Provider-to-credential and target selection live only here. One credential may serve multiple providers, but each provider appears in exactly one credential record globally; this makes provider-selected rotation and retirement unambiguous.

The secrets flake derives credential inventory consumers, `secrets.nix` recipients, and each dev identity's credential/provider projection from that registry. Recipients are the Nexus active/staged keys plus every target VM active/staged key. Unknown providers/targets, non-libvirt or non-dev targets, missing ciphertext, duplicate provider references, unsupported fields, and recipient drift fail evaluation. Empty public registries generate no Pi credential or provider artifacts.

Secrets exports its consumed inventory source. Archetypes exports the profiles, secrets, inventory, and secrets-consumed-inventory sources it composed and extends `composedLayerCheck` to accept all three deploy-pinned inputs. Deploy binds `archetypes.inputs.{profiles,secrets,inventory}` and `secrets.inputs.inventory` to its root inputs. The canary requires all four observed source paths to equal the corresponding root input and independently sabotages each follows redirect.

Inventory validates repository aliases and duplicate checkout paths against raw `machines`, including hypervisors instead of only guest `vm-specs.json`. Every hypervisor that composes Nexus host tooling must include the `profiles`, `secrets`, and `inventory` aliases. The public Nexus fixture remains exactly `nexus`, `inventory`, `secrets`, and `profiles`; a mutation witness proves this slice added only `profiles` and that removing any required host alias fails.

### Nexus CLI

```text
pi-provider [--dry-run] add <provider> --credential <credential> --url <https-url> --target <dev-vm>... (--model <id>[,<id>...] | --models-file <path>) [--api <adapter>] [--rotation-strategy <overlap|in-place>]
pi-provider [--dry-run] retarget <credential> --target <dev-vm>...
pi-provider [--dry-run] rotate <provider>
pi-provider [--dry-run] retire <provider>
pi-provider recover
```

`add` refuses an existing provider. A new credential triggers hidden token input and ciphertext creation. An existing credential may be shared only when its target set is unchanged; otherwise the command refuses before secret input and names `retarget`. Repeated targets/models accumulate. `--models-file` is mutually exclusive with `--model` and contains only the allowed model array.

`retarget` replaces one credential's complete target set, reports every shared provider added to or removed from each VM, and requires the owner to re-enter that credential so Age can encrypt to the new recipient set without decrypting the old ciphertext. It updates targets, recipients, and ciphertext in one recoverable operation; an empty target set is a refusal, not retirement.

`rotate` resolves the provider to its credential, reports every provider and target affected by that shared credential, then replaces only its ciphertext. `overlap` means the old remote token remains valid until rebuild and verification succeed; `in-place` warns that remote rollback may already be impossible. The command records neither token value.

`retire` removes the provider catalog entry and all credential-registry references to it. If its credential still serves another provider, the credential record and ciphertext remain. If no declared consumer remains, it also removes the credential record and current ciphertext. This stops future deployment but neither erases encrypted Git history nor revokes the remote token.

All live operations require clean private profiles and secrets checkouts and calculate every touched path before prompting. Token input uses hidden `read -r -s`, accepts exactly one non-empty RFC 6750 `b64token` matching `^[A-Za-z0-9._~+/-]+={0,}$`, and pipes directly to Age. Only ciphertext and non-secret JSON enter a `0700` tmpfs staging directory. There is no token flag, token-path input, force bypass, automatic commit/push/rebuild, or generic remote probe/revocation. `--dry-run` stops before secret input and prints only IDs, targets, paths, shared impact, and follow-up actions.

Before mutation the command durably records one non-secret transaction journal under the Nexus user's private state directory: operation, exact touched paths, pre-run repository HEADs, and which paths existed. It then validates and installs staged data. On an ordinary failure it restores tracked bytes from those recorded HEADs and removes only recorded paths absent there. A signal or power loss leaves the journal; every later live operation refuses until `recover` verifies both HEADs are unchanged and all dirt is confined to recorded paths, performs the same path-scoped restoration, and removes the journal. If either condition fails, recovery preserves evidence and requires human repair. Journal creation/removal is fsynced around mutation; no backup plaintext is stored.

### Dev VM and Pi reconciliation

Every dev VM imports the reconciler even when its desired provider set is empty. Each declared credential becomes a user-owned, group-user, mode-`0600` Age secret in volatile runtime storage. A managed home symlink gives the fixed helper command a credential path; plaintext is never copied into persistent home, generated JSON, argv, the environment, or the Nix store.

`auth.json` receives managed `api_key` entries whose `key` is Pi's `!<helper-command>` form. The helper validates a single non-empty line and writes it only to Pi's credential pipe. `models.json` receives the matching provider metadata. Pi caches a command result for the process lifetime, so rotation verification and remote revocation require a fresh Pi process started after rebuild; existing Pi sessions are explicitly stale. Existing unrelated entries and top-level settings remain byte-equivalent in meaning.

A private ownership manifest records the provider IDs and credential links last installed by this mechanism. Configured provider IDs are exclusively managed: the first activation refuses an existing auth/model ID absent from the manifest, while later activation may replace or retire its own recorded IDs; unrelated IDs are always preserved. Credential links are removed only when still pointing to their recorded managed targets, otherwise a warning preserves the operator replacement.

Activation uses Pi's `proper-lockfile` protocol for `auth.json`, holds its own locks for the remaining files, validates and stages every result, and rechecks inputs before atomic per-file replacement; partial failure restores exact pre-run bytes while locks remain held. If any desired Age credential is absent during provisioning, the whole reconciler warns and makes no mutation; a later activation with all credentials present completes it. An empty desired set needs no credential and removes `previously managed - currently desired` auth/model entries and stale links, then deletes the empty manifest. Malformed JSON, an unowned collision, an empty/malformed present credential, ownership-state corruption, or concurrent change fails without partial mutation.

## Agent Gates

- The agent stops after this plan until the owner says `go`; approval covers only public implementation and synthetic validation.
- Agents do not access, decrypt, migrate, rotate, retire, commit, or deploy private credentials or provider data.
- The owner creates a separate private rollout plan after the public interfaces settle. It owns migration from the current private module, private pin updates, host rebuilds, real inference, rollback, and provider-side revocation.
- Remote retirement occurs only after a fresh post-rebuild Pi process proves the replacement works and every pre-rebuild Pi session is closed. The tool may print this gate but cannot perform or attest to it.
- MicroVM support remains blocked on its credential-delivery path; no fallback writes plaintext to persistent guest storage.

## Acceptance Tests

```sh
cd <profiles-worktree> && nix build .#checks.x86_64-linux.pi-provider-catalog
cd <secrets-worktree> && nix build .#checks.x86_64-linux.credential-inventory .#checks.x86_64-linux.pi-credential-registry
cd <inventory-worktree> && nix build .#checks.x86_64-linux.repository-registry
cd <archetypes-worktree> && nix build .#checks.x86_64-linux.pi-provider-lifecycle
cd <nexus-worktree> && nix build .#checks.x86_64-linux.provisioning-contract
cd <deploy-worktree> && nix build .#checks.x86_64-linux.composed-layer
```

The lifecycle witness uses synthetic providers and a local HTTP server to observe Pi obtaining the fixture bearer through its helper for both adapters. Across install, update, partial retirement, last-provider retirement, rebuild, absent-then-present provisioning activation, reboot-shaped reactivation, and Nix garbage collection, it proves exact auth/models contents, runtime ownership/mode, stale-entry/link cleanup, unowned-collision refusal, preservation of unrelated IDs and replaced links, and absence of plaintext from persistent/generated/store artifacts. A concurrent Pi-compatible auth writer cannot be lost. A Pi process started before rotation remains demonstrably stale, while a fresh process uses the replacement. Empty registries and unsupported targets remain absent or fail closed as specified.

The Nexus witness inspects child argv/environment and every written regular file; plaintext may cross only hidden shell input, the Age stdin pipe, and the synthetic Pi/helper pipe. It proves dry-run, malformed input, shared-impact reporting, retargeting, orphan-versus-shared ciphertext retirement, ordinary rollback, and crash recovery after termination at every install and post-validation boundary. The inventory witness reads raw hypervisor data and sabotages each required Nexus alias. The deploy witness sabotages the archetypes profiles/secrets/inventory and secrets-inventory follows redirects independently. Each schema and lifecycle witness includes sabotage proving the check can fail.

## Rollback Plan

Before merge, close or amend the public PRs. After merge but before private adoption, revert the public PRs or leave private deploy pins unchanged; empty public registries produce no runtime change. If a Nexus operation is interrupted, run `pi-provider recover` before changing either data checkout. If a synthetic rollout fails between slices, pin the preceding compatible revisions and rerun the absent-resource checks.

Private migration and its rollback are intentionally not specified here: the private plan must name the legacy module, live data commits, rebuild target, provider-specific overlap behavior, and exact cutover witness without publishing them. Public `retire` never claims to revoke a remote credential or erase historical ciphertext.
