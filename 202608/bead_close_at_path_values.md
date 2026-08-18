---
tier: tale
title:
  Expand `@<path>` on the four bead free-text values that commit 771454166 missed, and
  recover the two close summaries it silently dropped
goal:
  "`sase bead close -n/-r`, `sase bead +1 -n`, and `sase bead snooze -r` read `@<path>`
  from that file like every other bead free-text value; a missing or unreadable path is
  a loud exit-1 that mutates nothing; a new option cannot be added to this family
  without a test forcing the decision; and the two 2026-08-18 close summaries currently
  stored as the literal strings `@/tmp/notes/close.txt` and `@/tmp/p5_close.md` are
  readable on `sase-pv.7` and `sase-p5` again."
size: medium
proposed_by: bbugyi200.athena.070
create_time: 2026-08-18 18:46:23
status: wip
---

# Plan: finish `@<path>` for bead free-text values

## Problem

`sase bead show sase-pv.7` ends on a note whose entire body is a file path:

```
[2026-08-18T22:29:19Z · sase-pv.7.f0] @/tmp/notes/close.txt
```

The stored note really is that 21-character literal. The 2.0 KB migration summary the
closing agent wrote into that file — the record of how the five flag beads were actually
migrated, which IDs replaced which, what was verified, and the tombstoned-stream
carry-over that `sase-pv.8` depends on — was never stored on the bead.

Commit `771454166` ("feat(bead): expand `@<path>` on free-text CLI values", 2026-08-18
13:49 EDT) was supposed to have fixed exactly this class of bug. It added the shared
resolver `read_at_path_value` in `src/sase/cli_file_values.py` and wired it into five
callsites. The `sase-pv.7` note was written at 18:29 EDT — **4h40m after that fix
landed** — and still stored the raw token, because the note's timestamp is identical to
the bead's close timestamp: it came from `sase bead close -n`, which the fix never
touched.

### What is and is not covered today

| Value                            | Handler                     | Expands? |
| -------------------------------- | --------------------------- | -------- |
| `bead create -d/--description`   | `cli_crud_create.py:126`    | yes      |
| `bead update -d/--description`   | `cli_crud_update.py:71`     | yes      |
| `bead update -n/--notes`         | `cli_crud_update.py:76`     | yes      |
| `bead note <id> <single-token>`  | `cli_crud_evidence.py:110`  | yes      |
| `-f/--field` (create and update) | `task_types/fields.py:113`  | yes      |
| **`bead close -n/--note`**       | `cli_crud_lifecycle.py:102` | **no**   |
| **`bead close -r/--reason`**     | `cli_crud_lifecycle.py:108` | **no**   |
| **`bead +1 -n/--note`**          | `cli_crud_evidence.py:49`   | **no**   |
| **`bead snooze -r/--reason`**    | `cli_crud_snooze.py:48`     | **no**   |

### This has already cost two real close summaries

Scanning every event stream in the bead store for a single-token value beginning with
`@` returns eight historical hits. Six were descriptions, and all six were repaired by
the prior plan — the current `issues.jsonl` projection has no literal `@` description.
The two that remain live are both `close -n` notes, i.e. both are this bug:

| Bead        | Status | Note stored             | Source file            | Size   | Written          |
| ----------- | ------ | ----------------------- | ---------------------- | ------ | ---------------- |
| `sase-p5`   | closed | `@/tmp/p5_close.md`     | `/tmp/p5_close.md`     | 6.8 KB | 2026-08-18 07:47 |
| `sase-pv.7` | closed | `@/tmp/notes/close.txt` | `/tmp/notes/close.txt` | 2.0 KB | 2026-08-18 18:29 |

**Both source files still exist as of this writing, so both are recoverable now.**
`/tmp` is cleared on reboot, so this window is not durable — the appendix embeds both
files verbatim so the repair survives one.

The `sase-p5` loss is the more expensive of the two: the missing 6.8 KB is that epic's
entire landing record — per-phase commit SHAs, the replay of all nine recorded
`dirty_work_discarded` failures against the landed code, and the disposition of every
child follow-up (which were filed, which were already fixed, which were declined and
why). None of that is reconstructible from the diff.

## Diagnosis

