---
tier: tale
title: Make /sase_new_task find duplicates with bead search instead of a full task dump
goal:
  The /sase_new_task skill directs agents to detect semantic duplicates with a few
  targeted `sase bead search` queries instead of dumping every task bead, while still
  using `sase bead list` for in-progress epics.
proposed_by: bbugyi200.athena.wb
create_time: 2026-08-09 07:41:31
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.wb](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.wb.md)
- **COMMITS:**
  - [fcc9be4](https://github.com/sase-org/sase/commit/fcc9be44f2cf5ea8b5a23ab505d50f94a508f970)
    — fix: scope task duplicate detection to bead search

# Plan: Duplicate detection in `/sase_new_task` moves to `sase bead search`

## Problem

Step 3 of `src/sase/xprompts/skills/sase_new_task.md` tells every agent that files a
task bead to run:

```bash
sase bead list --type task --format full --limit 0 --status open --status claimed --status ready --status in_progress --status closed
```

Measured against the current bead store, that command emits **~71,000 words (roughly
90k+ tokens)** across 173 task beads, and the agent must then read all of it to judge
duplication. Every `/sase_new_task` invocation pays that cost before it can even decide
whether a new bead is warranted.

`sase bead search` is the purpose-built alternative. Verified behavior:

- Case-insensitive **literal substring** match (not regex/glob) over indexed text: ID,
  title, description, notes, plan path, artifact refs, owner, assignee, model, size,
  Patch name/bug ID, status, type, tier.
- Searches **every** status by default. Confirmed empirically that this includes
  `snoozed` (e.g. `sase bead search "Recalibrate the scoped"` returns snoozed
  `sase-gk`), which the current `--status ...` list in the skill omits entirely — so the
  switch is a strict **coverage improvement**, not just a cost cut.
- `--limit` omitted means unlimited, so no `--limit 0` is needed.
- Compact output (the default) is one row per hit plus a match-line snippet: a real
  query measured **~475 words for 20 hits** versus ~71,000 for the full dump.

The epic branch (step 4) is unaffected:
`sase bead list --type plan --tier epic --status in_progress --format full --limit 0`
costs ~5,500 words, is bounded by the small number of live epics, and has no query term
to search on. It stays exactly as written.

## Design decisions

**Multiple short queries, not one sentence.** Because the match is a literal substring,
`sase bead search "flaky retry test under parallel pytest"` will almost always return
nothing even when a duplicate exists. The instruction must push agents toward a handful
of short, distinctive terms — a symbol, filename, command, or error fragment. This is
the single highest-risk failure mode of the change (a false "no duplicate found"
produces the duplicate bead the skill exists to prevent), so it gets explicit prose
rather than being left implicit.

**No full-list fallback.** A `--format compact` list of all tasks (~2,600 words) was
considered as a "searches came up empty" backstop and rejected: compact rows show titles
only, whereas search already indexes descriptions and notes, so the fallback is not a
recall superset of two or three good queries — it would add per-invocation cost while
inviting agents back into scan-everything behavior. Recall is instead bought with query
diversity.

**Prose budget.** The replacement adds roughly three sentences of guidance to the skill
body. That is a deliberate trade: ~40 extra words of static instruction in exchange for
removing a ~90k-token command output from every invocation.

## Scope

Two files. Nothing else in the repo hard-codes the skill's step-3 command.

1. `src/sase/xprompts/skills/sase_new_task.md`
2. `tests/main/test_init_skills_sources.py`

Explicitly out of scope, and confirmed to need no edit:

- `sase/memory/sase_beads.md` and the memory templates under
  `src/sase/main/init_memory/templates/` say the skill "checks all task statuses" /
  "checks every task status for semantic duplicates". Those statements stay true —
  `sase bead search` covers every status by default. **Do not edit any memory file**;
  the user granted no memory-edit permission for this work.
- `docs/beads.md`, `docs/getting_started.md`, `docs/sdd.md`, and the blog posts
  reference `/sase_new_task` only by name and by its create/refine commands. None
  reproduce the duplicate-search command, so there is no doc drift to repair.
- Chezmoi deployment. Per `sase/memory/generated_skills.md`, a deploy must happen from a
  clean tree already landed on the canonical branch. The implementing agent should
  preview with `sase skill init --diff` (read-only) but must **not** run
  `sase skill init --force` or `chezmoi apply`; deployment happens after this change
  lands.

## Step 1 — Rewrite step 3 of the skill source

In `src/sase/xprompts/skills/sase_new_task.md`, replace the step-3 heading line and its
code block (currently lines 23–28) with the search-based version. Keep the rest of step
3 — the semantic-duplicate definition, the `sase bead +1` command, and the "Do not
create a task / a reporter counts at most once" paragraph — byte-for-byte unchanged.

Replace:

````markdown
3. Inspect every existing task status, then show plausible matches:

   ```bash
   sase bead list --type task --format full --limit 0 --status open --status claimed --status ready --status in_progress --status closed
   sase bead show <plausible-task-id>
   ```
````

With:

````markdown
3. Search existing task beads for prior reports, then show plausible matches:

   ```bash
   sase bead search '<distinctive-term>' --type task
   sase bead show <plausible-task-id>
   ```

   Search is a case-insensitive substring match across every status, closed and snoozed
   included. Run a few short, distinctive queries — a symbol, filename, command, or
   error fragment — because a whole sentence rarely matches verbatim. Do not list every
   task bead.
````

Authoring constraints:

- Prose wraps at the repo Markdown width of 88 columns (`DEFAULT_MARKDOWN_PRINT_WIDTH`);
  the new paragraph must respect it. The new command lines are well under that anyway,
  which also removes the file's current 136-column line.
- The new paragraph belongs before the existing "A semantic duplicate has the same
  underlying defect/root cause..." paragraph, so the sequence reads: command → how to
  query → how to judge a hit → how to corroborate.
- Change nothing in the frontmatter, in steps 1, 2, 4, or 5, or in step 4's
  `sase bead list --type plan --tier epic ...` invocation.

## Step 2 — Update the skill-content assertions

`tests/main/test_init_skills_sources.py` pins expected phrases per skill in the
`@pytest.mark.parametrize` block at line 138; the `sase_new_task` entry begins at line
219 and currently asserts the removed command:

```python
"sase bead list --type task --format full --limit 0",
```

Replace that single tuple element with assertions on the new command, e.g.:

```python
"sase bead search '<distinctive-term>' --type task",
```

Leave every other phrase in that tuple as-is — in particular
`"sase bead list --type plan --tier epic --status in_progress"`, which must keep passing
and is the guard that step 4 was not disturbed. Note that the test compares
whitespace-collapsed text (`_collapse_whitespace`), so wrapped prose phrases are safe to
assert.

## Step 3 — Add a regression guard against the full dump

Add one focused test to `tests/main/test_init_skills_sources.py`, modeled on the
existing negative-assertion test for the commit skills at line 422
(`assert "sase commit -M" not in body`). Load the shipped `sase_new_task` source the
same way the parametrized test does (`get_sase_package_skills_dir()`, then
`parse_yaml_front_matter`), collapse whitespace, and assert:

- `"sase bead search"` is present;
- `"sase bead list --type task"` is **absent** — this is the regression that matters,
  since it is the exact prefix of the ~90k-token dump;
- `"sase bead list --type plan --tier epic"` is still present, so the guard cannot be
  satisfied by deleting the epic step.

Give the test a docstring stating why: duplicate detection must stay query-scoped, and
the epic check must stay a list.

## Verification

```bash
just install
sase skill init --diff            # read-only preview of the rendered skill; do not --force
just check
```

`just check` covers the whole-repo lint gates plus the diff-scoped test lane, which will
select `tests/main/test_init_skills_sources.py`. If the scoped selection escalates or
reports anything unusual, fall back to `just check-full`.

Also sanity-check the new instruction against the live store to confirm the command in
the skill is copy-pasteable and cheap:

```bash
sase bead search 'flaky' --type task | head -20
```

## Done when

- `/sase_new_task` step 3 detects duplicates with `sase bead search --type task` and
  explicitly warns against listing every task bead.
- Step 4 still uses `sase bead list` for in-progress epics, unchanged.
- The skill's step-3 guidance tells agents to run several short, distinctive substring
  queries.
- `tests/main/test_init_skills_sources.py` asserts the new command, retains the
  epic-list assertion, and fails if `sase bead list --type task` ever returns to the
  skill.
- `just check` passes, and no memory file, doc, or chezmoi-deployed skill file was
  modified.
