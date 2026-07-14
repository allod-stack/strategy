# Nexus External Backup SSH Rotation Gates

## Tracking Issue

https://forge.anarch.diy/allod/nexus/issues/4

Multi-repo work across `allod/secrets` and `allod/nexus`. Two PRs:

- `allod/secrets`: registry schema + synthetic template entries — `Refs allod/nexus#4`.
- `allod/nexus`: consumer logic, gates, tests, runbook — carries `Closes allod/nexus#4`.

The nexus PR carries the closing keyword because it delivers the behavior the
acceptance criteria describe. It must no-op safely on an empty registry, so the
landing order of the two PRs is not load-bearing.

## Goal

Make `nexus-host-key` treat declared external backup SSH targets as first-class
rotation gates: print exact per-target setup commands at `stage`, prove the
staged key against them before `activate` swaps `~/.ssh/host`, and prove the
installed key against them before `retire`, with a typed per-target override for
intentionally abandoned hosts.

## Scope

In scope:

- `allod/secrets`: add an `externalSshTrustTargets` attrset to `identity.nix`,
  reachable through the existing `lib.identity`, referencing `sshHosts` aliases.
  The public template ships synthetic example entries only. Add a flake check
  (alongside `credential-inventory`, which references neither `sshHosts` nor
  the new attrset) asserting every entry's `sshHost` resolves in `sshHosts`,
  `recovery` is one of the three known values, and `authorizedKeysPath` is a
  string — a fork operator hand-edits `identity.nix`, and without this the
  first feedback on a typo is a fail-closed die mid-rotation instead of a
  failed `nix flake check` at commit time.
- `allod/nexus`:
  - a resolver that reads `externalSshTrustTargets` and joins each entry to its
    `sshHosts` connection record (new helper in `scripts/lib/rotation-common.sh`);
  - an external-host phase wired into `do_stage` / `do_activate` / `do_retire`
    in `scripts/nexus-host-key`;
  - a `--accept-unverified-external-host <name>` override with typed confirmation;
  - external-host cases in `tests/nexus-host-key.sh`;
  - runbook text in `README.md` documenting external backup targets as rotation
    gates, replacing the current "update external SSH hosts" follow-up reminders
    in the printed activate/retire steps.

Out of scope:

- Real external-target hostnames, users, or addresses in any public repo —
  synthetic placeholders only (privacy boundary).
- Managed VM verification (already covered by `verify_all_vm_ssh_with_key`) and
  Forgejo git-over-SSH verification (already covered by
  `host_git_ls_remote_with_key`); external gates must not duplicate or re-run
  those loops.
- The backup scripts themselves (service-VPS dump/pull, offsite upload) and
  their timers/scheduling.
- The age/secrets re-encryption flow, the missing-old-key recovery path
  (`--accept-missing-old-key`), and the tmpfs workdir handling.
- Any provider-console or provider-support automation; those stay human gates.

## Risk Assessment

Residual risk (per milestone, multi-PR):

| PR / milestone | Risk | Reason | Human scrutiny |
|---|---|---|---|
| secrets: registry schema + synthetic entries | R2 Medium | Additive data in the identity template, but not inert: each synthetic gate needs a synthetic `sshHosts` alias, and `home-shared.nix` maps every `sshHosts` entry into `programs.ssh.matchBlocks`, so the entries become literal `Host` blocks on every dev VM once profiles bumps its secrets input. Still R2: the template already ships synthetic `192.0.2.x` entries whose blocks are inert and this adds more of the same, it lives in the identity-bearing repo, and it defines the cross-repo contract. | Confirm the public template carries only synthetic hosts; confirm the shape matches what nexus consumes; confirm the added `sshHosts` aliases follow the existing synthetic-address convention. |
| nexus: gates + resolver + tests + runbook | R3 High | Adds gating logic into the host-key rotation state machine (auth + secrets + the `~/.ssh/host` swap). The failure that matters is failing *open* — letting `retire` proceed while a required target still trusts only the retired key — which strands offsite backup access, a security/infrastructure boundary with a slow recovery path. | Read the activate/retire gate ordering; confirm auth-rejection and unreachable are both fatal for external gates; confirm the override cannot be satisfied without a typed per-target confirmation; inspect the printed commands for a real target class. |

