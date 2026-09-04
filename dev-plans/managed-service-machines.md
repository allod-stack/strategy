# Managed Service Machines

## Tracking Issue

`allod/strategy#45` tracks this public, cross-repository plan. Each PR references
that issue; the final compatibility-removal PR closes it. `allod/nexus#39` owns
operator-invoked generation rollback, and `allod/memory#47` records the recovery
principle this design implements.

This is the public leaf of a cross-boundary arc. It contains no concrete host,
provider, address, credential, application, or rollout data. Downstream plans
may consume these contracts without being prerequisites for this work.

## Goal

Allod composes and administers service machines consistently whether they run
under a local hypervisor or on externally provided hardware, while application
deployment flakes remain independently usable and durable service state remains
an explicit application concern.

## Scope

In scope: a focused `allod/services` module library; a placement-aware inventory
schema; an application-agnostic service archetype; Nexus operations for install,
deploy, verify, and configuration rollback; compatibility adapters; and one
synthetic stateless end-to-end example.

Out of scope: concrete deployments and identities, provider APIs, DNS, application
builds, backup schedules, automatic rollback, high availability, service-data
rollback, and a generic stateful-service lifecycle. One service per machine is a
reasonable deployment default, not a framework invariant.

`dev-plans/buzz-service-archetype.md` must not be executed as written: its VM-only
inventory and application-specific archetype assumptions are superseded here.
Its application work may resume later as a consumer of the interfaces below.

## Milestones and Landing Order

### M1 — Establish the service-module boundary

Entry: `allod/strategy#45` accepted; a human creates public `allod/services`.

- Give the new flake the narrow charter “reusable NixOS service behavior and
  operational policy.” It contains no profiles, concrete machines, identities,
  deploy state, or single-consumer application logic.
- Move the reusable public-host and generator-independent static-site modules
  from `allod/archetypes`; retain temporary re-exports so current consumers build.
- Leave site-generator build helpers with their application deployment flakes.
- Characterize `rentedKvmGuest`: move it to `allod/vm` only if it is a reusable
  machine-placement module; otherwise leave it with the concrete deployment.

Exit: the new flake checks independently, archetypes' pre-migration public output
paths still evaluate through compatibility exports, and ownership is documented.

### M2 — Generalize inventory around placement

Entry: M1 merged.

- Split common managed-machine facts from a discriminated placement record.
- Support the existing libvirt and microVM placements plus an `external` placement
  that requires no fake hypervisor, runtime, MAC, volume, or VM sizing fields.
- Derive legacy `vmSpecs` and `vmFacts` views for current consumers during migration.
- Add positive and mutation-style negative fixtures for each placement.

Exit: every existing machine produces the same generated VM facts/specs, an
external synthetic machine evaluates, and impossible field combinations fail.

### M3 — Make service an application-agnostic archetype

Entry: M2 merged.

- Implement `mkService` in `allod/archetypes` as profile composition: shared
  managed-machine baseline plus the profile's NixOS/home modules.
- Add `service` to profile-definition validation and one synthetic definition in
  `allod/profiles`; do not hardwire a web server, forge, relay, or state policy.
- Consume reusable behavior from `allod/services`. A standalone application flake
  can export a NixOS module; a profile selects it without the application depending
  on inventory, profiles, archetypes, Nexus, or deploy.
- Preserve capability absence: no forge write credentials, backup, or outbound
  identity appears unless the concrete profile requests it.

Exit: the synthetic service toplevel builds for local and external placement;
existing configurations retain drvPath parity apart from intentional input moves.

### M4 — Generalize Nexus machine operations

Entry: M2 schema merged; M3 composition contract available.

- Resolve all targets through common management facts and dispatch placement-only
  bootstrap mechanics behind adapters.
- Keep four operator concepts distinct: initial install, routine deployment,
  verification, and rollback to the previous NixOS generation.
- Preserve existing VM commands as compatibility entry points while adding generic
  managed-machine entry points. All remote mutation originates from Nexus.
- Implement `allod/nexus#39`; rollback changes system configuration only and is
  always operator-selected after a failed deployment or health check.

Exit: fixture targets prove dispatch and command rendering without network writes;
existing VM flows are unchanged; the synthetic external target covers all four
operations, including selection of the previous generation.

### M5 — Prove composition and remove shims

Entry: M1–M4 merged and one downstream deployment has adopted the new interfaces.

- Build and boot-test a synthetic stateless HTTPS service from standalone module
  through profile, archetype, inventory, and deploy composition.
- Document the ownership map and application integration contract.
- Remove archetypes' service-module re-exports and other migration adapters only
  after repository-wide reference searches show no consumers.
- Rebase the future buzz work onto this model or close its old tracking issue as
  superseded; do not add state abstractions during this milestone.

Exit: the end-to-end check passes, compatibility references are zero, and the
tracking issue records downstream adoption evidence.

## Risk

Overall R3 High: this changes schemas and composition seams shared by the fleet.
M1 is R2 (module relocation with aliases); M2–M4 are R3 (inventory, builders, and
remote mutation); M5 is R2. No milestone changes concrete credentials or deployed
state. Land one repository at a time behind compatibility views and compare
generated artifacts before removing any old interface.

## Interface Contracts

- `allod/services` exports reusable NixOS modules; it never exports concrete
  `nixosConfigurations` or application content builders.
- A machine has common platform, profile selection, and management endpoint facts,
  plus exactly one placement variant. The final field names bind in M2 after a
  fixture-backed schema review; semantic ownership does not.
- A service profile supplies modules to `mkService`; statefulness is not an
  archetype and does not alter the inventory type.
- A standalone application deployment may depend on focused reusable modules but
  must build and deploy without the Allod inventory/profile/deploy graph.
- Initial install may invoke placement-specific tooling; routine deployment targets
  an installed NixOS machine. Verification never silently triggers rollback.
- Durable state, backup, restore, and recovery objectives are declared by each
  stateful application. Framework generalization waits for a second proven consumer.

## Agent Gates

- Human: create `allod/services`, merge public PRs, and perform any live target test.
- Agent: stop at dry-run/generated-artifact validation for commands that mutate a
  real machine; never invent provider, identity, or state requirements.
- Owner review: required before M2 schema names and M4 generic CLI names become
  compatibility commitments. Contract semantics above are already decided.
- Public work prepared from a restricted environment is relayed by a human-capable
  environment; inability to push does not justify a private or temporary public API.

## Acceptance Tests

- All affected flakes pass their native checks on supported platforms.
- Snapshot comparisons show existing VM specs/facts and configuration drvPaths are
  unchanged at each compatibility milestone.
- Schema fixtures accept every placement and reject missing discriminators,
  cross-placement fields, and missing management facts.
- The synthetic service boots, accepts SSH only by default, and serves its declared
  content after enabling the reusable module.
- Nexus fixture tests distinguish install/deploy/verify/rollback and prove target
  resolution without contacting a network endpoint.
- A rollback fixture selects the prior NixOS generation and leaves application data
  untouched; failed verification reports failure and awaits operator choice.
- Final `rg` checks find no compatibility exports or references scheduled for removal.

## Rollback

Before shim removal, revert the affected repository PR and retain the compatibility
view. After removal, restore the shim in a patch release before reverting downstream
consumers. Nexus operational changes land without deleting old VM entry points, so
operators can fall back while defects are repaired. No public-plan rollback mutates
a concrete machine or application data.
