---
tier: tale
title: Add a flexible, beautiful `sase plan show` detail command
goal:
  Any way a user can name a plan — an explicit path, a `plans:` reference, a pending-approval prefix, a bare slug, or a
  bead id — resolves to exactly one plan and renders it as a colored, section-structured detail view that matches the
  ACE TUI's PLAN lane, with machine-readable `json` and byte-faithful `raw` output alongside it.
proposed_by: bbugyi200.athena.ud
create_time: 2026-08-06 15:48:40
status: wip
---

- **PROMPT:**
  [prompts/202608/plan_show_command.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/plan_show_command.md)

# Plan: Add a flexible, beautiful `sase plan show` detail command

## Why this is a tale

This is one cohesive CLI detail view built almost entirely from existing, already-tested primitives: the Rust plan
reference resolver, the Rust plan-search corpus walker, the shared `PlanDisplay` loader/renderer that the ACE TUI PLAN
lane already uses, and the pending-plan selector that `sase plan approve`/`reject` already share. The work is a single
serial chain — record model, resolver, renderer, CLI wiring, tests, docs — where every later step consumes the previous
step's type. Splitting it into phases would add handoff cost without unlocking any parallelism, and there is no Rust
core API change, no bead semantics change, and no cross-repository dependency to sequence.

## Current state and constraints

### What already exists (reuse it; do not reimplement it)

- **Reference resolution is Rust-owned.**
  `sase.sdd.plan_refs.resolve_plan_reference(value, workspace_dir=…, workspace_num=…)` and
  `resolve_plan_reference_from_roots(value, roots=…)` call `plan_reference_resolve` in
  `crates/sase_core/src/plan/refs.rs` and return a `PlanReferenceResolution` with `status` in
  `exact | drifted | ambiguous | missing`, a `resolved_path`, and ordered `candidates`. The core already accepts
  `plans:<shard>/<name>.md` **and** legacy marker paths (`sase/repos/plans/…`, `.sase/sdd/plans/…`, `sdd/plans/…`,
  `plans/…`), already resolves absolute paths, and already recovers six-digit month drift (`plans:202607/x.md` found
  under `202608/`) — reporting `drifted` for a single hit and `ambiguous` for several. A bare single-segment payload
  (`x.md`) deliberately does **not** drift-resolve.
- **`resolve_plan_roots(workspace_dir, workspace_num)`** returns the two ordered plan roots: the active store's `plans`
  kind root, then the machine-local `~/.sase/plans/` archive. `canonicalize_plan_reference_from_roots(path, roots=…)`
  turns a path below either root back into a canonical `plans:` reference, or returns `None`.
- **Corpus discovery is Rust-owned.** `sase.plan_search.facade.search(...)` returns ranked `PlanSearchMatch` values
  whose `Plan` carries `source` (`repo`/`local`), `kind`, `path`, `relpath`, `name`, `title`, `status`, `created_at`,
  `prompt_link`, `summary`, `body`, and `frontmatter`. It accepts `repo_root`/`local_dir`/`cwd` overrides, which is how
  the existing tests stay hermetic.
- **Plan loading and presentation are already shared with the TUI.** `sase.sdd.plan_display` publicly exports
  `load_plan_display`, `plan_file_metadata_from_content`, `unavailable_plan_metadata`, the `PlanDisplay` /
  `PlanFileMetadata` / `PlanDisplayPhase` values, the `COLOR_PLAN_*` palette, and the renderers `plan_field_rows`,
  `plan_provenance_rows`, `plan_lane_details`, `plan_phase_metadata`, `plan_logical_text`, `render_plan_document`, and
  `render_plan_lines`. `PlanDisplay` already carries title, goal, authored tier, existence/readability, phase
  availability, normalized epic phases, validation state, and the parsed PLAN/PROMPT/PARENT/BEAD/AGENTS/COMMITS
  provenance sections. `sase.sdd.plan_waves.plan_phase_waves` groups phases into dependency waves.
- **The pending-proposal selector is already shared.** `sase.main.plan_pending.resolve_pending_plan(selector)` resolves
  a notification id, a unique prefix, or (with `None`) the sole visible pending proposal, raising
  `PlanApprovalActionError` with the same messages `approve`/`reject` produce. `sase.main.plan_inventory` builds
  `ProposedPlan` rows carrying `id_prefix`, `agent`, `project`, `provider_model`, `plan_path`, `title`, `tier`, `age`,
  and `response_dir`.