Why R3 and not R4 for the nexus PR: the risky operation is local and explicit
(the operator runs `stage`/`activate`/`retire` deliberately), every code path
except the real-host proof is exercised by fixtures, rollback is a straight
`git revert`, and a bug fails toward *more* gating (a spurious gate blocks
`retire` visibly) far more readily than toward silently skipping one. The
residual R4-flavored concern — proving the gate against the real VPS and offsite
destination — is a human-only validation gate (see Agent Gates), not something
the diff itself can guarantee.

The plan rests on one asserted premise no public repo can confirm: that the
external backup targets trust `~/.ssh/host` as the SSH *client* identity the
backup jobs present — that is what makes rotating it a threat to backups and
the probe's `-i <rotation-key>` the right proof. Nothing in `nexus`,
`secrets`, `profiles`, or `inventory` references the backup scripts or their
key; the premise comes from the tracking issue. If a target actually
authenticates backups with a dedicated key, the gate proves the wrong thing
for that target and rotation never threatened it. Agent Gates carries the
operator confirmation; the resolver's `identityFile` cross-check (Interface
Contracts 2) surfaces registry evidence either way.

Human scrutiny first: the `do_activate` and `do_retire` gate-insertion points
and the reachable-vs-auth-failure classification for external targets.

## Interface Contracts

1. **Registry schema** — in `secrets/identity.nix`, exposed as
   `lib.identity.externalSshTrustTargets`:

   ```nix
   externalSshTrustTargets = {
     <name> = {
       sshHost = "<alias>";              # key into identity.sshHosts
       authorizedKeysPath = "<path on target>";
       recovery = "old-key" | "provider-console" | "provider-support";
     };
   };
   ```

   Every declared entry is a rotation gate: declaring a target is what makes
   it gate-worthy. The issue's example shape carried `requiredFor` and
   `verify`; both are deliberately omitted. `requiredFor` would be a list
   filtered for a single membership token with rotation as the only consumer,
   and an entry with a typo'd or missing token would silently stop gating — the
   exact fail-open class this plan exists to remove; a consumer tag returns
   when a second consumer exists. `verify` had one legal value; the SSH probe
   is hardcoded until a second verifier kind actually exists (see Agent Gates
   for the restricted-shell caveat that would force one). An absent
   `externalSshTrustTargets` attribute ⇒ empty set ⇒ no external gate (today's
   behavior). An eval failure of the attribute ⇒ die (fail closed); it is
   never treated as empty.

2. **Connection resolution** — one `nix eval --json` invocation resolves the
   whole registry, joined to `identity.sshHosts` inside the eval:

   ```
   nix eval --json "path:${IDENTITY_CONFIG}#lib.identity" --apply 'i:
     builtins.mapAttrs (name: t: {
       inherit (t) authorizedKeysPath recovery;
       hostname = i.sshHosts.${t.sshHost}.hostname;
       user = i.sshHosts.${t.sshHost}.user;
       port = i.sshHosts.${t.sshHost}.port or 22;
     }) (i.externalSshTrustTargets or {})'
   ```

   The `or {}` / in-eval join shape is load-bearing, not style. The
   per-attribute pattern used by `resolve_forge_connection`
   (`nix eval --raw ...#lib.identity.<attr> 2>/dev/null || die`) cannot
   implement contract 1: `nix eval` of an absent attribute path exits nonzero
   exactly like a real eval error, so mirroring it either dies on an absent
   registry (breaking the no-op-on-absent landing-order and rollback claims)
   or, "fixed" with `|| true`, treats every eval error as an empty registry —
   the silent fail-open this plan exists to prevent. With the single-eval
   shape, `or {}` catches only the missing attribute: a missing `sshHost`
   alias, a missing required field, or any throw inside the values still
   fails the eval (verified against nix 2.x behavior). Bash side: exit 0 ⇒
   parse the JSON; nonzero exit ⇒ die and surface nix's stderr — never
   swallow it with `2>/dev/null`, and never infer emptiness from empty
   output. A missing optional `port` defaults to 22 inside the eval (a
   per-field eval would spuriously die on portless entries). After parsing,
   the resolver validates `recovery` against the three known values and dies
   on anything else — remediation text must never silently print empty. Add
   the helper to `rotation-common.sh` (e.g. `resolve_external_targets`); the
   PR pins the name. One cheap premise guard: when a gate's `sshHosts` alias
   declares an `identityFile` that is not the rotated identity path, print a
   warning naming both — deployment data saying the operator reaches this
   host with a different key is exactly the signal that the target may not
   trust `~/.ssh/host` at all (warn, not die: a target may legitimately trust
   both keys).

