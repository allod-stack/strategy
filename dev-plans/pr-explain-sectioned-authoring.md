# PR Explain: Sectioned Authoring

## Tracking Issue

https://forge.anarch.diy/allod/tools/issues/138 — one implementation PR in `allod/tools`, `Refs` only. The issue tracks the report program across slices and closes later.

Trigger: the ninth datapoint on the issue. Three consecutive owner reads show report register decaying with output position despite the merged reader-contract prompt; the fifth datapoint pre-committed per-section authoring as the fix when that happened. Rationale and citations live on the issue, not here.

## Goal

A generated report holds the reader contract at full strength through its last section, because every section is authored by its own provider call with the contract re-anchored a short distance from the text it governs.

## Scope

In scope, all in `allod/tools`:

- `pr-explain/explain` and `pr-explain/lib.sh`: replace the single author pass with outline → per-section → quiz passes plus mechanical assembly.
- Prompt assets: split `pr-explain/prompt.md` into per-pass prompts sharing one reader-contract block; add the characterization paragraph.
- `docs/pr-explain.md` and `--help`: new pass table and provider-call ceiling.
- `tests/pr-explain/`: extend the mock-runner suite for the multi-call pipeline.

Out of scope:

- Validator rules, `report.css`, `report.js`, component gallery (unchanged).
- Reader model, quiz capture, card export (slice 2); SVG vocabulary (slice 3); primer library (follows this slice; its parked watch item is discharged by this plan).
- The Forgejo file-view iframe sizing defect (host-side; recorded on the issue).

## Risk Assessment

Residual risk: R2 Medium

Why:

- Behavior change is localized to report generation in one CLI; no secrets, provisioning, persistent state, or cross-repo interfaces move.
- The pipeline glue is deterministic and covered by the existing mocked-runner fixture suite, extended for the multi-call flow; every new mechanical guard is demonstrated to fail on sabotaged input.
- Integration uncertainty is real — `lib.sh` is large and the author pass is being restructured — but rollback is a straight revert and a failed run publishes nothing.
- The one thing local tests cannot witness is the goal itself: register quality of a real report, which needs a live provider run and an owner read.

Human scrutiny: the consent/disclosure text (call ceiling must be true), the assembly path (validator must see one complete body identical in shape to today's), and the next real report read.

## Interface Contracts

1. **Pipeline order**: triage → T0 short-circuit → outline → section calls in document order, strictly sequential → quiz-and-provenance call → mechanical assembly → slop → validate → at most one repair. All tiers use the same pipeline; T1 simply has few sections. `--force-tier`, `--no-slop`, `--no-repair`, `--dry-run` keep their meanings.
2. **Provider-call ceiling**: triage 1 + outline 1 + sections ≤ 12 (the triage `max_sections` cap) + quiz 1 + slop 1 + repair 1 = **at most 17 calls**; a typical run spends `sections + 5`. The pre-run consent text and `--help` state the ceiling; `docs/pr-explain.md` replaces the four-pass table.
3. **`outline.json`**, written by the outline call to the job dir and named by `ALLOD_PR_EXPLAIN_OUTLINE` for later calls:

   ```json
   {
     "sections": [ { "id": "kebab-case", "title": "...", "layer": "concept|mechanism|receipts",
                     "objectives": ["obj-1"], "concepts": ["slug"], "gist": "one sentence" } ],
     "notes": "free-form evidence notes for later calls (optional)"
   }
   ```

   Tool-side mechanical validation before any section call, failing the run closed: 1–`max_sections` entries; unique kebab-case ids; layer order never goes backward with at least one `concept`; every triage objective id claimed by at least one section; every listed objective and concept exists in `triage.json`. There is no outline repair pass.
4. **Front matter** is written by the outline call as its own fragment: skip link, masthead, summary, TOC, objectives section. Mechanical check: TOC hrefs resolve exactly to the outline's section ids, in order.
5. **Section call k** receives the shared contract block, the job files, `outline.json`, and every previously captured fragment verbatim, and writes exactly one `<section>` fragment to its own tool-named path (`ALLOD_PR_EXPLAIN_REPORT_BODY` points at that call's fragment, per call). Mechanical check per fragment before the next call runs: non-empty regular file; opening tag carries the outline's id, `data-layer`, and `data-objective`; no document-shell, `script`, or `style` tags. Full grammar remains the assembled-body validator's job.
6. **Quiz-and-provenance call** receives the complete assembled-so-far body and writes the final quiz section and the footer (provenance, sources, limits). All quiz items live in that final section; the global quiz rules (3–7 items, letter balance, objective coverage, design-reasoning item) bind one call that can see them whole.
7. **Assembly is code, not a provider call**: the tool concatenates front matter + section fragments in outline order + the final fragment into the staged body. Slop, validation, and repair operate on that assembled body exactly as they do today; repair prompts and cage postconditions are unchanged in kind, with postconditions re-checked after every provider call.
8. **Prompt assets**: the reader contract, voice rules, and a new short characterization paragraph (the register anchor, in the spirit of the plan-review template's reviewer intro) live in one shared block the tool prepends to every authoring prompt; per-pass prompts carry only their pass's rules (heading and TOC rules with the outline pass, learning mechanisms and component vocabulary with the section pass, quiz rules with the quiz pass). No rule is duplicated across pass prompts.
9. **Failure behavior**: any invalid or missing artifact fails the run before the next provider call, preserving the job dir with per-call captures and diagnostics; the previous report stays byte-for-byte untouched.

## Agent Gates

- The implementing agent runs no live provider. The decisive witness for the goal — register held to the last section — is the owner reading the next real report, generated from the branch via the tools-checkout fallback before merge or after it.
- Merging the PR is the human's act, as always.

## Acceptance Tests

```sh
cd allod/tools
bash tests/pr-explain/command.sh
bash tests/pr-explain/validation.sh
bash tests/pr-explain/components.sh
```

New cases the suite must carry, each guard proven by a sabotage fixture that fails:

- Full-pipeline fixture: scripted mock runner serves triage, outline, N sections, quiz, and slop in order; the run publishes; the assembled body passes `allod pr _validate-report`; the call count and per-call environment (fragment path, outline path, prior fragments present) are asserted.
- Outline sabotage: malformed JSON, duplicate ids, backward layer order, and an unclaimed objective each fail the run before any section call, preserving diagnostics.
- Fragment sabotage: wrong section id, missing `data-layer`, and an embedded `script` tag each fail at the mechanical check.
- Interruption mid-pipeline: killing the runner during a section call fails closed with the job dir preserved and no output written (extends the existing interruption harness).
- Disclosure: `--help` and the pre-run consent text state the 17-call ceiling; the old "at most four provider" assertions are replaced, not deleted.
- Repair path: a validator-rejected assembled body triggers exactly one repair call on the full body, and a repaired body publishes.

## Rollback Plan

Revert the implementation PR in `allod/tools`; the pipeline returns to the four-pass form. No persistent state, migrations, or cross-repo interfaces are involved. Partially failed runs already preserve their job dir and leave the destination untouched, so no cleanup path changes.