- **Two sibling `show` commands define the house style.** `sase xprompt show` (`src/sase/xprompt/cli_show*.py`) resolves
  a name into a schema-versioned record, prints rule-separated uppercase sections through a color-aware `rich` console,
  supports `--format {full,json,raw}`, prints `unknown …:` plus `difflib` suggestions on a miss, and ends with a dim
  next-command hint. `sase bead show` (`src/sase/bead/cli_query.py`, `src/sase/main/parser_bead_queries.py`) supplies
  the option vocabulary: `-c/--color {auto,always,never}`, `-f/--format {compact,json,full}`, `-w/--wrap WIDTH`.
- **`sase agent prompts show PROMPT`** (`resolve_prompt_archive_file` in
  `src/sase/agents_sync/prompt_archive/validation.py`) is the precedent for flexible target normalization: it strips a
  `prompts:` prefix, a `./` prefix, a corpus-directory prefix, and a `.md` suffix, then matches the remainder against
  the bare name, `<month>/<name>`, and the corpus-relative path — raising a listing error when several entries match.

### Constraints

- **Do not reimplement core behavior in Python.** Reference parsing/resolution and corpus discovery/ranking stay in the
  Rust core; this command composes those bindings plus host-only stores (notifications, beads). No Rust changes are in
  scope.
- **Do not import `_`-prefixed modules or symbols across files.** `sase.sdd._plan_display_loading`,
  `_plan_display_models`, and `_plan_display_rendering` are private; import only from the public `sase.sdd.plan_display`
  facade. Symvision fails the build on cross-file private imports, and every new public symbol needs a real non-test
  consumer.
- **`-h/--help` must be excellent, subcommands and options alphabetically sorted, and every public long option must have
  a short alias.** `tests/main/test_parser_plan.py` enforces the sorted-subcommand set, help completeness, and the
  short-alias rule for the plan group.
- **`show` displays; `validate` judges.** A malformed plan must still render, with its diagnostics shown, and must not
  change the exit code. Validation for display uses `mode="launch"` (what `plan_file_metadata_from_content` and the TUI
  already use) so committed plans carrying SASE-managed frontmatter and provenance bullets do not produce noisy false
  alarms; the strict authoring verdict remains `sase plan validate`'s job and is pointed at from the hint line.
- Prefer colored output; keep every color behind `--color` and the `no_color` console flag so `never` and non-TTY output
  stay plain.

## Command surface

```
sase plan show [TARGET] [-c {auto,always,never}] [-f {compact,full,json,raw}]
               [-t {auto,bead,name,path,proposal,ref}] [-w WIDTH]
```

Positional `TARGET` (`nargs="?"`, metavar `TARGET`): the plan to show. Omitted means the sole visible pending plan
proposal, exactly as `sase plan approve` and `sase plan reject` treat an omitted `SELECTOR`.

Options, alphabetical, each with a short alias:

| Option           | Values                                            | Default | Meaning                                                                                                                                                                          |
| ---------------- | ------------------------------------------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `-c`, `--color`  | `auto`, `always`, `never`                         | `auto`  | Color output, matching `sase bead show`.                                                                                                                                         |
| `-f`, `--format` | `compact`, `full`, `json`, `raw`                  | `full`  | `compact` is the one-line row `sase plan search` prints; `full` is the section view; `json` is the schema-versioned envelope; `raw` writes the plan file's exact text to stdout. |
| `-t`, `--target` | `auto`, `bead`, `name`, `path`, `proposal`, `ref` | `auto`  | Force one interpretation of `TARGET` instead of walking the ladder.                                                                                                              |
| `-w`, `--wrap`   | integer >= 20, `auto`, `none`, `0`                | `120`   | Wrap goal, phase descriptions, and diagnostics prose, reusing `wrap_width` from `sase.main.parser_bead_common` and `DEFAULT_PROSE_WRAP_WIDTH` from `sase.markdown_wrap`.         |

Help text must state the accepted target forms and the ladder order, and the epilog must carry these examples:

```
examples:
  sase plan show
  sase plan show abcdef12
  sase plan show plans:202608/plan_show_command.md
  sase plan show 202608/plan_show_command
  sase plan show sase_plan_feature.md
  sase plan show sase-64
  sase plan show plans:202608/plan_show_command.md --format json
  sase plan show plan_show_command --format raw > plan.md
```

Exit codes: `0` when a plan was resolved and rendered (**including** a plan that fails validation); `1` when the target
misses, is ambiguous, or the resolved file cannot be read; `2` for invalid command usage (argparse). This mirrors
`sase plan validate`'s documented `0/1/2` contract, with the deliberate difference that plan invalidity is not a `show`
failure.

## Resolution ladder

In `auto` mode the rungs are tried in this fixed order and the **first definitive match wins**. Each rung either
resolves to exactly one path, reports an ambiguity (which stops the ladder), or declines and falls through.

1. **`path`** — after `~` expansion, `TARGET` names an existing file (absolute, or relative to the cwd). An explicit
   path a shell completed cannot mean anything else, so it wins outright.
2. **`ref`** — `resolve_plan_reference(TARGET, workspace_dir=…, workspace_num=…)` using
   `workspace_context_for_plan_resolution(Path.cwd())` for the workspace context. `exact` and `drifted` resolve (drift
   is surfaced in the output, never hidden); `ambiguous` reports the candidates; `missing` falls through. This rung is
   what accepts `plans:…` references, every legacy marker path, and month drift.
3. **`proposal`** — `resolve_pending_plan(TARGET)`; the notification's plan path is the target and the proposal's
   metadata is attached to the record. Skipped when `TARGET` contains `/` or ends in `.md`, so a path-shaped string is
   never probed against notification ids.
4. **`name`** — a shard/slug lookup across the discovered corpus. Normalize `TARGET` by stripping a `plans:` prefix, a
   leading `./`, and a trailing `.md`; then match case-insensitively against each discovered plan's `name`,
   `<shard>/<name>`, and corpus-relative path minus its kind directory and `.md` — the same normalization
   `resolve_prompt_archive_file` uses for prompts. Candidates come from `plan_search.facade.search(None, limit=0)`
   restricted to plan kinds (`tale`, `epic`, and the local archive kind), so discovery stays Rust-owned. One match
   resolves; several report an ambiguity.
5. **`bead`** — `TARGET` is an existing bead id whose `design` field is set; the stored reference is resolved through
   `describe_design_reference` (`sase.sdd.plan_ref_display`) so the CLI says exactly what `sase bead show` says about
   where a plan reference lands. Skipped when `TARGET` contains `/` or ends in `.md`.

Rationale for the order, which the help text and `docs/cli.md` must state: `path` and `ref` are exact and syntactic;
`proposal` is the freshest, most actionable context and reuses the selector vocabulary the sibling plan commands already
teach; `name` is a corpus search; `bead` is an indirection to a reference someone else already recorded.

With an explicit `-t/--target`, only that rung runs and its failure is a hard miss — no fallthrough — so scripts get
deterministic behavior. `-t auto` is the default and equals the ladder above.

### Miss output (stderr, exit 1)

Mirror `sase xprompt show`'s miss shape: a one-line `unknown plan: <target>` followed by a `suggestions:` block.
Suggestions come from `difflib.get_close_matches` over discovered plan names rendered as canonical `plans:` references,
plus the `id_prefix` of every visible pending proposal when `TARGET` is selector-shaped (no `/`, no `.md`). Print
nothing but the error line when there is nothing to suggest.

### Ambiguity output (stderr, exit 1)

Never guess. Print `ambiguous plan: <target> — N plans match`, then one aligned row per candidate carrying its canonical
`plans:` reference, tier, created date, and title, then a closing line telling the user to pass one of the printed
references or narrow with `--target`. The printed references must be directly re-runnable as `sase plan show` arguments.

## Record and JSON envelope

One immutable record, built once and consumed by every format:

- `target` — `raw` (the string the user typed, or `null`), `kind` (which rung matched), `status` (`exact`/`drifted`),
  and `candidates` (paths considered).
- `plan` — `reference` (canonical `plans:…`, or `null` outside both roots), `path`, `relpath`, `source`
  (`repo`/`local`/`file`), `exists`, `tier`, `status`, `title`, `goal`, `created_at`, `frontmatter`, `body`,
  `validation` (`ok` plus `diagnostics`), `provenance` (kind, entries, targets, omitted), `phases` (id, title,
  `depends_on`, size, model, description), and `waves`.