3. **Verification command** — always uses the *rotation* key, never the
   operator's `identityFile`:

   ```
   ssh -F /dev/null -i <rotation-key> -o IdentitiesOnly=yes -o BatchMode=yes \
       -o StrictHostKeyChecking=yes -o ConnectTimeout=<n> [-p <port>] \
       <user>@<hostname> true
   ```

   `<rotation-key>` is the decrypted staged key at `activate` and the installed
   `~/.ssh/host` at `retire`. `-F /dev/null` makes the probe config-independent
   (a command-line `-F` skips the system-wide `/etc/ssh/ssh_config` as well as
   the user config), and `-i` + `IdentitiesOnly=yes` make the rotation key the
   only identity ssh will try — OpenSSH adds default identities only when none
   is specified, and `BatchMode=yes` excludes interactive fallbacks — so the
   probe cannot succeed via an agent or default key that is not the rotation
   key. Target host-key trust comes from the operator's default
   `~/.ssh/known_hosts`, so a missing or changed target host key fails the
   probe by design (the backup channel is exactly where a MITM must block, not
   auto-accept).

   Every probe failure is a hard stop; classification affects only the
   message and recovery text, and there are three classes, not two:

   - *Unreachable* — the existing `vm_ssh_unreachable` predicate, shared
     unchanged with the VM loop; only the disposition differs (the VM loop
     skips, external gates die). Recovery text names
     `--accept-unverified-external-host` as the deliberate path for a target
     that is gone for good — a decommissioned host whose DNS is dropped reads
     as "unreachable", and the override is the operator's recourse.
   - *Host-key verification failure* — a new external-only match on ssh's
     host-key diagnostics (`Host key verification failed`,
     `REMOTE HOST IDENTIFICATION HAS CHANGED`, `No ... host key is known`).
     Without this class, a fresh operator whose `known_hosts` lacks the target
     hits first contact and is told the target "rejected the staged key" —
     wrong, and an invitation to reach for `StrictHostKeyChecking=no`.
     Recovery text: verify the target's host-key fingerprint out-of-band
     (provider console), then seed `~/.ssh/known_hosts` explicitly; for the
     changed-key case, say plainly this is the MITM posture working and the
     key must be verified before trust is updated. The text must never
     suggest disabling strict host-key checking.
   - *Auth rejection* — the fallthrough; the target answered and refused the
     key. Recovery text is the per-target `recovery`-variant setup command.

