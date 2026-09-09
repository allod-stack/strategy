# forge-groom: weekly self-hosted tracker triage with an emailed report

## Tracking Issue

https://forge.anarch.diy/allod/strategy/issues/48. Four slices, each its own PR set; the private deployment PR that enables the timer carries `Closes allod/strategy#48`. Earlier PRs use `Refs allod/strategy#48`.

## Goal

Every week, every open issue and pull request across the allod repos is checked against master, the ones a merged `Closes` line has settled are closed, the rest are labelled, and the owner receives one short email listing what closed, what is ready to merge, and the decisions only they can make.

## Scope

In scope:

- `allod/memory` `skills/forge-groom/SKILL.md`: the portable skill, the whole behavioural contract, runnable by hand from any harness.
- `allod/tools` `forge pr list -s open|closed|all`: the one forge capability the close rule needs, since master is fast-forward only and merged PR bodies exist nowhere but the forge.
- `allod/archetypes`: a public module `forge-groom.nix` providing a systemd user timer that runs `pi` headless with the skill, mails the report through msmtp, and reads its forge token and SMTP credential from agenix paths the deployment supplies.
- Private deployment (`profiles`, `secrets`, `inventory`): which VM runs it, the recipient address, the SMTP relay, the SMTP credential, and a dedicated forge token scoped to issue write and repository read.

Out of scope:

- Any autonomy beyond the skill's table: not-planned closes, PR merges, thread comments, body edits, pushes.
- Changing how issues are written (`issue-writing.md`), or reviving the retired `allod pm` board. The forge is the board.
- Reports posted to the forge. The owner ruled that a rolling tracking issue bloats and gets ignored.

## Decisions taken with the owner (2026-09-09)

- Runtime: a systemd user timer on a dev VM, not the host, not a cloud routine.
- Model: `finite/deepseek-v4-flash-0731-thinking` at thinking `medium`, self-hosted, so inference is free and runs are bounded by `timeout`.
- Autonomy: report, label, and close only on a merged `Closes`. `Refs` is reported, never closed on, because memory reserves it for the non-final PRs of a chain.
- Output: email through an msmtp relay with an agenix-delivered credential.
- Scope and cadence: every public allod repo, weekly.
- Portability: one skill in `allod/memory/skills`, the timer is one caller.

## Slices

| Slice | Delivers | Witness | Unblocks |
| --- | --- | --- | --- |
| 1 memory | The skill | A by-hand run from Claude Code over `allod/tools` reproduces the 2026-09-09 manual pass: the same five closes flagged, the site chain under ready-to-merge, the three decisions listed | 3 |
| 2 tools | `forge pr list -s open\|closed\|all` with a merged marker | Fixture tests: a closed-unmerged PR is not listed as merged; page-2 results appear (pagination is allod/tools#89; land the flag on one page and say so, or land pagination first) | 3 |
| 3 archetypes | `forge-groom.nix`: timer, wrapper, msmtp config, options for token path, credential path, recipient, sender, relay, repos, schedule | Generated unit and timer inspected; a manual `systemctl --user start forge-groom.service` on a VM with a stub relay writes the report and produces one mail in the stub | 4 |
| 4 private | Enables the module on one VM with real values; creates the scoped token and SMTP credential | First real email received; two consecutive weekly runs compared by hand against the trackers before any autonomy is widened | done |

Contracts bind when a slice starts. Slice 3's option names are written in its PR, informed by what slice 1 needed on the command line.

## Risk Assessment

Residual risk: R3 High

Why:

- The timer mutates shared forge state (issue closes, labels) on a schedule with no human in the loop, using a model weaker than the ones that write the code. The close rule is narrow and every close names its merged PR, but a wrong close is a real, if reversible, change to a shared tracker.
- A systemd unit with two credentials: the forge token and the SMTP password. Both are agenix-delivered and read from files; the wrapper must not export either into the model's environment, and the skill's `--no-*` battery keeps repository content from loading extensions.
- Rollback is a straight revert of the enabling PR plus a token revocation; reopening a wrongly closed issue is one forge command.

Human scrutiny:

- The skill's action table and the wrapper's environment: what the model can reach, and that the token it holds cannot push or merge.
- The generated unit: `Environment=` lines, `LoadCredential=` or file paths, and that a failed run mails a failure line rather than nothing.
- The first two real emails, read against the trackers.

| PR or milestone | Risk | Reason | Human scrutiny |
| --- | --- | --- | --- |
| 1 memory skill | R1 | Prose contract, no runtime | Action table completeness |
| 2 forge pr list -s | R1 | Read-only flag, fixture-tested | Merged marker correctness |
| 3 archetypes module | R3 | Systemd unit, two credentials, mail path | Environment, credential paths, failure mailing |
| 4 private enable | R3 | Live token with write scope on a schedule | Token scope on the forge, first emails |

## Interface Contracts

- Skill to caller: the skill reads its repo list and scratch path from the prompt and writes one plain-text report to `$GROOM_REPORT` or stdout; exit non-zero with a one-line report when the forge is unreachable or the token fails. Mailing is the caller's.
- Wrapper to skill: `FORGE_TOKEN_FILE` names the scoped token file; `FORGEJO_TOKEN` is never set (forge refuses it after allod/tools#178). `HOME` is the service user's, so `pi` finds its own provider configuration.
- Token scope: Forgejo token with `write:issue` and `read:repository` only. No `write:repository`, no `write:package`, no admin. Verified with `forge auth status` and by a push attempt that must fail.
- Mail: the wrapper pipes the report to `msmtp -t` with a fixed `From:`, `To:`, and `Subject: forge groom <date>: <summary line>`. A run that produced no report mails the unit's last 20 log lines under `Subject: forge groom <date>: FAILED`.
- `forge pr list -s`: `-s open` is today's output unchanged; `-s closed` and `-s all` add a state column and mark merged PRs `merged`; the flag mirrors `issue list -s`.

## Agent Gates

- Creating the scoped forge token, the SMTP credential, and their agenix entries: human, at the forge and in the private secrets repo. Blocks slice 4.
- Choosing the SMTP relay and sender address: human. Blocks slice 3's real-relay witness; the stub-relay witness needs neither.
- Rebuilding the VM after slice 4: human, host-side. Blocks the first real run.
- Creating the five groom labels once per repo: human or agent with label write; the skill reports missing labels rather than creating them.

## Acceptance Tests

```sh
# Slice 1: by-hand run reproduces the manual pass
# (from Claude Code in ~/work, after reading the skill)
#   "Run forge-groom over allod/tools, report only, no actions."
# Compare against the 2026-09-09 pass recorded on allod/strategy#48.

# Slice 2
cd ~/work/allod/tools && nix flake check
forge -R allod/tools pr list -s closed | grep -c merged

# Slice 3: generated artifacts on a dev VM with the module enabled
systemctl --user cat forge-groom.service forge-groom.timer
systemctl --user show forge-groom.service -p Environment -p LoadCredential
GROOM_RELAY_STUB=1 systemctl --user start forge-groom.service && ls "$XDG_RUNTIME_DIR"/forge-groom/report.txt
# a deliberately unreachable forge URL must produce the FAILED mail, not silence

# Slice 4
forge auth status            # names the groom token file
git push origin HEAD:refs/heads/groom-push-probe   # must be refused by the forge
```

## Rollback Plan

Disable the timer by reverting the private enabling PR and rebuilding; revoke the groom token at the forge. Reopen any wrongly closed issue in the web UI (`forge` has no reopen command yet) using the close comment, which names the PR the groom believed settled it. Slices 1 and 2 are inert without slice 4 and need no rollback.
