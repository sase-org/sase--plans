---
tier: tale
title: Stop research-highlights from capturing non-report Markdown
goal:
  Only month-root research reports and canonical <name>/<name>.md topic reports reach
  bob ref create.
size: small
decisions:
  topic_dir_scope:
    ask:
      Should bob_research_capture also skip topic-dir Markdown other than
      <name>/<name>.md?
    choices:
      canonical: Notes like acceptance.md and epic-agent drafts never become refs
      depth_only:
        Only asset folders and deeper files are blocked; acceptance.md-style notes still
        become refs
    default: canonical
    why:
      "The research layout publishes one reader-facing file per topic dir:
      <name>/<name>.md"
    answer: canonical
  vault_cleanup:
    ask: Should the implementer propose removing the junk refs already in the Bob vault?
    choices:
      report:
        Vault untouched; the final summary lists each junk ref and task anchor to
        abandon
      gate:
        One /sase_gate proposes removing each junk ref's note, PDF, source, and task for
        approval
    default: report
    why:
      Vault edits sync to three machines; you already abandoned the Oct 7 README by hand
    answer: gate
proposed_by: bbugyi200.apollo.b8
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 11:58:07
status: wip
---

# Stop the research-highlights hook from capturing non-report Markdown

## Problem

The `research-highlights` file hook runs `bob_research_capture`, which runs
`bob ref create`. Over the last ~18 hours it captured Markdown files that are not
research reports. Each one became a Bob reference with a Highlights PDF and a reading
task.

Evidence comes from the file-hook batch records under `~/.sase/file_hooks/batches/`,
their run logs, and Bob vault commits titled `bob ref create: ingest chat/<id>`. All
times are EDT. Refs created in the window:

| Time                        | Agent              | Captured file (research sidecar)                                                                                 | Ref                                    | Verdict                          |
| --------------------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------- | -------------------------------------- | -------------------------------- |
| 10-29 18:45                 | research.2w.linker | `202610/bob_gcal_daily_events/bob_gcal_daily_events.md`                                                          | bob_gcal_daily_events                  | wanted                           |
| 10-29 18:51                 | research.2u.final  | `202610/ref_create_table_layout_for_highlights/ref_create_table_layout_for_highlights__final_assets/README.md`   | README_2                               | **junk**                         |
| 10-29 18:51                 | research.2u.final  | `202610/ref_create_table_layout_for_highlights/ref_create_tables_vs_pdfkit_cell_selection__cld_assets/README.md` | README_3                               | **junk**                         |
| 10-29 18:51                 | research.2u.final  | `202610/ref_create_table_layout_for_highlights/ref_create_tables_vs_pdfkit_cell_selection__cld_assets/test.md`   | test                                   | **junk**                         |
| 10-29 19:19                 | research.2u.linker | `202610/ref_create_table_layout_for_highlights/ref_create_table_layout_for_highlights.md`                        | ref_create_table_layout_for_highlights | wanted                           |
| 10-29 20:14                 | bob-cli-eb.5       | `202610/ref_create_table_layout_for_highlights/pdfkit_table_selection_report_drafts.md`                          | pdfkit_table_selection_report_drafts   | **junk**                         |
| 10-30 02:01                 | bob-cli-eb.6--3    | `202610/ref_create_table_layout_for_highlights/acceptance.md`                                                    | acceptance                             | **junk**                         |
| 10-29 22:29, 23:49 (athena) | sase agents        | `202610/tui_gc_fixed_soak_20261029.md`, `202610/sase_1pt_1pu_5_1_epic_conflict_check.md`                         | same stems                             | month-root report, by design     |
| 10-30 11:36                 | b7 (tailnet)       | `202610/mac_slowdown_memory_pressure_and_test_io.md`                                                             | same stem                              | user-requested report, by design |

The same leak happened once before. The 2026-10-07 swarm commit `c7ee973` in the bob-cli
research sidecar added `__cdx_assets/README.md`, `demo.md`, and
`__final_assets/README.md`. That produced the `README` and `demo` refs, and Bryan
abandoned `README` by hand. On Oct 29 the README asset ran in `mode=reread` because
`bob_research_capture` resolved the bare stem `README` to that unrelated Oct 7 ref.

## Root cause

The template in `sase-research-artifacts` (`RESEARCH_HIGHLIGHTS_HOOK_SPEC` in
`src/sase_research_artifacts/provider.py`) has a **deny-list** path filter. It matches
`20*/**/*.md` (every Markdown file at any depth under a month dir), then vetoes only
`__` drafts at depths 1–2 and named companions (`_infographic`, `_narration`,
`.diagrams`, `*.png.md`, and so on). Any other Markdown under a month dir counts as a
report. Two kinds of writer produced such files:

