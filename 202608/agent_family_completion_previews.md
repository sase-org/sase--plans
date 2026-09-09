---
tier: epic
title: Plan-aware agent-family completion previews
goal: "Agent-family completion entries in the ACE prompt input and in external editors
  lead with the tale/epic they belong to — tier, title, and epic phase structure — and
  fall back to the launch prompt instead of a list of member names.

  "
phases:
  - id: preview
    title: Shared family plan-preview value and TUI resolution cache
    depends_on: []
    size: medium
    description:
      "preview: add the surface-neutral AgentFamilyPlanPreview value plus its shared
      text formatters, and the TTL-cached TUI resolver that warms previews from agent
      rows off the render path."
  - id: rows
    title: Prompt-input completion rows and panel subtitle
    depends_on:
      - preview
    size: medium
    description:
      "rows: schedule the deferred preview warmup after Agents loads, carry the preview
      onto completion candidates, and render the new family row preview plus the
      selected-family panel subtitle."
  - id: editor
    title: Editor-helper agent catalog detail and documentation
    depends_on:
      - preview
    size: medium
    description:
      "editor: enrich family entries in the agent-catalog helper with a plan-aware
      detail line and a markdown documentation block, resolved from the artifact
      snapshot under a recency cap."
  - id: lspdoc
    title: sase-core LSP documentation passthrough
    depends_on:
      - preview
    size: small
    description:
      "lspdoc: add the optional documentation field to the Rust agent-catalog wire entry
      and surface it on agent completion items."
proposed_by: bbugyi200.athena.03u
bead_id: sase-n9
create_time: 2026-09-09 19:49:45
status: wip
---

- **PROMPT:**
  [prompts/202608/agent_family_completion_previews.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agent_family_completion_previews.md)
- **BEAD:**
  [sase-n9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-n9/README.md)

# Plan: Plan-aware agent-family completion previews

## Problem

Agent-family entries in the prompt-target completion menu render as:

```
▸ F 03s                         family · 2      ● 03s--plan, 03s--code
```

The preview column repeats the family's own name twice with role suffixes appended. That
is the least informative thing we know about the family: the member names are derivable
from the name already shown in the same row. What the user actually needs, when picking
a `%wait:` or `#fork:` target, is _what that family is working on_.

The same weakness exists in external editors. `sase editor-helper agent-catalog` emits
`detail: "family · 2 members"` for every family, and the Rust xprompt LSP renders that
as the completion item's detail and label description. Agent completion items never set
`documentation`, so the editor's documentation popup is empty for every family.

Nearly all of the missing information is already resolved and rendered elsewhere.
Selecting the same family in the Agents tab shows a `SASE CONTEXT` `PLAN` lane with the
tier (`tale`), the plan `Title:`, the `Goal:`, and — for an epic — every phase with its
size and description.

## Design

### Preview ladder

One ordered ladder decides what a family row shows. Each surface renders the same rung
with its own typography, so the TUI and the editor never disagree about _which_ fact is
being shown.

1. **Plan identity** — the family resolves to an authored plan. Show a tier chip
   (`Tale`, `Epic`, or the generic `Plan`) and the plan title. For an epic also show its
   phase structure.
2. **Bead identity** — the family has no authored plan but does resolve to a phase or
   task bead. Show a `Phase` / `Task` chip and the bead title.
3. **Tier without a title** — the plan file is missing, unreadable, or has no `title:`,
   but the tier is still known. Show the chip and fall back to the prompt snippet in
   place of the title. The user still learns "epic or tale", which is the question the
   chip exists to answer.
4. **Prompt snippet** — nothing resolves. Show a dimmed snippet of the prompt the user
   wrote for the family's initial agent. This replaces the member-name list in the row.

Member names are not deleted, they move: rung 4 rows surface them in the panel subtitle
(below), where they are the next layer of detail rather than the headline.

### Preview resolution

`resolve_agent_plan_enrichment` (`sase/ace/tui/models/agent_associated_plan.py`) already
produces exactly the two shapes the ladder needs — an `AssociatedPlanSummary`
(`PlanDisplay`) for rungs 1/3 and a `BeadSummary` for rung 2 — and it is already the
resolver behind the Agents-tab `PLAN` lane. The epic's phase structure comes from
`PlanDisplay.phases` plus the pure `plan_phase_waves` helper (`sase/sdd/plan_waves.py`);
no bead statuses are read.

