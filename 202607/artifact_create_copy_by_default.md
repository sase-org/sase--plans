---
tier: tale
title: Make `sase artifact create` copy by default and stop double-capturing what it declares
goal:
  Declaring a file with `sase artifact create` never destroys it. The command copies by default, `-m/--move` opts back
  into relocation, the output names both the source and the stored path, and a file an agent declares is captured and
  attached exactly once even though it now survives in the workspace.
create_time: 2026-07-30 10:30:37
status: wip
---

- **PROMPT:** [prompts/202607/artifact_create_copy_by_default.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/artifact_create_copy_by_default.md)

# Plan: Make `sase artifact create` copy by default

This implements recommendation **1** of
`research:202607/artifact_capture_and_retention/artifact_capture_and_retention.md` — "Make `sase artifact create` copy
by default" (§5.1) — together with the causal argument in §4 that names it "the precondition that makes 'declare, don't
sweep' a viable policy."

## Problem

`sase artifact create` deletes its input.

`src/sase/artifact_cli/create.py:36` hardcodes `move=True` when calling `store_explicit_artifact_file`. That flows into
`_store_file` (`src/sase/core/artifact_file_explicit.py:258` and `:272`), which calls `source.unlink()` on both the
already-stored and the freshly-copied branch. There is no `--move` or `--copy` flag: relocation is unconditional and
unannounced.

Three facts make this worse than a rough edge:

- **The library already defaults to copy.** `store_explicit_artifact_file(..., move: bool = False)` and its docstring
  say so explicitly, and the only other in-repo caller — `store_followup_prompt_artifact` in
  `src/sase/axe/run_agent_exec_plan_artifacts.py:46` — takes that default. The CLI is the sole outlier, and it overrides
  a safe default with a destructive one.
- **The skill instructs agents straight into the trap.** `src/sase/xprompts/skills/sase_artifact_file.md:19-24` says
  "Create the requested file in your workspace" then "Register it as an explicit artifact file" — a two-step sequence
  whose second step silently deletes the first step's output. An agent that declares a plan, a research note, or a doc
  it just committed leaves a dangling deletion in the working tree.
- **The docs teach a workaround instead of fixing the cause.** `docs/getting_started.md:101` has the tutorial run
  `cp notes.md workspace-note.md && sase artifact create -p workspace-note.md`, and `:111` explains that the copy is
  deliberate "so the tracked `notes.md` stays in the workspace." The onboarding path ships a defensive `cp` because the
  command is unsafe.

The research report draws the causal link (§4, "The causal link neither report drew"): because declaration destroys the
file, it is unusable for exactly the artifacts that matter most, which plausibly explains both why the explicit cohort
is small in bytes (217 rows / 5.56 MB) and why a permissive automatic sweep exists to compensate. Declaration is being
adopted — explicit creations per day over the last measured week were 20, 24, 15, 18, 36, 34 — so this fixes a path
agents actively use.

## The consequence that makes this more than a two-line flip

The research report sizes this as "two lines plus docs." That is right about the flip and wrong about the blast radius,
because the destructive behavior is currently load-bearing for deduplication.

Today, moving the source out of the workspace means nothing downstream can see it again. Once the source survives, **one
declared file can be captured twice and attached twice**:

1. **A second index row.** `_collect_default_artifacts` in `src/sase/axe/run_agent_exec_finalize.py:283` builds
   `image_paths`/`video_paths` from `collect_agent_image_paths`, which scans tracked changes, untracked files, saved
   diffs, and optionally `HEAD`. A declared PNG or GIF that stays in the workspace now appears in that list and reaches
   `persist_default_artifact_files` (`src/sase/core/artifact_file_defaults.py:166`), which writes a second `default:`
   row for the same bytes. Nothing filters it: the explicit row's `path` is the stored path under
   `~/.sase/artifacts/agents/…` while the default row's `path` is the workspace path, so `artifact_file_dedupe_key`
   (`src/sase/core/artifact_file_helpers.py:133`) sees two distinct keys and `dedupe_artifact_files` keeps both. The
   Files pane and `sase artifact list` show the same file twice.
