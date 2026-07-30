---
tier: epic
title: Commit-time file hooks + bob highlights PDF pipeline
goal: 'Files that sase agents add/modify/remove — via VCS commits or `sase artifact
  create` — automatically trigger user-configured per-file hook commands with rich
  success/failure notifications, and the first configured hook turns new consolidated
  research reports into Highlights-ready PDFs in the Obsidian vault.

  '
phases:
- id: highlights-create
  title: Add `bob highlights create <md_file>` to bob-cli
  depends_on: []
  size: medium
  description: 'highlights-create: add a Rust clap subcommand to the bob-cli highlights
    group that converts a markdown file into a beautiful, TOC-indexed PDF under ~/bob/lib/chat/
    via a pandoc shell-out, embeds the page-1 /Text Highlights marker annotation with
    lopdf so `bob highlights scan` later creates the Obsidian ref note, and ships
    polished colored success/failure output, docs, and tests.'
- id: hooks-config
  title: file_hooks config section, matcher, and list CLI
  depends_on: []
  size: medium
  description: 'hooks-config: add the `file_hooks` section to the sase config schema
    and default config, a fail-soft typed loader mirroring mentor_profiles, a wcmatch-based
    path matcher with negative-glob support, the `sase file-hook list` command, docs,
    and unit tests. No commit-flow integration yet.'
- id: hooks-engine
  title: Commit/artifact event capture, detached runner, notifications
  depends_on:
  - hooks-config
  size: medium
  description: 'hooks-engine: capture per-file ADD/MODIFY/REMOVE events at both commit
    seams (CommitWorkflow and commit_sdd_files) and at `sase artifact create`, match
    them against configured file_hooks, execute matched commands once per file in
    a detached batch runner (`sase file-hook exec-batch`), and send a sase notification
    per run with attached output; never gate or slow the commit.'
- id: deploy-verify
  title: Configure the research-highlights hook and verify end to end
  depends_on:
  - highlights-create
  - hooks-engine
  size: small
  description: 'deploy-verify: add the research-highlights file_hooks entry to the
    chezmoi-managed global sase.yml, apply it, install the new bob binary, run `bob
    highlights create` against the real 202607 beads research report, verify the PDF
    and then the ref note produced by `bob highlights scan`, and exercise the sase
    hook engine end to end including its notification.'
create_time: 2026-07-30 13:32:26
status: wip
bead_id: sase-bc
---

- **PROMPT:** [202607/prompts/commit_file_hooks.md](prompts/commit_file_hooks.md)
- **BEAD:** [sase-bc](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bc/README.md)

# Plan: Commit-time file hooks + `bob highlights create`

## Problem

SASE has no way to react to _specific files_ an agent adds, modifies, or removes. The existing `commit_hooks`
(`before`/`after`) config runs a single command string with no file context, no filtering, and gating semantics. Bryan
wants user-defined commands to run once per matching file whenever `sase commit` lands changes (or
`sase artifact create` registers a file), filtered by project, sidecar repo, path globs (with `!` negatives), and
operation (ADD/MODIFY/REMOVE) — plus a sase notification reporting each hook run's success or failure.

First use-case: every **new** consolidated research report (`20*/*/*.md`, excluding `!20*/*/*__*.md` swarm drafts) in a
`research` sidecar should be rendered to a TOC-indexed PDF in `~/bob/lib/chat/` by a new `bob highlights create`
command, marker-annotated so `bob highlights scan` creates its Obsidian ref note.

## Design overview

New top-level `file_hooks` config section (array of hook objects, mirroring the `mentor_profiles` idiom):

```yaml
file_hooks:
  - name: research-highlights # required — unique slug, shown in notifications and `list`
    description: Render new research reports into Highlights PDFs. # optional
    command: bob highlights create # required — matched file path appended as final shell-quoted arg
    projects: [sase] # optional — user-facing project names; omitted = all projects
    sidecars: [research] # optional — sidecar role names; omitted = any repo (primary too)
    globs: ["20*/*/*.md", "!20*/*/*__*.md"] # optional — repo-relative; omitted = all files
    ops: [ADD] # optional — subset of ADD|MODIFY|REMOVE; omitted = all three
    timeout: 120s # optional — per-run duration string; default 120s
```

Core semantics (identical wording should land in `docs/configuration.md`):

- **Event sources.** (1) Commits created by `sase commit` (`CommitWorkflow`), for whatever repo the commit runs in —
  primary workspace, or a sidecar/linked/external clone as cwd. (2) SDD sidecar commits made through
  `commit_sdd_files()` — this is how research/plans/beads sidecar files actually land (agent commit finalizer, bead CLI,
  TUI actions all call it and bypass `CommitWorkflow`); a runner living only in `sase commit`'s workflow would silently
  miss every research commit. (3) `sase artifact create`, treated as `ADD`.
- **Ops.** Derived from `git diff --name-status` of the created commit (`<sha>~1..<sha>`; root commit = all ADD),
  canonicalized the way `src/sase/ace/deltas/compute.py` already does: renames split into REMOVE + ADD, unknown letters
  fold to MODIFY.
- **Glob matching.** Matched against the file's repo-root-relative POSIX path. Positive globs are OR-ed (at least one
  must match); any matching `!`-negative veto-excludes the file. `*` does not cross `/`; `**` is available. A
  negative-only list implies "everything except". Implemented with the `wcmatch` library (new dependency) — `fnmatch` is
  wrong here because its `*` crosses `/`, and no negative-glob support exists in the repo today.
- **Filters.** `projects` compares alias-resolved, user-facing project names (never ProjectSpec keys). Hooks declared in
  a _project-local_ `sase/sase.yml` are auto-scoped to that project (mirror `src/sase/config/mentor.py:268-275`).
  `sidecars` compares sidecar role names (`research`, `plans`, `beads`, ...); an omitted `sidecars` places no
  restriction. All filter dimensions AND together.
- **Execution.** Post-commit and non-gating: hook failures never fail or block a commit. The in-process engine only
  matches events and, when at least one run matched, writes a batch JSON file and spawns a **detached** runner process
  (`sase file-hook exec-batch <batch>`). Detachment is mandatory: `commit_sdd_files` is called from TUI actions, and
  rendering a PDF synchronously there would freeze the TUI (see `sase/memory/tui_perf.md`). The runner executes each
  matched (hook, file) pair sequentially — `<command> <shell-quoted absolute path>` via `subprocess`, `shell=True`,
  `cwd=<repo root>`, inheriting env, with the hook's timeout — then sends one notification per run.
- **Notifications.** One per hook run, success and failure, following the established thin-notification pattern:
  `notes[0]` is a tight summary (inbox truncates at 50 chars), full stdout/stderr is written to a run log attached via
  `files`, failures get `action="ViewErrorReport"` + `action_data["error_report_path"]`.
- **Reliability.** Fail-soft config loading (warn + skip invalid entries, like mentor profiles); `CommitWorkflow`
  records a checkpoint step so `--resume` never double-fires hooks; batch/log files under `~/.sase/file_hooks/` give a
  durable audit trail and get opportunistically pruned.

No Rust-core (`sase-core`) changes are needed: the commit path is pure Python, and the Config Center's Rust validator
consumes the JSON Schema generically. The schema addition must stay within the keyword subset the Rust validator
supports (`type`, `enum`, `properties`, `required`, `additionalProperties`, `items`, `oneOf`/`anyOf`, numeric bounds,
`minLength`/`maxLength`/`pattern` — no `if/then`, `allOf`, `patternProperties`, `const`, `uniqueItems`, `minItems`).

Config-merge behavior to be aware of (and document): the user layer (`~/.config/sase/sase.yml`) _replaces_ the
bundled-default list, while machine overlays (`sase_<machine>.yml`) and project-local `sase/sase.yml` _concatenate_ onto
it — same as `mentor_profiles`.

---

## Phase highlights-create — `bob highlights create <md_file>` (repo: gh:bobs-org/bob-cli)

Open the repo with the `/sase_repo` skill (`sase repo open gh:bobs-org/bob-cli -r "..."`). This is a **Rust** CLI (clap
4.5 builder API, edition 2024). Read `sase/memory/cli_rules.md` via `/sase_memory_read` before starting:
subcommands/options sorted alphabetically, every public long option gets a short alias, help output must be excellent,
colored output preferred.

### Behavior

`bob highlights create <md_file>` converts `some/path/to/<name>.md` into `~/bob/lib/chat/<name>.pdf`:

1. **Read + title.** Title = markdown frontmatter `title:`, else first `# H1`, else the file stem. Refuse missing or
   non-`.md` input with a clear error.
2. **Render PDF** via pandoc shell-out (no md→PDF code exists in the repo; `/usr/bin/pandoc` + `xelatex` are installed
   on athena). Baseline invocation:
   `pandoc <md> -o <tmp> --toc --toc-depth=3 --pdf-engine=xelatex --resource-path=<md dir> -V colorlinks=true -V geometry:margin=1in --metadata title=<title>`
   — this yields a hyperlinked TOC page _and_ PDF `/Outlines` bookmarks (the "indexed" TOC) for free. Tune the defaults
   for beautiful, readable output (margins, link colors, a clean serif/sans pairing available via fontspec); the
   acceptance bar is that the beads research report (tables, code fences, deep heading nesting) renders cleanly. Resolve
   the pandoc binary through an env override (e.g. `BOB_PANDOC_COMMAND`, mirroring the `OB_COMMAND` precedent in
   `src/native/ob.rs`); if pandoc or the engine is missing or rendering fails, exit 1 with a styled error that includes
   pandoc's stderr — this output feeds the sase failure notification verbatim, so make it self-explanatory.
3. **Embed the Highlights marker** — a page-1 standalone `/Text` annotation whose `/Contents` is the `- key: value`
   list. Reuse the exact lopdf shape from the test helper `write_highlights_pdf_pages` (`tests/cli.rs:14232`) and
   `pdf_text_string` (`mod.rs:6138`); write with `atomic_save_pdf` (`mod.rs:6142`). Default marker: `- status: ready`,
   `- parent: obsidian_ref`, `- title: <title>`. Validate what you embed with the existing
   `parse_marker_with_normalization` (`mod.rs:6161`) + `validate_required_marker_keys` (`mod.rs:6241`) +
   `validate_marker_parent_value` (`mod.rs:6300`) before saving (parent must be a _bare_ target, not a wikilink). Verify
   the `obsidian_ref` default against the live vault's existing `~/bob/ref/chat/*.md` notes before hardcoding.
4. **Write** to `<lib_dir>/<ref_type>/<name>.pdf` (defaults `~/bob/lib/chat/`, honoring `BOB_DIR` /
   `BOB_HIGHLIGHTS_LIB_DIR`). Safety rails: refuse to overwrite an existing PDF without `--force`; refuse if a
   `<name>.md` already sits beside the target PDF (scan's `discover_sidecar_path` at `mod.rs:2780` checks
   `<stem>.md`-next-to-PDF _first_, so a stray source copy would be parsed as a Highlights sidecar); never copy the
   source markdown into the vault.
5. **Report.** Styled output via `Styler` (`src/native/style.rs`): on success print the `success_prefix` ok-line, the
   written PDF path, title/status/parent marker summary, page count, and a `next: bob highlights scan` hint. Failures go
   through the existing `report_result`/`CommandError` pattern (`mod.rs:626`), exit 1.

### Flags

`-d/--dry-run` (print planned paths + marker, `writes: none`, mirroring `sync`/`scan`), `-f/--force` (overwrite existing
PDF), `-P/--parent <note>` (default `obsidian_ref`), `-s/--status <status>` (default `ready`, validated against the
marker status set), `-t/--ref-type <dir>` (default `chat`), plus the shared config args (`with_config_args`).

### Wiring + chores

- `mod.rs:5011` `build_cli()`: add the `create` subcommand (alphabetically before `doctor`); dispatch arm at
  `mod.rs:562`; `run_create` beside `run_scan` (`mod.rs:571`) using `Config::from_matches` (`mod.rs:5162`).
- Add a pandoc availability check to `bob highlights doctor` (mirroring its `ob` check).
- Add a `bob highlights create` example to the top-level `Examples:` block (`src/runner.rs:265-289`).
- Docs: `README.md` highlights section (~line 589) and `docs/highlights-ref-sync.md` (~line 36); `justfile`
  `install-smoke` gets a `create --help` assertion (~line 52).
- Tests: unit tests for title extraction, output-path/ref-type derivation, marker composition/validation, and the
  overwrite/sidecar-collision refusals; integration tests in `tests/cli.rs` for dry-run and the full render (skip
  gracefully with a clear message when pandoc is unavailable in the environment, so CI without pandoc stays green —
  check what the repo's GitHub Actions runners provide and install pandoc there if cheap).
- Validation: `just all` (fmt + clippy + test) must pass. Commit via the `/sase_git_commit` skill flow from the opened
  checkout.

---

## Phase hooks-config — config section, matcher, list CLI (repo: sase)

Follow the `mentor_profiles` idiom throughout (`src/sase/config/mentor.py` — dataclass at :43, memoized loader keyed on
`current_config_token()` at :120, per-item parse with `ValueError`, fail-soft public accessor at :288, project
auto-scoping at :268).

1. **Schema** (`src/sase/config/sase.schema.json`): add `definitions.fileHook` and a root `file_hooks` array property
   (root is `additionalProperties: false`, so this is mandatory). Shape: `required: ["name", "command"]`,
   `additionalProperties: false`; `name` with a slug `pattern`; `command` with `minLength: 1`; `projects`/`sidecars`/
   `globs` as string arrays; `ops` items `enum: ["ADD", "MODIFY", "REMOVE"]`; `timeout` with a duration `pattern` (e.g.
   `^[0-9]+(ms|s|m|h)$`). Stay inside the Rust-validator keyword subset listed in the design overview. Add
   `file_hooks: []` handling consistent with how `mentor_profiles` treats absence (optional, no default entry needed in
   `src/sase/default_config.yml` unless the schema test requires it).
2. **Typed loader**: new `src/sase/config/file_hooks.py` with a frozen `FileHookConfig` dataclass (`name`,
   `description`, `command`, `projects`, `sidecars`, `globs`, `ops`, `timeout_seconds`), `__post_init__` validation
   (non-empty name/command, known ops, parseable timeout), `_load_file_hooks()` memoized on `current_config_token()`,
   per-entry warn-and-skip on invalid entries, public `get_all_file_hooks()` that never raises. Auto-scope project-local
   declarations to the detected project. Export through `src/sase/config/__init__.py`.
3. **Matcher**: pure functions (same module or a sibling) implementing the semantics from the design overview:
   `hook_matches_event(hook, event) -> bool` over an event carrying `(project, repo_kind, sidecar_role, rel_path, op)`,
   and `match_events(hooks, events) -> list[PlannedRun]`. Glob logic uses `wcmatch.glob.globmatch` with
   `GLOBSTAR | NEGATE | DOTGLOB` (+ the flag that makes negative-only pattern lists mean "everything except" — verify
   against wcmatch docs). Add `wcmatch` to `pyproject.toml` dependencies.
4. **CLI**: new `sase file-hook` command group with a `list` child (`-j/--json` flag; bare `sase file-hook` delegates to
   `list` via the central `_default_list_subcommands()` in `src/sase/main/parser.py` — do not re-wire per command).
   `list` renders each configured hook: name, description, command, filters, timeout, and the config layer it came from,
   colored per house style. Read `sase/memory/cli_rules.md` via `/sase_memory_read` first. Reserve space in the group
   for the internal `exec-batch` subcommand that the hooks-engine phase adds.
5. **Docs**: `docs/configuration.md` — new `### file_hooks` section (YAML example above, field table, matching
   semantics, merge behavior across layers, `Source:` line) + Table of Contents entry.
6. **Tests**: config-schema accept/reject cases (`tests/test_config_schema.py` + `tests/_config_schema_helpers.py`),
   loader fail-soft behavior, matcher unit tests covering: positive-OR, negative veto, negative-only lists, `*` vs `/`
   boundaries, `**`, ops filtering, projects/sidecars filters and their omitted-means-unrestricted defaults, project
   auto-scoping.
7. **Validation**: `just install` then `just check` must pass.

---

## Phase hooks-engine — event capture, detached runner, notifications (repo: sase)

### Event capture

Introduce a small engine module (e.g. `src/sase/file_hooks/` or `src/sase/workflows/commit/file_hook_engine.py`)
exposing `emit_file_hook_events(...)` used by all three producers. Each event: absolute path, repo-relative path, op,
repo root, repo kind (`primary` | `sidecar:<role>` | `linked:<name>` | `external:<name>`), project name.

1. **`CommitWorkflow`** (`src/sase/workflows/commit/workflow.py`): after `dispatch` succeeds (around the tracking steps
   at :296-370), take the returned commit SHA and derive per-file ops via
   `provider.diff_name_status("<sha>~1", "<sha>", cwd)` (`src/sase/vcs_provider/plugins/_git_query_ops.py:188`),
   canonicalized like `src/sase/ace/deltas/compute.py` (renames → REMOVE+ADD; root commit → all ADD). Reuse the existing
   cwd classification from `src/sase/workflows/commit/commit_tracking.py` (`_sdd_repo_name_for_commit_cwd` :394,
   `_external_repo_name_for_commit_cwd` :431) for repo kind/role, and `resolve_project_file()` for the project. Record a
   `file_hooks` step in the `CommitCheckpoint` `completed_steps` (`checkpoint.py:18-38`) so `--resume` never re-fires.
2. **`commit_sdd_files`** (`src/sase/sdd/_commit_store.py:33`): after a successful sidecar commit, same SHA-based
   name-status derivation; sidecar role comes from the commit target (`sdd_commit_targets` :131). This seam covers every
   research/plans/beads commit from the finalizer, bead CLI, publication, and TUI callers. Keep the added in-process
   work near-zero when no hooks are configured (memoized config, early return).
3. **`sase artifact create`** (`src/sase/artifact_cli/create.py:13` / `store_explicit_artifact_file`): emit one `ADD`
   event per stored artifact. Match globs against `source_path` made relative to the enclosing repo checkout when it is
   inside one (primary workspace or a `sase/repos/...` clone — derive kind/role the same way), else against the file's
   basename with no sidecar role. The hook command receives the **stored** artifact path (durable), not the
   possibly-moved source.

### Detached runner

- When matching yields ≥1 planned run, serialize a batch JSON (schema-versioned: events, matched hook names, resolved
  commands, repo root, project, commit SHA, timestamps) under `~/.sase/file_hooks/batches/`, then `subprocess.Popen` a
  detached `sase file-hook exec-batch <batch>` (`start_new_session=True`, stdout/stderr → a log under
  `~/.sase/file_hooks/logs/`). The producing process never waits. Commit latency and TUI responsiveness must be
  unaffected (`sase/memory/tui_perf.md` applies).
- `exec-batch` (internal subcommand: hidden from help, exempt from the short-alias rule) executes runs sequentially:
  `f"{hook.command} {shlex.quote(abs_path)}"`, `shell=True`, `cwd=repo_root`, inherited env, `timeout` enforced (timeout
  counts as failure). Each run's combined stdout/stderr goes to a per-run log file under `~/.sase/file_hooks/runs/`. The
  runner is crash-safe: a failure in one run still executes and reports the rest; finished batches are marked/moved so a
  re-invocation cannot double-run; batches/runs/logs older than ~30 days are opportunistically pruned on runner start.

### Notifications

One per hook run via `append_notification` (`src/sase/notifications/store.py:118`), modeled on the mentor-runner and
`notify_axe_error_digest` patterns (`src/sase/notifications/senders.py:136-171`):

- Common: `sender="file-hooks"`, `tags=["file-hooks", <hook name>]`, per-run log attached via `files` (the inbox list
  renders only `notes[0]` truncated to 50 chars, so the body must live in the attachment), `notes` lines carrying hook
  name, command line, file (repo-relative), op, project, repo/sidecar, duration, exit code.
- Success: `notes[0]` like `✅ research-highlights: <basename>`; no action; icon `✅`. Include the command's stdout in
  the attached log — for the research use-case that is where the written PDF path shows up.
- Failure (non-zero exit, timeout, spawn error): `notes[0]` like `❌ research-highlights failed: <basename>`; icon `❌`;
  `action="ViewErrorReport"`, `action_data["error_report_path"]=<run log>` so the full error output opens from the
  notification. Extend the sender sets in `src/sase/notifications/priority.py` so `file-hooks` failures classify as
  errors/priority alongside `axe`/`user-agent`.
- Update `docs/notifications.md` where senders/actions are cataloged.

### Tests + validation

- Unit: event derivation from real temp git repos (initial commit, adds/modifies/deletes, a rename), checkpoint/resume
  no-double-fire, batch serialization round-trip, runner success/failure/timeout paths writing correct notification rows
  (temp `HOME`), pruning, artifact-create event emission.
- Integration-style: end-to-end through `commit_sdd_files` in a temp store with a configured hook whose command is a
  tiny script — assert exactly one run per matched file with the file path as `argv[-1]`, and no runs (and no spawn)
  when nothing matches.
- `just install` then `just check` must pass.

---

## Phase deploy-verify — configure, install, verify end to end (runs on athena)

1. **Configure.** Open chezmoi via `/sase_repo` and add to `home/dot_config/sase/sase.yml` (top-level, per the design
   overview example — this is plain config, not a memory file):

   ```yaml
   file_hooks:
     - name: research-highlights
       description: Render new consolidated research reports into Highlights PDFs for the Obsidian reading queue.
       command: bob highlights create
       sidecars: [research]
       globs: ["20*/*/*.md", "!20*/*/*__*.md"]
       ops: [ADD]
       timeout: 120s
   ```

   Commit via `/sase_git_commit`, then run `chezmoi update -a --force` (required by the chezmoi repo's instructions).
   Note: `bob` resolves via `~/.cargo/bin` (chezmoi's own `bob_*` wrappers rely on this); if agent PATH turns out not to
   include it, use the absolute-path fallback form in `command`.

2. **Install the new bob.** From the bob-cli checkout (`/sase_repo`), install the binary produced by the
   highlights-create phase (`cargo install --path .` or the repo's `just` install target) and confirm
   `bob highlights create --help` renders.
3. **Verify `create` against the real report.** Run it on
   `<research sidecar root>/202607/sase_beads_close_integrity_and_capture/sase_beads_close_integrity_and_capture.md`
   (root from `sase repo path research --ensure`). Confirm: `~/bob/lib/chat/sase_beads_close_integrity_and_capture.pdf`
   exists; `pdfinfo`/a PDF read shows a title, multiple pages, and `/Outlines` bookmarks; the TOC page is present and
   hyperlinked; tables/code blocks render legibly; `bob highlights marker <pdf>` shows a valid marker (`status: ready`,
   bare `parent`, title). Fix-forward in bob-cli if the real report exposes rendering gaps.
4. **Verify `scan`.** Run `bob highlights scan` and confirm `~/bob/ref/chat/sase_beads_close_integrity_and_capture.md`
   is created with the managed frontmatter (`parent: "[[...]]"`, `type: "[[ref]]"`, `ref_type: chat`, `source_pdf`,
   lifecycle task line) and highlights region markers. `bob highlights doctor` stays clean.
5. **Verify the sase engine.** `sase file-hook list` shows `research-highlights` sourced from the user config layer.
   Then exercise the real pipeline: from the research sidecar checkout, create a throwaway file matching the globs (e.g.
   `202607/file_hook_e2e_check/file_hook_e2e_check.md` with a couple of headings), commit it through the normal sidecar
   commit path, and confirm: a batch file appears and completes, `~/bob/lib/chat/file_hook_e2e_check.pdf` is created,
   and a `file-hooks` success notification with the attached run log lands in the inbox (inspect via `/sase_notify`).
   Also verify the negative glob: a `..._e2e__x.md` sibling commit produces no run. Clean up the throwaway vault
   artifacts (PDF + ref note) and revert the throwaway research file afterward, leaving the real report's PDF/ref note
   in place.