Resolution is per _family_, not per row: resolve the family root entry first, and if it
yields nothing, walk `concrete_family_member_rows` in order and take the first non-empty
result. A family whose root predates plan metadata still gets its plan from its `--plan`
member.

When a family resolves both an authored plan and a bead, the authored plan wins — the
tale/epic title is what the user asked to see.

### Keeping the keystroke path clean

`sase/memory/tui_perf.md` rule 11: completion/typing paths are read-only and must not
call side-effectful or unbounded resolvers. `build_agent_completion_candidates` runs
synchronously when the menu opens, so it must read plan previews from memory only.

The repo already has three instances of the shape this needs —
`_loading_diff_badges.py`, `_loading_live_hints.py`, `_loading_bead_warmup.py`. This
epic adds a fourth lane in the same shape and deliberately does **not** refactor the
existing three.

Cached state lives in a module-level TTL cache keyed by the row fields that determine
the plan association, mirroring `_BeadDisplayCache` in
`sase/ace/tui/models/agent_bead.py`. A key-based cache survives agent-list reloads
without any carry-over pass, and it expresses the three states the renderer needs:

- **cache miss** — never resolved; render rung 4.
- **empty preview** — resolved, nothing to show; render rung 4 and stop retrying until
  the TTL expires.
- **preview** — render rungs 1–3.

The prompt snippet itself stays synchronous. `_candidate_from_agent` already reads
`raw_xprompt.md` through the mtime-keyed `ArtifactFileCache` for every visible row,
including family rows, so rung 4 costs nothing new; it is simply rendered instead of
discarded.

### TUI row anatomy

Group rows keep their existing column grid so families, clans, and tribes stay aligned.
Only the trailing preview column changes, and only for families.

```
  ▸ ␣ F ␣ <name:26> ␣␣ <badge:14> ␣␣ ● <preview…>
    ^glyph            ^"family · 2"    ^new
```

The preview budget becomes responsive: `max(24, inner_width - 50)`, where 50 is the
fixed prefix (2 selection marker + 2 glyph + 26 name + 2 + 14 badge + 2 + 2 status dot).
`inner_width` already reaches the row renderers via `build_completion_panel_content`;
today only the artifact-ref rows use it.

Composition, dropped only from the right, with **only the title/snippet segment ever
truncated** so the chip and the phase structure survive at any width:

```
● Epic · 6 phases · 3 waves · Plan-aware agent-family completion previews
● Tale · Complete common words from the middle of a word
● Phase · Prompt-input completion rows and panel subtitle
● Epic · 6 phases · <dim prompt snippet>
● <dim prompt snippet>
```

Degradation order when the budget is tight: drop ` · 3 waves`, then compress `6 phases`
to `6ph`, then drop the structure segment entirely, then ellipsize the title.

Chip colors reuse the existing cross-surface accents so the completion menu matches
toasts, notifications, and the `PLAN` lane: `PLAN_TIER_PRESENTATIONS`
(`sase/plan_tier_presentation.py`) for `Tale` (`#FFD75F`) and `Epic` (`#AF87FF`),
`GENERIC_PLAN_ACCENT` for `Plan`, and `BEAD_TYPE_PRESENTATIONS`
(`sase/bead_type_presentation.py`) for `Phase` (`#87D7FF`) and `Task` (`#D787FF`). All
five read clearly against the family identity blue `#00AFFF` already used for the glyph
and name.

### TUI panel subtitle

The row is the index; the panel border subtitle is the next layer down for the
_selected_ candidate. `model_completion_subtitle` already establishes this pattern for
the `%model` menu, and the `directive_arg` / `xprompt_arg_agent` menus currently leave
the subtitle empty.

The subtitle renders only for a selected **family** candidate (clan and tribe rows still
show their members in the row, so a subtitle there would be pure redundancy):

