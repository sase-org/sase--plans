---
tier: epic
title: Rich-text clan summaries via %clan
goal: 'Agent clans can be initialized with a Rich-markup summary — supplied inline
  (with a new double-colon directive shorthand) or produced by an executable script
  — that is persisted with the clan and rendered beautifully in the clan''s agent
  metadata panel; epic clans get a built-in summary script and the research_swarm
  xprompt shows its research prompt.

  '
phases:
- id: core
  title: sase-core clan_summary wire and resolver
  depends_on: []
  description: '''Phase: sase-core clan_summary wire and resolver'' section: add the
    clan_summary field to the Rust agent-meta scan wire and SQLite index refresh,
    and add a resolve_clan_summary binding mirroring resolve_clan_tribe.'
- id: directive
  title: '%clan summary arguments and :: shorthand'
  depends_on: []
  description: '''Phase: %clan summary arguments and :: shorthand'' section: extend
    %clan directive parsing with summary= / summary_script= named arguments and a
    generic double-colon text-block shorthand for directives, plus validation and
    TUI completion hints.'
- id: persist
  title: Launch-time resolution and persistence
  depends_on:
  - core
  - directive
  description: '''Phase: Launch-time resolution and persistence'' section: resolve
    the summary in the agent runner (literal or script execution with clan env vars),
    write clan_summary into agent_meta.json, and thread the field through the Python
    scan-wire mirror.'
- id: tui
  title: Clan panel rendering
  depends_on:
  - persist
  description: '''Phase: Clan panel rendering'' section: resolve the clan-level summary
    in the Agents-tab tree projection and render it as a Rich-markup block in the
    clan detail document, with PNG visual snapshots.'
- id: builtins
  title: Built-in epic summary script
  depends_on:
  - tui
  description: '''Phase: Built-in epic summary script'' section: ship the sase_clan_summary_epic
    console script that prints a beautiful epic overview, and emit summary_script=
    from the epic clan declaration in sase bead work.'
- id: swarm
  title: research_swarm summary in chezmoi
  depends_on:
  - tui
  description: '''Phase: research_swarm summary in chezmoi'' section: update the research_swarm
    xprompt in the chezmoi linked repo to declare its clan with a bold RESEARCH PROMPT
    summary.'
