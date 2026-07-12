---
create_time: 2026-07-11 09:02:30
status: done
prompt: 202607/prompts/init_change_preview.md
tier: tale
---
# `sase init` Change Preview: See Exactly What Will Change Before Saying Yes

## Problem

Bare `sase init` (and `sase init --check`) reports drift per subcommand (memory, sdd, skills) and then asks a
per-subcommand y/N question such as:

```
Run `sase init memory` now? This may commit and push generated project memory changes. [y/N]
```

The user must answer on blind trust:

- The "Needs attention" section caps the action listing at **3 lines per plan** (`_MAX_ACTION_DETAILS = 3` in
  `src/sase/main/init_onboarding.py`), collapsing the rest into `... N more actions`. A skills refresh can touch dozens
  of files across providers; the user sees three of them.
- Even for the visible actions, only `operation + path + detail` is shown. There is no way to see **what content** will
  be written, updated, or deleted — no diff, not even a magnitude ("is this a one-line tweak or a full rewrite?").
- The y/N prompt offers no inspection escape hatch. Your only options are trust or abort.

The raw material for full transparency already exists: **all three planners compute the exact expected file content at
plan time** and then throw it away before display:

- Memory: `MemoryExpectedFile.content` (`src/sase/main/init_memory/root_planning.py`) and provider-shim
  `_PlannedWrite.content` / `_PlannedDelete.expected_content` (`src/sase/amd/_shared.py`) are dropped when mapped to
  `MemoryFileChange` → `InitAction`.
- SDD: `expected_sdd_text_files()` / `expected_sdd_directory_map()` (`src/sase/sdd/_init_files.py`) carry full text and
  PNG bytes; `SddInitAction` drops them.
- Skills: `RenderedSkillTarget.content` (`src/sase/main/init_skills_handler.py`) is fully rendered (Jinja2 + prettier)
  before planning; `plan_init_skills` drops it. Notably, `sase skill init`'s standalone overwrite prompt **already**
  supports a `[y/n/d]` diff option (`_prompt_overwrite`) — the onboarding coordinator just never got the same treatment.

## Goals

1. **Complete inventory, always.** The "Needs attention" section lists _every_ planned action — no 3-line cap — with a
   color-coded operation, a readable path, a per-file diffstat (`+12 −3`), and the existing detail text.
2. **Exact diffs on demand.** The confirmation prompt becomes `[y/N/d]`: answering `d` prints a beautiful unified diff
   of every action in that plan, then re-asks. A new `-d/--diff` flag prints the same diffs inline for non-interactive
   contexts (`--check`, non-TTY, `--yes`).
3. **Beautiful output.** Rich-rendered: color-coded operation glyphs, aligned columns, green/red diffstats, styled diff
   bodies — degrading gracefully to plain text when stdout is not a TTY (the existing `_console_for` already handles
   color stripping).
4. **Additive and safe.** `InitPlan`/`InitAction` consumers (notably `sase doctor`'s `config.init` check in
   `src/sase/doctor/checks_config_init.py`) keep working untouched; the content field is optional.

## Non-Goals

- No change to what `sase init` actually _does_ when applied, nor to prompt ordering, exit codes, `--yes` semantics,
  chezmoi deploy deferral, or the SDD companion-repo confirmation flow (which stays its own explicit default-no gate).
- No pager integration; diffs print to stdout and rely on terminal scrollback (users can run
  `sase init --check --diff | less -R`).
- No attempt to guarantee the applied bytes are byte-identical to the previewed bytes if the environment changes between
  prompt and apply (runners re-plan internally today; that behavior is unchanged).

## UX Design

### 1. The inventory (replaces the capped action details)

```
SASE initialization check

Up to date:
  ok   init sdd      SDD provider storage, README files, and directory map are current

Needs attention:

  run  init memory   refresh 3 memory files and provider shims
       ~ update   memory/sase.md    +4 −1    generated SASE memory
       ~ update   AGENTS.md         +12 −3   managed agent instructions
       + create   CLAUDE.md         +210     provider shim for AGENTS.md

  run  init skills   overwrite 2 provider skill files
       ~ overwrite  ~/.claude/skills/sase_plan/SKILL.md   +6 −2   claude/sase_plan
       ~ overwrite  ~/.gemini/skills/sase_plan/SKILL.md   +6 −2   gemini/sase_plan

Tip: answer `d` at a prompt below to review the full diff before deciding.
```

Visual spec (Rich, `box=None` table or aligned Text so non-TTY output stays clean and grep-able):