| Preview rung | Subtitle                                             |
| ------------ | ---------------------------------------------------- |
| epic         | `◆ <phase title> · <phase title> · <phase title> +N` |
| tale / plan  | the plan goal                                        |
| phase / task | the parent epic title, else the bead description     |
| none         | `<member>, <member>, <member> +N`                    |

### Editor / LSP surface

`agent_catalog_response` (`sase/integrations/_editor_helper_agents.py`) already holds
everything needed: `snapshot.records` carry `agent_meta.plan_path`, `sdd_plan_path`,
`epic_plan_ref`, `epic_bead_id`, `phase_bead_id`, and a `raw_prompt_snippet`. It runs as
a short-lived subprocess per completion request behind a 5 s timeout in the Rust
`catalog_cache`, so process-level caches do not survive; work must be bounded per call
instead.

Two fields carry the preview:

- **`detail`** — one plain-text line. Already plumbed end to end: the Rust
  `agent_entry_detail` returns `entry.detail` verbatim when non-empty, and
  `completion_item` copies it to `CompletionItem.detail` while `agent_completion_label`
  also feeds it into `label_details.description` (prefixing `family · ` because the
  string starts with the tier word, which reads correctly). **This means the `editor`
  phase improves external editors with the currently released LSP binary, with no Rust
  change.**

  ```
  epic · 6 phases · 3 waves · Plan-aware agent-family completion previews
  tale · Complete common words from the middle of a word
  phase · Prompt-input completion rows and panel subtitle
  family · 2 members · Fix the flaky selection-health test
  ```

- **`documentation`** — a markdown block for the editor's documentation popup, where the
  epic's phase list finally has room. This is a new wire field; the `lspdoc` phase
  teaches Rust to read it.

Wire contract, fixed here so `editor` and `lspdoc` cannot drift:

> `AgentCompletionEntry` gains an optional string field named `documentation`. The
> Python helper omits it or sends `""` when there is nothing to show. Rust deserializes
> it with `#[serde(default)]` and sets `CompletionCandidate.documentation` to
> `Some(value)` only when non-empty. `AGENT_CATALOG_SCHEMA_VERSION` stays at `1`:
> `AgentCompletionEntry` does not use `deny_unknown_fields`, so an older LSP binary
> silently ignores the new key and a newer binary tolerates an older helper.

Documentation shape (bounded — at most 6 phases listed, goal clipped):

```markdown
**Epic** · 6 phases · 3 waves

## Plan-aware agent-family completion previews

Agent-family completion entries lead with the tale or epic they belong to.

- `preview` — Shared family plan-preview value and TUI resolution cache (medium)
- `rows` — Prompt-input completion rows and panel subtitle (medium)
- `editor` — Editor-helper agent catalog detail and documentation (medium)
- `lspdoc` — sase-core LSP documentation passthrough (small)

---

family · 5 members · RUNNING
```

## Non-goals

- Clan, tribe, and plain-agent completion rows keep their current preview. A clan is a
  set of agents rather than one sase agent with one job, so a single plan title would be
  misleading; plain agents already show their prompt snippet. Both can adopt the chip
  later if they gain a plan identity.
- The wait modal (`WaitModal` / `candidate_option`) keeps its dense column layout. It
  does not render prompt snippets today either, so extending it is a separate design
  question.
- Epic _progress_ (how many phase beads are done) is deliberately excluded. It would
  require a bead lookup per phase on both surfaces; phase and wave counts come free from
  the plan file and answer the size question.
- No Agents-tab row changes, no completion ordering or ranking changes.
- Releasing sase-core and bumping the `sase-core-rs` pin in `pyproject.toml` are outside
  this epic. Every phase must be useful without that release.

## preview: Shared family plan-preview value and TUI resolution cache

Add the vocabulary both surfaces share, plus the TUI-side resolver.

**New `src/sase/agent_family_plan_preview.py`** — surface-neutral, no ACE TUI model
imports, so the editor helper can use it:

- `AgentFamilyPlanPreviewKind = Literal["tale", "epic", "plan", "phase", "task"]`.
- `AgentFamilyPlanPreview` frozen slotted dataclass: `kind`, `title`, `goal`,
  `parent_title`, `phase_count`, `wave_count`, `phase_titles` (bounded tuple),
  `phase_ids`, `phase_sizes`, `size`; plus an `is_empty` property.