- `proposal` — the pending-approval context (`id`, `id_prefix`, `agent`, `project`, `provider_model`, `age`,
  `response_dir`) or `null`.
- `bead` — the bead id the plan was reached through, or `null`.

`--format json` prints this as `{"schema_version": 1, …}` following the `SHOW_SCHEMA_VERSION` precedent in
`sase.xprompt.cli_show_model`, with a module-level constant so the version has one home.

Build the record from a **single** file read: read the text once, pass it to the public
`plan_file_metadata_from_content(content)` for title/goal/tier/phases/validation/provenance, use
`sase.sdd.plan_tiers.parse_plan_frontmatter(content)` for the raw frontmatter map plus `status` and `create_time`, and
split the body below the frontmatter for `body`/`raw`. Assemble a `PlanDisplay` from that metadata with `actual_path`,
`display_path`, and `committed` filled from the resolution so the renderer consumes exactly the value type the TUI PLAN
lane consumes. On an unreadable or undecodable file, fall back to the public `unavailable_plan_metadata(...)` and let
the renderer say so plainly rather than raising.

Classify `source` by testing the resolved path against `resolve_plan_roots(...)`: below the first root is `repo`, below
the second is `local`, otherwise `file`.

## Output design

### `--format full` (default)

Section-structured like `sase xprompt show`: a header, then uppercase section titles separated by `rich.rule.Rule`,
using `SECTION_COLOR` from `sase.cli_show_palette` for titles and the `COLOR_PLAN_*` palette from
`sase.sdd.plan_display` for values, so the CLI reads as the same visual family as the ACE PLAN lane.

1. **Header** — the plan title on the left; on the right a dim chip strip: tier, plan status, phase count for epics, and
   `drifted` / `missing` / `invalid` when applicable. The goal follows on its own folded line.
2. **`PROPOSAL`** _(only when the target resolved through a pending proposal)_ — id prefix, agent, project,
   provider/model, age, and response dir. It comes first because a reader of a proposed plan is deciding whether to
   approve it.
3. **`PROPERTIES`** — `reference`, `tier`, `status`, `created`, `source`, `path`, `matched` (the rung plus the
   resolution status, e.g. `reference (month drift: authored 202607, found 202608)`), and `validation`. Reuse
   `plan_field_rows` for the canonical Title/Goal/Path rows so labels, alignment, and the dim-directory/bold-basename
   path styling match the TUI exactly.
4. **`PROVENANCE`** _(when the plan has header sections)_ — rendered from `plan_provenance_rows`, unchanged.
5. **`DIAGNOSTICS`** _(only when validation is not ok)_ — one wrapped line per diagnostic, before `PHASES`, because a
   failed validation is why phases may be unavailable.
6. **`PHASES`** _(epics only)_ — title line carries `N phases · M waves` from `plan_phase_waves`; each phase renders as
   ordinal, `◆`, title, size chip, then `plan_phase_metadata`'s `id · after … · model …` line and the folded
   description. Say `phases unavailable` plainly when `phase_availability` is `unavailable`.
7. **`BODY`** — the Markdown body below the frontmatter, printed through
   `rich.syntax.Syntax(body, "markdown", word_wrap=True)` the way `sase agent prompts show` prints an archived prompt.
   Print the whole body: `sase bead show` and `sase xprompt show` both print their full document, and `--format raw`
   exists for piping.
8. **Hint** — one dim trailing line, as `sase xprompt show` does: `sase plan approve <prefix>` /
   `sase plan reject <prefix>` for a proposal, otherwise `sase plan validate <path>`.

Omit any section with nothing to say; never print an empty section or a placeholder row.

### `--format compact`

The single row `sase plan search --format compact` prints for the same plan — status icon, tier/kind, display path,
title — so the two subcommands stay pixel-consistent.

### `--format raw`

Write the plan file's text to stdout unchanged, with no styling and no added or stripped trailing newline, for piping
and diffing. An unreadable file is exit `1` with the error on stderr.

## Implementation

### 1. Extract the shared plan status/kind style vocabulary

- Create `src/sase/plan_style.py` beside the existing `src/sase/cli_show_palette.py`, and move the status-icon,
  status-style, and kind-style tables and their three accessor functions out of `src/sase/main/plan_search_render.py`
  into it as public symbols.
