# memory-backpass: weekly trace review with a human-gated pull request

## Tracking Issue

https://forge.anarch.diy/allod/strategy/issues/55. Three slices deliver the qualification runs, public timer module, and private enablement; the private deployment PR that enables the timer carries `Closes allod/strategy#55`, and every earlier implementation PR uses `Refs allod/strategy#55`.

## Goal

Once a week, one VM reviews distilled session traces against the public and private memory indexes and opens one evidence-backed pull request on `agent-memory` for the owner to accept, revise, or reject.

## Scope

In scope:

- Two by-hand runs of the existing `memory-backpass` skill in `--report-only` mode, with both reports attached to allod/strategy#55 and compared before timer work starts.
- `allod/archetypes` `modules/memory-backpass.nix`: a systemd user service and weekly timer, a headless harness wrapper, checkout and schedule options, a report path, and report mail using the same msmtp shape planned for `forge-groom.nix`.
- Private deployment on one owner-selected VM, pointing the module at the public memory checkout, the `agent-memory` checkout, and the private traces checkout.

Prerequisites:

- The sibling `allod/memory` PR has landed the `memory-backpass` skill and the public memory rules it consumes, and the sibling `agent-memory` PR has landed the private index numbering and ledger structure.
- The sibling `allod/tools` PR has landed the trace distillation command, and the sibling `allod/archetypes` PR has landed the hourly trace timer.
- The owner has created the private traces repository and made its checkout available on the VM used for the two qualification runs.

The timer invokes the skill rather than reimplementing it. The skill pulls the private traces checkout, numbers the bullets of both memory indexes, sends each unanalyzed trace to a bulk-tier model with an analysis prompt, discards every finding whose verbatim quote is absent from the trace, appends gap, non-compliance, and harm sightings to each repository's `ledger.md`, folds the ledgers, and asks the strongest model for at most five edits that stay under the index byte cap and meet the two-session floor. A write-enabled run opens one pull request on `agent-memory` titled `memory-backpass: <date>` and includes the proposed edits, supporting quotes, and funnel; `--report-only` stops before every write. Every prompt begins with `<!-- memory-backpass:self -->` so the trace command excludes the backpass's own sessions.

Out of scope:

- Implementing or changing the `memory-backpass` skill, either memory index, either `ledger.md`, the memory budget hook, or the two-session evidence rule.
- Implementing or changing trace distillation, trace redaction, trace storage, or the hourly trace timer.
- Creating the private traces repository, choosing its contents, restructuring `agent-memory`, or changing model/provider configuration.
- Merging memory pull requests, pushing either memory repository's default branch, or granting the timer any new forge authority.

## Slices

| Slice | Delivers | Witness | Unblocks |
| --- | --- | --- | --- |
| 1 qualification | The landed skill is run by hand twice with `--report-only` against the configured public memory, `agent-memory`, and private traces checkouts | Both reports are attached to allod/strategy#55, every retained finding has a verbatim source quote, no repository or forge write occurs, and the reports are compared by hand | 2, only after the owner accepts the comparison |
| 2 public module | `modules/memory-backpass.nix` with the service, timer, wrapper, checkout options, schedule, report path, report-only default, and forge-groom-shaped mail | The archetypes check builds the real generated wrapper and units with fixture inputs, proves the marker and `--report-only` default, exercises success and failure mail, and proves missing checkouts fail loudly | 3 |
| 3 private enable | The private deployment enables the module on one VM with real checkout paths and the ordinary agent forge token already present there | Generated units show the approved paths and schedule; a manual service start writes and mails the report and, when write mode is owner-enabled, opens exactly one correctly titled pull request while the default branch stays unchanged | done |

Gate between slices 1 and 2: slice 2 does not start until the skill has completed two by-hand runs in `--report-only` mode, both reports are attached to allod/strategy#55, and the reports have been compared with each other and against their source traces. Unsupported quotes, missed write attempts, or material classification differences are fixed in the sibling skill and both qualification runs are repeated. Starting slice 2 is the owner's decision after reading the accepted reports.

Contracts bind when a slice starts. Slice 2's exact Nix option names are recorded in its PR after the qualification runs establish the paths and arguments the wrapper actually needs.

## Risk Assessment

Residual risk: R3 High

Why:

- The public module generates a systemd unit that runs a model with scheduled write access to branches in a private repository; no human participates between timer activation and pull-request creation.
- Quote checking, the five-edit ceiling, the byte cap, the two-session floor, two report-only qualification runs, and review through a pull request narrow what a faulty run can propose, but generated lifecycle behavior and the private/public checkout boundary still deserve high scrutiny.
- This is not R4: the timer can only open a pull request, never merge it or push a default branch, so authoritative memory remains human-controlled. Rollback is a revert of the private enablement plus disabling the timer; a bad pull request and its feature branch can be closed and deleted without reconstructing unique state.

Human scrutiny:

- Read the two qualification reports against their quoted traces first, then inspect the generated wrapper's prompt prefix, paths, mode, harness invocation, and absence of secret values in its environment.
- Inspect the generated service and timer, failure reporting, mail credential handling, and the first live pull request's title, evidence, funnel, edit count, byte-cap result, and unchanged default branch.

| PR or milestone | Risk | Reason | Human scrutiny |
| --- | --- | --- | --- |
| 1 qualification runs | R2 | Private traces are sent through the configured model path and reports are produced, but `--report-only` permits no repository or forge writes | Quote provenance, prompt self-marker, report differences, and evidence that all writes were skipped |
| 2 archetypes module | R3 | A wrapper and systemd user timer schedule a model process with private checkout paths, forge access, and a mail credential | Generated units and wrapper, report-only default, path isolation, failure behavior, and credential absence from `Environment=` |
| 3 private enable | R3 | One live VM gains unattended authority to push a feature branch and open a private pull request each week | Owner-approved mode, ordinary token authority, timer state, first mail and pull request, and unchanged default branch |

## Interface Contracts

- Harness and prompt: the wrapper invokes `pi` headlessly with `--no-session`; its top-level prompt begins with `<!-- memory-backpass:self -->`, names the skill path, checkout paths, mode, and report path, and instructs Pi to read and run the skill. The skill owns its bulk-tier analysis and strongest-model proposal calls, and every nested prompt preserves the same prefix.
- Runtime environment: the generated service supplies `MEMORY_BACKPASS_SKILL_PATH`, `MEMORY_BACKPASS_TRACES_CHECKOUT`, `MEMORY_BACKPASS_PUBLIC_MEMORY_CHECKOUT`, `MEMORY_BACKPASS_PRIVATE_MEMORY_CHECKOUT`, `MEMORY_BACKPASS_REPORT_PATH`, and `MEMORY_BACKPASS_MODE`, plus the agent user's ordinary `HOME` so Pi and forge find their existing configuration. `MEMORY_BACKPASS_MODE` accepts only `report-only` or `write`; the wrapper translates `report-only` to the literal `--report-only` skill argument and refuses any other value.
- Skill path: the wrapper receives or derives an absolute path to `skills/memory-backpass/SKILL.md` inside the configured public memory checkout and fails before invoking the model when that file is absent.
- Checkouts: separate options identify the public memory checkout, the `agent-memory` checkout, and the private traces checkout. The wrapper resolves and validates all three before the model starts; it does not discover repositories by scanning the workspace or substitute one checkout for another.
- Mode: the module defaults to report-only and the initial wrapper invocation passes `--report-only`. The private deployment may select write mode only in the owner-approved timer-enabling change after the slice 1 gate; absence of a mode value never silently enables writes.
- Report: one option names the report file. The wrapper creates its parent with user-only permissions, replaces the report atomically on success, exits non-zero if no report was produced, and leaves the previous report available when a failed run cannot replace it.
- Forge authority: no dedicated backpass token is created. The service runs as the agent user with its ordinary `HOME`, so `forge` uses the ordinary agent token already available on that machine; the wrapper never copies the token into a prompt, command argument, report, or `Environment=` value. That authority may create a feature branch and pull request on `agent-memory` but may not merge or update its default branch.
- Mail: the module follows the `forge-groom.nix` plan's msmtp contract with options for enabling mail, recipient, sender, relay, and credential file. The wrapper mails the completed report with subject `memory-backpass <date>: <summary line>`; if no report is produced, it mails a `memory-backpass <date>: FAILED` message containing only a bounded log tail. The mail credential is file-backed and never enters the model environment.
- Schedule and lifecycle: one option supplies the systemd calendar expression. The service is oneshot, the timer is enabled only when the module is enabled, and a missing skill, checkout, harness, report directory, or mail prerequisite fails loudly rather than falling back to another path or silently skipping the run.

