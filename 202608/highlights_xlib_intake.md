---
tier: tale
title: Create Highlights PDFs into xlib and intake them during scan
goal:
  "`bob highlights create` writes new PDFs into the Obsidian-Sync-tracked
  `~/bob/xlib/<ref-type>/` directory, and `bob highlights scan` moves every PDF found
  under `~/bob/xlib/` into the matching `~/bob/lib/` subdirectory before it plans or
  writes any reference note."
size: medium
proposed_by: bbugyi200.athena.zb
create_time: 2026-09-09 20:00:09
status: wip
---

# Plan: Create Highlights PDFs into `xlib` and intake them during `scan`

## Why

`bob highlights create` currently renders new PDFs straight into `~/bob/lib/<ref-type>/`
(default `~/bob/lib/chat/`). Bryan wants freshly created PDFs to land in a new
`~/bob/xlib/` directory that Obsidian Sync tracks, so a PDF rendered on the Linux host
reaches his synced devices. `~/bob/lib/` stays the durable library:
`bob highlights scan` intakes anything sitting in `xlib/` into the mirrored `lib/`
subdirectory, and only then plans and writes Obsidian reference notes — the intake
changes which notes should exist, so it must run first.

Mnemonic for the naming used throughout this plan: `xlib` is the _intake_ directory
(new, synced, transient); `lib` is the _library_ directory (durable, archival).

## Current behavior (verified in this repo)

All `bob highlights` code lives in `src/native/highlights_ref/`:

- `mod.rs` (~9.2k lines): `Config`, CLI wiring, `scan`, `sync`, `doctor`, `marker`,
  marker parsing, note rendering.
- `create.rs` (~840 lines): the `create` subcommand.

Key facts the implementation depends on:

- `Config` (`mod.rs:113-118`) holds `bob_dir`, `lib_dir`, `ref_dir`.
  `Config::from_matches` (`mod.rs:5182`) resolves `lib_dir`/`ref_dir` through
  `configured_path(matches, <arg>, <env>, <default>, &bob_dir)` (`mod.rs:5943`), which
  prefers the CLI flag, then the env var (`BOB_HIGHLIGHTS_LIB_DIR` /
  `BOB_HIGHLIGHTS_REF_DIR`), then the default (`lib` / `ref`), and resolves relative
  values under `bob_dir`.
- `configured_path` uses `matches.get_one::<OsString>(arg_name)`, which **panics** in
  clap when the argument is not defined for the invoked subcommand. Every subcommand
  that builds a `Config` (`create`, `doctor`, `marker`, `scan`, `sync`) therefore has to
  declare the new flag.
- CLI arg builders live at `mod.rs:5062-5180`: `with_config_args` (`doctor`, `marker`),
  `with_scan_args`, `with_sync_args`, plus `create::command()` in `create.rs:85`.
- `create.rs:297-302` computes
  `target = config.lib_dir.join(ref_type).join(stem).with_extension("pdf")` and
  `sidecar = target.with_extension("md")`, then refuses an existing sidecar
  unconditionally and an existing target unless `--force` (`create.rs:304-316`).
- `scan_library` (`mod.rs:668`) calls `collect_pdf_paths` (`mod.rs:2312`, lib only,
  recursive, skips `.git` via `should_skip_scan_dir`), then
  `validate_output_collisions`, then `plan_pdfs`, then `finalize_annotation_task_plans`,
  then `ensure_safe_to_write`, then `execute_pdf_sync` per PDF.
- `pdf_path_metadata` (`mod.rs:6761`) derives the reference-note path and `ref_type`
  from the PDF path relative to `lib_dir`; PDFs outside `lib_dir` fall back to a flat
  `ref/<stem>.md` target with no `ref_type`.
- `discover_sidecar_path` (`mod.rs:2800`) resolves a Highlights sidecar as `<stem>.md`
  (file) or `<stem>.textbundle/` (directory containing `text.md` or `text.markdown`,
  `TEXTBUNDLE_TEXT_FILES` at `mod.rs:77`).
- `ensure_safe_to_write` (`mod.rs:2444`) refuses to touch vault files that are dirty in
  the vault Git worktree; it only inspects paths the plans would write.
- Help ordering follows the SASE CLI rules note (`sase memory read cli_rules.md`):
  options are declared — and asserted in tests — in short-flag alphabetical order, so a
  new `-x, --xlib-dir` is declared **last** in every option list. Every public long
  option needs a short alias; `-x` is unused across `bob highlights`.

