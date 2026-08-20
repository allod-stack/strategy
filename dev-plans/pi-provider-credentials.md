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
- `allod/archetypes`: libvirt Age delivery, Pi's command-backed auth entries, atomic managed-entry reconciliation, ownership state, and stale symlink cleanup.
- `allod/nexus`: `pi-provider add`, `rotate`, `retire`, and `--dry-run`, with cross-repository rollback and fixture tests.
- `allod/deploy`: compatible public pins and a synthetic composition canary.

Out of scope:

- Live endpoints, provider/model identifiers, bearer values, target selections, or recipient keys in public history.
- Provider-side token issuance or revocation; those APIs are service-specific and remain human operations.
- OAuth, arbitrary headers, non-bearer authentication, HTTP endpoints, automatic discovery, privacy/service VMs, or microVM delivery.
- Automatically migrating or deleting the current private integration. Its one-shot cutover belongs to the private rollout plan.
- Hiding a runtime credential from agents on its target VM. Pi and agents share the same Unix account, so either can invoke the helper or read the user-owned runtime file; a broker would be a separate security design.
- Moving integration-specific extension behavior into public code. The public mechanism only owns safe installation and removal of links declared by a profile.

Ordered public slices:

1. `profiles` exports the provider catalog contract.
2. `secrets` exports the credential/target contract and recipient derivation.
3. `archetypes` joins both contracts and reconciles generated state.
4. `nexus` safely edits the two data registries and ciphertext.
5. `deploy` pins the compatible revisions.

## Risk Assessment

Residual risk: R3 High. The work crosses five repositories, accepts authentication material, changes Age recipients and Home Manager activation, and removes persistent configuration during retirement. Synthetic checks can prove the public lifecycle, including rollback and generated artifacts, but cannot prove any private endpoint or provider-side revocation. Human review should focus on source-of-truth separation, shared-credential impact, removal ownership, secret transport, and the stage/activate/retire order.

| Slice | Risk | Main scrutiny |
|---|---|---|
| Profiles/secrets contracts | R3 | No duplicated target fact; complete recipient and reference validation |
| Archetypes consumer | R3 | Only owned entries removed; empty desired state still cleans stale state |
| Nexus command | R3 | No plaintext boundary breach; exact rollback across two repos |
| Deploy composition | R3 | Compatible pins and absent-resource behavior |

## Interface Contracts

### Declarative data and ownership

`pi-providers.json` lives in profiles and is keyed by provider ID matching `^[a-z0-9][a-z0-9-]*$`. Each provider has an HTTPS `baseUrl`, a default `api` of `openai-completions` or `openai-responses`, and a non-empty unique model array. Models contain `id` and may contain `name`, `api`, `reasoning`, and `maxTokens`; unknown fields fail validation. Profiles may additionally declare managed extension links as Nix data, but the JSON catalog contains no executable paths.

`pi-credentials.json` lives in secrets and is keyed by credential ID using the same ID grammar. Each entry contains non-empty unique `targets`, non-empty unique `providers`, and `rotationStrategy = "overlap" | "in-place"`. The ciphertext path is derived as `secrets/pi-credentials/<credential>.age`; it is never repeated in the registry. Provider-to-credential and target selection live only here. A provider may share a credential with other providers, but one target may resolve a provider through only one credential.

The secrets flake derives credential inventory consumers, `secrets.nix` recipients, and each dev identity's credential/provider projection from that registry. Recipients are the Nexus active/staged keys plus every target VM active/staged key. Unknown providers/targets, non-libvirt or non-dev targets, missing ciphertext, duplicate resolution, unsupported fields, and recipient drift fail evaluation. Empty public registries generate no Pi credential or provider artifacts.

### Nexus CLI

```text
pi-provider [--dry-run] add <provider> --credential <credential> --url <https-url> --target <dev-vm>... (--model <id>[,<id>...] | --models-file <path>) [--api <adapter>] [--rotation-strategy <overlap|in-place>]
pi-provider [--dry-run] rotate <provider>
pi-provider [--dry-run] retire <provider>
```

`add` refuses an existing provider. A new credential triggers hidden token input and ciphertext creation. An existing credential may be shared only when its target set is unchanged; otherwise the command refuses before secret input and directs the owner through an explicit rekey/rotation. Repeated targets/models accumulate. `--models-file` is mutually exclusive with `--model` and contains only the allowed model array.

`rotate` resolves the provider to its credential, reports every provider and target affected by that shared credential, then replaces only its ciphertext. `overlap` means the old remote token remains valid until rebuild and verification succeed; `in-place` warns that remote rollback may already be impossible. The command records neither token value.

