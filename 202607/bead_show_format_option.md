---
tier: tale
title: Add -f/--format compact|full|json to sase bead show
goal: sase bead show accepts -f/--format with full (unchanged default), compact (the
  one-line list row for that bead), and json (a single-bead envelope carrying the
  resolved parent, children, dependency, blocker, and plan graph), with text and JSON
  rendered from one shared detail model so they cannot drift.
create_time: 2026-07-27 10:42:29
status: done
---

- **PROMPT:** [202607/prompts/bead_show_format_option.md](prompts/bead_show_format_option.md)
- **AGENTS:**
  - [bbugyi200.athena.m9](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.m9/README.md)
  - [bbugyi200.athena.m9--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.m9.md#member-code)
- **COMMITS:**
  - [d51475b](https://github.com/sase-org/sase/commit/d51475b249f77dbd8760df96cc268e24a9f7d2e1) — feat(bead): add show output formats

# Plan: `sase bead show` — `-f | --format <compact|full|json>`

## Goal

Give `sase bead show` the same output-format switch that `sase bead list` and `sase bead search` already have:

```
-f, --format {compact,full,json}   Output format (default: full)
```

- **`full`** (default) — byte-for-byte identical to today's detail block, so nothing existing changes.
- **`compact`** — the single `list` row for that bead (`{icon} {id} · {title}{ ← parent}`), so `show` composes into
  scripts, prompts, and status lines without post-processing.
- **`json`** — a stable single-bead envelope that carries everything the `full` block displays, including the resolved
  ancestor chain, children, dependencies, blockers, and plan link.

## Guiding principle (the thing that makes this intuitive)

**`--format` selects presentation only. It never changes what a command means, and the three format names mean the same
thing on every read command.** Each command keeps as its default the format it already produced; passing `--format` lets
you borrow another command's presentation.

| Command                     | `compact`             | `full`                      | `json`                        | Default   |
| --------------------------- | --------------------- | --------------------------- | ----------------------------- | --------- |
| `sase bead list` (today)    | rows                  | `show` blocks, 60-dash rule | collection envelope           | `compact` |
| `sase bead search` (today)  | rows + match snippets | `show` blocks, 60-dash rule | collection envelope + matches | `compact` |
| `sase bead show` (**this**) | **that bead's row**   | **today's detail block**    | **single-bead envelope**      | **full**  |

Two invariants fall out of that principle, and both are cheap to test:

1. `sase bead show <id> -f compact` is byte-identical to `<id>`'s line in `sase bead list -f compact`.
2. `sase bead show <id>` and `sase bead show <id> -f full` are byte-identical to today's output.

## Background / current behavior

- `handle_bead_show` lives in `src/sase/bead/cli_query.py`. It opens `get_read_view()`, resolves `args.id` (`KeyError` →
  `Error: issue not found: <id>` on stderr, `sys.exit(1)`), and prints
  `_render_issue_detail(view, issue, relativize_design=_design_paths_are_relative())`.
- `_render_issue_detail` (`cli_query.py:109-252`) is already the single text renderer shared by `show`,
  `list --format full`, and `search --format full` — it was extracted for exactly that reason in commit `672ecbb4c`
  ("feat(bead): add list output formats"). It emits, in order: the header/`Type:`/`Assignee:`/`Claimed by:`/`Model:`/
  `Size:` preamble, then `PARENT`, `CHILDREN` (split into `PHASES` and `CHILD EPICS`), `DEPENDS ON`, `BLOCKS`,
  `DESCRIPTION`, `NOTES`, `CHANGESPEC`, and one of `PLAN` / `EPIC PLAN` / `PARENT PLAN`.
- Relationship resolution is spread through that renderer: `_parent_lineage` walks ancestors (returning a trailing
  unresolved id for a dangling **or cyclic** parent), `view.get_epic_children` fetches children, each
  `issue.dependencies` entry is re-`show`n, and `BLOCKS` scans `view.list_issues()` over the whole store.
- The `show` subparser (`src/sase/main/parser_bead.py:237-239`) is two lines:
  `add_parser("show", help="Show issue details")` plus a positional `id`. It has no `description` and no `epilog`,
  unlike `list` and `search`.
- JSON today comes from the module-private `_issue_to_wire_dict` / `_dependency_to_wire_dict` helpers in `cli_query.py`,
  used by both `list --format json` and `search --format json`.
- `sase bead show` is deliberately **off** the Rust fast path: `try_handle_bead_fast_path`
  (`src/sase/main/bead_fast_path.py:26`) returns `None` for `argv[0] in {"list", "show"}`.
- Six test call sites construct the handler's namespace by hand as `argparse.Namespace(id=...)` and would therefore have
  no `format` attribute: `tests/test_bead/test_cli_show.py:89` (the `_show` helper),
  `tests/test_bead/test_cli_changespec.py:134,158,421,433`, and `tests/test_bead/test_claimed_status.py:97`.
  `tests/test_bead/test_cli_list.py:199` and `tests/test_bead/test_cli_search.py:167` already go through
  `create_parser().parse_args(...)`.

## Design decisions (please confirm at review)

1. **`full` is the default, and defaulting differs per command on purpose.** `show`'s existing behavior is the detail
   block, so `full` must stay its default exactly as `compact` stays `list`'s and `search`'s. The format _vocabulary_ is
   uniform; only the per-command default differs, and each default is "what this command already did". Every existing
   `show` golden (`show.stdout`, `show_phase_parent_epic_plan.stdout`, `show_missing.stderr`) and every `list -f full` /
   `search -f full` golden must keep passing untouched — that is the regression contract for this change.

2. **`compact` reuses the `list` row renderer.** `_render_list_compact([issue])` already produces
   `{icon} {id} · {title}{ ← parent}` with a trailing newline. `show -f compact` calls it with a one-element list rather
   than growing a second implementation, which is what makes invariant 2 above true by construction.
   - _Alternative considered:_ a bespoke one-liner for `show` (e.g. including status text). Rejected: it would make
     `show -f compact` and `list -f compact` disagree about the same bead, which is precisely the confusion this option
     should not create.

3. **`json` is lossless with respect to `full`.** Every fact the `full` block displays is present in the JSON as
   structured data. This is the decision that makes the option worth having: a bare `_issue_to_wire_dict(issue)` would
   make `sase bead show <id> -f json` exactly equal to `sase bead list -f json | jq '.results[] | select(.id=="<id>")'`,
   i.e. a redundant feature. `show`'s value over `list` is the _resolved graph_ — ancestors, split children,
   dependencies, blockers, and the inherited-plan resolution — so the JSON carries that graph.
   - "Lossless" means every fact, in stored (non-cwd-dependent) form; see decision 6 for paths and decision 7 for phase
     sizes.
   - _Alternative considered:_ emit the flat wire dict only. Rejected as above. It is also the strictly weaker option:
     the envelope below still contains that exact wire dict under `"issue"`, so nothing is lost by choosing the richer
     shape.

4. **One resolved detail model feeds both renderers (this is the reliability decision).** Rather than writing a second
   graph walk for JSON, resolve once into a structured value and render it twice:

   ```
   resolve_issue_detail(view, issue) -> IssueDetail     # ancestors, children, deps, blockers, plan link
   render_detail_text(detail, *, relativize_design) -> str
   render_detail_json(detail) -> str
   ```

   Text and JSON then cannot drift as bead fields evolve — a new section added to the text block is a field the JSON
   builder sees for free, and the "lossless" claim in decision 3 becomes structural rather than a promise a test has to
   police forever. It also avoids a second whole-store `list_issues()` scan for `BLOCKS`.
   - This does touch the renderer shared by `show`, `list -f full`, and `search -f full`, so it is the riskiest part of
     the diff. The guards are the untouched goldens (decision 1) plus
     `test_handle_bead_search_full_reuses_show_rendering` and `test_handle_bead_list_full_reuses_show_rendering`.
   - _Alternative considered:_ leave `_render_issue_detail` alone and write an independent JSON walker. Rejected: two
     graph walks over the same data, guaranteed to drift, and it doubles the `BLOCKS` full-store scan.

5. **Envelope shape** — fixed key order for diff stability, single-entity (no `count`/`total`, which belong to the
   collection envelopes):

   ```json
   {
     "issue": { "id": "beads-1.1", "title": "Build Alpha", "...": "..." },
     "ancestors": [
       {
         "id": "beads-1",
         "resolved": true,
         "title": "Alpha Epic",
         "status": "open",
         "issue_type": "plan",
         "tier": "epic",
         "size": null
       }
     ],
     "children": { "phases": [], "epics": [] },
     "depends_on": [],
     "blocks": [
       {
         "id": "beads-1.2",
         "resolved": true,
         "title": "Review Alpha",
         "status": "open",
         "issue_type": "phase",
         "tier": null,
         "size": null
       }
     ],
     "plan": {
       "section": "EPIC PLAN",
       "source": "parent",
       "path": "plans/alpha.md",
       "from": {
         "id": "beads-1",
         "resolved": true,
         "title": "Alpha Epic",
         "status": "open",
         "issue_type": "plan",
         "tier": "epic",
         "size": null
       }
     }
   }
   ```

   - `"issue"` is `_issue_to_wire_dict(issue)` **verbatim** — the exact per-issue schema `list --format json` and
     `search --format json` already emit, reused unchanged so all three commands stay in sync.
   - `"ancestors"` is nearest-first, matching the left-to-right order of the text `PARENT` lineage line. It is `[]` for
     a root bead. The text's `phase`/`epic`/`plan` lineage labels (`_lineage_kind`) are deliberately **not** duplicated
     as a field: they are a pure function of `issue_type` + `tier`, both present on every ref.
   - `"children"` mirrors the text's `CHILDREN` section and its two subsections; `{"phases": [], "epics": []}` when
     there are none. (`epics`, not `child_epics` — the nesting already says "child".)
   - `"plan"` is `null` when the text block prints no plan section. `"source"` is `"self"` when the bead's own `design`
     is used (text section `PLAN` or `EPIC PLAN`) and `"parent"` when it is inherited from the nearest plan ancestor
     (text section `EPIC PLAN` or `PARENT PLAN`); `"from"` is the ancestor ref for `"parent"` and `null` for `"self"`.
     Keeping `"section"` verbatim is what makes the block reconstructible without re-deriving the text's branching.

6. **Bead refs have fixed keys, and `resolved: false` is the one rule for dangling ids.** Every relationship entry
   (`ancestors`, `children.*`, `depends_on`, `blocks`, `plan.from`) is the same shape:

   ```json
   { "id": "beads-9", "resolved": false, "title": null, "status": null, "issue_type": null, "tier": null, "size": null }
   ```

   The text renders all of these cases as `<id> (not found)`, so one flag covers them all and consumers learn one rule.
   Fixed keys (nulls rather than omissions) keep `jq` paths total.
   - Note this is a deliberate, documented divergence from `_issue_to_wire_dict`, which _omits_ `size` when unset. The
     wire dict is a frozen shared schema this change must not touch; refs are new, so they get the better rule.
   - **Known quirk, mirrored faithfully:** `_parent_lineage` returns its trailing id for a _cyclic_ parent as well as a
     missing one, and the text prints `(not found)` for both. The JSON reports `resolved: false` identically. Fixing the
     cycle wording is out of scope; the JSON must not silently disagree with the text it mirrors.

7. **JSON reports stored values; only text relativizes paths.** `_display_design_path` shortens plan paths with
   `os.path.relpath` when the SDD store is in-tree — that is cwd-dependent _display_ state, so it applies to `full`
   only. `plan.path` carries the stored `design` value, matching the `design` field that `list --format json` already
   emits verbatim. Likewise `size` is the raw stored value (`null` when unset); the text's "phases with no stored size
   display `small`" rule is deterministic and documented, so nothing is lost.

8. **Errors stay format-independent.** An unknown id prints `Error: issue not found: <id>` to stderr and exits 1 under
   every format, leaving stdout empty. So `sase bead show <id> -f json` writes stdout that is either valid JSON or
   nothing at all — never a mixture, and never an error object a consumer has to discriminate.
   - _Alternative considered:_ emit `{"error": "..."}` on stdout with exit 1 under `json`. Rejected: it forces every
     consumer to branch on shape, and it contradicts the existing precedent (`search` with an empty query exits 2 with a
     stderr message regardless of format).

9. **`show` stays off the Rust fast path; Python-only change.** `try_handle_bead_fast_path` returns `None` for `show`,
   so `crates/sase_core/src/bead/cli.rs::handle_show` is unreachable from the CLI — and it has _already_ diverged
   substantially (no ancestor lineage chain, no `PHASES`/`CHILD EPICS` split, no `Size:` line, no `EPIC PLAN` /
   `PARENT PLAN` sections). Adding `--format` there would deepen a divergence nothing executes. This also matches the
   Rust-core boundary memo: the rendering here is presentation glue over an already-Rust-backed read.
   - _Alternative considered:_ port `handle_show` to parity and re-enable the `show` fast path. Rejected as a separate,
     larger change; noted in Out of scope.

10. **No `-c/--color` for `show`.** `search`'s `-c/--color` only takes effect on the Rust fast path, so a `--color` flag
    on Python-rendered `show` would be an inert option — worse than no option. Colorizing the shared Python detail
    renderer would also change `list -f full` and `search -f full` and rewrite every text golden, which is its own
    change. Flagged in Open questions since the CLI rules do prefer color where it helps.

## Behavior matrix

| Invocation                                           | Result                                                                       |
| ---------------------------------------------------- | ---------------------------------------------------------------------------- |
| `sase bead show <id>`                                | unchanged detail block (default `--format full`)                             |
| `sase bead show <id> -f full`                        | identical to `sase bead show <id>`                                           |
| `sase bead show <id> -f compact`                     | one line, identical to that bead's row in `sase bead list -f compact`        |
| `sase bead show <id> -f json`                        | envelope only, exit 0                                                        |
| `sase bead show <missing> -f json`                   | empty stdout; `Error: issue not found: <missing>` on stderr; exit 1          |
| `sase bead show <root-bead> -f json`                 | `"ancestors": []`, `"plan"` from the bead's own `design`, `"source": "self"` |
| `sase bead show <phase> -f json`                     | nearest-first `"ancestors"`, `"plan.source": "parent"`, `"plan.from"` set    |
| `sase bead show <bead-with-dangling-parent> -f json` | trailing `"ancestors"` entry with `"resolved": false`                        |
| `sase bead show <id> -f bogus`                       | argparse usage error, exit 2                                                 |
| `sase bead show` (no id)                             | unchanged argparse usage error, exit 2                                       |

## Implementation outline

### 1. Parser — `src/sase/main/parser_bead.py`

Replace the two-line `show` subparser (currently at the `# sase bead show` comment) with a declaration matching how
`list` and `search` are declared. Options stay alphabetical, and `-f` is the standard short alias:

```python
bead_show_parser = bead_subparsers.add_parser(
    "show",
    help="Show issue details",
    description=(
        "Show one bead's full detail block: status, type, tier, owner, "
        "assignee, model, phase size, parent lineage, children, "
        "dependencies, blockers, description, notes, ChangeSpec, and the "
        "linked plan. --format compact prints the same single row as "
        "'sase bead list'; --format json adds the resolved parent, child, "
        "dependency, blocker, and plan graph as machine-readable data."
    ),
    epilog=(
        "Examples:\n"
        "  sase bead show sase-64\n"
        "  sase bead show sase-64 --format compact\n"
        "  sase bead show sase-64 --format json"
    ),
    formatter_class=argparse.RawDescriptionHelpFormatter,
)
bead_show_parser.add_argument(
    "-f",
    "--format",
    choices=["compact", "json", "full"],
    default="full",
    help="Output format: compact, json, or full (default: full)",
)
bead_show_parser.add_argument("id", help="Issue ID")
```

Keep the `choices` list in the same order and the same help wording as `list`/`search`, changing only the parenthesized
default.

Note the parser interaction worth pinning with a test: `_default_list_subcommands()` in `src/sase/main/parser.py` copies
the `list` subparser's defaults (including `format="compact"`) onto the `bead` group parser. `argparse`'s
`_SubParsersAction` parses the chosen subcommand into a fresh namespace and then copies every key onto the parent
namespace, so `show`'s own `format="full"` default wins. No code change is needed — but assert it (step 6) rather than
trusting it, because a regression there would silently change `show`'s default output.

### 2. New module — `src/sase/bead/cli_detail.py`

`cli_query.py` is 561 lines and already mixes handlers, list renderers, search renderers, the detail renderer, and the
wire helpers; adding the JSON detail builder in place would push it toward the `toobig` info threshold (`just lint` runs
`toobig src 1000 850 700`). More importantly, "resolve a bead's graph and render it" is now a distinct, three-consumer
concern. Move it out:

- **Moved from `cli_query.py`, made public** (symvision forbids importing `_`-prefixed symbols across files; each of
  these gains a real non-test consumer in `cli_query.py`, so no pragmas are needed):
  - `render_issue_detail(detail, *, relativize_design) -> str` (from `_render_issue_detail`)
  - `design_paths_are_relative() -> bool` (from `_design_paths_are_relative`)
  - `issue_to_wire_dict(issue)` and `dependency_to_wire_dict(dep)` (from `_issue_to_wire_dict` /
    `_dependency_to_wire_dict`) — bodies unchanged, since `list`/`search` JSON output must not move.
- **New public:**
  - `resolve_issue_detail(view, issue) -> IssueDetail` — performs the walk once: ancestor chain (+ trailing unresolved
    id), `get_epic_children` split into phases and child epics, dependency targets, the single `view.list_issues()`
    blockers scan, and the plan-link resolution (`section`, `source`, `path`, `from`).
  - `render_issue_detail_json(detail) -> str` — `json.dumps(envelope, indent=2) + "\n"`.
- **New private, in-file only:** `IssueDetail` (a frozen dataclass), a `PlanLink` value, `_issue_ref(issue)` /
  `_unresolved_ref(issue_id)`, and the relocated `_parent_lineage`, `_lineage_kind`, `_phase_size_value`,
  `_display_design_path`.

`render_issue_detail` takes the resolved `IssueDetail` instead of `(view, issue)`, so it performs no lookups of its own
and its output stays byte-identical. Callers that render many beads (`list -f full`, `search -f full`) call
`resolve_issue_detail` per bead from their single open view, exactly as they call `_render_issue_detail` today.

### 3. Handlers — `src/sase/bead/cli_query.py`

- Import the public names from `cli_detail`; delete the moved definitions. Keep `_render_list_compact`,
  `_render_list_full`, `_render_list_json`, and the `_render_search_*` family here, updated to call
  `resolve_issue_detail` + `render_issue_detail`.
- Rewrite `handle_bead_show` to mirror `handle_bead_list` / `handle_bead_search`:

  ```python
  def handle_bead_show(args: argparse.Namespace) -> None:
      with get_read_view() as view:
          try:
              issue = view.show(args.id)
          except KeyError:
              print(f"Error: issue not found: {args.id}", file=sys.stderr)
              sys.exit(1)

          match args.format:
              case "compact":
                  print(_render_list_compact([issue]), end="")
              case "json":
                  print(render_issue_detail_json(resolve_issue_detail(view, issue)), end="")
              case "full":
                  print(
                      render_issue_detail(
                          resolve_issue_detail(view, issue),
                          relativize_design=design_paths_are_relative(),
                      ),
                      end="",
                  )
              case _:
                  raise AssertionError(f"unknown show format: {args.format}")
  ```

  The lookup and its error path stay ahead of the dispatch, so decision 8 holds by construction. `_render_list_compact`
  stays private in `cli_query.py` — `handle_bead_show` is an in-file caller, so no cross-file private import.

### 4. Onboard help — `src/sase/bead/cli_admin.py`

Add one aligned line to the `handle_bead_onboard` quick-start block, directly under the existing `sase bead show <id>`
entry and matching the `sase bead list --format=json` line added for `list`:

```
  sase bead show <id> --format=json              Machine-readable bead detail
```

### 5. Skill source — `src/sase/xprompts/skills/sase_beads.md`

- In the `### show` section, add the format examples and a short prose paragraph describing the three formats and the
  JSON envelope keys (`issue`, `ancestors`, `children`, `depends_on`, `blocks`, `plan`), the `resolved` flag on refs,
  and that `full` is the default — mirroring the treatment the `### list` and `### search` sections already have.
- Regenerate and deploy per the generated-skills workflow: `sase skill init --force`, then `chezmoi apply`. Do not
  hand-edit deployed `SKILL.md` files; open the chezmoi repo through `/sase_repo` if any path there must be inspected.

### 6. Tests — `tests/test_bead/`

**Namespace migration (do this first — it is what makes the rest compile).** Six call sites build the handler namespace
by hand and would raise `AttributeError` on `args.format`. Convert each to
`create_parser().parse_args(["bead", "show", <id>])`, which also makes them exercise the real parser:
`tests/test_bead/test_cli_show.py:89` (the `_show` helper), `tests/test_bead/test_cli_changespec.py:134,158,421,433`,
and `tests/test_bead/test_claimed_status.py:97`. Do **not** paper over this with `getattr(args, "format", "full")` in
the handler — that hides exactly this kind of drift, and the other two bead handlers read `args.format` directly.

**`tests/test_bead/test_cli_show.py` (parser):**

- `--format` and `-f` parse into `args.format`; `sase bead show <id>` defaults to `"full"`.
- `sase bead show <id> -f bogus` → `SystemExit(2)`.
- Group-default guard: `parse_args(["bead", "show", "x"]).format == "full"` while
  `parse_args(["bead"]).format == "compact"` — pins the `_default_list_subcommands()` interaction described in step 1.

**`tests/test_bead/test_cli_show.py` (handler, using the existing `nested_store` / `project_dir` fixtures):**

- `-f compact` on a phase equals that bead's line from `sase bead list -f compact` (invariant 2).
- `-f full` equals plain `sase bead show <id>` (invariant 1).
- `-f json` on the `nested_store` root epic: `issue.id`, `ancestors == []`, `children.phases` ids, `children.epics` ids,
  and `plan.source == "self"`.
- `-f json` on a nested phase: nearest-first `ancestors` ids matching the text lineage order, `plan.source == "parent"`,
  `plan.section == "EPIC PLAN"`, `plan.from.id` set.
- `-f json` on a bead with a dependency and a blocker: `depends_on` / `blocks` ids with `resolved is True`.
- `-f json` with a dangling parent and a dangling dependency: the corresponding refs carry `"resolved": false` and null
  fields, and the text block for the same bead prints `(not found)` — assert both in one test so the mirror is pinned.
- **Losslessness guard:** for each `nested_store` bead, assert every bead id appearing in the `-f full` text output also
  appears somewhere in the parsed `-f json` payload. This is the cheap, durable expression of decision 3.
- `-f json` on a missing id: `SystemExit(1)`, empty stdout, `Error: issue not found:` on stderr (decision 8).
- Skill-example contract test for the `### show` section of `src/sase/xprompts/skills/sase_beads.md`, mirroring
  `test_list_skill_examples_parse_against_cli_contract` in `test_cli_list.py`: collect the ordered `sase bead show ...`
  lines, assert the exact expected list, and parse each through `create_parser()`.

**`test_cli_golden.py` + `tests/test_bead/golden/cli/` (text contract):**

- New cases over the existing `current` fixture with new `.stdout` goldens: `show_compact`
  (`bead show beads-1 --format compact`), `show_json` (`bead show beads-1 --format json` — epic with children,
  ChangeSpec, and its own plan), and `show_phase_json` (`bead show beads-1.1 --format json` — lineage, blockers, and an
  inherited parent plan).
- All existing `show*.stdout` / `show_missing.stderr` / `list_full*.stdout` goldens stay byte-identical — do not
  regenerate them. That is the decision-1 and decision-4 regression contract.
- Golden JSON must be deterministic: the `current` fixture store (`golden/stores/current/`) supplies fixed `created_at`
  / `updated_at` values and a relative `design` (`plans/alpha.md`), and `_run_cli` rewrites the project root to
  `<ROOT>`, so no absolute paths leak. If any field turns out to be non-deterministic, drop that case from the goldens
  and cover it in `test_cli_show.py` instead rather than weakening the golden harness.

**Fast-path regression (`tests/main/test_bead_fast_path.py`):**

- Extend `test_fast_path_defers_show_to_argparse` with
  `assert try_handle_bead_fast_path(["show", "beads-1", "--format", "json"]) is None`, alongside the existing `show`
  deferral assertion — `show` must not start routing to Rust (decision 9).

### 7. Docs — `docs/beads.md` and `docs/configuration.md`

These are hand-maintained (no generator, no contract test), and the `list --format` change did not touch them.

- `docs/beads.md` → `### sase bead show <id>`: add a flag table for `-f, --format` and describe the three formats and
  the JSON envelope keys, mirroring the `### sase bead search <query>` section's treatment.
- `docs/configuration.md` → `#### sase bead show`: add the `-f, --format` row (`compact`, `json`, `full`; default
  `full`) to the existing table.
- **Backfill the adjacent `list` gap in the same change:** `docs/beads.md`'s `### sase bead list` table and
  `docs/configuration.md`'s `#### sase bead list` table are both missing the `-f, --format` row that shipped in
  `672ecbb4c` (and the `configuration.md` one is also missing `-n, --limit`). These tables sit immediately beside the
  ones being edited, so leaving them stale would make the new `show` rows look inconsistent. This is the one piece of
  scope beyond `show` itself; drop it at review if you would rather it be its own commit.

## Validation

- `just install` first (ephemeral workspace deps may be stale), then `just check`.
- `sase skill init --force` && `chezmoi apply` after editing the skill source.
- Manual smoke in a workspace with beads:
  - `sase bead show <id>` — unchanged; `sase bead show -h` reads well and options are alphabetical
  - `sase bead show <id> -f full` — identical to the line above
  - `sase bead show <id> -f compact` — matches that bead's row in `sase bead list -f compact`
  - `sase bead show <epic-id> -f json | jq '.issue.id, .children.phases[].id, .plan'`
  - `sase bead show <phase-id> -f json | jq '.ancestors[].id, .plan.source, .plan.from.id'`
  - `sase bead show nope -f json; echo $?` — exit 1, stderr only, empty stdout
  - `sase bead list -f full` and `sase bead search <q> -f full` still match `sase bead show <id>` (the decision-4
    refactor guard)

## Out of scope

- Porting `--format` into `sase_core`'s `handle_show` / re-enabling the Rust fast path for `show`, and closing the
  existing Python↔Rust `show` divergence (documented alternative in decision 9).
- `-c/--color` for `show`, and colorizing the shared Python detail renderers generally (decision 10).
- Changing the `full` text block, the per-issue JSON wire schema shared with `list`/`search`, or either collection
  envelope.
- Accepting multiple ids in one `sase bead show` invocation.
- Fixing the `(not found)` wording that `_parent_lineage` also produces for a _cyclic_ parent (decision 6).
- `--format` for `ready`, `blocked`, or `stats`.

## Open questions for the reviewer

1. Approve `full` as `show`'s default (decision 1) and the "uniform format vocabulary, per-command default" principle?
2. Approve the relationship-rich JSON envelope (decisions 3 and 5) over a bare `_issue_to_wire_dict`, which would make
   `show -f json` a duplicate of a `list -f json` slice?
3. Approve the `cli_detail.py` extraction and the shared resolved model (decision 4)? It is what guarantees text/JSON
   parity, but it is the part of the diff that touches `list -f full` and `search -f full`.
4. Approve backfilling the stale `list --format` docs rows alongside the new `show` rows (step 7), or split that out?
5. Color: the CLI rules prefer colored output where it helps, but this plan keeps `show` colorless (decision 10) because
   `search`'s `--color` is inert off the Rust fast path and colorizing the shared renderer would rewrite every text
   golden. Confirm color stays a separate change?
