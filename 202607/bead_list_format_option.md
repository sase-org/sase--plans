---
tier: tale
title: Add -f/--format compact|full|json to sase bead list
goal: sase bead list accepts -f/--format with compact (unchanged default), full (bead
  show detail per bead), and json (a stable machine-readable envelope), reusing one
  shared detail renderer with sase bead show and sase bead search.
create_time: 2026-07-27 08:40:13
status: done
---

- **PROMPT:** [202607/prompts/bead_list_format_option.md](prompts/bead_list_format_option.md)
- **AGENTS:**
  - [bbugyi200.athena.m5](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.m5/README.md)
  - [bbugyi200.athena.m5--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.m5.md#member-code)
- **COMMITS:**
  - [672ecbb](https://github.com/sase-org/sase/commit/672ecbb4c835507817b78af6c573c399145c3b08) — feat(bead): add list output formats

# Plan: `sase bead list` — `-f | --format <compact|full|json>`

## Goal

Give `sase bead list` the same output-format switch that `sase bead search` already has:

```
-f, --format {compact,full,json}   Output format (default: compact)
```

- **`compact`** (default) — byte-for-byte identical to today's listing, so nothing existing changes.
- **`full`** — the `sase bead show` detail block for every listed bead, sections joined by a 60-dash rule (exactly how
  `sase bead search --format full` renders today).
- **`json`** — a stable, machine-readable envelope so scripts/agents can consume `list` without screen-scraping.

## Background / current behavior

- `sase bead list` is handled by `handle_bead_list` in `src/sase/bead/cli_query.py`. It resolves `--status` / `--type` /
  `--tier` filters, falls back to `--status closed` when the default open/claimed/in-progress view is empty and no
  explicit `--status` was given, applies `--limit` (defaulting to 20 whenever closed beads are in scope), and prints one
  line per bead: `{icon} {id} · {title}{ ← parent}`.
- The `list` subparser lives in `src/sase/main/parser_bead.py` (`register_bead_parser`), around the `# sase bead list`
  comment. It currently declares `-n/--limit`, `-s/--status`, `--tier`, `-t/--type` — no `--format`, no `description`,
  no `epilog`.
- The sibling `search` subparser in the same file already declares `-f/--format` with
  `choices=["compact", "json", "full"]`, `default="compact"`, plus a description and an examples epilog. Its handler
  `handle_bead_search` dispatches on `args.format` to `_render_search_compact` / `_render_search_json` /
  `_render_search_full`, all in `src/sase/bead/cli_query.py`.
- `_render_search_json` emits `{"query", "count", "results": [{"issue": <wire dict>, "matched_fields": [...]}]}` built
  from the module-local `_issue_to_wire_dict` / `_dependency_to_wire_dict` helpers.
- `_render_search_full` renders each match by calling `handle_bead_show(argparse.Namespace(id=...))` inside
  `contextlib.redirect_stdout`, rstripping, and joining with `f"\n{'-' * 60}\n"`.
- `sase bead list` deliberately does **not** use the Rust fast path: `try_handle_bead_fast_path` in
  `src/sase/main/bead_fast_path.py` returns `None` for `argv[0] in {"list", "show"}`, and commit `4d3264c36`
  ("feat(bead): limit list results and fall back to closed") states the reason — _"Keep bead list on the argparse/Python
  path so the presentation behavior stays centralized."_ `sase bead search --format full` is likewise deferred to Python
  by `_search_uses_full_format`.
- Bare `sase bead` already delegates to `sase bead list` through `_default_list_subcommands()` in
  `src/sase/main/parser.py`, which also copies the `list` subparser's defaults onto the group parser.

## Design decisions (please confirm at review)

1. **Python-only change; no `sase-core` work.** `bead list` is intentionally off the Rust fast path (see Background), so
   `crates/sase_core/src/bead/cli.rs::handle_list` is unreachable from the CLI today and is already behind Python — it
   implements neither `--limit` nor the implicit-closed fallback. Adding `--format` there would deepen a divergence
   nothing executes. This also matches the Rust-core boundary memo: the JSON/full rendering here is presentation glue
   over an already-Rust-backed read (`BeadProject.list_issues`), and it reuses the wire helpers that `bead search`
   already keeps in Python.
   - _Alternative considered:_ mirror `--format` into `sase_core`'s `handle_list` and re-enable the `list` fast path.
     Rejected as a separate, larger change (it would also have to port `--limit` + the closed fallback + the notice
     text, and re-open the "presentation stays centralized" decision). Easy to schedule later if you want `list` fast.

2. **`compact` is the default and is byte-identical to today.** Every existing golden (`list.stdout`,
   `list_limit.stdout`, `list_implicit_closed*.stdout`, `list_closed_*.stdout`, `list_empty.stdout`) must keep passing
   untouched. This is the regression contract for the change.

3. **`full` reuses the `show` renderer, so `list -f full` == concatenated `bead show` output.** Same 60-dash separator
   as `search --format full`. This keeps one detail-rendering implementation for `show`, `search -f full`, and
   `list -f full`.

4. **Extract a shared `_render_issue_detail(view, issue, ...) -> str` (required, not cosmetic).** Today
   `_render_search_full` re-enters `handle_bead_show` per match, and `handle_bead_show` opens its **own**
   `get_read_view()` (which runs `get_project()` → `resolve_beads_location` → possible store materialization) _and_
   calls `view.list_issues()` over the whole store to compute the `BLOCKS` section. For `search` that is bounded by the
   query; for `list` it is unbounded — `sase bead list -f full` on a store with N listed beads would do N store opens
   and N full-store scans (O(N²)). Extracting a pure renderer that takes an already-open `view` lets `list -f full` and
   `search -f full` render every bead from a single open view, and removes the `contextlib.redirect_stdout` hack.
   - `handle_bead_show` becomes: open view → resolve issue (same `KeyError` → stderr + `exit(1)` behavior) →
     `print(_render_issue_detail(...), end="")`.
   - Output must stay byte-identical; `test_handle_bead_search_full_reuses_show_rendering` in
     `tests/test_bead/test_cli_search.py` already pins `search -f full` == `show`, and is the guard for this refactor.
   - Also hoist the design-path decision: `_display_design_path` calls `resolve_sdd_store(Path.cwd(), 1)` on every
     invocation, so compute the "relativize?" boolean once per command and thread it into `_render_issue_detail`.

5. **`json` never mixes prose into stdout.** The two human affordances change shape under `json`:
   - `No issues found.` → an envelope with `"count": 0, "total": 0, "results": []` and exit code 0.
   - `No open beads to show — defaulting to --status closed.` → **not printed at all**; the fallback is reported as the
     `"implied_status_closed": true` field. (Not stderr: that would be noise for `... | jq` pipelines, and
     `search --format json` likewise prints nothing but the envelope.)

6. **JSON envelope shape** — fixed key order for diff stability, modeled on `sase prompt search --format json` (which
   uses `count`/`total` to make truncation explicit) rather than on `bead search`'s `results[].issue` nesting; there is
   no `matched_fields` to attach, so results are flat wire dicts:

   ```json
   {
     "count": 2,
     "total": 37,
     "statuses": ["closed"],
     "implied_status_closed": true,
     "results": [{ "id": "beads-1", "title": "...", "status": "closed", "issue_type": "plan", "...": "..." }]
   }
   ```

   - `total` = beads matching the filters **before** `--limit`; `count` = beads actually emitted. This makes the
     implicit 20-row cap on closed listings visible to consumers.
   - `statuses` = the statuses actually queried (so the closed fallback is self-describing).
   - `implied_status_closed` distinguishes the fallback from an explicit `--status closed` (both yield
     `statuses == ["closed"]`).
   - Each element is `_issue_to_wire_dict(issue)` — the exact per-issue schema `bead search --format json` already
     emits, reused verbatim so both commands stay in sync.

7. **`--limit` semantics are unchanged for every format.** The limit (explicit, or the implicit 20 when closed beads are
   in scope) is applied to the chosen result set before rendering, in all three formats.

8. **No `--color` for `list`.** `search`'s `-c/--color` only takes effect on the Rust fast path; Python's renderers are
   colorless. Colorizing `list` is a separate change (noted in Out of scope).

## Behavior matrix

| Invocation                                          | Result                                                                                       |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `sase bead list`                                    | unchanged compact rows (default `--format compact`)                                          |
| `sase bead` (bare)                                  | delegation notice + unchanged compact rows                                                   |
| `sase bead list -f compact`                         | identical to `sase bead list`                                                                |
| `sase bead list -f full`                            | `show` block per bead, joined by `\n` + 60 dashes + `\n`                                     |
| `sase bead list -f json`                            | envelope only, exit 0                                                                        |
| `sase bead list -f full` (no matches)               | `No issues found.`                                                                           |
| `sase bead list -f json` (no matches)               | `{"count": 0, "total": 0, "statuses": [...], "implied_status_closed": false, "results": []}` |
| `sase bead list -f full` (implicit closed fallback) | notice line, then the `show` blocks                                                          |
| `sase bead list -f json` (implicit closed fallback) | no notice; `"implied_status_closed": true`, `"statuses": ["closed"]`                         |
| `sase bead list -f json -n 2` over 37 closed        | `"count": 2, "total": 37`                                                                    |
| `sase bead list --status closed -f json`            | `"statuses": ["closed"]`, `"implied_status_closed": false`, `count <= 20`                    |
| `sase bead list -f bogus`                           | argparse usage error, exit 2                                                                 |

Handler ordering stays exactly as today, with rendering swapped in at the end: resolve filters → default query → (if
empty and no explicit `--status`) closed query + set implicit flag → record `total` → resolve limit → slice → render per
`args.format`. Emptiness and fallback decisions are format-independent; only the final rendering differs.

## Implementation outline

### 1. Parser — `src/sase/main/parser_bead.py`

- Add to `bead_list_parser`, placed alphabetically **before** `-n/--limit` (CLI rules: options sorted, every public long
  option gets a short alias):

  ```python
  bead_list_parser.add_argument(
      "-f",
      "--format",
      choices=["compact", "json", "full"],
      default="compact",
      help="Output format: compact, json, or full (default: compact)",
  )
  ```

  Match `search`'s `choices` list order and help wording verbatim.

- Upgrade the `list` subparser's help per the CLI rules ("make `-h|--help` output excellent"), mirroring how `search` is
  declared: keep `help="List issues"`, and add a `description` (default view = open/claimed/in-progress; implicit
  `--status closed` fallback; closed listings capped at 20 unless `--limit 0`; bare `sase bead` defaults here) plus a
  `RawDescriptionHelpFormatter` `epilog` with examples:

  ```
  sase bead list
  sase bead list --status open --type phase
  sase bead list --format json
  sase bead list --format full --limit 3
  sase bead list --status closed --limit 0
  ```

- Sanity note (no code needed): `_default_list_subcommands()` copies `list`'s defaults onto the `bead` group parser, so
  bare `sase bead` gets `format="compact"`, and other bead subcommands gain a harmless unused `format` default exactly
  as they already do for `limit`/`status`/`tier`/`type`.

### 2. Detail-renderer extraction — `src/sase/bead/cli_query.py`

- Add `_render_issue_detail(view: BeadProject, issue: Issue, *, relativize_design: bool) -> str` containing the current
  body of `handle_bead_show` from the header line through the plan/parent-plan block, appending to a list of lines and
  returning `"\n".join(lines) + "\n"` — output byte-identical to today's prints.
- Add a `_design_paths_are_relative() -> bool` helper wrapping the existing
  `resolve_sdd_store(Path.cwd(), 1).is_in_tree` check; `_display_design_path` becomes
  `_display_design_path(design: str, *, relativize: bool)`.
- Rewrite `handle_bead_show` to open one view, resolve the issue (unchanged `KeyError` → `Error: issue not found: <id>`
  on stderr, `exit(1)`), and print the rendered string.
- Rewrite `_render_search_full` to take the open `view` and call `_render_issue_detail` per match, dropping
  `contextlib.redirect_stdout` (and the now-unused `contextlib` / `io` imports). This requires moving the rendering
  inside `handle_bead_search`'s `with get_read_view() as view:` block; do that for all three formats so the structure
  stays uniform.

### 3. Handler — `src/sase/bead/cli_query.py` (`handle_bead_list`)

- Keep the existing filter/fallback/limit logic; capture `total = len(issues)` immediately before slicing.
- Replace the trailing print loop with a `match args.format:` dispatch mirroring `handle_bead_search`, including the
  `case _: raise AssertionError(f"unknown list format: {args.format}")` guard.
- Move the empty-result and notice handling into the format dispatch so `json` emits an envelope instead of prose:
  - `compact`/`full`: `No issues found.` on empty; notice line before rows when `implied_status_closed`.
  - `json`: always the envelope.
- New module-private renderers next to the `_render_search_*` family:
  - `_render_list_compact(issues) -> str` — today's `{icon} {id} · {title}{ ← parent}` lines.
  - `_render_list_full(view, issues, *, relativize_design) -> str` — `_render_issue_detail` per bead joined by
    `f"\n{'-' * 60}\n"`.
  - `_render_list_json(issues, *, total, statuses, implied_status_closed) -> str` — the envelope from decision 6,
    `json.dumps(..., indent=2) + "\n"`.
- Because `full` needs the open view, keep all rendering inside `handle_bead_list`'s `with get_read_view() as view:`
  block (it already is).

### 4. Onboard help — `src/sase/bead/cli_admin.py`

Add one aligned line to the `handle_bead_onboard` quick-start block, next to the other `sase bead list` entries:

```
  sase bead list --format=json                   Machine-readable listing
```

### 5. Skill source — `src/sase/xprompts/skills/sase_beads.md`

- In the `### list` section, add format examples and a short prose paragraph describing the three formats and the JSON
  envelope keys (`count`, `total`, `statuses`, `implied_status_closed`, `results`), mirroring the `### search` section's
  treatment.
- **`tests/test_bead/test_cli_list.py::test_list_skill_examples_parse_against_cli_contract` asserts the exact ordered
  list of `sase bead list` example lines in that section** — update that expected list in the same commit or the test
  fails. Note the assertion collects every line starting with `sase bead list`, so new examples must be appended in the
  order they appear in the doc.
- Regenerate and deploy per the generated-skills workflow: `sase skill init --force`, then `chezmoi apply`. Do not
  hand-edit deployed `SKILL.md` files; open the chezmoi repo through `/sase_repo` if any path there must be inspected.

### 6. Tests — `tests/test_bead/`

**`test_cli_list.py` (parser):**

- `--format` / `-f` parse into `args.format`; default is `"compact"`.
- `sase bead list -f bogus` → `SystemExit(2)`.
- Updated skill-example contract list (see step 5).

**`test_cli_list.py` (handler, using the existing `project_dir` fixture from `tests/test_bead/conftest.py` in the style
of `test_cli_search.py`):**

- `-f json` on a store with beads: parse stdout with `json.loads`; assert `count`, `total`, `statuses`,
  `implied_status_closed is False`, and `results[0]["id"]`.
- `-f json` on an empty store: valid envelope, `count == 0`, `results == []`, no prose, exit 0.
- `-f json` with only closed beads: `implied_status_closed is True`, `statuses == ["closed"]`, and **no** notice text in
  stdout (`json.loads` on the whole stream must succeed — this is the regression guard for decision 5).
- `-f json -n 1` over ≥2 matches: `count == 1`, `total == 2`.
- `-f full` on a single-bead store equals `sase bead show <id>` output (mirrors
  `test_handle_bead_search_full_reuses_show_rendering`).
- `-f compact` output equals plain `sase bead list` output.

**`test_cli_golden.py` + `tests/test_bead/golden/cli/` (text contract):**

- New cases over the existing `current` / `closed_only` / `closed_many` / `initialized` fixtures with new `.stdout`
  goldens: `list_full`, `list_json`, `list_json_limit` (count < total), `list_implicit_closed_json`, `list_empty_json`,
  and `list_implicit_closed_full` (notice + detail blocks).
- All existing `list*.stdout` goldens stay byte-identical — do not regenerate them.
- Golden JSON must be stable: `_run_cli` already rewrites the project root to `<ROOT>`, and design paths are relativized
  for in-tree stores, so no absolute paths should leak; verify the `created_at`/`updated_at` values come from the
  checked-in fixture stores (they do — `golden/stores/*/issues.jsonl`) and are therefore deterministic. If any field
  turns out to be non-deterministic, drop that case from the goldens and cover it in `test_cli_list.py` instead rather
  than weakening the golden harness.

**Fast-path regression (`tests/main/test_bead_fast_path.py`):**

- Assert `try_handle_bead_fast_path(["list", "--format", "json"])` still returns `None`, alongside the existing `list`
  deferral assertion — `list` must not start routing to Rust.

## Validation

- `just install` first (ephemeral workspace deps may be stale), then `just check`.
- `sase skill init --force` && `chezmoi apply` after editing the skill source.
- Manual smoke in a workspace with beads:
  - `sase bead list` (unchanged), `sase bead -h` / `sase bead list -h` (help reads well, options alphabetical)
  - `sase bead list -f compact`, `sase bead list -f full -n 2`
  - `sase bead list -f json | jq '.count, .total, .implied_status_closed'`
  - `sase bead list --status closed -f json | jq '.count, .total'` (shows the implicit 20 cap)
  - a closed-only store: `sase bead list -f json | jq -e .implied_status_closed`
  - `sase bead search auth -f full` still matches `sase bead show <id>` (refactor guard)

## Out of scope

- Adding `--format` to `sase_core`'s `handle_list` / re-enabling the Rust fast path for `list` (documented alternative
  in decision 1).
- `-c/--color` for `list`, and colorizing Python's bead renderers generally.
- `--format` for `ready`, `blocked`, or `stats`.
- Changing the compact row format, the closed-fallback rule, the 20-row default, or the per-issue JSON wire schema.

## Open questions for the reviewer

1. Approve the Python-only scope (decision 1), leaving Rust's unreachable `handle_list` untouched?
2. Approve the envelope shape in decision 6 — specifically flat `results: [<issue>]` (like `prompt search`) rather than
   `results: [{"issue": ...}]` (like `bead search`, whose nesting exists only to carry `matched_fields`)?
3. Approve suppressing the implicit-closed notice under `json` in favor of `implied_status_closed` (decision 5), rather
   than routing the notice to stderr?
4. Approve the `_render_issue_detail` extraction (decision 4) as part of this change — it is what makes `-f full` safe
   on unbounded listings, but it does touch `show` and `search -f full`, so it is the riskiest part of the diff.
