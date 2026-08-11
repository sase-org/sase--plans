---
tier: epic
title: Stitch origin indicators on the Artifacts Stitches sub-tab
goal: 'Every row on the Artifacts Stitches sub-tab carries a distinct, self-documenting
  indicator for how the commit was created — through `sase stitch create`, automatically
  by another `sase` command, or by hand — backed by a provenance invariant that makes
  the classification reliable rather than heuristic, and exposed through a matching
  `origin:` filter, the shared `sase stitch log` renderers, and the core wire.

  '
phases:
- id: core
  title: Core stitch-origin classifier
  depends_on: []
  size: medium
  description: 'core: add the `CommitOriginWire` enum, the footer-driven `classify_commit_origin`
    function, and the `origin` field on the VCS-log commit wire in the linked `sase-core`
    repo, bump the VCS-log wire schema version, and expose both through the PyO3 binding
    with Rust unit and parity tests.

    '
- id: invariant
  title: Tracked-commit provenance invariant
  depends_on: []
  size: medium
  description: 'invariant: stamp `SASE_TYPE=stitch` on every commit created through
    the tracked `sase stitch create` workflow, audit every remaining commit-creating
    call site so each stamps a `SASE_TYPE=` value, and add a contract test that keeps
    new call sites from regressing the invariant.

    '
- id: wire
  title: Python wire and golden-contract parity
  depends_on:
  - core
  size: small
  description: 'wire: mirror the new `origin` field in the Python VCS-log wire records,
    teach the pure-Python golden-contract parser to compute the same value, and update
    the shared commit fixtures and core wire tests.

    '
- id: render
  title: Timeline, legend, and detail rendering
  depends_on:
  - wire
  size: medium
  description: 'render: add the shared origin glyph/label/style module, the fixed
    origin column on timeline rows, the adaptive legend key, the Origin line in the
    Stitches detail panel, and the origin field in every `sase stitch log` output
    format, with unit and PNG visual coverage.

    '
- id: filter
  title: origin filter and CLI flag
  depends_on:
  - wire
  size: medium
  description: 'filter: add the repeatable, negatable `origin:` key to the Stitches
    filter query language with completions and canonical rendering, plus the matching
    `sase stitch log --origin` flag and post-collection filtering.

    '
- id: docs
  title: Docs, help modal, and configuration reference
  depends_on:
  - invariant
  - render
  - filter
  size: small
  description: 'docs: document the origin taxonomy, glyphs, provenance invariant,
    and `origin:` filter across the ACE guide, commit-workflow guide, CLI reference,
    configuration reference, and the ACE help modal.'
proposed_by: bbugyi200.athena.xv
create_time: 2026-08-11 06:57:35
status: wip
bead_id: sase-jo
---

- **PROMPT:** [prompts/202608/stitch_origin_badges.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/stitch_origin_badges.md)
- **BEAD:** [sase-jo](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jo/README.md)

# Plan: Stitch origin indicators on the Artifacts Stitches sub-tab

## Problem

The Artifacts → Stitches sub-tab renders one row per commit across the primary repo,
linked repos, and (on request) sidecars. Nothing on a row says **how** the commit came
to exist. Three mechanisms produce commits in a SASE project, and they mean very
different things to a reader:

1. An agent (or a human) ran `sase stitch create` — the tracked workflow. Only these
   commits get a numeric stitch record in a Patch's `STITCHES:` section, so only these
   are stitches in the glossary sense.
2. Another `sase` command committed automatically — `sase init`, memory/skill deploys,
   bead-store sync, SDD/plan bookkeeping, config writes, xprompt saves.
3. A human ran `git commit` themselves.

Today a reader has to open a commit and read its message footer to tell these apart, and
even that is guesswork because the footer vocabulary was never designed to answer this
question.

## Design

### Taxonomy

One closed, three-valued taxonomy named **stitch origin**, with these canonical string
values used identically on the wire, in the query language, and in JSON output:

| Value    | Meaning                                                      |
| -------- | ------------------------------------------------------------ |
| `stitch` | Created through `sase stitch create` (the tracked workflow). |
| `auto`   | Created automatically by some other `sase` command.          |
| `manual` | Created by hand; no SASE provenance in the commit message.   |

`stitch` is deliberately the name of the first bucket: those commits are the only ones
that become numbered stitches inside a Patch, so the badge teaches the glossary instead
of inventing a parallel vocabulary. There is no fourth `unknown` value — the taxonomy is
exhaustive by construction (see the invariant below), and a fourth state would only
invite rows that explain nothing.