- Re-point `plan_search_render.py` at the new module. This is a pure move: no icon, color, or fallback behavior may
  change, and the existing plan-search renderer tests must pass untouched.
- Both `plan_search_render.py` and the new show renderer import from it, so each symbol has two real non-test consumers
  and Symvision stays satisfied.

### 2. Add the record model

- Create `src/sase/plan_show/__init__.py` and `src/sase/plan_show/model.py` with frozen, slotted dataclasses for the
  record, the resolved-target descriptor, the proposal context, the miss, and the ambiguity, plus
  `SHOW_SCHEMA_VERSION = 1` and a `to_json_dict()` that emits the envelope described above.
- Keep the model free of I/O and of `rich`, so it can be asserted directly in tests.

### 3. Build the loader

- Create `src/sase/plan_show/load.py` turning one resolved path (plus resolution provenance, optional proposal, and
  optional bead id) into a record, using the single-read strategy, the public `plan_display` facade, and the root-based
  `source` classification and canonical-reference lookup described above.

### 4. Build the resolver

- Create `src/sase/plan_show/resolve.py` exposing one entry point that takes the raw target and the forced target kind
  and returns a record, a miss, or an ambiguity — never raising for user input.
- Implement the five rungs exactly as specified, including the `/`-and-`.md` guards that skip the `proposal` and `bead`
  rungs, and the no-fallthrough behavior under an explicit `--target`.
- Accept optional root/corpus/cwd overrides on the entry point so tests can drive it hermetically the way
  `tests/test_plan_search_facade.py` drives the facade.
- Translate `PlanApprovalActionError` from `resolve_pending_plan` into a miss or ambiguity value; do not let it escape
  as a traceback.

### 5. Build the renderer

- Create `src/sase/main/plan_show_render.py` with the full and compact renderers plus the miss and ambiguity printers,
  built on `rich` and gated by `resolve_color` from `sase.bead.cli_dep_render`. Build the console locally, modeled on
  the private console helper in `sase.xprompt.cli_show` (terminal-width aware, `markup`/`emoji`/`highlight` off) — copy
  the shape, do not import the private symbol.
- Compose sections from the public `plan_display` renderers (`plan_field_rows`, `plan_provenance_rows`,
  `plan_phase_metadata`, the `COLOR_PLAN_*` palette) and `phase_size_chip` from `sase.phase_size_presentation`. Do not
  restyle values that already have a shared style.
- Apply `--wrap` to goal, phase descriptions, and diagnostics prose through the existing wrap helpers; leave the
  syntax-highlighted body to `rich`'s own word wrapping.

### 6. Wire the CLI

- Add `register_plan_show_parser`-style registration inside `src/sase/main/parser_plan.py` with the description, epilog,
  and options above, keeping the subcommand registration and the option definitions alphabetically ordered.
- Create `src/sase/main/plan_show_handler.py` to validate args, call the resolver, dispatch on `--format`, and return
  the documented exit codes.
- Dispatch `show` from `handle_plan_command` in `src/sase/main/plan_command_handler.py`, following the existing
  `sys.exit(...)` shape used by the `list` and `search` branches.

### 7. Documentation

- Add `sase plan show` rows to both `sase plan` tables in `docs/cli.md`, keeping each table's existing ordering.
- Add a `sase plan show` paragraph to `docs/cli.md`'s plan narrative section next to the `sase plan search` and
  `sase plan validate` paragraphs, documenting the accepted target forms, the ladder order, the formats, and the exit
  codes.
- Cross-reference the command from the SDD plan section of `docs/sdd.md` where `sase plan search` and
  `sase plan validate` are already described.

## Testing

- **`tests/main/test_parser_plan.py`** (extend): add `show` to the expected sorted-subcommand set and to the short-alias
  loop; assert the choice lists render as `{compact,full,json,raw}` and `{auto,bead,name,path,proposal,ref}`, that
  `TARGET` is optional, and that the documented examples appear in the epilog.