## Desired behavior

1. `bob highlights create report.md` writes `<xlib-dir>/<ref-type>/report.pdf` (default
   `~/bob/xlib/chat/report.pdf`) and refuses up front if the eventual library
   destination `<lib-dir>/<ref-type>/report.pdf` (or its sidecar) already exists.
2. `bob highlights scan` first moves every PDF under `<xlib-dir>` — with its Highlights
   sidecar, when present — to the mirrored path under `<lib-dir>`, and only then
   collects library PDFs, plans, and writes reference notes.
3. `<xlib-dir>` is tracked by Obsidian Sync (verification steps below; no vault content
   changes are needed).

## Implementation

### Step 1 — Add the `xlib` directory to `Config` and the CLI (`mod.rs`)

- Add constants beside the existing ones (`mod.rs:33-37`):
  `const DEFAULT_XLIB_DIR: &str = "xlib";` and
  `const ENV_XLIB_DIR: &str = "BOB_HIGHLIGHTS_XLIB_DIR";`.
- Add `xlib_dir: PathBuf` to `Config` and resolve it in `Config::from_matches` with
  `configured_path(matches, "xlib-dir", ENV_XLIB_DIR, DEFAULT_XLIB_DIR, &bob_dir)`.
- Add `fn xlib_dir_arg() -> Arg` next to `lib_dir_arg`: `--xlib-dir` / `-x`,
  `value_name("PATH")`, `value_parser(OsStringValueParser::new())`, help text
  `"Highlights PDF intake directory; defaults to BOB_HIGHLIGHTS_XLIB_DIR or xlib"`.
- Declare it **last** in `with_config_args`, `with_scan_args`, `with_sync_args`, and
  `create::command()` so `-x` sorts after `-w` / `-t` in help output.
- `print_config_report` (`mod.rs:5017`): print `xlib_dir: <path>` after `ref_dir`.
- Reject an overlapping configuration once, in a shared helper
  (`fn validate_library_layout(config: &Config) -> Result<()>`): fail when
  `xlib_dir == lib_dir`, when `xlib_dir` is inside `lib_dir`, or when `lib_dir` is
  inside `xlib_dir`, because intake would then move PDFs within a scanned tree or scan
  the same PDF twice. Call it at the top of `scan_library` and from `doctor_vault`.

### Step 2 — `create` renders into `xlib` (`create.rs`)

- In `plan_create` (`create.rs:261`), build
  `target = config.xlib_dir.join(&options.ref_type).join(stem).with_extension("pdf")`.
  `sidecar` stays `target.with_extension("md")`.
- Add `library_destination: PathBuf` to `CreatePlan`, computed as
  `config.lib_dir.join(&options.ref_type).join(stem).with_extension("pdf")`.
- Keep the existing guards against the `xlib` target (`--force` required) and the
  same-stem Markdown sidecar beside the `xlib` target (always refused).
- Add a new guard, **not** overridable by `--force`: refuse when `library_destination`
  exists, or when its `.md` / `.textbundle` sidecar exists. Error text must name both
  paths and the remedy, e.g.
  `refusing to create <xlib target> because the library destination already exists: <lib destination>; remove or rename the archived copy before recreating it (bob highlights scan would refuse to move the new PDF over it)`.
  Rationale: the library copy carries prior annotations and its sidecar; silently
  clobbering it during intake would destroy them, and `--force` at create time cannot
  make the later intake safe.
- `print_plan` (`create.rs:633`) prints `library_destination: <path>` after
  `sidecar_guard:` so `--dry-run` shows where `scan` will move the PDF.
- Update the `create` `after_help` (`create.rs:145`) to say the PDF is written to the
  intake directory and that `bob highlights scan` moves it into the library.
- Update the `config()` test helper in `create.rs`'s test module (`create.rs:688`) to
  set `xlib_dir: root.join("xlib")`.

### Step 3 — `scan` intakes `xlib` PDFs before planning (`mod.rs`)

Add an intake pre-pass with the same "plan, validate, then execute" shape the rest of
the command uses.

- `struct IntakeMove { source: PathBuf, destination: PathBuf, companions: Vec<(PathBuf, PathBuf)> }`
  where `companions` are `(source, destination)` pairs for the PDF's Highlights sidecar.
