# Pi Provider Redundant-Name Adoption

This follow-up lets an existing provider enter authenticated discovery when its only extra model metadata is a redundant display name equal to the model ID; custom names and other hand-authored fields remain outside discovery.

## Tracking Issue

[allod/nexus#34](https://forge.anarch.diy/allod/nexus/issues/34). The strategy PR uses `Refs allod/nexus#34`; the Nexus implementation PR uses `Closes allod/nexus#34`.

## Goal

An operator can use `discover-refresh` to normalize away redundant `name == id` fields without weakening the pre-bearer refusal for metadata discovery cannot reproduce.

## Scope

In scope:

- `allod/nexus/scripts/pi-provider`: refresh validation for redundant names and actionable refusal text.
- `allod/nexus/tests/pi-provider.sh`: successful adoption and pre-bearer custom-name/unsupported-field refusals through the production command.
- `allod/nexus/docs/credentials.md`: the adoption rule and executable remediation.

Out of scope:

- Persisting discovered names, accepting custom display names, or accepting other fields outside the discovery allowlist.
- Turning manual `add` into an update command or changing discovery HTTP, bearer, removal, transaction, and recovery behavior.
- Private providers, inference, deployment, rebuilds, remote token mutation, or private data.

## Risk Assessment

Residual risk: **R3 High**.

Why:

- The change relaxes a validation gate immediately before an authenticated bearer read, so an over-broad condition could discard hand-authored model metadata.
- The accepted case is narrow and reversible: every present `name` must be a string exactly equal to the already-validated `id`; the ordinary refresh transaction then replaces the catalogue without `name`.
- Synthetic command-level witnesses cover the accepted case and prove custom names and unrelated fields still fail before Age or HTTP; rollback is a straight source revert.

Human scrutiny should focus on the exact all-model equality predicate, whether the accepted refresh actually removes `name`, and whether each refusal still precedes bearer access.

## Interface Contracts

`pi-provider discover-refresh <provider>` keeps the existing discovery-compatible model fields. It additionally accepts a stored `name` field only when every model carrying `name` has `name == id`. Models without `name` may coexist with redundant-name models. The next non-dry refresh compares the original models, reports affected retained IDs as changed, and replaces them with discovery-translated models that omit `name`.

A custom name, non-string name, or `name` unequal to `id` fails before bearer prompting/decryption or HTTP, even when sibling models have absent or redundant names. Every other field outside `api`, `contextWindow`, `id`, `input`, `maxTokens`, and `reasoning` remains a pre-bearer refusal. Diagnostics tell the operator to edit and commit the existing profiles catalogue to normalize/remove `name` or remove unsupported fields, then retry `discover-refresh`; they do not recommend `add` or `--models-file`, which cannot update an existing provider. The operator may instead keep the hand-authored catalogue and not use discovery refresh.

No CLI syntax, JSON schema, network route, bearer handling, credential state, removal confirmation, transaction, or recovery interface changes.

## Agent Gates

The agent uses only generated HTTP, Age, inventory, and repository fixtures. The human decides whether and when to run refresh against private provider data; private discovery, deployment, inference, and provider-side changes are not evidence for this PR.

## Acceptance Tests

```sh
cd <nexus-worktree>
bash -n scripts/pi-provider tests/pi-provider.sh
shellcheck -x --source-path=SCRIPTDIR scripts/pi-provider tests/pi-provider.sh
bash tests/pi-provider.sh
nix build .#checks.x86_64-linux.provisioning-contract
```

The production-command witness must show that a mixed catalogue of absent and redundant names refreshes successfully, reports the redundant-name model as changed, and persists no `name`. A custom-name sabotage must include an absent-name or redundant-name sibling so an existential predicate fails the witness; it and a separate unsupported-field fixture must fail without invoking Age or HTTP. Both diagnostics must direct the operator to edit and commit the existing catalogue before retrying and must not recommend `add` or `--models-file`. The existing discovery and manual lifecycle suite remains green.

## Rollback Plan

Before merge, amend or close the implementation PR. After merge, revert the Nexus commit; this restores the stricter refusal and does not require a data migration. If a private refresh already normalized redundant names away, those names carried no information and need not be restored; any desired display names remain manual provider metadata and can be restored from the prior profiles commit.