Note that `origin:` already exists in the **Patch** query language with the values
`sase`, `external`, and `unknown` (PR origin — "who opened this PR?"). Reusing the word
for stitches is intentional and consistent: in both places `origin:` answers "what
created this thing?". The two live in separate filter grammars over separate entities,
so the value sets do not collide.

### The provenance invariant

Classification reads the commit message footer, which is self-describing, travels with
the commit into every repo and clone, needs no cross-file joins, and is already parsed
by the shared Rust footer grammar (`parse_commit_footer`). The design rests on one
invariant, established by the `invariant` phase:

> **Every commit SASE creates carries a `SASE_TYPE=` footer tag.** Tracked
> `sase stitch create` commits carry `SASE_TYPE=stitch`; every automatic commit carries
> `SASE_TYPE=<kind>` naming the automation. A commit with no `SASE_TYPE=` was not
> created by SASE.

That single key makes the classifier a lookup rather than a heuristic:

1. `TYPE == "stitch"` → `stitch`.
2. `TYPE` present with any other value → `auto`.
3. No `TYPE`, but `AGENT`, `BEAD`, or `PLAN` present → `stitch`. This rule exists purely
   for history predating the invariant: every agent-authored commit already carries a
   linked `SASE_AGENT=` tag, so existing timelines classify correctly the day this
   ships.
4. Otherwise → `manual`.

Keys are compared in canonical (unprefixed) form, so the legacy `TYPE=` spelling and the
current `SASE_TYPE=` spelling both resolve. `TYPE` already has no behavioral consumers —
only display styling in `src/sase/vcs_log/_tag_style.py` — so widening its meaning from
"which automation produced this" to "which SASE mechanism produced this" is safe.

Accepted consequences, all of which should be stated in the docs phase rather than
engineered around:

- A squash-merge of a SASE PR carries the original footer, so it classifies as `stitch`
  even though a human pressed the merge button. That is the honest answer: the content
  originated from a tracked stitch.
- Commits in a linked repo authored upstream by other people classify as `manual`, which
  is correct.
- A human who runs `sase stitch create` gets `stitch`. The badge names the mechanism,
  not the actor; `SASE_AGENT=` in the detail panel still names the actor when there was
  one.

### Visual language

Origin is rendered as a **fixed one-glyph column immediately to the left of the
subject**, after the conditional merge column. A column, not an inline chip: subjects
stay aligned, the eye can scan straight down, and a row never shifts because its
neighbor gained a badge. Unlike the merge column, the origin column is always present,
because every commit has an origin.

| Origin   | Glyph | Style          | Rationale                                                        |
| -------- | ----- | -------------- | ---------------------------------------------------------------- |
| `stitch` | `✦`   | `#FFD700`      | The established SASE agent gold, already the `AGENT=` tag color. |
| `auto`   | `↻`   | `dim #8A8A8A`  | Machinery, deliberately receding — it is the least notable case. |
| `manual` | `✎`   | `bold #FFAF5F` | A hand touched this; rare, so it is the one that speaks up.      |

All three glyphs are already used elsewhere in this codebase (`✦` in pickers and the
Beads pane, `↻` in retry badges and the stitch-log progress line, `✎` in the
`sase stitch list` dirty marker and prompt-bar chrome), so they are known to render in
the pinned Fira Code used by the visual snapshot suite and are known to be single-cell.
`✎` for `manual` deliberately echoes `✎ dirty` in `sase stitch list` — both mean "a
person edited this".

The glyphs avoid every mark already on a timeline row: presence (`● ↑ ↓ ·`) and merge
(`◆`, magenta). The three shapes are distinct at a glance even in monochrome, so the
indicator survives `--color never` and colorblind readers; color is reinforcement, never
the sole channel.

The legend is what makes the column self-teaching. The shared legend already lists the
presence keys and conditionally lists the merge key; it gains an origin cluster on the
same principle — **only origins actually present in the displayed commits appear**, the
same adaptive rule as `visible_repos_only` and the merge key. A timeline of nothing but
agent work shows `✦ stitch` and nothing else, so the legend never grows into noise.

### Where it appears

- Timeline rows in the Stitches sub-tab and in `sase stitch log --format pretty` — the
  same `commit_line` builder serves both.
- The legend shared by the TUI info line and both console CLI formats.
- The Stitches detail panel, as a full `Origin` line under `Author`. For `auto` it names
  the automation from the `TYPE` value (`↻ auto · sase init`); for `stitch` it says the
  commit came from `sase stitch create`; for `manual` it says there is no SASE
  provenance.
- `sase stitch log --format full` (header marker) and `--format oneline` (compact
  token).