- `fn plan_xlib_intake(config: &Config) -> Result<Vec<IntakeMove>>`:
  - Return `Ok(Vec::new())` when `config.xlib_dir` is not a directory (a fresh vault has
    no `xlib/` until `create` makes one) — this must not be an error.
  - Reuse `collect_pdf_paths_from_dir` to walk `xlib_dir` recursively (it already skips
    `.git`), and sort for deterministic output.
  - For each PDF: `relative = source.strip_prefix(&config.xlib_dir)`,
    `destination = config.lib_dir.join(relative)`.
  - Companions: the same-stem `<stem>.md` file and the same-stem `<stem>.textbundle`
    directory when either exists (mirror `discover_sidecar_path`'s notion of a sidecar,
    but move the whole `.textbundle` directory rather than the inner text file, so
    TextBundle image assets travel with it). Moving sidecars is required for
    correctness: leaving one behind makes the moved PDF look sidecar-less, and the next
    sync would regenerate the note's managed body with its highlights removed.
  - Collect conflicts instead of failing on the first one: any `destination` (PDF or
    companion) that already exists. If any conflict exists, return a single
    `CommandError` listing every `source -> destination` pair and the remedy, in the
    style of the existing `output path collision(s) detected before writes:` message.
    This is a hard preflight failure for the whole scan — consistent with
    `validate_output_collisions`, and it keeps a half-applied intake from leaving the
    library in a mixed state.
- `fn execute_xlib_intake(moves: &[IntakeMove]) -> Result<()>`: for each move,
  `fs::create_dir_all(destination.parent())` then `fs::rename` the PDF and each
  companion, wrapping errors as `move <source> to <destination>: {error}`. `fs::rename`
  is correct here because both directories live inside the vault (same filesystem).
- Wire into `scan_library` (`mod.rs:668`), before `collect_pdf_paths`:

  ```rust
  validate_library_layout(config)?;
  let intake = plan_xlib_intake(config)?;
  if !options.dry_run {
      execute_xlib_intake(&intake)?;
  }
  let pdfs = collect_pdf_paths(config, options.dry_run)?;
  ```

- Intake deliberately runs **before** `ensure_safe_to_write`'s dirty-worktree check and
  is not subject to it: a PDF that `create` just rendered is untracked in the vault Git
  repo (so always "dirty"), and intake never overwrites an existing path, so a move
  cannot lose committed content. A tracked `xlib` PDF (synced from another device and
  committed by `bob bulk-git-commit`) moves as an ordinary Git rename. State this in the
  doc update.

### Step 4 — Keep dry-run previews faithful (`mod.rs`)

Without this step, `scan --dry-run` would report the planned move but omit the reference
notes that the move makes due — the exact ordering problem this plan exists to fix,
mirrored into the preview.

- `pdf_path_metadata` (`mod.rs:6761`): after the existing
  `strip_prefix(&config.lib_dir)` branch, add an equivalent
  `strip_prefix(&config.xlib_dir)` branch that derives `note_relative_path` and
  `ref_type` the same way. An `xlib` PDF therefore resolves to the same reference note
  it will own after intake. (`Step 1`'s overlap validation keeps the two branches
  unambiguous.)
