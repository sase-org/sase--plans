---
tier: tale
title: Migrate build_and_run core memory into a lint_and_test reference note
goal:
  sase/memory/build_and_run.md is replaced by a type:reference
  sase/memory/lint_and_test.md whose description makes reading it mandatory for any
  agent that changed a file in the sase repo, cutting ~610 tokens from every turn's Tier
  1 context without weakening the verification mandate.
size: small
proposed_by: bbugyi200.athena.0fo
---

# Plan: Migrate `build_and_run.md` to a `lint_and_test.md` reference note

## Why

`sase/memory/build_and_run.md` is `type: core`, so its 694 tokens are inlined into
`AGENTS.md` and all four provider shims and paid on **every** turn by **every** agent in
this repo. It is 13.6% of the project's 5,098-token always-loaded memory budget. Almost
all of that body is procedure an agent needs only at one moment — after it has changed
files and is deciding how to verify — which is exactly what Tier 2 is for.

The migration has one real risk, and the plan is built around it: the note's
load-bearing instruction is "if you changed files, verify before you finish." A Tier 2
body is read only on demand, so an agent that never reads the note could finish
unverified. The mitigation is that **the description is the Tier 1 residue**.
Reference-note descriptions render verbatim into the Tier 2 section of every instruction
root, so a description that states the trigger and the mandate keeps the obligation
always-loaded at ~80 tokens instead of ~694.

## Scope Notes

- No source code changes. `grep` confirms nothing under `src/` references the
  `build_and_run` note; `src/` only mentions `just check` in unrelated CLI help text.
- The three tests that contain the string `build_and_run` (`tests/test_memory_notes.py`,
  `tests/main/test_inline_memory.py`, `tests/main/test_section_numbers.py`) use it as an
  inline fixture string, never as a path on disk. They keep passing untouched. **Do not
  rename those fixtures** — churn without benefit.
- `sase/memory/symvision.md` stays parented under `AGENTS.md`. Re-parenting lint-gate
  notes under the new note is a different change; this plan only adds a prose
  cross-reference.

## Step 1 — Create `sase/memory/lint_and_test.md`

Write the file below verbatim. It is the current body with five deliberate edits, each
justified under "Content Decisions".

````markdown
---
type: reference
parent: AGENTS.md
description: |
  IMPORTANT: if you changed ANY file in the sase repo, you MUST read this note before
  you finish your turn. Verification is not optional here and the lanes are not
  interchangeable: this note covers the `just` command surface, the two-speed rule that
  makes `just check` the agent default and `just check-full` a monitor-only landing
  gate, the `just install` prerequisite for ephemeral workspace clones, and the PNG
  snapshot suite.
---

# Linting And Testing

```bash
just install       # Install in editable mode with dev deps
just fmt           # Auto-format Python + Markdown
just lint          # Every whole-repo lint gate (ruff, mypy, symvision, toobig, ...)
just check         # Agent default: whole-repo lint gates + a diff-scoped
                   # test lane that never queues behind another agent's run
just check-full    # Exhaustive verification: every lint gate + the full
                   # test suite; run before landing and in CI
just test          # Fast parallel pytest run (excludes PNG visual snapshots)
just test-cov      # pytest with coverage + 50% gate (used by CI); also
                   # excludes the visual snapshot suite
```

## Two-Speed Verification: Run `just check` If You Changed Files

If you made file changes in this repo (the sase repo), make sure to run the `just check`
command before terminating / replying to the user.

`just check` runs every whole-repo lint gate plus a diff-scoped test lane
(`just test-scoped`) that selects tests via a static import-graph closure. The scoped
run is serial unless a middle gear wins it a small, bounded suite-gate lease, and it
never queues behind other agents' runs either way. Selection is a heuristic backstopped
by CI: `tools/select_tests --explain` shows why a test was or was not chosen, and
`just selection-health` shows whether the heuristic has ever been wrong.

Run `just check-full` instead — every lint gate plus the full test suite — before
landing an epic's combined tree, when the change touches the broadening set, or any time
`just check`'s scoped run escalated or reported an unusual selection.

`just check-full` routinely outruns a single agent turn, so run it **only** through your
`/sase_monitor` skill, never inline, using the `TESTING` / `TESTED` status pair.
`just check` may be run inline, but hand it to a monitor the same way whenever it is
taking a long time. `sase memory read decisions:two-speed-verification` has the host
capacity measurements that make this rule non-negotiable.

**IMPORTANT**: SASE agents run from ephemeral `sase_<N>` workspace clones that each own
an isolated virtualenv, so you MAY need to run `just install` before `just check` — this
workspace may have sat unused while pinned dependencies changed.

## Gate-Specific Help

`sase memory read symvision.md` covers the `symvision` unused/misused-symbol gate, whose
failures are the ones least often fixed correctly by deleting the reported symbol.

## PNG Snapshot Tests

Run `just test-visual` for the dedicated ACE PNG snapshot suite; goldens live in
`tests/ace/tui/visual/snapshots/png/`. On failures, inspect `.pytest_cache/sase-visual/`
for actual/expected/diff/source artifacts, and use `--sase-update-visual-snapshots` to
accept intentional visual changes. Local runs use exact pixel equality by default, while
CI allows a small ratio-only renderer drift tolerance; the visual fixtures pin color and
fontconfig/Fira Code to keep rendering deterministic.
````