`retire` removes the provider catalog entry and all credential-registry references to it. If its credential still serves another provider, the credential record and ciphertext remain. If no declared consumer remains, it also removes the credential record and current ciphertext. This stops future deployment but neither erases encrypted Git history nor revokes the remote token.

All live operations require clean private profiles and secrets checkouts and calculate every touched path before prompting. Token input uses hidden `read -r -s`, accepts exactly one non-empty RFC 6750 `b64token` matching `^[A-Za-z0-9._~+/-]+={0,}$`, and pipes directly to Age. Only ciphertext and non-secret JSON enter a `0700` tmpfs staging directory. Installation across both repositories is transactional: every pre-run byte and absence is recorded, partial failure restores it exactly, and validation runs before and after mutation. There is no token flag, token-path input, force bypass, automatic commit/push/rebuild, or generic remote probe/revocation. `--dry-run` stops before secret input and prints only IDs, targets, paths, shared impact, and follow-up actions.

### Dev VM and Pi reconciliation

Every dev VM imports the reconciler even when its desired provider set is empty. Each declared credential becomes a user-owned, group-user, mode-`0600` Age secret in volatile runtime storage. A managed home symlink gives the fixed helper command a credential path; plaintext is never copied into persistent home, generated JSON/TypeScript, argv, the environment, or the Nix store.

`auth.json` receives managed `api_key` entries whose `key` is Pi's `!<helper-command>` form. The helper validates a single non-empty line and writes it only to Pi's credential pipe. `models.json` receives the matching provider metadata. Existing unrelated entries and top-level settings remain byte-equivalent in meaning.

A private ownership manifest records the provider IDs, credential links, and extension links last installed by this mechanism. Activation locks all managed files, validates and stages every result before mutation, atomically replaces individual files, and restores exact pre-run bytes after a partial install failure. It removes `previously managed - currently desired` auth/model entries. It removes a stale link only when the path is still a symlink to its recorded managed target; an operator replacement is preserved with a loud warning. After the last provider is retired, the empty desired state performs this cleanup and removes the empty manifest. Malformed JSON, an unowned collision, a missing credential, or ownership-state corruption fails without partial mutation.

## Agent Gates

- The agent stops after this plan until the owner says `go`; approval covers only public implementation and synthetic validation.
- Agents do not access, decrypt, migrate, rotate, retire, commit, or deploy private credentials or provider data.
- The owner creates a separate private rollout plan after the public interfaces settle. It owns migration from the current private module, private pin updates, host rebuilds, real inference, rollback, and provider-side revocation.
- Remote retirement occurs only after the rebuilt target proves the replacement works. The tool may print this gate but cannot perform or attest to it.
- MicroVM support remains blocked on its credential-delivery path; no fallback writes plaintext to persistent guest storage.

## Acceptance Tests

```sh
cd <profiles-worktree> && nix build .#checks.x86_64-linux.pi-provider-catalog
cd <secrets-worktree> && nix build .#checks.x86_64-linux.credential-inventory .#checks.x86_64-linux.pi-credential-registry
cd <archetypes-worktree> && nix build .#checks.x86_64-linux.pi-provider-lifecycle
cd <nexus-worktree> && nix build .#checks.x86_64-linux.provisioning-contract
cd <deploy-worktree> && nix build .#checks.x86_64-linux.composed-layer
```

The lifecycle witness uses synthetic providers and a local HTTP server to observe Pi obtaining the fixture bearer through its helper for both adapters. Across install, update, shared-credential rotation, partial retirement, last-provider retirement, rebuild, reboot-shaped reactivation, and Nix garbage collection, it proves exact auth/models contents, runtime ownership/mode, stale-entry/link cleanup, preservation of unrelated or operator-replaced state, and absence of plaintext from persistent/generated/store artifacts. Empty registries and unsupported targets remain absent or fail closed as specified.

The Nexus witness inspects child argv/environment and every written regular file; plaintext may cross only hidden shell input, the Age stdin pipe, and the synthetic Pi/helper pipe. It proves dry-run, malformed input, shared-impact reporting, two-repository atomic rollback, orphan-versus-shared ciphertext retirement, and exact no-op behavior after every simulated failure. Each schema and lifecycle witness includes sabotage proving the check can fail.

## Rollback Plan

Before merge, close or amend the public PRs. After merge but before private adoption, revert the public PRs or leave private deploy pins unchanged; empty public registries produce no runtime change. If a synthetic rollout fails between slices, pin the preceding compatible revisions and rerun the absent-resource checks.

Private migration and its rollback are intentionally not specified here: the private plan must name the legacy module, live data commits, rebuild target, provider-specific overlap behavior, and exact cutover witness without publishing them. Public `retire` never claims to revoke a remote credential or erase historical ciphertext.