- `sase stitch log --format json`, as a plain `"origin"` string field.

### Query surface

`origin:` joins the Stitches filter grammar as a repeatable, comma-listable, negatable
key alongside `repo:` and `author:` — `origin:stitch`, `origin:auto,manual`,
`-origin:auto` — with `sase stitch log --origin` as the CLI equivalent. Origin cannot be
pushed down to `git log`, so it is applied after collection, exactly like the exact
`since`/`until` refinement already applied there; the phase must preserve that ordering
so a provider-side limit cannot silently swallow matches.

### Where the code lives

The classifier is domain logic, not presentation: the CLI, the TUI, and the
`sase_gateway` crate must all agree on it, so it belongs in the Rust core per the
backend boundary rule. It is computed once during `parse_git_log` and carried on the
commit wire beside `presence`, which is the existing precedent for a classified field.
Presentation — glyphs, colors, labels, column placement — stays in this repo.

Open the `sase-core` checkout with the `/sase_repo` skill (`sase repo open sase-core`)
and use the path it prints. Do **not** hand-edit the `sase-core-rs` requirement window
in `pyproject.toml`: per `docs/rust_backend.md`, that window is owned by the release
automation, and a source checkout builds the extension regardless of the declared
window. The only ordering requirement is that the `core` phase lands in `sase-core`
before the `wire` phase's CI runs.

## Core stitch-origin classifier

In the linked `sase-core` repo:

- Add `crates/sase_core/src/vcs_log/origin.rs` with a serde `CommitOriginWire` enum
  (`Stitch`, `Auto`, `Manual`, serialized lowercase as `stitch`/`auto`/`manual`) and
  `classify_commit_origin(subject: &str, body: &str) -> CommitOriginWire` implementing
  the four precedence rules above over `commit_footer::parse_commit_footer`. Reuse the
  footer module's canonical key handling so both the prefixed and legacy spellings
  match; do not re-implement prefix stripping.
- Register the module in `crates/sase_core/src/vcs_log/mod.rs`.
- Add `origin: CommitOriginWire` to `VcsCommitWire` in
  `crates/sase_core/src/vcs_log/wire.rs`, populate it in `parse_git_log` (`parsers.rs`)
  where subject and body are already in hand, and bump `VCS_LOG_WIRE_SCHEMA_VERSION`
  from 3 to 4. Confirm the flattened `AggregatedCommitWire` JSON carries the new key.
- Expose `classify_commit_origin(message: &str) -> &str` as a standalone PyO3 binding in
  `crates/sase_core_py/src/lib.rs` (registered alongside the existing commit-footer and
  vcs-log bindings, and listed in the module docstring binding inventory) so hosts can
  classify a single message that did not come from a log stream.
- Rust tests: one case per precedence rule, including `TYPE=stitch` beating a co-present
  `AGENT=`, an `AGENT=`-only legacy commit, a `BEAD=`-only commit, a `PLAN=`-only
  commit, a footer-shaped line in the middle of a body (must not be treated as a
  footer), an empty message, and a message whose only tag is an unrelated key.

Follow the `sase-core` repo's own contribution and release conventions; a `feat`-typed
commit there releases as a patch bump inside the current `0.24.x` window.

## Tracked-commit provenance invariant

In this repo, establish the invariant the classifier depends on:

- Stamp `SASE_TYPE=stitch` on tracked commits. `CommitWorkflow` in
  `src/sase/workflows/commit/workflow.py` already calls `apply_runtime_commit_tags` on
  the `create_commit` branch; that branch must also set the type tag. Prefer extending
  `src/sase/workflows/commit/runtime_tags.py` with an explicit helper (for example
  `apply_tracked_commit_tags`) that composes `TYPE=stitch` with the runtime provenance
  tags in one footer update, so the tag block is rendered once and `TYPE` stays owned.
  Do **not** touch the `create_pull_request` branch: those tags describe a PR body, not
  a commit.
- Verify the ordering against the plan and bead hooks that write `PLAN=` and `BEAD=`
  (`src/sase/workflows/commit/plan_hooks.py`, `bead_hooks.py`) — the resulting footer
  must keep `TYPE` first per the existing `_KEY_PRIORITY` display ordering, and no hook
  may drop `TYPE` when it rewrites the block.