- An `EMPTY_AGENT_FAMILY_PLAN_PREVIEW` singleton for "resolved, nothing to show".
- `agent_family_plan_preview_from_plan(plan: PlanDisplay)` — maps `effective_tier`,
  `title`, `goal`, and, when `phase_availability == "available"`, `len(phases)` plus
  `len(plan_phase_waves(phases))` (`plan_phase_waves` returns `None` on a dependency
  cycle; leave `wave_count` unset in that case rather than inventing a number).
- `agent_family_plan_preview_from_bead(*, bead_type, title, parent_title, size)` — takes
  primitives, not the TUI's `BeadSummary`, to keep layering clean.
- Shared formatters used by every surface so the wording cannot drift:
  `agent_family_plan_preview_label` (`"Epic"`), `agent_family_plan_preview_accent` (hex,
  from the existing tier/bead presentation tables),
  `agent_family_plan_structure_text(preview, *, compact)` (`"6 phases · 3 waves"` /
  `"6ph"` / `""`), `agent_family_plan_preview_detail(...)` (the one-line editor detail),
  and `agent_family_plan_preview_documentation(...)` (the markdown block).

**New `src/sase/ace/tui/models/agent_family_preview_cache.py`**:

- A TTL-bounded LRU modeled on `_BeadDisplayCache`
  (`sase/ace/tui/models/agent_bead.py`): a longer TTL for resolved previews, a shorter
  one for empty results, a bounded entry count, an `RLock`, and a distinct
  `FAMILY_PREVIEW_CACHE_MISS` sentinel.
- Cache key derived purely from in-memory row fields that can change the plan
  association — the family reference name plus `plan_path`, `archived_plan_path`,
  `sdd_plan_path`, `epic_plan_ref`, `plan_committed`, `plan_action`, `epic_bead_id`,
  `phase_bead_id`. Model it on `associated_plan_cache_key`. It must be stable across an
  Agents reload that rebuilds `Agent` objects, and it must change when the association
  changes.
- `cached_family_plan_preview(agent)` — pure memory read, safe from a render or
  keystroke path, returning the sentinel / `None` / a preview.
- `should_resolve_family_plan_preview(agent)` — false for non-family rows and for keys
  with a live entry.
- `warm_family_plan_previews(agents)` — the off-thread entry point. Opens one
  `BeadIssueLookupSession` for the whole batch and threads it through
  `resolve_agent_plan_enrichment`, resolves root-then-members as described in the Design
  section, writes the cache, and returns a key-indexed mapping so the caller can decide
  what changed. Never raises: one bad plan file or bead store must not lose the rest of
  the batch.

**`Justfile`** — this phase's public symbols have no non-test consumer until `rows` and
`editor` land. Add a `--epic-symbol "<this epic's bead id>(<Symbol>)"` entry to
`_lint-symvision` for each symbol `just check` reports, per `sase/memory/symvision.md`.
Do not pre-add entries for symbols that are not actually flagged. Phases `rows` and
`editor` remove the entries their code consumes.

Tests: preview construction from each `PlanDisplay` shape (tale, epic, epic with a
dependency cycle, unavailable phases, missing title) and from each bead type; cache key
stability across a simulated reload and invalidation when an association field changes;
three-state cache semantics; root-then-member resolution order; a batch containing an
unreadable plan still returns the other previews.

## rows: Prompt-input completion rows and panel subtitle

Wire the preview into the ACE prompt input.

**New `src/sase/ace/tui/actions/agents/_loading_family_previews.py`** —
`AgentFamilyPreviewMixin`, following `_loading_bead_warmup.py` closely:
scheduled/running/pending/source coalescing flags, `spawn_pump_free_task` with its own
`registry_attr`, a nav-gate deferral with a `set_timer` retry, the work body in
`asyncio.to_thread(warm_family_plan_previews, candidates)`, a `tui_trace` span,
`log.exception` on failure, and a trailing re-arm. Candidates are the visible family
rows for which `should_resolve_family_plan_preview` is true. Because the cache is keyed
rather than row-attached, this lane needs neither a carry-over pass nor a
result-application-by-identity step: nothing currently painted depends on a preview, so
a finished batch triggers no refresh at all. The next completion menu build simply finds
the cache warm.