- **`tests/plan_show/test_resolve.py`** (new): one test per rung against a temp store plus a temp local archive —
  explicit relative and absolute paths; a `plans:` reference; each legacy marker form; month drift reported as
  `drifted`; a `plans:` reference that resolves in two shards reported as an ambiguity; bare-slug and `<shard>/<slug>`
  matches with and without `.md`; a slug matching in both corpora reported as an ambiguity; a pending-proposal id and
  unique prefix; an ambiguous proposal prefix; the omitted-target case with zero, one, and several visible proposals; a
  bead id with and without a `design` value; the `/`-and-`.md` guards skipping the proposal and bead rungs; a forced
  `--target` failing without falling through; and a miss carrying close-match suggestions.
- **`tests/plan_show/test_load.py`** (new): `source` classification across repo, local, and outside-both-roots paths;
  canonical reference present in a root and `None` outside; frontmatter, status, created, and body extraction; epic
  phases and waves populated; a plan with no valid tier and a plan with broken frontmatter degrading to an honest record
  with diagnostics rather than raising; an unreadable file handled through `unavailable_plan_metadata`.
- **`tests/main/test_plan_show_render.py`** (new): the full render emits exactly the expected sections in order for a
  tale, an epic, an invalid plan, and a proposal-resolved plan; omitted sections really are absent; the drift note and
  the `missing` marker appear; `--color never` output contains no ANSI while `--color always` does; `--wrap` narrows
  prose; the compact row matches the plan-search row for the same plan.
- **`tests/main/test_plan_show_handler.py`** (new): exit `0` for a valid plan, `0` for an invalid one, `1` for a miss,
  an ambiguity, and an unreadable file; the JSON envelope's `schema_version` and full key set, with `proposal`/`bead`
  `null` when unused; `raw` reproduces the file's exact text; miss and ambiguity text goes to stderr and never to
  stdout.
- Drive resolution through the real Rust bindings over temp trees, as `tests/test_plan_search_facade.py` does, using
  `tests/sdd_policy_helpers.set_sdd_policy` and `write_sdd_store_record`; stub only the notification and bead stores.

## Verification

1. Run `just install` first — this is an ephemeral SASE workspace whose dependencies may have drifted.
2. Iterate with the focused parser, resolver, loader, renderer, and handler tests.
3. Eyeball the real thing before handing off: run `sase plan show` against a committed tale, a committed epic, and a
   `--color never` capture, and confirm the section hierarchy, alignment, and color read well at 80 and 120 columns.
4. Run `just check`, then `just check-full` before landing, since this change adds a CLI surface and moves a shared
   renderer module.

## Acceptance criteria

- `sase plan show` with no argument shows the sole visible pending proposal, and errors with the same wording as
  `sase plan approve`/`reject` when there is not exactly one.
- Every documented target form — explicit path, `plans:` reference, legacy marker path, month-drifted reference,
  `<shard>/<slug>`, bare slug with or without `.md`, pending-approval id or unique prefix, and bead id — resolves to the
  right plan, and `-t/--target` forces exactly one interpretation with no fallthrough.
- Nothing is ever silently guessed: every ambiguity prints its candidates as re-runnable `plans:` references and exits
  `1`, and every miss prints close-match suggestions.
- A plan that fails validation still renders in full with its diagnostics shown and exits `0`; only resolution and read
  failures exit `1`.
- `--format full` renders the header, the applicable sections in the documented order, and the trailing hint, reusing
  the shared `plan_display` renderers so the CLI and the ACE PLAN lane agree on labels, palette, and path styling;
  `--format compact` matches the `sase plan search` row; `--format json` emits the schema-versioned envelope; and
  `--format raw` reproduces the file byte for byte.
- `sase plan show --help` lists options alphabetically, gives every public long option a short alias, states the
  accepted target forms and ladder order, and carries the documented examples.
- No Rust core change, no cross-file private import, no reimplementation of reference resolution or corpus discovery in
  Python, `docs/cli.md` and `docs/sdd.md` are updated, and `just check-full` passes.

## Out of scope

- Resolving a plan by agent name through the TUI's associated-plan machinery. It is genuinely useful but pulls the ACE
  agent loader into a CLI path, and one agent can be associated with several plans, so it needs its own disambiguation
  design. Worth a follow-up task bead once this ladder is in use.
- A `--project` selector. The plan group has no cross-project precedent today (`sase plan search` scopes with `--source`
  only), and adding one here would be the first.
- Fetching a plan from a hosted URL, editing or opening the plan in `$EDITOR`, and any ACE TUI surface for this view.