- Audit every call site that creates a commit and make each one stamp a `SASE_TYPE=`.
  Sites already compliant: `bare_git_init.py`, `config_commit.py`,
  `init_workspace_handler.py`, `init_memory/commit_message.py`,
  `init_memory/project_deploy.py`, `_init_chezmoi_deploy.py`, `bead/_sync_git.py`,
  `bead_pages/refresh.py`, `bead_pages/publication.py`, `sdd/_commit_store.py`,
  `sdd/_commit_bare_git.py`, `axe/run_agent_exec_plan_sdd.py`,
  `llm_provider/commit_finalizer_git.py`, `xprompt/write_targets.py`. Sites to fix:
  `src/sase/ace/revert_agent_transaction.py` (the single squashed revert commit — give
  it its own type, e.g. `revert`) and
  `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt_git.py` (verify
  whether the message it builds already routes through a tag helper; if not, stamp
  `xprompt`). Confirm that the amend paths in
  `src/sase/vcs_provider/plugins/_git_sync_ops.py`, `_git_core_ops.py`, and
  `_git_commit_dispatch.py` preserve an existing footer rather than replacing it — an
  amend that drops `TYPE` would silently reclassify a commit as `manual`.
- Add a contract test (`pytestmark = pytest.mark.contract`) that enumerates modules
  invoking `git commit` (or a provider `create_commit`) and asserts each one's message
  passes through a `runtime_tags` helper, with an explicit, commented allowlist for any
  intentional exception. This is what keeps the invariant true as new commands are
  added; without it the badge degrades quietly.
- Unit tests: a tracked commit message ends up with `SASE_TYPE=stitch`; a tracked commit
  with a bead and an agent keeps all three tags with `TYPE` first; re-running the tag
  application is idempotent; the revert and xprompt-save messages carry their new types.

Where a new `TYPE` value is introduced, add a color for it in `_TYPE_COLORS` in
`src/sase/vcs_log/_tag_style.py`, including `stitch` itself.

## Python wire and golden-contract parity

- Add `CommitOrigin = Literal["stitch", "auto", "manual"]` and an `origin` field to
  `VcsCommitWire` in `src/sase/core/vcs_log_wire.py`, defaulting to `"manual"` (the
  no-SASE-footer state), with an `_origin_from_dict` helper that mirrors
  `_presence_from_dict` and falls back to `"manual"` for an unrecognized value. Bump
  `VCS_LOG_WIRE_SCHEMA_VERSION` to 4 and update the module docstring's JSON shape list.
- Teach `_parse_git_log_python` in `src/sase/core/vcs_log_facade.py` to compute the same
  value so the golden-contract reference stays in parity with Rust; it may call the
  existing `parse_commit_footer` binding rather than re-implementing the grammar. Add a
  thin `classify_commit_origin` facade wrapper over the new binding for hosts that need
  to classify a standalone message.
- Update the shared fixtures and helpers that build commit wires so the new field is
  exercised rather than always defaulted: `src/sase/ace/testing/fixtures.py`,
  `tests/_vcs_log_render_helpers.py`, `tests/_vcs_log_collect_helpers.py`, and
  `tests/ace/tui/_commits_pane_helpers.py`.
- Extend `tests/test_core_vcs_log.py` with round-trip and schema-version coverage, and
  check `tests/test_core_health.py` and any test asserting the VCS-log schema version.

## Timeline, legend, and detail rendering

- Add `src/sase/vcs_log/_origin_style.py` as the single source of origin presentation:
  glyph, Rich style, short label, and the sentence used by detail views, plus a helper
  that builds the legend cluster for a set of observed origins. Every renderer reads
  from here; no glyph or hex color is duplicated at a call site.
- `src/sase/vcs_log/_render_console.py`:
  - `commit_line` gains a fixed origin column between the merge column and the subject.
    It is unconditional. Keep the row single-line and ellipsized as it is today.
  - `legend` gains the origin cluster, listing only origins present in the passed
    commits, placed after the presence keys and the conditional merge key.
  - `_full_commit_lines` shows the origin glyph and label in the commit header.
- `src/sase/vcs_log/_render_plain.py`: add `"origin"` to `_commit_json`, and a compact
  origin token to `render_oneline`.
- `src/sase/ace/tui/widgets/artifacts/commits_rendering.py`: add an `Origin` line to
  `build_commit_detail` directly under `Author`, rendering the glyph, the value, and the
  explanatory sentence — including the `TYPE` value for `auto` (`↻ auto · sase init`),
  read from the already-parsed `commit_tag_view`.
- Check `src/sase/ace/tui/modals/commit_view_modal.py` and the agent prompt-panel commit
  header (`src/sase/ace/tui/widgets/prompt_panel/`) for commit headers that should carry
  the same marker, and keep any that already show commit metadata consistent with the
  detail panel.