1. **Swarm asset folders.** Researchers sometimes save supporting files in
   `<stem>__<suffix>_assets/`, and the lead writes `<name>__final_assets/`. Files inside
   them (`README.md`, `test.md`, `demo.md`) have no `__` in their basename, so no path
   veto fires. When a researcher commits its own assets, the agent-name veto
   (`!research.*.cld`, and so on) hides them. In commit `a1f5d5f` (Oct 29), though, the
   lead `research.2u.final` **moved** the cld researcher's assets into the topic dir.
   SASE splits renames into REMOVE + ADD, so the ADD is attributed to the mover.
   `research.*.final` is deliberately not vetoed, so the move re-admitted the files.
2. **Non-research agents writing working notes into a topic dir.** bob-cli epic phases
   `bob-cli-eb.5` and `bob-cli-eb.6--3` committed
   `pdfkit_table_selection_report_drafts.md` and `acceptance.md` next to the published
   report. These sit at depth 2 with no `__`, so nothing vetoes them.

Ruled out as causes:

- No configuration or code changed in the window. The template was last touched on Oct
  24, the chezmoi `file_hooks` entry and `bob_research_capture` on Oct 27, and every
  wanted capture between Oct 27 and 29 behaved.
- SASE dispatch matched exactly as configured. Every run matches the effective globs,
  and finalizer reconciliation reused existing batches (`batch_already_present`), so no
  duplicate runs happened.

The glob language (`wcmatch`) cannot express "basename equals parent dir name". So the
template can bound depth and veto asset folders, but `<name>/<name>.md` vs.
`<name>/acceptance.md` has to be decided by the command, which sees the absolute path.

## Changes

Both changes live in linked repos. Open each with `/sase_repo`
(`sase repo open sase-research-artifacts`, `sase repo open chezmoi`), read its
`AGENTS.md`, and work only in the printed path. No change is needed in the sase repo
itself.

### 1. sase-research-artifacts: allow only report depths in the hook template

In `src/sase_research_artifacts/provider.py`,
`RESEARCH_HIGHLIGHTS_HOOK_SPEC["file_hook"]["filters"]["path_globs"]`:

- Replace the positive glob `"20*/**/*.md"` with two depth-bounded positives:
  `"20*/*.md"` (month-root reports) and `"20*/*/*.md"` (topic-dir files).
- Add the veto `"!20*/**/*_assets/**"` so asset folders are excluded even at depth 2
  (for example, a solo `#research` agent writing `202610/foo_assets/README.md`).
- Keep the existing `!20*/*__*.md`, `!20*/*/*__*.md`, and companion vetoes, plus the
  `agent_name_globs`, `ops`, `producers`, and `sidecars` filters, unchanged.
- Leave `_RESEARCH_INVENTORY_GLOBS` (the `@research` ref inventory) unchanged; the
  divergence is intentional.
- Update the module docstring and the inline comments: the hook admits only month-root
  and one-level topic-dir Markdown and vetoes asset folders, and moved files are
  attributed to the mover. Say that telling `<name>/<name>.md` apart from other
  topic-dir notes is the command's job, because globs cannot compare a basename with its
  parent dir.

In `tests/test_filters.py`:

- Add a hook-only candidate set; do not touch `_CANDIDATES`, so the inventory test stays
  unchanged. It must include `202608/widgets/widgets__final_assets/README.md`,
  `202608/widgets/widgets__cld_assets/test.md`, `202608/widgets__cld_assets/README.md`,
  `202608/solo_assets/README.md`, and `202608/widgets/sub/notes.md`. Assert that every
  one is filtered, while `202608/widgets/widgets.md` and `202608/solo_report.md` stay
  allowed.
- Add a `hook_matches_event` regression that builds the hook from the full template
  filters (path and agent globs together). An `ADD` of
  `202610/topic/topic__cld_assets/README.md` attributed to `research.2u.final` must
  return `False`, and an `ADD` of `202610/topic/topic.md` by `research.2u.linker` must
  return `True`.
- Keep the existing assertions passing. `test_file_hook_globs_exclude_swarm_drafts`
  still expects `("202608/widgets/widgets.md", "202608/solo_report.md")` as the allowed
  set.

Docs: update the hook description in `README.md` ("The `research-highlights` file hook")
and `docs/configuration.md` ("Enabling the `research-highlights` file hook") to the new
`path_globs`, with one sentence on the asset-folder veto and the command-side canonical
check.

Use a `fix(research-highlights): ...` conventional commit so release-please cuts a patch
release for machines that install the published wheel. Verify with `sase tool run check`
in that repo; `check` is guarded, so do not run bare `just check`.