Wiring:

- `src/sase/ace/tui/actions/agents/_loading.py` — add the mixin to `AgentLoadingMixin`.
- `src/sase/ace/tui/actions/_state_init_runtime.py` — initialize the four coalescing
  flags and the task registry set alongside the diff-badge and bead-warmup ones.
- `src/sase/ace/tui/actions/agents/_loading_apply.py` — schedule with `source="apply"`
  next to `_schedule_diff_badge_classification`.
- `src/sase/ace/tui/actions/agents/_wait_actions.py` — in
  `visible_agent_completion_candidates`, schedule with `source="completion"` when any
  family candidate is still unresolved. Scheduling only sets flags and spawns a
  pump-free task, so the menu still opens synchronously; the resolved preview appears
  the next time the menu is built.

Candidate model and builder:

- `src/sase/ace/tui/_agent_completion_models.py` — add
  `plan_preview: AgentFamilyPlanPreview | None = None` to `AgentCompletionCandidate`.
  Keep `member_names`; it now feeds the subtitle. Include the preview title in
  `search_text` so typing a plan title finds its family.
- `src/sase/ace/tui/_agent_completion_candidates.py` —
  `_build_family_completion_candidates` reads `cached_family_plan_preview` (memory only)
  and attaches the result. Extend `_raw_prompt_for_agent` so a family falls back to the
  first concrete member's raw prompt when the root has none, giving rung 4 the initial
  agent's prompt as the user asked. Add the preview title to `search_aliases` where that
  is the established mechanism.

Rendering:

- `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows_agents.py` —
  `_append_group_completion_row` takes the responsive preview budget and, for
  `kind == "family"`, renders the ladder instead of `_member_preview`. Keep
  `_member_preview` for clan and tribe rows. Truncate only the title/snippet segment.
- `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel_content.py` — thread
  `inner_width` into both agent-row call sites (`kinds.xprompt_arg_agent` and the
  `directive_arg` branch).
- `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel_labels.py` — new
  `agent_completion_subtitle(rows, selected_index, inner_width)` implementing the
  subtitle table, returning an empty `Text` for any non-family selection.
- `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py` — call it in the
  `else` branch that currently blanks the subtitle, guarded so the wait menu's
  `priority=` / `runners=` / `time=` keyword rows and the delete-hint subtitle keep
  their current behavior.

Check whether the `?` help modal documents the completion menu's glyph or column legend
(`src/sase/ace/tui/modals/help_modal.py`, 57-character box rule in
`src/sase/ace/CLAUDE.md`). Add a legend line only if such a section already exists; do
not invent one.

Tests: row rendering for each ladder rung and each degradation step of the budget; the
chip and structure survive truncation while the title ellipsizes; clan and tribe rows
are unchanged; the subtitle for each rung and for a non-family selection; the
initial-agent prompt fallback; the completion path performs no plan or bead I/O (assert
with a resolver patched to raise). Update
`tests/ace/tui/visual/test_ace_png_snapshots_prompt_target_completion.py` so
`_TARGET_ROWS` includes an epic-backed family, a tale-backed family, and a snippet-only
family, then regenerate the goldens with
`just test-visual --sase-update-visual-snapshots` and inspect the diffs in
`.pytest_cache/sase-visual/` before accepting them.

## editor: Editor-helper agent catalog detail and documentation

Give external editors the same preview through the agent catalog.

**New `src/sase/integrations/_editor_helper_agent_plans.py`** — snapshot-based
resolution, kept out of `_editor_helper_agents.py` so neither file grows past the
`toobig` warn tiers:

- Input is the newest-generation `_CatalogMember` list `_family_entries` already
  computes, plus the matching `AgentMetaWire` records.
- Resolve a plan path from member metadata in root-then-member order: `plan_path`,
  `archived_plan_path`, `sdd_plan_path`, `epic_plan_ref`. Load it with
  `sase.sdd.plan_display.load_plan_display`, which never raises for a missing,
  unreadable, or invalid file, and map it with `agent_family_plan_preview_from_plan`.