2. **A duplicate notification attachment.** In `src/sase/axe/run_agent_runner_finalize.py:254`,
   `_completion_explicit_artifact_paths` returns each explicit row's **stored** path, and
   `append_unique_paths(explicit_artifact_paths, extra_files)` dedupes by absolute path against the already-collected
   workspace `image_paths`. Stored path ≠ workspace path, so the same image is attached twice.

Both are new regressions introduced by the flip, both are user-visible, and both undercut the very goal of the change:
recommendation 1 exists to make declaration the obviously-correct path, and shipping it with "your declared file now
shows up twice everywhere" would teach the opposite lesson. Closing them is in scope.

Note the shape of the interaction with `sase-b7` (below): the VCS-backed capture policy landing right now would classify
a _committed_ declared file as `reference` — a byte-free row — so the duplicate is cheap in bytes but still duplicated
in the two surfaces users actually read. Suppression, not accounting, is the fix.

## Relationship to the `sase-b7` epic (in flight)

`sase-b7` — "Make artifact capture mean authorship and stop copying what version control stores" — is landing now.
Phases `sase-b7.1` through `sase-b7.4` are closed (`d309f9537`, `c9edec561`, `94daa1ebd`); `sase-b7.5` ("Docs, skill,
and configuration reference") is **in progress**. Two consequences for this work:

- **File conflicts are likely.** `sase-b7.5` edits `src/sase/xprompts/skills/sase_artifact_file.md`, the artifact docs,
  and the configuration reference — the same files this plan edits. Land this work **after** `sase-b7.5`, or rebase onto
  it and reconcile before touching those files. Do not open a competing edit of the skill source while `sase-b7.5` is
  open.
- **No overlap in behavior.** `sase-b7` governs the _automatic_ (`explicit=False`) capture path only: `decide_captures`
  in `src/sase/core/artifact_capture_policy.py` is reached exclusively from `persist_default_artifact_files`.
  `store_explicit_artifact_file` never consults it. This plan changes only the explicit path plus the suppression seam
  between the two.

## Deliberately out of scope

- **VCS-backed byte-free rows for explicit artifacts.** `sase-b7` gives automatic captures the option of a record with
  `vcs_repo`/`vcs_sha`/`vcs_relpath` and no stored bytes. Do **not** extend that to `create`. An explicit declaration is
  a deliberate signal that the bytes should be kept — recommendation 3 of the research report makes `explicit=True` a
  hard protection against pruning for exactly that reason — and the whole explicit cohort is 5.56 MB, 0.84% of the
  store. There is no byte problem here to solve, and making declaration byte-free would re-introduce the "declaring
  loses my content" failure mode in a subtler form.
- **The legacy `synthesize_default_artifact_files` fallback.** That branch runs only for agents whose `done.json` lacks
  `default_artifacts_persisted` (`src/sase/core/artifact_file_defaults.py:129`). It is a best-effort path for runs that
  predate persistent capture; leave it alone.
- **Retention, pruning, and `doctor` economics.** Recommendation 3 of the research report. Separate work.

## Rust core boundary

No `sase-core` change. Per `docs/rust_backend.md:75`, explicit and automatic artifact-file _storage_ is deliberately
Python-owned because it copies or moves files on the local filesystem; `sase-core` owns the record wire schema, the
reference grammar, and VCS materialization, none of which change here. Whether the CLI copies or moves is local
filesystem policy on the Python side of the declared boundary.

## Step 1 — `-m/--move` opt-in, copy default

**`src/sase/main/parser_artifact.py`.** Add a `-m/--move` flag to `create_parser`, positioned between `--label` and
`--path` so the block stays alphabetical, per `sase/memory/cli_rules.md` (options sorted alphabetically; every public
long option gets a short alias — `-m` is free alongside `-k`, `-l`, `-p`).

```python
create_parser.add_argument(
    "-m",
    "--move",
    action="store_true",
    help="Remove the source file after storing it (default: copy)",
)
```

Update `--path`'s help text, which currently reads "Source file to **move** into durable artifact storage", and the
subcommand's own `help=` ("Store a file as an explicit artifact for the current agent" is already accurate).

**`src/sase/artifact_cli/create.py`.** Pass `move=args.move` instead of `move=True`. Read the attribute directly rather
than through `getattr(args, "move", ...)`: argparse always supplies it, and a default would silently paper over a
hand-built `Namespace` that forgot the field.

Also print the source path. The research report asks for "both source and stored paths"; emit a fixed four-line shape in
both modes so the output stays trivially parseable:

```text
id: explicit:<hash>
source: /abs/path/to/report.md
path: /home/<user>/.sase/artifacts/agents/<project>/<ts>/report-<digest>.md
ref: file:explicit:<hash>
```

`source:` names where the artifact came from, which is true in copy and move mode alike; the docs, not a variable output
shape, carry the "with `--move` the source no longer exists" caveat. Print the resolved absolute source
(`source_path.expanduser()` as already computed at `create.py:26`), not the raw argument. Nothing in the repo parses
this output except `tests/main/test_artifact_handler.py:202`, so inserting a line is safe.

## Step 2 — Capture a declared file once

**`src/sase/core/artifact_file_defaults.py`.** In `persist_default_artifact_files`, drop candidates that an explicit row
for this same run already claims.

- Build the exclusion set from `list_indexed_artifact_files(artifacts_dir, index_path=index_path)` filtered to
  `explicit=True` (or `list_explicit_artifact_files` with the same `index_path` — it must honor the injected index path
  so tests stay hermetic), keying on `path_key(row.source_path)` for every row that has a `source_path`.
- Apply it after `_existing_media_candidates` and before `_capture_decisions`, so excluded files never reach the git
  probe. That keeps the policy's per-run `max_stored_per_agent` budget spent on files that actually need it.
- Count the drops and surface them in the existing `print_summary` line, which already reports
  `stored=… referenced=… skipped=… cap_fired=…`. Add a `declared=…` term rather than folding them into `skipped`, so the
  summary distinguishes "the policy rejected this" from "an agent already declared this."

Keep the ordering guarantee intact: `persist_default_artifact_files` runs from `finalize_loop`, after the workflow loop
and therefore after every in-run `sase artifact create` has already written its row. Also keep the function's documented
idempotency — re-running over the same workspace must still yield the same set.

**`src/sase/axe/run_agent_runner_finalize.py`.** Stop attaching a declared file twice.
`_completion_explicit_artifact_paths` currently returns stored paths only. Give it access to each row's `source_path`
and, at the call site (`:254`), skip an explicit row whose `source_path` resolves to a path already present in
`extra_files`. The workspace copy is already collected and ordered earlier; dropping the redundant stored path preserves
today's attachment order and count exactly. Compare with the same normalization `append_unique_paths` uses
(`os.path.abspath(os.path.expanduser(...))`) so the two agree.

Both sites must stay best-effort: `_completion_explicit_artifact_paths` already swallows exceptions, and
`persist_default_artifact_files` is wrapped in a `try` at `run_agent_exec_finalize.py:301`. A failure to read the index
should degrade to "capture everything" rather than to "capture nothing."

## Step 3 — Skill and documentation

Wait for `sase-b7.5` to land (or rebase onto it) before editing these files.

**`src/sase/xprompts/skills/sase_artifact_file.md`** — the highest-value edit, because it is what agents actually read.

- Delete the standing hazard at `:36`: "The command moves the file into SASE artifact-file storage, so do not edit the
  original path after registration."
- State the new contract: `create` copies, the source stays where the agent put it, and the stored copy is a snapshot —
  edits made after registration do not propagate, so re-run `create` to refresh.
- Document `-m/--move` in the options list, in alphabetical position, with the one case it is for: a scratch file that
  should not be left behind. Warn that `--move` on a tracked file leaves a deletion in the working tree.
- Add `source:` to the reported output at `:26`.

**Docs.**

- `docs/agent_images.md:156` — "It moves the source file into persistent SASE artifact storage" → copies; document
  `-m/--move` and the new `source:` output line in the "Explicit Artifact Contract" section (`:131`).
- `docs/configuration.md:3210` — "It moves a generated file into persistent SASE artifact storage" → copies; add
  `-m/--move` to the `sase artifact create` flags row at `:3219`.
- `docs/cli.md:275` — "Move an explicit file into persistent agent artifact storage" → "Copy".
- `docs/getting_started.md:101,111` — the tutorial's defensive `cp notes.md workspace-note.md` exists only to survive
  the move. Simplify the command to declare `notes.md` directly and rewrite the explanation at `:111` to say the file
  stays in the workspace. This is the clearest single demonstration that the fix landed.
- `docs/rust_backend.md:75` ("copies or moves files") stays accurate — no edit.
- Check `docs/notifications.md:129,147`, `docs/axe.md:825,842`, and `docs/ace.md:328,935` while you are here; they cite
  the command without claiming it moves, so they likely need nothing.

**Deployment.** Per `sase/memory/generated_skills.md`, the chezmoi `SKILL.md` files are generated, never hand-edited.
Preview with `sase skill init --diff` while iterating, then land the source change on the canonical branch, and only
from that clean, merged tree run `sase skill init --force` (followed by `chezmoi apply` if it was skipped). Deploying
from a dirty or unmerged tree is refused by design and would revert another agent's deployment.

## Step 4 — Tests

**`tests/main/test_artifact_handler.py`**

- `test_public_long_options_are_alphabetical_and_have_short_aliases` (`:117`) — add `"--move"` between `"--label"` and
  `"--path"`. The generic short-alias assertion at `:134` then covers `-m` for free.
- `_create_args` (`:16`) — add `move=False`.
- Rename and rewrite `test_create_moves_file_records_association_and_prints_reference` (`:161`). Its
  `assert not source.exists()` at `:198` is the exact assertion that must invert. Split it in two:
  - copy path — source still exists, stored file exists, contents match, four output lines in the documented order with
    `source:` naming the absolute source, index row has `explicit=True`.
  - move path (`move=True`) — source gone, everything else identical.
- Add a regression test that the stored copy is independent: register, mutate the source, and assert the stored bytes
  and the recorded `sha256` are unchanged.

**`tests/test_artifact_file_e2e.py`** — `_artifact_create_args` (`:130`) needs `move=False`. The existing assertions at
`:166-174` only check `id:`/`ref:` substrings and survive the extra line.

**New coverage for Step 2** — the point of these is that they fail loudly on the old behavior, so write them to assert
counts, not just membership:

- A declared image left in the workspace and also present in `image_paths` produces **exactly one** index row for that
  file, and it is the `explicit=True` one.
- The `print_summary` line reports the drop under the new `declared=` term.
- A non-declared image in `image_paths` is still captured — the filter must not over-reach.
- An explicit row with `source_path=None` (or pointing at a path outside the workspace) does not poison the exclusion
  set.
- Attachment assembly: a declared image already in `image_paths` yields one attachment, not two; a declared markdown
  report that no other collector found still yields its one attachment.

**`tests/main/test_init_skills_sources.py:129`** — the `sase_artifact_file` phrase tuple asserts required substrings in
the rendered skill. Add `--move` so the flag cannot silently disappear from the skill source.

## Verification

1. `just install` first — this workspace may be stale (see `CLAUDE.md` on ephemeral `sase_<N>` directories).
2. `just check`.
3. Manual smoke inside an agent run, since the destructive behavior is the thing being retired:

   ```bash
   printf '# smoke\n' > /tmp/smoke_artifact.md
   sase artifact create -p /tmp/smoke_artifact.md -l 'Copy smoke'   # source survives
   test -f /tmp/smoke_artifact.md && echo "copy default OK"
   sase artifact create -p /tmp/smoke_artifact.md -l 'Move smoke' -m # source removed
   test ! -f /tmp/smoke_artifact.md && echo "--move opt-in OK"
   ```

4. Confirm `sase artifact create -h` reads cleanly and lists `-k`, `-l`, `-m`, `-p` in order.

## Done when

- `sase artifact create` copies by default; `-m/--move` is the only way to remove the source.
- Output carries `id:`, `source:`, `path:`, `ref:` in that order, in both modes.
- A file declared during a run yields exactly one artifact row and exactly one notification attachment, even though it
  now survives in the workspace.
- The `sase_artifact_file` skill source no longer tells agents the command moves their file, documents `-m/--move`, and
  has been regenerated and deployed from a clean, landed tree.
- `docs/getting_started.md` declares `notes.md` directly, with no defensive `cp`.
- `just check` passes.