### Root cause: the resolver was wired per-callsite, so three handlers were simply never visited

`read_at_path_value` is correct and complete. Nothing about it is per-option. The defect
is purely in its distribution: commit `771454166` enumerated the callsites it knew about
and edited those five, and there is nothing in the codebase that would notice a sixth.
`create`/`update` live in `cli_crud_create.py`/`cli_crud_update.py`, which the commit
touched; `close`/`snooze` live in `cli_crud_lifecycle.py`/`cli_crud_snooze.py`, which it
did not. `+1` and `note` share `cli_crud_evidence.py`, and only `note` was wired — the
handler imports the resolver at line 16 and uses it once, twenty lines below an
unresolved `args.note`.

Concretely, `handle_bead_close` reads both values straight off the namespace inside the
mutation block and hands them to the store:

```python
note = getattr(args, "note", None)          # cli_crud_lifecycle.py:102
...
closed = mutation.project.close(
    resolved_ids,
    reason=args.reason,                     # :108
    ...
    note=note,
)
```

`handle_bead_plus_one` does the same with `args.note` at `cli_crud_evidence.py:49`, and
`handle_bead_snooze` with `getattr(args, "reason", None) or ""` at
`cli_crud_snooze.py:48`.

### The failure is silent by construction

None of these three values is validated, so a path that does not exist is stored just as
happily as one that does. The user sees `✓ Closed: sase-pv.7` and a `Noted:` line, the
mutation commits and pushes, and the discrepancy only surfaces when someone later reads
the bead. Every one of the eight historical hits was written by an agent that believed
it had succeeded.

### Why the fix is Python-only

This does not cross the Rust core boundary. `execute_bead_cli` in
`crates/sase_core/src/bead/cli.rs:89` matches `list`, `show`, `search`, `ready`,
`blocked`, `stats`, `create`, `open`, `ref`, `update`, `close`, `dep`, `rm` — there is
**no arm for `+1`, `snooze`, or `note`**, so all three hit `_ => Ok(defer())` and land
in the Python handlers. And `close`, although Rust implements it, is force-deferred
before Rust is ever consulted (`bead_fast_path.py:35`, for unrelated close-rendering
reasons). So all four broken values are reached only through Python today.

That is a fact about the present, not a guarantee. `_argv_requests_at_path` exists
precisely because `update` _is_ Rust-handled, and its allowlist is `{"update", "note"}`
— if someone later adds a Rust `+1` or `snooze` arm, the raw token silently comes back.
The guard should be widened now, while the reason is in view.

## Scope

In scope:

- `@<path>` / `@@` expansion on `close -n`, `close -r`, `+1 -n`, `snooze -r`.
- A regression guard so option number nine cannot repeat this.
- Parser help, docs, and completion-snapshot churn for the four options.
- Recovering the two lost close summaries.

Out of scope:

- **Multi-token `bead note` text.** `handle_bead_note` resolves only when
  `len(text) == 1`, and the parser help deliberately documents it as "a single-token
  `@<path>`". An unquoted `@/tmp/a.md and more words` should stay literal; changing that
  is a separate decision.
- **`-t/--title`, `-a/--assignee`, `-D/--design`, `-x/--external-ref`.** These are short
  identifiers, not prose. Step 5's guard records them as deliberate exclusions rather
  than leaving them unclassified.
- **Any `sase-core` change.** See the diagnosis above.
- **Rewriting the two bad notes in place.** The store is append-only, and `sase-pv.7`'s
  own `BLOCKED` note is the record of what happens to an agent that tries to edit event
  history. Step 8 appends the recovered text instead.

## Implementation

Follow the established shape at `cli_crud_update.py:68-81` for every handler: resolve
_before_ opening `bead_store_mutation`, catch `CliFileValueError`, print
`f"Error: {exc}"` to stderr, and `sys.exit(1)`. Resolving first is what makes "a bad
path mutates nothing" true rather than merely likely.

### 1. `handle_bead_close` — `src/sase/bead/cli_crud_lifecycle.py`

Resolve `args.note` (target `-n/--note`) and `args.reason` (target `-r/--reason`) at the
top of the function, before `with bead_store_mutation(...)`, and use the resolved values
at lines 102 and 108.