## Content Decisions

Five edits, and one deliberate non-edit. Everything else is byte-identical to the
current body, because it was verified accurate against `Justfile` and CI.

1. **The description carries the mandate, not just a pointer.** It names the trigger in
   the agent's own terms ("if you changed ANY file in the sase repo"), states the
   obligation ("MUST read... before you finish your turn"), and previews the four things
   the body decides. It names `just check` / `just check-full` on purpose: an agent that
   ignores the read instruction still learns the command exists, while an agent that
   reads it learns which lane and how to run it. `decisions.md` already keeps a one-line
   `two-speed-verification` summary in Tier 1, which reinforces the same rule from a
   second always-loaded surface.

2. **`just lint` was wrong and is corrected.** The old note said `# ruff check + mypy`;
   `Justfile:270` runs ten gates (ruff, mypy, feature flags, pyscripts, test waits,
   changelog, patch/stitch terminology, symvision, toobig, keep-sorted). An agent that
   believed the old comment would think `just lint` clean meant lint clean.

3. **The `just install` paragraph stops referencing "the sase.md file in this
   directory."** That phrasing only worked when this note was inlined next to `sase.md`
   in `AGENTS.md`. A Tier 2 note is read standalone, so the paragraph now states the
   ephemeral-workspace fact directly instead of forwarding to a sibling file.

4. **A pointer to `decisions:two-speed-verification` replaces re-explaining the why.**
   The decision record owns the capacity numbers (~200-400 full-suite runs/day against
   ~46,000 gated worker-minutes at ~61 worker-minutes per run). This note stays
   operational and links rather than duplicates.

5. **A `symvision.md` cross-reference is added.** Roughly 30 tokens in a note only read
   when a gate is about to run, and it forecloses the single most common wrong fix
   (deleting a symbol symvision flagged).

**Non-edit:** `just test` and `just test-cov` stay, verified accurate —
`tools/run_pytest` passes `--cov-fail-under=50` and `.github/workflows/ci.yml:203` runs
`just test-cov`. Listing `just test` slightly tempts an agent to bypass the two-speed
rule, but deleting accurate command documentation to nudge behavior is the worse trade
when the two-speed section sits directly beneath it.

## Step 2 — Delete `sase/memory/build_and_run.md`

```bash
git rm sase/memory/build_and_run.md
```

## Step 3 — Fix the stale example in `docs/memory.md`

`docs/memory.md:10-11` illustrates generated section numbering with headings that will
no longer exist:

```
  `### 1.1 Build & Run Commands (build_and_run)` and
  `#### 1.1.1 IMPORTANT: Two-Speed Verification`
```

Replace them with the `sase.md` note, which SASE always generates and pins to
`priority: 10`, so the example stays true for as long as the memory system exists:

```
  `### 1.1 SASE = Structured Agentic Software Engineering (sase)` and
  `#### 1.1.1 SASE Memory`
```

Change only those two heading strings; leave the surrounding sentence intact.

## Step 4 — Regenerate memory artifacts

The user's request to change a SASE memory file carries authorization for this step; do
not open a gate for it.

```bash
sase memory init
```

This rewrites `AGENTS.md`, the `CLAUDE.md` / `GEMINI.md` / `QWEN.md` / `OPENCODE.md`
shims, and `sase/memory/README.md`. Expect Tier 1 to lose its `Build & Run Commands`
section and renumber (`decisions` becomes 1.2 through `task_types` at 1.6), and Tier 2
to gain `sase/memory/lint_and_test.md` alphabetically between `generated_skills.md` and
`sase_artifacts.md`. Do not hand-edit any generated file.

## Step 5 — Format and verify

`sase/memory/` is not in `.prettierignore`, so the new note is prettier-formatted
(`proseWrap: always`, `printWidth: 88`, per `package.json`). Run `just fmt` and let
prettier reflow the prose and the description block rather than hand-wrapping; the
fenced `bash` block is preserved as-is.

```bash
just install   # only if this workspace is stale
just fmt
just check
```

`just check` is the right lane: the diff touches Markdown plus generated instruction
files, none of the broadening set. Its `SASE validation` stage runs
`sase memory init --check`, which fails if Step 4 was skipped or a generated file was
edited by hand.

## Acceptance Criteria

1. `sase/memory/lint_and_test.md` exists with `type: reference`, `parent: AGENTS.md`,
   and a multi-line `description`; `sase/memory/build_and_run.md` is gone.
2. `sase memory list` reports `sase/memory/lint_and_test.md` as `referenced` (not
   `available`, which would mean nothing reaches it), and `Approx loaded tokens` drops
   by roughly 600 (5,098 before; expect ~4,500). Treat a drop far smaller than that as a
   signal the note stayed core by mistake.
3. `grep -rn "build_and_run" --include="*.md" .` matches nothing outside `tests/` — no
   hits in `AGENTS.md`, the four shims, `sase/memory/README.md`, or `docs/memory.md`.
4. `sase memory read lint_and_test.md -r "verify migration"` prints the body with
   frontmatter stripped.
5. `just check` passes.

## Rollback

Single-commit, self-contained: `git revert` restores the core note, and a follow-up
`sase memory init` regenerates the instruction roots.