- **Operation glyph + word, color-coded**: `+ create` green, `~ update`/`~ overwrite` yellow, `− delete` red, procedural
  ops (companion repo create/connect, legacy import) cyan. The `run`/`hold` row prefix keeps its meaning; `run` renders
  green, `hold` red.
- **Path**: existing `_display_path` relativization (cwd-relative, then `~/`-prefixed), rendered bold; columns aligned
  across rows within one plan.
- **Diffstat**: computed lazily at render time by reading the on-disk file and diffing against the planned content —
  `+N` green / `−N` red for text; `binary` (dim, with human sizes, e.g. `binary 34.1 kB → 35.0 kB`) for bytes content
  such as the SDD directory-map PNG; `–` for procedural actions with no file content (e.g. "create or connect the
  provider companion SDD repository"). Deletes show `−N` from the on-disk line count.
- **Detail**: existing detail string, dim.
- **No cap**: every action is listed. A from-scratch onboarding may print a long list — that is the point. (If this ever
  proves noisy we can group skill rows by skill name later; explicitly out of scope now.)
- The tip line prints only when an interactive prompt will follow (TTY, no `--yes`, no `--check`).

### 2. The prompt (default answer stays No)

```
Run `sase init memory` now? This may commit and push generated project memory changes. [y/N/d]
```

- `y`/`yes` → apply; `n`/`no`/empty → skip (unchanged default); `d`/`diff` → print the plan's full diff, then re-prompt;
  anything else → one-line hint (`y = apply, n = skip, d = show diff`) and re-prompt.
- EOF still means No; Ctrl-C still aborts the whole run. The memory-commit and companion-repo warning sentences are
  preserved verbatim.

### 3. The diff view (`d` at the prompt, or `--diff` anywhere)

Per action, a Rich `Rule` header then the body:

```
── ~ update AGENTS.md ────────────────────────────────────────────
@@ -12,7 +12,9 @@
 ### 1. Build & Run Commands (build_and_run)
-just install       # Install in editable mode
+just install       # Install in editable mode with dev deps
+just lint          # ruff check + mypy
...

── + create CLAUDE.md ────────────────────────────────────────────
  New file, 210 lines:
+<full content, green>

── − delete GEMINI.md ────────────────────────────────────────────
  Removes 34 lines (provider shim no longer referenced).

── binary ~ update sdd/assets/sdd-directory-map.png ──────────────
  Binary file differs: 34.1 kB on disk → 35.0 kB generated.

── ● create or connect the provider companion SDD repository ─────
  Remote/procedural action — no local file diff. A separate y/N
  confirmation guards companion repository creation.
```

- Unified diffs via `difflib.unified_diff`, colored by hand-rolled line styling (`+` green, `-` red, `@@` cyan, context
  dim) on a Rich `Text` — full control, clean no-color fallback, no pygments dependency games.
- Creates render the whole new file as added lines; deletes summarize (or show the removed content — spec: show the
  removed lines as a red diff, consistent with everything else).
- Binary and procedural actions get one-sentence summaries instead of diffs.

### 4. New flag: `sase init -d/--diff`

- Prints the full diff view inline directly under each changed plan's inventory. Composes with everything:
  `--check --diff` (inspect without a TTY), bare interactive (diffs shown up front, prompts still ask), `--yes --diff`
  (audit trail of what was applied), `--all --diff`.
- Registered on the bare `init` parser and on the check-capable subcommand parsers (`init memory`, `init sdd`,
  `init skills` via `add_skills_init_arguments`, which also serves `sase skill init`), so
  `sase init memory --check --diff` works. Handlers read it with `getattr(args, "diff", False)`.
- Per CLI rules: short alias `-d` (free everywhere it is added), help text kept crisp, options remain alphabetically
  ordered in `-h` output.

## Technical Design

### Data flow: thread `new_content` through the planners (additive)

`InitAction` (`src/sase/main/init_plan.py`) gains one optional field:

```python
@dataclass(frozen=True)
class InitAction:
    path: Path
    operation: InitOperation
    detail: str = ""
    new_content: str | bytes | None = None  # planned post-apply content, when known
```

`None` means "no content diff available" (procedural actions, deletes). Old content is **not** stored — the renderer
reads the on-disk file lazily when producing a diffstat or diff, keeping plans light and `sase doctor` unaffected.

Per planner:

- **Memory** (`src/sase/main/init_memory/models.py`, `root_planning.py`, `init_memory_handler.py`): `MemoryFileChange`
  gains the same optional field. `_compare_expected_memory_files` passes `expected.content`; `_provider_shim_changes`
  passes `write.content` for writes and leaves deletes as `None` (the renderer diffs on-disk → nothing).
  `plan_init_memory` forwards the field into `InitAction`. The `--enable-project-memory` synthetic sase.yml action stays
  content-less (it is applied before planning; its one-line detail already says exactly what it does — optionally
  upgrade later).
- **SDD** (`src/sase/sdd/_types.py`, `_init_files.py`, `src/sase/main/sdd_handler.py`): `SddInitAction` gains the field;
  `plan_sdd_init_actions` passes text content for the READMEs and PNG bytes for the directory map. Companion-repo and
  legacy-import actions stay `None`.
- **Skills** (`src/sase/main/init_skills_handler.py`): `plan_init_skills` passes `target.content` (already
  prettier-formatted, so the preview matches the applied bytes exactly).

No changes in `src/sase/amd/_shared.py` are needed — write content is already exposed on its plan objects.

### New module: `src/sase/main/init_preview.py`

Owns everything display-related so `init_onboarding.py` stays a coordinator:

- `action_diffstat(action) -> Diffstat | None` — reads the current file (missing → empty), returns `(added, removed)`
  line counts for text, a `binary` marker with sizes for bytes, `None` for content-less actions.
- `render_plan_inventory(console, plan)` — the full aligned, color-coded action listing (replaces
  `_render_action_details`; `_MAX_ACTION_DETAILS` is deleted).
- `render_plan_diff(console, plan)` — the per-action diff view described above.

`init_onboarding.py` changes:

- `_render_plans` calls `render_plan_inventory` for every changed plan, and `render_plan_diff` too when `--diff` is set;
  prints the `d`-tip line when prompts will follow.
- `_prompt_for_plan` becomes the `[y/N/d]` loop (taking the console so `d` can render diffs), preserving the existing
  warning sentences, EOF→No, and Ctrl-C→abort behaviors.
- `run_init_check` honors `--diff` via the same path.

For UX consistency, `sase skill init`'s standalone `_prompt_overwrite` diff branch switches to the same
`render_plan_diff` styling so a diff looks identical everywhere (small, contained follow-through — its `[y/n/d]`
semantics are unchanged).

### Boundary note

This is CLI presentation over locally computed plan data; the planners and the drift model live (and stay) in this
repo's Python. Nothing here crosses into shared domain behavior consumed by other frontends, so no Rust core
(`sase_core`) changes are involved.

## Tests

Update/extend (primarily `tests/main/`):

- `test_init_onboarding_flow.py`:
  - Replace `test_needs_attention_output_snapshot_caps_path_details` with a full-inventory snapshot (no cap; diffstat
    column present; procedural SDD action shows `–`).
  - New: prompt answers `d` → diff body appears (added/removed lines visible) → re-prompt → `n` skips; invalid answer
    re-prompts with hint; EOF/Ctrl-C behaviors unchanged.
  - New: `--check --diff` prints diffs and still exits 1 on drift; `--yes` path never prompts (unchanged).
  - Existing prompt-wording tests (companion-repo/memory-commit sentences) updated only for the `[y/N/d]` suffix.
- `test_init_skills_plan.py` / SDD planner tests / memory planner tests: planned actions now carry `new_content`
  matching the rendered targets (text and bytes cases).
- `init_preview` unit tests: diffstat math (create/update/delete/binary/missing-on-disk), delete rendering from on-disk
  content, binary size formatting, no-color output stability.
- `tests/doctor/test_checks_config_init.py`: untouched and green (proves the additive field is safe).
- Parser tests: `-d/--diff` accepted on `sase init`, `sase init memory|sdd|skills`, and `sase skill init`; help output
  stays alphabetized.

`just check` gates the change as usual.

## Risks & Notes

- **Long output on first-run onboarding** (many `create` rows, and `--diff` prints entire new files). Accepted:
  exactness is the feature; scrollback and `--check --diff | less -R` cover the rest. Future polish could group skill
  rows or elide very large create bodies, deliberately not done now.
- **Preview vs. apply drift**: runners re-plan internally, so if the environment changes between preview and `y`, the
  applied content could differ from the preview. Pre-existing behavior; unchanged risk, now at least visible.
- **Windows/encoding edge cases**: unreadable/undecodable current files already map to `update`/`overwrite` operations;
  the renderer treats unreadable old content as empty and labels the diff accordingly.