Note the interaction with the already-closed path: re-closing a bead with a `--reason`
that disagrees with the recorded one exits non-zero (`docs/beads.md:1067`). That
comparison must run against the **expanded** text, which resolving before the mutation
block gives for free — do not move the resolution inside.

### 2. `handle_bead_plus_one` — `src/sase/bead/cli_crud_evidence.py`

Resolve `args.note` (target `-n/--note`) before `with bead_store_mutation(...)` and pass
the resolved value to `project.plus_one` at line 49. The module already imports
`CliFileValueError` and `read_at_path_value` at line 16; no new import is needed.

`+1 -n` is `required=True` and its whole purpose is prose evidence, so this is the
option in the set most likely to be handed a file.

### 3. `handle_bead_snooze` — `src/sase/bead/cli_crud_snooze.py`

Resolve at line 48, where `reason` is already read, keeping the `or ""` default for the
absent case. This is above both the `--cancel` conflict check and the
`bead_store_mutation` block, so ordering is already correct.

One ordering decision to make explicitly: `sase bead snooze --cancel -r @/nope.md`
currently fails the "`--cancel` takes no wake conditions" check. Resolve the path
_after_ that check so the user gets the accurate complaint about `--cancel` rather than
a file-not-found error for a value that was going to be rejected anyway. Cover this with
a test.

### 4. Widen the fast-path guard — `src/sase/main/bead_fast_path.py`

Replace the inline `{"update", "note"}` at line 42 with a module-level frozenset naming
every verb that carries an `@<path>` value —
`{"+1", "close", "note", "snooze", "update"}` — with a comment saying it must list every
such verb regardless of whether Rust currently implements one.

Today this only skips a Rust round-trip that would have deferred anyway (and `close`
already returns earlier), so it is behavior-preserving. Its value is that a future Rust
`+1` arm cannot silently reintroduce raw-token storage.

### 5. Regression guard — `tests/test_bead/test_cli_at_path_values.py`

The point of this plan is that enumerating callsites by hand failed once already. Add a
test that walks the real argparse tree from `create_parser()` and asserts that every
free-text `bead` option is in exactly one of two explicit sets:

- **expanded** — `create -d`, `create -f`, `update -d`, `update -n`, `update -f`,
  `note`, `close -n`, `close -r`, `+1 -n`, `snooze -r`
- **deliberately literal** — `-t/--title`, `-a/--assignee`, `-D/--design`,
  `-x/--external-ref`, and the rest, each with a one-line reason in a comment

Fail on any option in neither set. A tenth free-text option then cannot be added without
someone classifying it, which is the check that was missing when `close -n` was added.

Keep it structural: assert over option `dest`s and the parser tree, not over help
strings, so it does not double as a brittle copy of step 6.

### 6. Parser help — `src/sase/main/parser_bead_lifecycle.py`

Append the existing `_AT_PATH_READS_IT` constant (line 9) to the help for `close -n`
(line 116), `close -r` (line 131), `+1 -n` (line 47), and `snooze -r` (line 454). Do not
write the sentence out by hand — reuse the constant, which is what makes the phrasing
identical to `create`/`update` and keeps `tests/main/test_parser_command_help.py:372`
meaningful.

Also add `f"Free-text values accept {AT_PATH_PREFIX}<path>."` to the `close` parser
description, matching `create` (line 156). Extend
`test_bead_create_and_update_help_document_at_path` (or add a sibling) to cover the four
new options, and refresh the completion snapshot with `just sync-completion-spec` — the
`close`/`+1`/`snooze` `description_digest` entries in
`tests/completion/snapshots/cli_spec.json` will drift.

### 7. Docs — `docs/beads.md`

Extend the option-table rows for `-n, --note` (line 1114) and `-r, --reason` (line 1116)
under `sase bead close`, the `-r, --reason TEXT` row under `sase bead snooze` (line
1621), and the `sase bead +1` prose at line 1036, each noting that `@<path>` reads the
value from that file. Match the wording already used for `-f, --field` at line 1127.

`src/sase/xprompts/skills/sase_new_task.md` needs no change: it documents only
`create -d` and `-f`, both of which stay accurate.