4. **Phase behavior**. Every phase resolves the registry once, at the start,
   before any state mutation, and prints the resolved gate count — including
   an explicit `external trust gates: none declared` line when empty. The
   count line is the operator-visible signal for the one seam fail-closed
   resolution cannot cover: a fork typo of the top-level attribute name is
   indistinguishable from "no gates declared", and an operator who declared
   gates must be able to notice them missing from the phase output. Gates are
   processed in sorted-name order so multi-gate output and stdin-fed
   confirmations are deterministic.
   - `stage`: for each gate, print copy/paste setup commands using the staged
     public key and the target's `authorizedKeysPath`, plus the `recovery`-
     specific variant — `old-key`: append the staged key over SSH using the
     current/old key; `provider-console`: one-line
     `echo '<staged-pub>' >> <authorizedKeysPath>` for a console/recovery shell;
     `provider-support`: a support-request block embedding the staged public key.
     No verification at `stage` (the target may not trust the key yet).
     Resolve-at-start matters most here: `do_stage` mutates the secrets
     checkout (`update_staged_json`, re-encryption) before it prints, so a
     registry that only failed at print time would leave a dirty secrets
     checkout plus a staged backup that block the next run until manually
     cleaned up. A malformed registry must die before stage writes anything.
   - `activate`: after the existing staged-key decrypt / VM / Forgejo checks and
     **before** installing `OLD_BACKUP` and swapping `~/.ssh/host`, prove the
     staged key against every gate not covered by an accepted override. Any
     failure stops activation with actionable per-target recovery text and
     leaves `~/.ssh/host` untouched.
   - `retire`: **before** `promote_staged_json` and re-encryption, prove the
     installed `~/.ssh/host` against every gate not covered by an accepted
     override. Run the external gate after `assert_identity_public_matches`
     confirms the installed key is the staged key and before the `OLD_BACKUP`
     handling, so a probe failure or a mistyped override confirmation dies
     before the operator is asked to type the missing-old-key confirmation.
     Any failure stops retirement with a named human gate before any secret
     is re-encrypted.

5. **Override** — `--accept-unverified-external-host <name>` (repeatable), valid
   on `activate` and `retire`. The activate-time form exists for transient
   outages: activation can proceed past a temporarily-down target while the
   retire gate still proves that target once it recovers. A permanently
   decommissioned target should instead be deleted from the fork's registry;
   the override is per-run and leaves the registry authoritative. For each
   named target the command reads a typed confirmation from stdin (e.g.
   `abandon external target <name>`, reusing the
   `confirm_missing_old_key_retire` prompt pattern) and echoes the exact
   abandoned target (`name` + resolved `user@host`) into the phase output. The
   confirmation string embeds the specific target name, so it cannot be
   satisfied by a generic yes or by a confirmation typed for a different
   target. A named target not present in the registry ⇒ die. An unmatched
   confirmation ⇒ die.

6. **New env overrides** — `NEXUS_HOST_KEY_EXTERNAL_CONNECT_TIMEOUT` (seconds,
   default aligned with the existing VM connect-timeout default of 5). Registry
   and connection source remains `IDENTITY_CONFIG`. Document in `usage()`.

## Agent Gates

- **Real-target proof is human-only.** The agent cannot reach the service-backup
  VPS or the offsite backup destination; proving the printed commands and gates
  against them during a real or dry-run rotation is an operator step. This
  blocks the final acceptance criterion (prove printed commands and gates cover
  the external backup targets before old-key deletion).
- **The client-key premise is operator-confirmed, per target.** Before
  declaring a target in the private fork, the operator confirms (a) the backup
  jobs authenticate to it with `~/.ssh/host` — if they use a dedicated backup
  key, host-key rotation does not threaten that target and it does not belong
  in this registry — and (b) `ssh <user>@<host> true` is a valid no-op there:
  a restricted storage-box shell that refuses remote commands would fail the
  probe against a healthy key, and such a target needs a second verifier kind
  added before the gate is meaningful for it.
- **Authorizing the staged key on external targets** — provider console,
  recovery shell, SSH append, or provider support — is human, on real
  infrastructure.
- **`allod/secrets` is a protected repo.** The agent edits it through
  `allod change begin -d <desc> secrets`; a human reviews and merges. Real
  external-target values are added by the operator in their private fork, never
  in the public template.

## Acceptance Tests

Agent-runnable, extending the existing fixture harness in
`tests/nexus-host-key.sh` (fake `ssh` / `nix` / `virsh` / `git` / `age` stubs,
env overrides):

```sh
# Full nexus host-key suite including the new external-host cases
bash tests/nexus-host-key.sh

# Whole flake check (runs the suite + the packaging assertion)
nix flake check

# In secrets: registry shape check (alias resolution, recovery enum) over the
# synthetic template entries
nix flake check
```