- When no plan path resolves and a member carries `epic_bead_id` or `phase_bead_id`,
  fall back to one bead lookup through a single `BeadIssueLookupSession` shared by the
  whole call, and map it with `agent_family_plan_preview_from_bead`.
- Bound the work, because this runs per completion request in a fresh subprocess: dedupe
  plan-file reads by resolved path, and enrich only the most recent
  `_FAMILY_PREVIEW_LIMIT` families (a named module constant; start at 40) ranked by
  newest member timestamp. Families past the cap keep today's `family · N members`
  detail.
- Wrap the whole enrichment in the same defensive posture `_derive_group_entries`
  already uses: any failure degrades to the current detail and must never hide real
  agents or fail the response.

**`src/sase/integrations/_editor_helper_agents.py`** — `_family_entries` sets `detail`
from `agent_family_plan_preview_detail` and adds `documentation` from
`agent_family_plan_preview_documentation`, using the record's `raw_prompt_snippet` for
the fallback rung. `raw_prompt_snippet` is only the first 200 bytes of `raw_xprompt.md`
and still carries frontmatter, leading `%directives`, and the `#gh:` VCS tag, so strip
it the same way `sase/ace/tui/_agent_completion_prompt.py:prompt_snippet` does before
using it; if the strip leaves nothing usable, emit the plain member-count detail rather
than a snippet of directives.

Remove this phase's `--epic-symbol` entries from the `Justfile`.

Tests: extend `tests/test_editor_helper_agent_catalog.py` with fixture families for each
rung, asserting the exact `detail` strings and the documentation structure; the recency
cap leaves older families on the member-count detail; a missing or unreadable plan file
degrades instead of raising; an unavailable bead store degrades; `schema_version` stays
`1`. Time `sase editor-helper agent-catalog` before and after against a realistic
artifact store and record both numbers in the phase bead notes; added wall time should
stay under roughly 150 ms, and if it does not, lower `_FAMILY_PREVIEW_LIMIT` rather than
shipping a slower completion.

## lspdoc: sase-core LSP documentation passthrough

Land the Rust half in the sibling core repo. Open it with
`sase repo open sase-core -r "<reason>"` and work only from the printed path, per
`sase/memory/` repository rules.

- `crates/sase_core/src/editor/wire.rs` — add
  `#[serde(default)] pub documentation: String` to `AgentCompletionEntry`, documented as
  an optional markdown block supplied by the Python helper. Leave
  `AGENT_CATALOG_SCHEMA_VERSION` at `1`.
- `crates/sase_core/src/editor/completion.rs` — in `build_agent_completion_candidates`,
  replace the hardcoded `documentation: None` with the entry's value when it is
  non-empty. The existing `markdown_doc` conversion in
  `crates/sase_xprompt_lsp/src/lsp_convert.rs:completion_item` already turns it into a
  `CompletionItem.documentation`, so no change is needed there.
- Update any fixture literals of `AgentCompletionEntry` that construct all fields
  positionally or exhaustively.

Tests: a Rust unit test that a catalog entry carrying `documentation` produces a
candidate with `Some(...)`, and one that an entry without it stays `None`; extend the
existing `wait_completion_uses_kind_aware_agent_catalog` server test to cover an entry
with documentation. Verify with `just check` in sase-core (`fmt`, `clippy`, `test`).

Do not release sase-core and do not touch the `sase-core-rs` pin in `pyproject.toml`;
the `editor` phase's `detail` improvement already works against the released binary, and
this field activates whenever the next release is consumed.

## Verification

Every Python phase, from an ephemeral workspace:

```bash
just install
just check
```

`just check-full` before landing the epic's combined tree, run through `/sase_monitor`
with a `--next` action rather than inline, because it routinely outruns a single agent
turn. The `rows` phase additionally runs `just test-visual` and inspects
`.pytest_cache/sase-visual/` before accepting any regenerated golden. The `lspdoc` phase
runs `just check` inside the sase-core checkout.

Manual confirmation for `rows`: open `sase ace`, type `%wait:` in the prompt input, and
confirm that a family working an epic shows its tier, phase structure, and title, that
the selected family's subtitle lists its phases, and that a family with no plan shows
its launch prompt rather than its member names.