- id: smoke
  title: End-to-end summary smoke exercises
  depends_on:
  - builtins
  description: '''Phase: End-to-end summary smoke exercises'' section: end-to-end
    tests that launch summary-declaring clans (literal and script forms) and assert
    persistence and panel rendering.'
  model: haiku
create_time: 2026-07-19 19:10:17
status: wip
bead_id: sase-7r
---

# Plan: Rich-text clan summaries via %clan

## Context

Agent clans (`%clan` / `%c`) currently carry only a name, generation, and optional tribe. The clan detail document in
the agent metadata panel (`AgentPromptPanel`, rendered by `build_clan_detail_text` in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`) shows structured fields (Name / Tribes / Status /
Runtime / Members) plus aggregated member sections, but has no way to say _what the clan is about_.

This epic adds a **clan summary**: a block of Rich markup attached to the clan at launch and rendered in the clan's
metadata panel. Three authoring forms are supported:

1. **Inline literal** — `%clan(<name>, summary=<value>)`, where `<value>` is a quoted string or a `[[ ... ]]` multiline
   text block (already supported by the directive arg grammar in `src/sase/xprompt/_parsing_args.py`).
2. **Double-colon shorthand** — `%clan:<name>:: <text...>` or `%clan(<args>):: <text...>`, a new directive-side analog
   of the existing `#xprompt:: text` shorthand, rewritten into `summary=[[...]]` during directive parsing.
3. **Executable script** — `%clan(<name>, summary_script=<path-or-name>)`; the runner executes it at launch, and its
   stdout becomes the summary markup.

Two consumers land with the feature: epic clans created by `sase bead work` declare a built-in `summary_script`, and the
`research_swarm` xprompt (chezmoi repo) declares a literal summary showing its research prompt.

### Key existing mechanics (verified)

- Directive parsing is pure Python in the sase repo: regex + aliases in `src/sase/xprompt/_directive_types.py`
  (`_DIRECTIVE_PATTERN`, `%c` → `clan`), match collection in `src/sase/xprompt/_directive_collect.py`
  (`_collect_clan_paren_args` currently allows only `tribe=`), value resolution in
  `src/sase/xprompt/_directive_values.py` (`resolve_clan_tribe`), assembly in `src/sase/xprompt/_directive_extract.py`
  into `PromptDirectives`.
- The xprompt `::` shorthand lives in `src/sase/xprompt/_parsing_shorthand.py` (`DOUBLE_COLON_SHORTHAND_PATTERN`,
  `find_double_colon_text_end`, `_format_as_text_block`); `%` directives have **no** `::` form today.
- The declaring member's runner writes clan fields into `agent_meta.json` at `src/sase/axe/run_agent_directives.py` (the
  block that writes `AGENT_CLAN_FIELD`, `AGENT_CLAN_GENERATION_FIELD`, and `clan_tribe`).
- `agent_meta.json` is parsed by the **Rust** scanner in the sase-core repo
  (`crates/sase_core/src/agent_scan/scanner.rs::agent_meta_from_object`, explicit field-by-field) into
  `crates/sase_core/src/agent_scan/wire.rs::AgentMetaWire`, serialized generically back to Python (no per-field pyo3
  code). The SQLite artifact index stores the full record as a `record_json` blob; `clan_tribe` has no dedicated column.
  Schema version is `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION = 13` in `crates/sase_core/src/agent_scan/index.rs`; the v12
  no-op "record_json refresh" migration is the exact precedent for adding an agent-meta field (it added `clan_tribe`).
- Clan-level attribute resolution ("latest explicit declaration wins") is the Rust binding `resolve_clan_tribe`
  (`crates/sase_core/src/agent_clan_tribe.rs`, pyo3 registration in `crates/sase_core_py/src/lib.rs`), consumed via the
  Python facade `src/sase/core/agent_clan_tribe.py` from `_container_for_clan` in
  `src/sase/ace/tui/models/_agent_tree.py`.
- Epic clans are declared programmatically by `_clan_identity_directives` in `src/sase/bead/work.py`
  (`%clan(<epic_id>, tribe=epic)` on the first phase segment).
- Built-in executable scripts ship in `src/sase/scripts/` with `[project.scripts]` entry points in `pyproject.toml`
  (`sase_chop_*` pattern); chop discovery (`src/sase/axe/chop_script_runner.py::discover_chop_script`) is the precedent
  for resolving a configured script name to an executable (path → interpreter bin → PATH).
- The sase repo pins `sase-core-rs>=0.9.0,<0.10.0` and mirrors `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` in
  `src/sase/core/agent_scan_wire.py` (re-export).

## Design decisions

**Syntax.**

- `%clan(<name>, summary=<value>)` — literal Rich markup; quoted string or `[[ ... ]]` block. Composes with `tribe=`.
  Recommend `[[ <markup> ]]` with spaces inside the delimiters so markup that begins with `[` never forms a `[[[` run.
- `%clan(<name>, summary_script=<value>)` — executable reference. `summary=` and `summary_script=` are **mutually
  exclusive** (targeted parse error if both given).
- `::` shorthand: `%clan:<name>:: <text>` and `%clan(<args>):: <text>` (the `:: ` must include the trailing space,
  matching the xprompt convention). The captured text maps to `summary=`. Implement generically: a small allowlist
  mapping directive → target named arg (initially `{"clan": "summary"}`), so future directives can opt in;
  non-allowlisted directives followed by `:: ` keep today's behavior (no rewrite, no new errors).
- The `%c` alias participates in all forms.
- `%id(..., clan=...)` joiners cannot set a summary — the existing `%id` keyword allowlist already rejects unknown keys,
  and the existing `%clan`-vs-`%id` contract validation stays unchanged. Only a declaring `%clan` can attach a summary,
  mirroring `tribe=`.

**Double-colon capture semantics.** Text runs from after `:: ` to EOF or to the next line that _starts_ with a `%`
directive or `#` xprompt reference (a directive-aware variant of `_NEXT_DIRECTIVE_PATTERN`; blank lines are included,
exactly like the xprompt `::` form). Because capture is line-oriented, anything after `:: ` on the same line is part of
the summary — document that `::` should end its line's directive cluster, and use `summary=[[...]]` instead when the
declaration shares a line with more prompt text. The rewrite happens inside directive extraction (before match
collection) so every caller of `extract_prompt_directives` gets identical behavior and the captured block is stripped
from the prompt the model sees. The rewrite must respect the same literal zones as directive parsing (fenced code,
`%xprompts_enabled:false ... :true`).

**Execution model (reliability).** The summary is resolved **once, at launch, by the declaring member's runner**
(`run_agent_directives.py`) — the single site that already writes `clan_tribe`. Scripts never run in the TUI process and
never run at render time (TUI perf rule: render paths do no disk/subprocess work). Script contract:

- Resolution: values containing `/` resolve as paths (with `~` expansion, relative to the agent's working directory);
  bare names resolve via the interpreter's `bin/` then `PATH` (factor or mirror `discover_chop_script`; a shared helper
  is preferred if it stays clean).
- Invocation: no arguments; env includes `SASE_CLAN_NAME`, `SASE_CLAN_GENERATION`, and `SASE_CLAN_TRIBE` (when declared)
  on top of the inherited environment; stdout is the summary; stderr goes to the agent log.
- Guardrails: ~10s timeout, output capped (32 KiB, also applied to literals at meta-write), trailing whitespace
  stripped. Any failure (not found, non-zero exit, timeout, empty output) logs a clear warning and simply omits the
  summary — a broken summary script must never fail or delay an agent launch.
- Launch validation (`src/sase/agent/launch_validation.py` path) checks syntax only (mutual exclusion, non-empty
  values); it does not resolve or execute scripts.

**Persistence and resolution.** The runner writes `agent_meta["clan_summary"]` next to `clan_tribe`. The field flows
through the Rust scanner → scan wire → Python `AgentMetaWire` → `Agent` rows. Clan-level resolution reuses the tribe
semantics ("latest explicit declaration wins, omissions never clear") via a new `resolve_clan_summary` Rust binding —
per the core-boundary rule, this domain behavior belongs in sase-core so any frontend resolves identically. No
`agent_name_registry.json` changes: durability comes from `agent_meta.json`, which registry rebuilds and index rebuilds
already re-scan.

**Rendering (beauty + safety).** The clan detail document renders the summary as its own block directly beneath the
structured field block (after `Members:`, before the fold header/roster), at **every** fold level — it is identity-level
information, like `Name:`. No section heading is added: summaries self-label (the research swarm supplies its own bold
`RESEARCH PROMPT:` prefix; the epic script prints its own header). Markup is parsed with
`rich.text.Text.from_markup(...)` inside a `try/except MarkupError` that falls back to appending the raw text unparsed —
user markup can never crash or corrupt the panel.

**Out of scope / notes.**

- Member rows' metadata panels are unchanged; the summary appears only on the clan container row.
- Do **not** edit `sase/memory/*.md`, `AGENTS.md`, or provider shims in any phase (memory updates require separate
  explicit user approval). After the epic lands, the user may want to ask for an xprompts memory update documenting the
  new syntax.
- No new keybindings, footer entries, or `sase ace` options are introduced, so the help modal and footer stay untouched.
- A prompt whose interpolated text contains `]]` breaks out of a `[[ ... ]]` block; this is a pre-existing property of
  the text-block grammar. Ensure the resulting parse error message is clear; do not try to escape inside blocks.

## Phase: sase-core clan_summary wire and resolver

Work in the **sase-core linked repo** (open it with the `/sase_repo` skill; never edit it through any other path).
Mirror the `clan_tribe` precedent end to end:

1. `crates/sase_core/src/agent_scan/wire.rs`: add `#[serde(default)] pub clan_summary: Option<String>` to
   `AgentMetaWire` beside `clan_tribe`.
2. `crates/sase_core/src/agent_scan/scanner.rs::agent_meta_from_object`: parse
   `clan_summary: coerce_str(data.get("clan_summary"))`.
3. `crates/sase_core/src/agent_scan/index.rs`: bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` to 14 and add the no-op
   `migrate_record_json_refresh_v14` (doc-comment why: refresh `record_json` with `agent_meta.clan_summary`), following
   the v12/v13 pattern and its `prior_version < 14` gate. No dedicated column — the field rides in `record_json` like
   `clan_tribe`.
4. `crates/sase_core/src/agent_clan_tribe.rs`: add `clan_summary` support. Extend `ClanTribeMemberWire` with
   `#[serde(default)] clan_summary: Option<String>` and add `resolve_clan_summary(request) -> ClanSummaryResolutionWire`
   with identical filter/latest-wins/tie-break logic (factor a private helper shared with `resolve_clan_tribe` rather
   than duplicating the selection loop). Keep `resolve_clan_tribe`'s public shape unchanged.
5. `crates/sase_core_py/src/lib.rs`: add and register `py_resolve_clan_summary`
   (`#[pyo3(name = "resolve_clan_summary")]`) next to `py_resolve_clan_tribe`; update the module docstring listing. The
   scan payload needs no binding changes (generic serde).
6. Check `crates/sase_xprompt_lsp` for `%clan` keyword/argument tables and add the new `summary` / `summary_script`
   keywords if such tables exist.
7. Tests: inline unit tests beside `resolve_clan_summary` (mirror the four tribe cases); extend
   `crates/sase_core/tests/agent_scan_parity.rs` fixtures/assertions and `crates/sase_core/tests/python_wire_parity.rs`
   round-trips with `clan_summary`; keep the schema-version parity assertions passing.
8. Update the sase_core_py CHANGELOG entry per repo convention so the release that ships this is identifiable; note the
   released version for the persist phase (the sase repo pins `sase-core-rs>=0.9.0,<0.10.0` and will raise its floor to
   the first version carrying this change).

## Phase: %clan summary arguments and :: shorthand

All in the sase repo's xprompt/directive layer, mirroring how `tribe=` and `%wait(...)` keywords are wired:

1. `src/sase/xprompt/_directive_collect.py`: extend `_collect_clan_paren_args` to accept `summary=` and
   `summary_script=` (update the unknown-keyword allowlist and error text); stash `clan_summary_arg` /
   `clan_summary_script_arg` (+ presence flags) on `_CollectedDirectives`; reject the combination of both with a
   targeted message. Colon form `%clan:<name>` stays summary-less.
2. `src/sase/xprompt/_directive_values.py`: add `resolve_clan_summary(...)` / `resolve_clan_summary_script(...)`
   validators (non-empty after strip; `[[...]]` dedenting already handled by `_process_text_block`).
3. `src/sase/xprompt/_directive_extract.py` + `PromptDirectives` (`src/sase/xprompt/_directive_types.py`): thread new
   `clan_summary: str | None` and `clan_summary_script: str | None` fields (populated only when `%clan` declares).
4. Directive `::` shorthand: add a preprocessing pass in the directive-parsing path (so all `extract_prompt_directives`
   callers share it) that finds allowlisted directives — `{"clan": "summary"}` — followed by `:: `, captures text to EOF
   or the next line-start `%`/`#` reference using a directive-aware sibling of `find_double_colon_text_end`, and
   rewrites to the paren form with `summary=[[...]]` (reuse `_format_as_text_block`). Handle both `%clan:<name>:: ` and
   `%clan(<args>):: ` (including `%c`), respect literal zones, and error clearly if the rewrite would duplicate an
   explicit `summary=` / `summary_script=`.
5. Completion/hints: `src/sase/ace/tui/widgets/directive_completion.py` (`_DIRECTIVE_ARGUMENT_HINTS["clan"]`,
   `_CLAN_KEYWORD_ARGUMENTS`, description) and `src/sase/ace/tui/widgets/_directive_completion_tokens.py`
   (`_extract_clan_paren_arg_token` currently rejects fragments containing `=`) so typing `%clan(x, su...` completes
   both keywords.
6. Tests (pure parsing, no runner): literal quoted + `[[...]]` forms; `::` after colon and paren forms; `%c` alias;
   capture termination at line-start `%` and `#` boundaries and at EOF; blank-line inclusion; stripped prompt excludes
   the captured block; mutual-exclusion and duplicate-argument errors; `%id(..., summary=...)` still rejected;
   fenced-code and xprompts-disabled zones untouched.

## Phase: Launch-time resolution and persistence

1. Script execution helper (new module near the runner, e.g. `src/sase/axe/clan_summary_script.py`): discovery (path vs
   bare name per the design contract, sharing/mirroring `discover_chop_script`), subprocess execution with the
   `SASE_CLAN_*` env vars, 10s timeout, 32 KiB stdout cap, stderr → agent log, and a warning-not-error failure mode.
2. `src/sase/axe/run_agent_directives.py`: in the clan-membership block that writes `clan_tribe`, resolve the summary —
   literal from `directives.clan_summary`, else execute `directives.clan_summary_script` — and write
   `agent_meta["clan_summary"]` (declaring member only, cap applied). A missing/failed summary writes nothing.
3. Python wire mirror: add `clan_summary` to the `AgentMetaWire` dataclass in
   `src/sase/core/agent_scan_wire_markers.py`, map it in the scan-wire from/to-dict helpers, and bump the Python-side
   `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` mirror (re-exported through `src/sase/core/agent_scan_wire.py`) to 14 in
   lockstep with core.
4. Extend the Python facade `src/sase/core/agent_clan_tribe.py` with `resolve_clan_summary` (+ `clan_summary` on
   `ClanTribeMemberWire`), calling the new binding via `require_rust_binding`.
5. Raise the `sase-core-rs` floor in `pyproject.toml` to the release produced by the core phase.
6. Tests: runner-level tests with a fixture script (success, non-zero exit, timeout, empty output, oversized output,
   unresolvable name) asserting `agent_meta.json` contents and that launches never fail; wire round-trip tests for the
   new field; facade test against the real binding.

## Phase: Clan panel rendering

1. Model: add `clan_summary` to `AgentState` (`src/sase/ace/tui/models/_agent_state.py`), populate it in the agent
   loader from `AgentMetaWire`, and reset appropriately in `_reset_tree_projection` if needed.
2. Tree projection: in `_container_for_clan` (`src/sase/ace/tui/models/_agent_tree.py`), when any row carries an
   explicit `clan_summary`, call the new `resolve_clan_summary` facade (same member-wire construction as the tribe call)
   and set the resolved value on the container `Agent`.
3. Rendering: in `build_clan_detail_text` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`), append the
   summary block after the `Members:` line and before the fold header — blank line, then `Text.from_markup(summary)`
   guarded by `try/except MarkupError` falling back to the raw string, then blank line. Visible at all fold levels. No
   disk reads or subprocess calls: the value arrives on the `Agent` row via the scan wire (TUI perf rules hold).
4. Tests: unit tests for container summary resolution (declared vs undeclared, latest-wins via mixed members, invalid
   markup fallback); PNG visual snapshots — extend the clan fixtures in
   `tests/ace/tui/visual/test_ace_png_snapshots_agents.py` and add cases to
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py` covering a multiline styled summary
   (research-prompt-shaped) and fold-level cycling; regenerate affected goldens intentionally with
   `--sase-update-visual-snapshots` and inspect them for beauty (spacing, wrapping, color) before accepting.

## Phase: Built-in epic summary script

1. New `src/sase/scripts/sase_clan_summary_epic.py` with a `main()` entry point registered as `sase_clan_summary_epic`
   in `[project.scripts]` (alphabetical order per CLI rules). Behavior: read `SASE_CLAN_NAME` (the epic bead id),
   resolve the epic bead and its PHASE beads through the existing bead APIs from the current working directory (the
   runner executes it inside the project workspace), and print a launch-time-stable Rich-markup overview — a bold,
   colored `EPIC <id> · <title>` header, the epic's one-line goal (dimmed), and a numbered phase-title list. Keep it
   launch-stable (no live statuses — the panel's Status/Members fields already cover liveness), width-friendly (~76
   cols), and consistent with the TUI's existing color idiom. On any lookup failure, print a minimal
   `[bold]EPIC <id>[/]` fallback and exit 0 so epics always get some summary.
2. `src/sase/bead/work.py::_clan_identity_directives`: extend the declaring segment to
   `%clan(<epic_id>, tribe=epic, summary_script=sase_clan_summary_epic)`.
3. Tests: script unit tests against a fixture bead store (happy path + fallback); an emission test asserting the
   rendered multi-prompt contains the `summary_script=` argument; a PNG snapshot of the clan panel fed a fixture summary
   in the script's output style, so the "renders beautifully" bar is pinned by a golden.

## Phase: research_swarm summary in chezmoi

Work in the **chezmoi linked repo** (open with `/sase_repo`; path `home/sase/xprompts/research_swarm.md`). Change the
first segment's declaration to:

```text
%clan(research.@, tribe=research, summary=[[ [bold]RESEARCH PROMPT:[/bold] {{ prompt }} ]]) %id:research.@.cdx %model:@research_a {{ prompt }} #research
```

Notes: the spaces inside `[[ ... ]]` are deliberate (avoid a `[[[` run before `[bold]`); the inline `summary=[[...]]`
form is required here because the declaration shares its line with the rest of the segment, so the `::` shorthand would
swallow it. Only the declaring segment changes; `%id(..., clan=research.@)` joiners are untouched. After committing,
follow the chezmoi repo's own instructions for applying changes. Verify with a real (or fakey-provider)
`#research_swarm` launch that the clan panel shows the bold-prefixed prompt, including a prompt containing commas and
`[` characters (markup-fallback path).

## Phase: End-to-end summary smoke exercises

Add end-to-end coverage in the style of the existing axe smoke suites (see the recent outage/recovery smoke tests under
`tests/`): using the fakey provider, launch (a) a small clan whose declaring segment uses `summary=[[...]]`, (b) one
using `summary_script=` pointing at a fixture script, and (c) an epic via the `sase bead work` path. Assert the
declaring member's `agent_meta.json` carries the expected `clan_summary`, that scan/index round-trips surface it, that
the clan container in the Agents tab renders the block, and that a deliberately failing summary script still yields a
successful launch with a logged warning and no summary.

## Testing & verification

- Each sase-repo phase runs `just install` then `just check` before finishing; visual phases run `just test-visual` and
  inspect `.pytest_cache/sase-visual/` artifacts before accepting golden updates.
- sase-core phase runs that repo's own build/test workflow (cargo tests including the parity suites).
- The chezmoi phase makes no sase-repo file changes, so `just check` does not apply there.

## Risks

- **Wheel sequencing**: the persist phase needs a released `sase-core-rs` wheel containing the core phase. The pin bump
  is the synchronization point; if the wheel is not yet published when persist starts, that phase must block on it
  rather than feature-detecting.
- **Index schema bump**: v14 triggers a background full index rebuild on first open (established, deliberate behavior of
  the no-op-migration pattern); no action needed beyond the version gate.
- **Markup safety**: user markup is untrusted; the `from_markup` fallback must be exercised by tests so bad markup
  degrades to plain text instead of breaking the panel.
- **`::` capture surprises**: same-line content after `:: ` is captured by design; the parse-time behavior is documented
  and the research_swarm phase deliberately uses the inline form. Completion hints should steer users accordingly.