### 2. chezmoi: canonical-path guard in `bob_research_capture`

In `home/bin/executable_bob_research_capture`, add a guard that runs before any `bob`
invocation, right after the stable stem is derived:

- Find the nearest ancestor directory of the target path whose name matches `^20\d{4}$`
  (a research month dir). Use the path as given, made absolute with `os.path.abspath`;
  do not resolve symlinks. Artifact-store copies live under 14-digit timestamp dirs such
  as `.../20261030113020/` and never match.
- If there is **no** month ancestor, keep today's behavior unchanged. This covers manual
  runs and stored copies with hash suffixes, which the existing tests use.
- If there **is** one, take the path parts below it:
  - 1 part (`<YYYYMM>/<stem>.md`): proceed.
  - 2 parts: proceed only when the topic dir name equals the stable stem
    (`<YYYYMM>/<name>/<name>.md`).
    > [!decision] topic_dir_scope = depth_only Proceed for every 2-part path instead;
    > only 3+ parts are skipped.
  - 3+ parts: skip.
  - Under `canonical`, the only other topic-dir captures in the hook history are the Oct
    10 `sase-1jo.*` epic outputs
    `202610/databricks_nyc_cv_and_role_pitches/databricks_nyc_cv_and_role_pitches_interview_prep.md`
    and `..._v2.md`. Those would now be skipped as well.
- A skip prints one stderr line, such as
  `bob-research-capture: skip: <path> is not a research report (expected <YYYYMM>/<stem>.md or <YYYYMM>/<name>/<name>.md)`,
  and exits **0** without calling `bob`. The hook is non-gating, and a skip is not a
  failure.
- Update the module docstring to describe the guard.

In `tests/bash/bob_research_capture_test.sh`, add offline tests that use the existing
fake-`bob` harness and assert zero create calls plus exit 0 for:

- `202610/topic/topic__final_assets/README.md`
- `202610/topic/topic__cld_assets/test.md`
- `202610/topic/acceptance.md` (under `canonical`)
- `202610/topic/pdfkit_table_selection_report_drafts.md` (under `canonical`)

They must also assert one create call for `202610/topic/topic.md` and
`202610/solo_report.md`. Existing tests must keep passing unchanged. They write reports
directly under a `mktemp` dir with no month ancestor, which exercises the "no month
ancestor" branch. Run the bash suite with the repo's `test-bash` Justfile recipe (or the
narrowest equivalent for this file).

Deploy according to the chezmoi repo's `AGENTS.md` gotcha: run
`chezmoi update -a --force` once the commit has landed. If this turn cannot run it after
host finalization, propose it with `/sase_gate` and say so in the final response.

### 3. Verify the effective behavior

- Run `sase file-hook list --json` and confirm that the effective `research-highlights`
  `path_globs` show `20*/*.md`, `20*/*/*.md`, and `!20*/**/*_assets/**`. On apollo the
  plugin is an editable install from Bryan's primary `sase-research-artifacts` checkout,
  so the new globs take effect only once that checkout includes the commit. If the list
  still shows `20*/**/*.md`, report that the primary checkout needs a fast-forward; do
  not hand-edit it. Also say that athena picks up the template from its own install or
  the patch release.
- Replay the incident with the real matcher. Use a short Python snippet with
  `sase.config.file_hooks.hook_matches_event` and the updated template filters. Feed it
  the five junk paths from the table, with their original agent names, plus the two
  wanted linker paths. Report the results: junk is `False`, wanted is `True`.

### 4. Already-created junk refs

Junk refs: `README_2`, `README_3`, `test`, `pdfkit_table_selection_report_drafts`, and
`acceptance` from this window, plus `README` (already abandoned) and `demo` from Oct 7.
Their files are `ref/chat/<id>.md`, `lib/chat/<id>.pdf`,
`.bob/ref-sources/chat/<id>--<hash>/`, and a reading task anchored `^ref-<id-kebab>` in
`bob.md` or `done/bob_done.md`.

> [!decision] vault_cleanup = report Do not touch `~/bob`. List these refs and their
> task anchors in the final response.

> [!decision] vault_cleanup = gate Read the `obsidian` reference memory first. Propose a
> single `/sase_gate` with an explicit, reviewable command list that removes exactly
> these refs' notes, PDFs, saved sources, and task lines, then runs `bob vault-sync`.
> Never delete anything outside the gate.

## Out of scope

- Changing the `#research` or `#research_swarm` prompts so agents stop creating
  `_assets` folders, or bob-cli epic conventions about writing into research topic dirs.
  The filter fix makes those harmless.
- A generic SASE file-hook filter feature for backreference-style path matching. One
  consumer does not justify that machinery.
- The `@research` ref inventory globs.