## Agent Gates

- The owner must create the private traces repository and make its checkout available to the qualification environment. This blocks slice 1; the agent does not create or populate that repository.
- The sibling skill, trace command, and hourly trace timer PRs must land before the relevant qualification or integration witness can pass. These are prerequisites, not work performed by this plan.
- The first two runs are by hand in `--report-only` mode. Their reports must be attached to allod/strategy#55 and compared before slice 2 starts; the owner decides whether the evidence is good enough to cross that gate.
- The owner chooses the deployment VM, mail values, schedule, and whether the private enabling change may select write mode; rebuilding the VM and enabling the timer are owner actions. This blocks slice 3's live witness and is the only decision that grants unattended execution.

## Acceptance Tests

```sh
# Slice 1: run twice from the public memory checkout with the three checkout and report variables set by the operator.
test -f "$PUBLIC_MEMORY_CHECKOUT/skills/memory-backpass/SKILL.md"
test -d "$PRIVATE_MEMORY_CHECKOUT/.git"
test -d "$TRACES_CHECKOUT/.git"
pi --no-session -p "<!-- memory-backpass:self -->
Read $PUBLIC_MEMORY_CHECKOUT/skills/memory-backpass/SKILL.md and run memory-backpass --report-only with public memory at $PUBLIC_MEMORY_CHECKOUT, private memory at $PRIVATE_MEMORY_CHECKOUT, traces at $TRACES_CHECKOUT, and report at $FIRST_REPORT."
pi --no-session -p "<!-- memory-backpass:self -->
Read $PUBLIC_MEMORY_CHECKOUT/skills/memory-backpass/SKILL.md and run memory-backpass --report-only with public memory at $PUBLIC_MEMORY_CHECKOUT, private memory at $PRIVATE_MEMORY_CHECKOUT, traces at $TRACES_CHECKOUT, and report at $SECOND_REPORT."
git -C "$PUBLIC_MEMORY_CHECKOUT" status --short
git -C "$PRIVATE_MEMORY_CHECKOUT" status --short
git -C "$TRACES_CHECKOUT" status --short
diff -u "$FIRST_REPORT" "$SECOND_REPORT" > "$REPORT_COMPARISON"
diff_status=$?
test "$diff_status" -le 1
sed -n '1,240p' "$REPORT_COMPARISON"  # review every difference; equality is not required

# Slice 2: repository and generated-artifact checks.
cd "$ARCHETYPES_CHECKOUT"
nix flake check
system=$(nix eval --raw --impure --expr builtins.currentSystem)
nix build ".#checks.${system}.memory-backpass"

# Slice 3: inspect and exercise the deployed generated lifecycle before waiting for the schedule.
systemctl --user cat memory-backpass.service memory-backpass.timer
systemctl --user show memory-backpass.service -p Environment -p ExecStart -p LoadCredential
systemctl --user list-timers memory-backpass.timer
cd "$PRIVATE_MEMORY_CHECKOUT"
git fetch origin master
default_before=$(git rev-parse origin/master)
systemctl --user start memory-backpass.service
systemctl --user status memory-backpass.service --no-pager
test -s "$MEMORY_BACKPASS_REPORT_PATH"
forge pr list -s open | rg "memory-backpass: $(date -u +%F)"
git fetch origin master
default_after=$(git rev-parse origin/master)
test "$default_before" = "$default_after"
```

The slice 2 `memory-backpass` check uses the production module with fixture inputs and must fail when a required checkout is absent, assert that the first prompt bytes are the self-marker, assert that report-only is the default, inspect the generated calendar and checkout paths, exercise atomic report replacement, and exercise both successful and failed mail without a real forge token or network access.

## Rollback Plan

Stop and disable `memory-backpass.timer` immediately, revert the private enabling PR, and rebuild the VM so declarative state agrees with the stopped runtime. Close any pull request created by a bad run and delete its feature branch; the default branch requires no repair because the timer cannot update or merge it. If the fault is in the public module, revert that PR after private enablement is off; the sibling skill, trace command, hourly trace timer, private traces repository, and ordinary agent token remain untouched. Preserve the failed report and bounded service log until the cause is understood, then remove only generated report files no longer referenced by the reverted deployment.