- `collect_pdf_paths` gains an `include_xlib: bool` parameter (pass `options.dry_run`
  from `scan_library`, `false` from `doctor_vault`'s existing call at `mod.rs:2188`).
  When `true` and `xlib_dir` is a directory, append its PDFs and re-sort. In a non-dry
  run `xlib` is already empty by this point, so behavior is unchanged.
- `FIELD_SOURCE_PDF` is a pipeline field rewritten on every sync (`PIPELINE_FIELDS`,
  `mod.rs:89`), so an `xlib`-relative `source_pdf` written by a direct
  `bob highlights sync <xlib pdf>` self-heals on the next scan after intake. No extra
  handling needed; mention it in the doc.

### Step 5 — Report the intake (`mod.rs`)

- Verbose (`print_verbose_scan_plan_report`, `mod.rs:1452`): after `pdf_count:`, print
  `intake_moves: <N>` and one line per move,
  `intake: <would-move|moved> <vault-relative source> -> <vault-relative destination>`.
  Reuse `vault_relative_path_value` (`mod.rs:6819`) and `display_path` (`mod.rs:1503`)
  for stable, forward-slash output.
- Concise default output: before the `Scanning N PDFs in lib` header
  (`print_scan_header`, `mod.rs:1473`), print one styled line per move — `moved` /
  `would move`, the vault-relative source, an arrow, and the vault-relative destination
  — using `Styler` (`src/native/style.rs`) so piped output stays ANSI-free
  (`assert_stdout_has_no_ansi` in the tests depends on this). Print nothing extra when
  there are no moves.
- Do not change the scan summary counters; intake counts are reported separately from
  note/PDF write counts.

### Step 6 — `doctor` reports the intake directory (`mod.rs:2159`)

- `print_path_check("xlib_dir", &config.xlib_dir, config.xlib_dir.is_dir())` after the
  `ref_dir` check, but treat a missing `xlib_dir` as a **warning**, not a failure:
  `bob highlights create` creates it on demand.
- Print `xlib_pending: <N>` (PDFs awaiting intake) and, when `plan_xlib_intake` reports
  conflicts, surface that error as a doctor failure so the problem is visible before the
  next scan aborts.
- Call `validate_library_layout` and report a misconfigured overlap as a failure.

### Step 7 — Confirm Obsidian Sync tracks `xlib` (verification, no code)

This host syncs through `obsidian-headless` (`ob`), not a GUI Obsidian session. Verify —
do not silently reconfigure:

1. `ob sync-status --path ~/bob` must list `pdf` in `File types` (it does today:
   `image, audio, pdf, video`).
2. The headless client's per-vault config
   (`~/.config/obsidian-headless/sync/<vault-id>/config.json`) currently has no
   `excludedFolders` key, so nothing under the vault is excluded and `xlib/` syncs as
   soon as it exists. If exclusions are ever introduced, `xlib` must not be among them:
   `ob sync-config --path ~/bob --excluded-folders "<comma-separated list without xlib>"`.
3. The vault repo's `.gitignore` un-ignores `*.pdf` at any depth and allows directory
   traversal, so `xlib/**/*.pdf` is Git-tracked exactly like `lib/**/*.pdf`. No
   vault-side change is required.
4. Owner-side manual check (cannot be done from this host): in Obsidian on the
   Mac/iPhone/iPad, Settings → Sync → Excluded folders must not list `xlib`.
5. Finding to report, not to act on: on this Linux host `lib/` is **not** excluded from
   Obsidian Sync today — the headless sync log shows recent
   `Upload complete lib/chat/*.pdf` entries. Excluding `lib/` is out of scope for this
   plan; flag it to the owner instead of changing sync configuration.

No migration of existing `~/bob/lib/chat/*.pdf` files: they stay where they are, and
`scan` keeps processing them from `lib/` exactly as before.

### Step 8 — Docs

- `README.md`
  - Both `bob highlights` synopsis blocks: add `[-x|--xlib-dir PATH]` (last) to
    `create`, `doctor`, `marker`, `scan`, and `sync`.
  - Line ~714: `TOC-indexed PDF at lib/chat/<basename>.pdf` becomes
    `xlib/chat/<basename>.pdf`, and note that `scan` moves it to
    `lib/chat/<basename>.pdf`.
  - In the `scan` paragraph, describe the intake step, its ordering before note writes,
    and the refusal when a library destination already exists.
- `docs/highlights-ref-sync.md`
  - `MVP Status`: add intake to the `scan` description and update the command list (line
    ~37-42) with `-x, --xlib-dir`.
  - The `create` paragraph (line ~45): `<xlib-dir>/<ref-type>/<basename>.pdf`, default
    `~/bob/xlib/chat/<basename>.pdf`, plus the new library-destination refusal.
  - `Default Paths`: document `BOB_HIGHLIGHTS_XLIB_DIR=xlib`, resolution rules identical
    to the other two, and the `xlib/<rel> -> lib/<rel>` intake mapping with a worked
    example (`~/bob/xlib/chat/report.pdf` -> `~/bob/lib/chat/report.pdf` ->
    `~/bob/ref/chat/report.md`).
  - `Scan, Safety, and Git/ob Behavior`: intake runs first; sidecars move with the PDF;
    an existing destination aborts the scan before any write; dry runs move nothing but
    list planned moves and still preview the notes those moves would make due; intake is
    exempt from the dirty-target refusal and why.
  - Check `MacBook Setup Guide` and `Scheduled Scan` for `lib/chat` references and
    update any that describe where `create` writes.

### Step 9 — Tests

Unit tests in `create.rs`'s test module:

- `plan_derives_ref_type_output_and_valid_marker`: expect `xlib/books/report.pdf` and
  `xlib/books/report.md`.
- `plan_refuses_existing_pdf_without_force` /
  `plan_refuses_highlights_markdown_sidecar_even_with_force`: retarget to
  `xlib/chat/...`.
- New: existing `lib/chat/report.pdf` refuses creation even with `--force`, and the
  error names both paths.
- New: existing `lib/chat/report.md` sidecar refuses creation.

Unit tests in `mod.rs`'s test module:

- `plan_xlib_intake` maps nested paths (`xlib/chat/a/b.pdf` -> `lib/chat/a/b.pdf`),
  includes `<stem>.md` and `<stem>.textbundle` companions, and returns empty when `xlib`
  is missing.
- Conflict detection lists every colliding destination.
- `pdf_path_metadata` derives the same note path and `ref_type` for `xlib/chat/x.pdf` as
  for `lib/chat/x.pdf`.
- `validate_library_layout` rejects equal and nested `lib`/`xlib` values.

Integration tests in `tests/cli.rs` (helpers already present: `bob_command`,
`write_highlights_pdf`, `write_file`, `assert_success`, `stdout`, `assert_text_order`,
`assert_stdout_has_no_ansi`, `TempDir`):

- Update existing expectations that hard-code `create`'s output directory: the dry-run
  assertion at ~line 9427 (`lib/papers/...` -> `xlib/papers/...`), and the render test
  at ~lines 9517 and 9576 (`lib/chat/...` -> `xlib/chat/...`).
- Update `highlights_create_help_lists_options_alphabetically`,
  `highlights_ref_scan_help_lists_options_alphabetically`, and
  `highlights_ref_sync_help_lists_options_alphabetically` to expect `-x, --xlib-dir`
  last; extend `highlights_ref_short_options_are_accepted` with `-x xlib`.
- New: `scan` moves `xlib/chat/x.pdf` to `lib/chat/x.pdf` **and** writes `ref/chat/x.md`
  in the same run (the ordering requirement), leaving nothing behind in `xlib`.
- New: `scan --dry-run` reports the planned move, previews the reference note it would
  create, writes nothing, and leaves the PDF in `xlib`.
- New: `scan` moves a `<stem>.md` sidecar with its PDF, and the generated note contains
  the sidecar's highlights (proving the sidecar was not orphaned). Cover
  `<stem>.textbundle` too.
- New: `scan` fails, before writing any note, when `lib/chat/x.pdf` already exists while
  `xlib/chat/x.pdf` is present; assert the PDF stays in `xlib` and no note is created.
- New: `create` refuses when `lib/chat/report.pdf` exists, with and without `--force`.
- New: `doctor` prints `xlib_dir` and `xlib_pending` and stays zero-exit when `xlib/`
  does not exist.

## Verification

```bash
just fmt            # cargo fmt --check
just lint           # cargo clippy --all-targets --all-features
just test           # cargo test
```

Then a read-only smoke check against a scratch vault (never `~/bob`):

```bash
bob highlights create --dry-run -b /tmp/scratch-vault some.md   # shows xlib/chat/... target
bob highlights scan --dry-run -b /tmp/scratch-vault --verbose   # shows intake lines + note preview
bob highlights doctor -b /tmp/scratch-vault
```

Do not run a writing `bob highlights scan` against `~/bob` as part of implementation;
leave that to the owner.

## Decisions and rejected alternatives

- **Hard preflight failure on an intake conflict**, rather than a per-PDF `plan_failure`
  that lets the rest of the scan proceed. It matches the existing all-or-nothing
  `validate_output_collisions` preflight, avoids a partially applied intake, and the
  `create`-time guard makes the case rare.
- **`create` refuses an existing library destination even with `--force`.** `--force`
  means "replace the target I am writing now"; it cannot authorize the later intake to
  overwrite an archived, possibly annotated library copy. The owner deletes or renames
  that copy deliberately.
- **Sidecars move with the PDF.** Strictly, the request names PDFs only, but a sidecar
  left in `xlib` would silently strip highlights from the generated note on the next
  sync.
- **`pdf_path_metadata` learns about `xlib`** so dry runs preview the real post-intake
  note set. The alternative — previewing only the pre-move library — reproduces the
  ordering bug inside `--dry-run`.
- **No `bob nightly` change.** `nightly` does not invoke `bob highlights`; `scan` stays
  user- or scheduler-invoked.