### 8. Recover the two lost close summaries

Do this **first**, before any code change, and confirm each `/tmp` file's bytes match
the appendix before using it — the appendix is the fallback if `/tmp` was cleared, not
the preferred source.

For each bead, append the recovered text as a note attributed to the recovering agent,
prefixed with one line explaining the provenance, e.g.:

> RECOVERED: the note above this one was stored as the literal token
> `@/tmp/notes/close.txt` because `sase bead close -n` did not expand `@<path>`. This is
> that file's contents.

- `sase-p5` ← `/tmp/p5_close.md` (6.8 KB, appendix A)
- `sase-pv.7` ← `/tmp/notes/close.txt` (2.0 KB, appendix B)

Both beads are closed; appending a note to a closed bead is a normal non-cascading
operation and must not reopen either one. Verify with `sase bead show <id>` that status
is still `closed` afterward.

Do not attempt to remove or rewrite the two literal-token notes. The store's append-only
integrity guard rejects it — `sase-pv.7`'s `BLOCKED` note documents thirteen failed
attempts and a stream-corrupting failure mode.

Once both notes are on their beads, `/tmp/p5_close.md` and `/tmp/notes/close.txt` are
redundant; leave them alone rather than cleaning up.

### 9. Tests — `tests/test_bead/test_cli_at_path_values.py`

Add cases alongside the existing `create`/`update`/`note` coverage, reusing the
`project_dir` fixture and the `_flake_create_argv` helper. For each of the four options:

- a file's contents are stored, and the stored value is not the `@` token
- `@@literal` stores one literal leading `@`
- a missing path exits 1 **and the bead is unchanged** — assert against the reloaded
  store, not just the exit code, since "mutates nothing" is the part that is easy to
  break

Plus the specific cases the general shape misses:

- `close -r @path` on an **already-closed** bead compares the expanded text against the
  recorded reason (step 1's ordering constraint)
- `snooze --cancel -r @/nope.md` reports the `--cancel` conflict, not a file error (step
  3's ordering decision)
- `+1 -n @path` — resolved before the `--verified-after-close` status check, so a bad
  path on a non-closed bead reports the file error without a partial mutation

## Verification

- `just check` green. Expect churn in `tests/completion/snapshots/cli_spec.json`,
  `tests/main/test_parser_command_help.py`, and `docs/beads.md`.
- Because this touches the parser tree and the completion snapshot, finish with
  `just check-full` through `/sase_monitor` (never inline) before landing.
- `sase bead show sase-p5` and `sase bead show sase-pv.7` each render their recovered
  summary, and both remain `closed`.
- Re-run the store scan from the Problem section: zero single-token `@` values in the
  `issues.jsonl` projection's `notes`, `close_reason`, and `description` fields.
- Manual smoke on a throwaway bead: `sase bead close <id> -n @/tmp/x.md` stores the
  file, and `sase bead close <id> -n @/tmp/does-not-exist.md` exits 1 leaving the bead
  open.

## Risks

- **`/tmp` is cleared before step 8 runs.** Mitigated by appendices A and B, which are
  the verbatim bytes.
- **Snapshot drift beyond the four options.** `just sync-completion-spec` rewrites the
  whole file; confirm the diff touches only `close`, `+1`, and `snooze` digests.
- **Over-broad expansion.** `@` is a plausible first character for real prose (`@agent`,
  an email). The `@@` escape is the documented out and is already tested; step 5's
  explicit "deliberately literal" set is what keeps the blast radius from growing by
  accident.

## Appendix A — verbatim `/tmp/p5_close.md`, for `sase-p5`

```text
LANDED.

VERIFIED (step 1) — read every child note and the source, not just the reports. All
five phases closed done, each with a landed commit: sase-p5.1/22e5444bf (restamp),
sase-p5.2/1519d20f2 (ledger), sase-p5.3/0d3621714 (guard), sase-p5.4/aaa09eba9
(shared), sase-p5.5/af951d1f9 (hardening). Confirmed in the code at HEAD af951d1f9:
_restamp_missing_footer_tags runs before finalize_commit and aborts with HEAD
unpushed when it cannot restamp (workflow_resume.py:147-218); resolve_head_commit_sha/
resolve_head_tree_id are threaded through both write_result_marker call sites AND the
resume path after finalize_commit, closing the CONFLICT-originated "result": null gap
(workflow_resume.py:112-113, workflow.py:274-275/397-398/432-433,
commit_tracking.py:531-578); discarded_dirty_work_evidence now accepts footer OR
agents-sync TYPE OR a ledger SHA/tree match, degrading to footer-only on any unreadable
ledger (commit_finalizer_git_progress.py:313-430); _published_store_state_is_exempt
covers kind="external" as well as "sdd", treats a foreign SASE_AGENT footer as a race,
and drops the strict ahead==0 requirement, behind commit_finalizer_shared_clone_exempt
(default on, flag bead sase-pk, registry + schema entries present) with
_legacy_published_store_state_is_exempt retained as the kill switch; kind="main"/"sibling"
is still never exempt, so a genuine discard still fails. discarded_dirty_work_message
now names repo, HEAD before/after, each newly reachable commit with its attributed
agent, whether the ledger was consulted, and a reason-specific next step. Docs refreshed
(docs/commit_workflows.md resume step 4 + new "Discarded-Work Guard" section). Tests:
77 pass across the nine affected suites, including the three real-git end-to-end
regressions in test_commit_finalizer_resume_provenance_e2e.py.

VERIFIED (plan's own acceptance test) — replayed all nine recorded dirty_work_discarded
failures from the project's artifacts tree. Counts still match the plan exactly (3284
finalized/clean_after_pass, 55 dirty_after_max_passes, 9 dirty_work_discarded = 8
missing_agent_provenance + 1 head_not_advanced). Classified against the landed code:
five name the machine-wide agents sidecar and one names an sdd beads clone -> all six
are now race/published exemptions (Cluster B); the remaining two name the agent's own
"main:" workspace and both are method=resume runs whose commit_result.json recorded
result: null with no SHA -> exactly Cluster A, now covered by restamp plus the resume
ledger write. The single head_not_advanced case still produces evidence and still fails,
unchanged as the plan required. Live confirmation: every finalizer run since the epic
landed writes real commit_sha/commit_tree into commit_results.json (e.g. e4319dbb4,
aaa09eba9, 134839e82), and zero new dirty_work_discarded failures have been recorded
since 2026-08-17 17:43, before the first fix landed.

VERIFIED (step 2, integration) — reviewed all 48 non-epic commits landed since
22e5444bf. Only one touches any file this epic touched: 11fddd525 added the
epic_resume_gate member to feature_flags/registry.py, orthogonal to the
commit_finalizer_shared_clone_exempt member added by aaa09eba9; both are present and
correctly ordered. No commit duplicates or conflicts with the ledger, the guard, or the
restamp; nothing outside the finalizer reads commit_results.json's new fields yet; the
message-only git_log_commit_messages helper was fully replaced by git_log_commit_records
with no dangling callers. Nothing needed rewiring.

VERIFICATION GATES — just fmt-py-check clean; targeted suites 77/77; full just test
33079 passed, 12 skipped, 1 failed. The one failure and the one lint failure are both
pre-existing and unrelated (see below); did not run just check-full, since its symvision
gate is red on that same unrelated debt regardless of this epic. sase bead epic-symbols
sase-p5 is empty and the only remaining Justfile --epic-symbol entries are keyed to
sase-n4 / sase-n4.5, both still in progress, so no re-keying or cleanup was owed.

FOLLOW-UPS (every child PROPOSED FOLLOW-UP plus the epic's own DISCOVERED ISSUE,
resolved):
- sase-p5.1--1, sase-p5.1--2, sase-p5.3 all proposed removing stale --epic-symbol
  entries for closed bead sase-p1.2/sase-p1.6 (six Glossary* symbols). ALREADY FIXED by
  another agent before this landing; verified gone from the Justfile. No task filed.
- sase-p5.5 proposed fixing a just fmt-py-check failure on src/sase/amd/_memory.py:398-400.
  ALREADY FIXED; just fmt-py-check reports 6975 files already formatted. No task filed.
- sase-p5.5 proposed resolving two unused public symbols (long_memory_entry_path,
  normalize_long_memory_description_lines in src/sase/amd/_agents_doc.py). STILL LIVE and
  already filed by another agent as sase-pm; corroborated with an independent
  reproduction on this tree (+2 reports) rather than filing a duplicate.
- This epic's own DISCOVERED ISSUE (05l): live flag bead sase-pk had no definition on
  other agents' trees, reddening their feature-flag lint. The instance RESOLVED when
  aaa09eba9 landed (tools/check_feature_flags is green here). The underlying window
  recurs with every new flag and had been reported only as notes on four separate epics
  (sase-p5, sase-oo.2, sase-oo.3, sase-p3.10), so filed it as sase-pp (bug, medium).

ALSO FILED:
- sase-pn (ci, small) — tests/main/test_init_memory_glossary.py::
  test_memory_plan_renders_glossary_terms_block_in_tier2 fails deterministically on
  master; the assertion expects the unwrapped literal "Aliases follow in parentheses."
  but the rendered Tier 2 block wraps it. Introduced by the non-bead commit 445afde7c,
  not by this epic; reproduced twice (in the parallel lane and in isolation).
- sase-po (feature, medium) — the one plan item this epic did NOT deliver: the
  "Also tighten the resume identity check itself" paragraph of phase restamp. sase-p5.1
  shipped the footer restamp but left the subject-only gate at
  workflow_resume.py:63-78 unchanged and recorded no descoping note. Filed rather than
  finished here because it is outside this epic's goal statement (which is about the
  discarded-work guard's false failures, not about finalizing an unrelated commit), the
  loose gate predates the epic, and closing it needs a new VCS-agnostic
  upstream-containment primitive across the provider protocol, the git plugin, and the
  test fakes. Recorded in the bead that the restamp does raise the stakes: HEAD is now
  stamped with this run's provenance before the push, so a wrong HEAD that clears the
  subject gate is mis-attributed rather than merely unattributed.

NOT FILED / DECLINED: nothing else. No child proposal was declined on the merits; the
three stale-lint proposals were already resolved and are recorded above instead.
```

## Appendix B — verbatim `/tmp/notes/close.txt`, for `sase-pv.7`

```text
Migrated the five live flag beads by the owner-directed create-new + delete-old route, replacing the plan's in-place event-stream rewrite (which the store's append-only guard makes impossible; see this bead's BLOCKED note). Each old bead was removed with `sase bead rm` and re-created through `create_flag_bead` -- the path `sase flag new` uses -- preserving title, size, kind and both removal thresholds exactly: sase-nw->sase-qe, sase-nx->sase-qf, sase-om->sase-qg, sase-pa->sase-qh, sase-pk->sase-qi. Each new bead carries a PROVENANCE note naming the bead it replaces. `when_enabled`/`when_disabled`/`remove_when` are the plan's drafts after per-flag verification against the code (the `coder_inherits_planner_chat` "On" text was corrected: the fork prefix is prepended in addition to the plan-file reference, not instead of it). Repointed the only two files that named the old IDs: `src/sase/feature_flags/registry.py` and `tests/feature_flags/test_consumers.py`.

VERIFIED: `sase bead list -T flag` shows all five with unchanged countdowns (88d/88d/89d/89d/90d, all v0.18.0) and sizes; `sase bead show sase-qe` renders the typed flag body block; pages regenerated for all five; `just _lint-flags` green; `sase bead doctor` reports nothing flag-related; `tests/feature_flags` + `tests/test_bead/test_flag_fields.py` 77 passed; `just test-scoped` 4700 passed, 1 skipped. `issues.jsonl` and the compatibility mirror have zero `issue_type: flag` rows. `just check` fails only on the pre-existing `_lint-toobig` violation in `tests/_suite_gate.py` (recorded as a PROPOSED FOLLOW-UP on this bead), which aborts the gate before the test lane -- hence the separate `just test-scoped` run above.

CARRY-OVER: `sase bead rm` leaves a tombstoned event stream, so five streams still hold a flag-typed `issue_created` event. sase-pv.8 has a note with the full analysis and the two constraints on fixing it; that phase must prune them before `IssueTypeWire::Flag` can be deleted.
```