Stub work is load-bearing, not copy-paste. The fake `ssh` today hard-requires
`UserKnownHostsFile=${fixture}/known_hosts_vms` on every invocation (exit 69),
so it would reject every external probe as written. It must learn to
distinguish target classes by `<user>@<host>` and assert a *per-class probe
shape*: VM probes keep today's assertions (required `UserKnownHostsFile`);
external probes must **require `-F /dev/null` and forbid `UserKnownHostsFile`**
— an implementation that lazily reuses the VM probe would silently move
external host-key trust into `KNOWN_HOSTS_VMS`, and the stub must catch
exactly that. Both classes keep the `BatchMode`/`IdentitiesOnly`/
`StrictHostKeyChecking` assertions and the rotation-key `-i` assertion (staged
tmpdir key at `activate`, installed `~/.ssh/host` at `retire`). Outcomes must
be controllable per target: `ssh-unreachable` is already a per-IP list, but
`ssh-fail` is global — and since the VM loop runs before the external gate at
`activate`, a global auth-failure toggle dies in the VM loop and an
"external auth failure" case would go green without ever exercising the gate.
Auth-reject and the new host-key-failure outcome need per-target toggles. The
fake `nix` exits 65 on unknown queries: teach it the registry eval (serving
per-fixture registry JSON) plus an eval-error toggle, or every new case below
dies at exit 65 before testing anything.

Cases the suite must add:

- **empty/absent registry** → `stage`/`activate`/`retire` behave exactly as
  today apart from the single `external trust gates: none declared` line (no
  gate, no per-target output blocks).
- **stage output** → contains a setup command per gate embedding the staged
  public key and the target's `authorizedKeysPath`, with the correct `recovery`
  variant text.
- **activate success** → all gates accept the staged key → activation proceeds
  and swaps `~/.ssh/host`.
- **activate auth failure** → a gate rejects the staged key → activation stops,
  `~/.ssh/host` unchanged and no `~/.ssh/host.pre-rotation` created (the state
  pair that proves the gate fired before the swap), recovery text names the
  target.
- **activate unreachable** → a gate is network-unreachable → activation stops
  (distinct from the VM skip semantics), `~/.ssh/host` unchanged, and the
  stop text names the override flag.
- **activate host-key failure** → a gate's target host key is unknown or
  changed → activation stops, `~/.ssh/host` unchanged, and the text gives
  verify-then-seed `known_hosts` guidance distinct from the auth-rejection
  text (and never suggests `StrictHostKeyChecking=no`).
- **retire success** → all gates accept the installed key → retire proceeds.
- **retire auth failure** → a gate rejects the installed key → retire stops
  with a named human gate; `nexus-host-key.json` still has staged state and
  the old backup key still decrypts every declared secret (the observable
  pair that proves neither `promote_staged_json` nor re-encryption ran).
- **override** → `--accept-unverified-external-host <name>` with a matching
  typed confirmation → phase proceeds and records the abandoned `name` +
  `user@host`; a missing/mismatched confirmation or an unknown name → die.
- **fail-closed resolution** → a gate whose `sshHost` alias is absent from
  `sshHosts`, and an `externalSshTrustTargets` eval error → die (never silently
  skipped).

## Rollback Plan

- Plan text: revert the plan commit.
- `allod/nexus` PR: straight `git revert`; rotation returns to current behavior
  (external hosts as printed manual follow-up, no gate). Because the consumer
  no-ops on an absent registry, reverting nexus while the secrets schema remains
  is safe.
- `allod/secrets` PR: straight `git revert` of the additive attrset. If both are
  reverted, revert nexus before secrets so no live consumer references a removed
  attribute; if only secrets is reverted, the nexus resolver treats the absent
  attribute as an empty registry (no gate), not an error.
- Partial state: a failed `activate` leaves `~/.ssh/host` and all rotation state
  untouched (the proof runs before the swap). A failed `retire` stops before
  `promote_staged_json` and re-encryption, leaving staged state intact for a
  retry after the target is fixed or explicitly abandoned.