- Tests: extend `tests/test_vcs_log_render_pretty.py`, `test_vcs_log_render_full.py`,
  and `test_vcs_log_render_compact.py` for the column, the adaptive legend (present
  origins only), and the JSON/oneline fields; add a focused test for `_origin_style.py`;
  add a test asserting every origin glyph is one cell wide, so a future glyph swap
  cannot break column alignment.
- Visual snapshots: add a Stitches PNG case whose timeline contains all three origins
  (and refresh the existing `artifacts_stitches_*` and `copy_as_stitches_*` goldens that
  shift). Verify the 80x24 case (`artifacts_stitches_persistent_filter_80x24.png`) still
  reads well; if the legend cluster crowds that width, shorten the legend labels rather
  than dropping the column. Accept intentional changes with
  `--sase-update-visual-snapshots` and inspect the diff artifacts before doing so.

## origin filter and CLI flag

- `src/sase/vcs_log/filter_query.py`: add `origin` to `_FILTER_KEYS`,
  `_REPEATABLE_KEYS`, and `_NEGATABLE_KEYS`; parse and validate values against the three
  canonical strings with a token-anchored error message in the style of
  `_parse_merges_value`; add `origins` and `excluded_origins` to
  `CommitLogFilterValues`; render them in `to_query_tokens` in stable order (after
  `author:`, before `sidecar:`); enforce them in `compile_commit_matcher`; and add the
  `origin` completion kind.
- `src/sase/vcs_log/models.py`: add origin selection to
  `CommitFilters`/`CommitFilterSpec` and apply it in `ordered_commits` in
  `src/sase/vcs_log/_render_util.py` alongside the exact date refinement, before the
  user-visible cap. Include it in `filter_summary` and `empty_message` so an empty
  result explains itself.
- `src/sase/ace/tui/widgets/artifacts/commit_filter_bar.py`: add the key description,
  the value candidates (`stitch`, `auto`, `manual`), and the value-hint text.
- `src/sase/main/parser_stitch.py`: add a repeatable `--origin` option to
  `_add_log_options` with `choices`, and thread it through the `sase stitch log` handler
  into the filter spec.
- `src/sase/config/sase.schema.json`: update the `ace.artifacts.stitches.default_query`
  description to mention `origin:`.
- Tests: extend `tests/test_vcs_log_filter_query.py` for parse, canonical round-trip,
  negation, comma lists, invalid values, and completions; add CLI coverage for
  `--origin` including that it composes with `--limit` without dropping matches.

## Docs, help modal, and configuration reference

- `docs/ace.md`: extend the Stitches query paragraph with `origin:`, and document the
  origin column, the three glyphs, and the adaptive legend in the same section that
  explains the `[P/N]` legend.
- `docs/commit_workflows.md`: document the provenance invariant — every SASE-created
  commit carries `SASE_TYPE=`, `stitch` marks the tracked workflow — and the accepted
  consequences listed in the Design section above (squash merges, upstream commits,
  human-run `sase stitch create`).
- `docs/cli.md`: document `sase stitch log --origin` and the `origin` field in
  `--format json`.
- `docs/configuration.md`: mention `origin:` in the `default_query` row for the Stitches
  pane.
- `docs/rust_backend.md`: if the new binding belongs in the golden-contract table or the
  binding inventory, add it.
- `src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py`: add an
  `origin:stitch/auto/manual` row to the Stitches Pane section, honoring the
  57-character box width and the 32-character description limit documented in
  `src/sase/ace/CLAUDE.md`.
- Update `CHANGELOG.md` if this repo's convention requires it, and re-run
  `just check-full` for the combined tree.

## Risks and non-goals

- **Wire schema bump.** Adding `origin` moves the VCS-log wire from 3 to 4. Both sides
  pin the constant and the rehydrators fail loudly on a mismatch, so the failure mode is
  a clear error, not silent corruption — but the `core` phase must land in `sase-core`
  before the `wire` phase's CI runs.
- **Glyph rendering.** All three glyphs are already used in this codebase, so Fira Code
  coverage is established. Still verify cell width in the `render` phase; a two-cell
  glyph would break column alignment everywhere.
- **Legend width.** The legend already carries three presence keys and a conditional
  merge key. The adaptive rule keeps the common case to one added key, but the 80-column
  snapshot is the check that matters.
- **Not in scope.** No join against Patch `STITCHES:` records to identify stitches: the
  message footer is the single source of truth, and a cross-file join would be
  expensive, incomplete for archived or remote Patches, and unavailable for sidecar and
  linked repos. No backfill or rewrite of existing commit messages — rule 3 of the
  classifier covers history. No change to the Patch `origin:` (PR origin) property or
  its chip.
