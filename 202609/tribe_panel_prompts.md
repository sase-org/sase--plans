---
tier: tale
title: Agent tribe PROMPTS section — the tribe's intent map
goal:
  Selecting an agent tribe panel shows every member agent's prompt in a fold-aware
  PROMPTS section whose default glance view tells the user at a glance what each agent
  in the tribe was asked to do, with deeper fold levels revealing labels, sizes,
  previews, and full prompt bodies, and with no measurable TUI slowdown.
size: medium
proposed_by: bbugyi200.athena.0pw
create_time: 2026-09-23 10:00:10
status: wip
---

# Plan: Agent tribe PROMPTS section — the tribe's intent map

## Context

Whole-panel focus on an Agents-tab tribe panel renders a fold-aware `TRIBE` document
through `AgentPromptPanel` (`build_tribe_detail_text` / `_append_tribe_body` in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py`). Its body today is:
`NEEDS ATTENTION`, the numbered `TRIBE MEMBERS` roster, `ERRORS`, `OUTPUT VARIABLES`,
`WORKFLOW VARIABLES`, `REPLIES`, `SLOW TOOL CALLS`, and (level 4 only)
`RUNTIME STATISTICS`. Tribe fold levels are 1 Glance (`COLLAPSED`, the default
`panel_fold_level`), 2 Triage (`EXPANDED`), 3 Inspect (`FULLY_EXPANDED`), and 4
Forensics (`EXHAUSTIVE`) — see the tribe table in `docs/ace.md`.

`REPLIES` is disk-backed: `build_tribe_enrichment`
(`widgets/prompt_panel/_agent_tribe_aggregation.py`) runs in the coalesced,
last-request-wins tribe worker (`widgets/prompt_panel/_agent_display_async_groups.py`),
loads per-member content through `build_agent_group_disk_snapshot` and its mtime-keyed
per-member cache (`widgets/prompt_panel/_agent_clan_aggregation.py`,
`_agent_clan_member_content.py`), reuses cached clan snapshots, and repaints through
`TribeSectionSnapshotLoaded` → the debounced detail update. Sections stay hidden until
loaded, behind the single `⋯ scanning member data…` tail.

Clan documents already have a `PROMPTS` section, and the shared per-member loader
already supports the `"prompts"` disk section (`_load_member_prompts` returns an
`AGENT XPROMPT` entry from `raw_xprompt.md` and an `AGENT PROMPT` entry from the
selected `*_prompt.md`). Tribes never request or render it. This plan adds a tribe
`PROMPTS` section built on that pipeline.

Everything here is presentation logic over already-loaded rows and existing artifact
files, so it stays in this repo. No `sase-core` changes are needed (same conclusion as
the original tribe-panel epic). No keymaps, config keys, or feature flags change: the
section uses the existing `z` fold chords, `Ctrl+J`/`Ctrl+K` section stops, and digit
member jumps. The tale lands the whole feature at once, so it needs no beta flag.

### What real prompts look like (design inputs)

Measured on recent `raw_xprompt.md` artifacts:

- A launch preamble of directives and a VCS tag usually comes first (`#gh:…`,
  `%id(3, clan=sase-16t, bead=sase-16t.3)`, `%model:@medium`, `%auto`, `%wait:…`,
  `%q(w=0.25)`). Those lines are launch mechanics, not intent.
- Many prompts are only xprompt invocations (`#bd/work_phase_bead:sase-16t.3`,
  `#bd/land_epic:sase-16n.11`). Their arguments carry the key information, so the
  invocation itself is the headline.
- Human prompts are hard-wrapped prose
  (`Can you help me split the \`src/…/_settings.py\` file up into multiple files? …`),
  so the first line alone often cuts a sentence mid-way.
- Routine/chop tribes relaunch the same body many times, with only the directive
  preamble (ids, waits) differing.
- Monitor follow-ups start with `%xprompts_enabled:false` and then a Markdown heading
  (`# Monitored command finished`).
- `summarize_prompt_for_list(raw).xprompts` reports false-positive chips such as
  `#D75FFF` when a directive argument contains Rich color markup
  (`%clan(…, summary=[bold #D75FFF]…)`). Computing chips from the preamble-stripped body
  avoids this for tribe prompts.

## Design

### Principles

1. **Intent at a glance.** At the default level (1, Glance), `PROMPTS` is a compact
   intent map: one headline per distinct prompt, tied to the roster by the same number
   chips. This is a deliberate exception to "level 1 shows only headings for most
   sections". Like `NEEDS ATTENTION` and the roster, prompts answer the first question a
   user has about a tribe ("what are these agents doing?"). Document the exception.
2. **Never repeat a wall of text.** Identical prompt bodies (same project, same text
   once the launch preamble is stripped and whitespace is collapsed) are listed once,
   with an `×N` badge and a shared-by list.
3. **Progressive disclosure at two scales.** Section level and per-entry level both
   follow the tribe fold ladder. Each step adds something visible, and a deeper level
   never shows fewer entries than a shallower one.
4. **Render paths are pure appends.** All I/O, parsing, humanizing, grouping, and
   xprompt tokenization happen in the existing tribe worker thread. The renderer only
   appends precomputed strings and replays precomputed style spans.

### Placement

Render `PROMPTS` immediately after `TRIBE MEMBERS` and before `ERRORS`. The document
then reads who → what they were asked → what went wrong → what they produced (variables,
replies). Existing fixtures without prompt artifacts render byte-identically, because a
known-empty `PROMPTS` section produces no output.

### Section heading

`append_fold_heading(title="PROMPTS", section_id="tribe:prompts", level, count=<agents with a prompt>)`.
When the number of distinct entries is smaller than that count, add a dim summary
` · <M> distinct`. For example, `▸ PROMPTS · 9 · 6 distinct`. Add
`summary: str | None = None` to the tribe `append_fold_heading` in
`_agent_display_tribe_common.py` (rendered as a dim ` · <summary>` after the count) and
`prompts: str = "tribe:prompts"` to `_TribeSectionIds`.

### Entries: which prompt belongs to which row

Walk `TribeUnitSource`s in roster order, and within each source walk `source.rows` in
order. Match each row to its `ClanDiskMemberSnapshot` by `member_identity`. For each
row:

- Skip monitors, gates, and proc shells (`row.is_monitor`, `row.is_gate`,
  `row.is_proc_shell`).
- Workflow step children (`row.is_workflow_step_child`): use only the `AGENT PROMPT`
  entry, and only when `row.step_type == "agent"`. Their shared `raw_xprompt.md` belongs
  to the parent, so never attribute it to a step.
- Every other row: prefer `AGENT XPROMPT` (the authored prompt). Fall back to
  `AGENT PROMPT` only when no raw xprompt exists (historical rows).

Group rows by `group_key` in first-appearance order. The first member is the
representative whose digest renders.

### Prompt digest (worker-side, pure, cached)

New module `src/sase/ace/tui/widgets/prompt_panel/_agent_tribe_prompts.py` (keep it
under the toobig 700-line warning threshold). It owns:

```python
type StyleSpan = tuple[str, int, int]  # (rich style, start, end) relative to one string

@dataclass(frozen=True, slots=True)
class PromptDigest:            # content-only; cached by raw-text hash
    group_key: str             # 12-hex blake2b of f"{project}\0{' '.join(body.split())}" (pre-humanize body)
    headline: str              # humanized, <=120 chars
    headline_spans: tuple[StyleSpan, ...]
    body: str                  # humanized, preamble-stripped, rstripped; line structure preserved
    body_spans: tuple[StyleSpan, ...]
    body_line_count: int
    launch: str                # humanized, whitespace-collapsed preamble (no frontmatter); "" when none
    launch_spans: tuple[StyleSpan, ...]
    xprompts: tuple[str, ...]  # <=3 xprompt chips from the body, excluding ones already visible in the headline
    project: str | None        # short display name of the prompt's VCS/project target

@dataclass(frozen=True, slots=True)
class TribePromptMember:
    unit_identity: ClanAgentIdentity
    unit_label: str
    member_identity: ClanAgentIdentity
    member_label: str          # source.labels value: unit label for the unit root, ".3" / "--code" for nested rows
    @property
    def is_unit_root(self) -> bool: ...

@dataclass(frozen=True, slots=True)
class TribePromptGroup:
    digest: PromptDigest       # the representative (first) member's digest; its launch line is that member's
    members: tuple[TribePromptMember, ...]

@dataclass(frozen=True, slots=True)
class TribePromptsSnapshot:
    groups: tuple[TribePromptGroup, ...]
    agent_count: int           # members across all groups
    multi_project: bool        # >1 distinct non-None project across groups

def build_tribe_prompts(
    sources: Sequence[TribeUnitSource],
    member_snapshots: Mapping[ClanAgentIdentity, ClanDiskMemberSnapshot],
) -> TribePromptsSnapshot: ...
```

A digest is computed per raw text, so members whose preambles differ get different
digests that share one `group_key`. The group renders its representative's digest, and
the launch line appears only for single-member groups.

Digest algorithm for one raw prompt text:

1. `preamble, body = split_prompt_preamble(raw)`: a new public helper in
   `src/sase/ace/tui/_agent_completion_prompt.py` that composes the existing private
   `_strip_frontmatter`, `_strip_leading_prompt_directives`, and
   `_strip_leading_vcs_tag` (directives → VCS tag → directives, exactly the
   `prompt_snippet` order). It preserves newlines in the body, and the preamble keeps
   the frontmatter out. The VCS-tag pattern is `^`-anchored, so the body is always a
   suffix of the frontmatter-stripped text, and
   `preamble = text[: len(text) - len(body)].strip()`. Refactor `prompt_snippet` to call
   it. Behavior must not change; its existing tests cover this.
2. `project`: `extract_vcs_workflow_tag(raw)` → `extract_project_from_vcs_tag(tag)` (as
   in `_agent_xprompt_highlighting._agent_project_and_workspace`), shortened with
   `sase.project_display_names.project_display_name_for_ref`. Fail open to the raw ref,
   or `None`.
3. Headline source: the first paragraph of the body. Skip leading blank lines, then join
   consecutive non-blank lines up to the first blank line (or a fence line starting with
   three backticks), collapsing whitespace. Strip a leading Markdown heading marker
   (`^#{1,6}\s+`). An xprompt-only body therefore keeps its invocation with arguments
   (`#bd/work_phase_bead:sase-16t.3`). If the body is empty (a directives-only prompt),
   use the collapsed preamble. Truncate to 120 characters at a word boundary with `…`.
4. Humanize every displayed string (headline, body, launch) with the existing
   `humanize_prompt_body` (`_agent_display_clan_sections_common.py`), so project
   references render like every other prompt surface (D5/D6).
5. Style spans: add a public
   `xprompt_overlay_spans(source, *, known_skills=frozenset()) -> tuple[tuple[str, int, int], ...]`
   to `src/sase/ace/tui/util/xprompt_syntax.py`. It returns, in application order, the
   xprompt/alt-token overlays and the project-tag overlays that `apply_xprompt_overlays`
   applies today. Refactor `apply_xprompt_overlays` to delegate to it with identical
   behavior, including the byte cap. Compute `headline_spans`, `body_spans`, and
   `launch_spans` with it on the humanized strings. Known skills stay empty (skill-token
   styling is out of scope).
6. `xprompts`: `summarize_prompt_for_list(body).xprompts`, taken from the body so
   directive-argument false positives (`#D75FFF`) cannot appear. Drop chips already
   present in the headline text and keep at most 3.
7. `body_line_count = body.count("\n") + 1` when the body is non-empty.

Reliability and performance rules for the builder:

- **Fail open per prompt.** Wrap each digest computation. On any exception, return a
  plain digest (headline = `first_meaningful_line(raw)`, body = raw, no spans, no
  chips). One malformed prompt must never fail the tribe worker, because a failed worker
  leaves the whole document stuck behind the scanning tail.
- **Content-hash LRU.** Memoize digests in a module-level `OrderedDict` guarded by a
  `threading.Lock`: 1024 entries, keyed by
  `(blake2b(raw).hexdigest(), peek_project_tag_catalog_signature())` (the same catalog
  signature `util/xprompt_syntax.py` uses, so tag accents refresh once the catalog
  warms). Steady-state 10 s re-enrichments then only re-tokenize new or changed prompts,
  which keeps GIL time stolen from the event loop near zero. Both the base panel and a
  zoom-modal panel may run workers concurrently, hence the lock.

### Aggregation wiring (`_agent_tribe_aggregation.py`)

- Add `"prompts"` to `TribeEnrichmentSection` and `_TRIBE_DISK_SECTIONS`, and add a
  `prompts: TribePromptsSnapshot | None` field to `_TribeDiskSnapshot` (default an empty
  snapshot so construction sites stay simple).
- In `build_tribe_enrichment`, when `"prompts"` is requested, collect each source's
  per-member snapshots from the `ClanDiskSnapshot` that `_disk_snapshot_for_source`
  returns (`disk.members`, keyed by `member_identity`). Reused cached clan snapshots
  already carry `"prompts"`, because clan documents load all four disk sections. Then
  call `build_tribe_prompts(sources, member_snapshots)`.
- `tribe_enrichment_sections_for_fold_state` (`_agent_display_tribe.py`) always requires
  `"prompts"`, alongside `"replies"` and `"slow-tool-calls"`, because the section has
  content at level 1. `_has_pending_enrichment` then covers it automatically: hidden
  plus scanning tail until loaded, and no section at all when loaded with zero prompts
  (the known-empty convention).

### Rendering (new `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe_prompts.py`)

`append_prompts(text, section_snapshot, *, level, overrides, unit_numbers)` is called
from `_append_tribe_body` right after `append_member_roster`.
`unit_numbers = {t.member_identity: t.number for t in jump_map.targets}` comes from the
roster's returned `MemberJumpMap`, whether or not a publisher was supplied, so prompt
chips always carry the exact digits that jump to those units. If the disk section is not
loaded or has zero groups, render nothing.

Constants: `PROMPT_GLANCE_LIMIT = TRIAGE_LIMIT` (8), `PROMPT_LIST_LIMIT = 24`,
`PROMPT_PREVIEW_LINES = 10`, `PROMPT_BODY_SAFETY_LINES = 500`,
`PROMPT_ENTRY_ANCHOR_PREFIX = "tribe:prompt:"`.

**Entry-count ladder (by the section's effective level):**

| Section level | Entries shown   | Tail              |
| ------------- | --------------- | ----------------- |
| 1 Glance      | first 8 groups  | `  +N more` (dim) |
| 2 Triage      | first 24 groups | `  +N more`       |
| 3 Inspect     | first 24 groups | `  +N more`       |
| 4 Forensics   | all groups      | —                 |

**Per-entry ladder.** Each entry is an independently foldable, non-navigable anchor
(`append_fold_anchor(..., section_id=f"tribe:prompt:{group_key}")`, exactly like roster
rows). Its effective level is `effective_level(anchor, section_level)`: an entry
override, otherwise the section's effective level, clamped to `TRIBE_FOLD_SCALE`. So
`za`/`zA` with an entry at the viewport top open one prompt without changing the others.

- **Who part** (every level): one gold chip per distinct unit of the group's members, in
  first-appearance order, styled exactly like roster chips (`f" {n} "`,
  `bold black on {TRIBE_IDENTITY_COLOR}`, then a space). Show at most 3 chips, then a
  dim `+K`. A unit with no roster number (roster capacity exceeded) gets a dim `•`
  instead. Then:
  - single member, nested row → `› {member_label}` (dim `›`, label in the member style
    `bold #D75FFF`);
  - single member, unit root → the unit label (`bold {TRIBE_IDENTITY_COLOR}`) at entry
    level ≥ 2 only. At level 1 the chip alone points at the roster row just above, which
    keeps the glance line short;
  - multiple members → `×N` (`bold #5FD7FF`).
- **Entry level 1:** who part + `  ` + headline (base style `#D7D7FF`, headline spans
  replayed).
- **Entry level 2:** level 1 plus the unit label for unit-root members, and dim
  `·`-separated tags after the headline: xprompt chips (`#87D787`, the xprompt
  invocation color), `N lines` when `body_line_count > 1`, and `+{project}` styled with
  `sase.project_tag_style.project_column_style` only when `multi_project` is true.
- **Entry level 3:** the entry line becomes who part + tags, with no headline because
  the body follows. Then a gutter-quoted body preview of the first 10 body lines. Each
  line is prefixed with `   │` in `dim #8787AF`, the text is `#D7D7FF`, and body spans
  are replayed and clipped to the slice. When truncated, add `    │ … +N more lines`
  (dim italic). For multi-member groups, add `    ↳ shared by <labels>` (dim), where
  labels use the existing `unit › member` form (`_tribe_member_label`), at most 8 then
  `+N`. Put a blank line between level-3 and level-4 entries.
- **Entry level 4:** as level 3, but the whole body (safety cap 500 lines, with the same
  `… +N more lines` tail and the hint `open the agent (press its number) for the rest`).
  For single-member groups, prepend a `    launch  ` line with the humanized,
  highlighted preamble (label in `FIELD_LABEL_STYLE`), omitted when the preamble is
  empty. A directives-only prompt shows the launch line in place of an empty body.

Implement slicing without re-tokenizing. Find the character offset of the Nth newline,
build `Text(body[:cut], style=body_style)`, replay spans with `start < cut` clipped to
`cut`, then `split("\n", allow_blank=True)` and indent each line (as
`_append_tag_highlighted_lines` does for clans).

Illustrative level 1 (glance) rendering in the default 120×40 layout:

```text
▸ PROMPTS · 7 · 6 distinct
 0  Add a PROMPTS section to agent tribe documents so users can see what each agent in the tribe was asked to…
 0  › --code  @plan:202609/tribe_prompts.md The above plan has been reviewed and approved. Implement it…
 1  › .1  #bd/work_phase_bead:sase-16t.1
 1  › .2  #bd/work_phase_bead:sase-16t.2
 2  3  ×2  Inspect the documentation changes made by the update agent for sase.
```

Illustrative level 3 (inspect) entry for a 14-line prompt (the gutter shows the first 10
lines, elided here):

```text
 0  visual-tribe-build · #plan · 14 lines
    │ Add a PROMPTS section to agent tribe documents so users can see what
    │ each agent in the tribe was asked to do.
    │ (…8 more preview lines…)
    │ … +4 more lines
```

### Docs

Update `docs/ace.md` (tribe summary section):

- Tribe level table: add prompt content per level. Glance: "prompt headline digest (up
  to 8 distinct prompts)". Triage: "every distinct prompt (up to 24) with labels,
  xprompt chips, and sizes". Inspect: "10-line prompt previews". Forensics: "full prompt
  bodies (500-line safety cap per prompt) and launch directives".
- One short paragraph on `PROMPTS`. It sits after `TRIBE MEMBERS`. Its number chips are
  the same digits as the roster jump targets. Identical bodies are listed once with `×N`
  and a shared-by list. `za`/`zA` on a prompt entry opens just that prompt. It is the
  level-1 exception.
- Extend the "Reply and slow-call presence enrichment…" paragraph to include prompts.

Keep line widths and the existing table style (`just fmt` formats Markdown).

## Files

- `src/sase/ace/tui/_agent_completion_prompt.py` — public `split_prompt_preamble`, and
  `prompt_snippet` reuses it.
- `src/sase/ace/tui/util/xprompt_syntax.py` — public `xprompt_overlay_spans`, and
  `apply_xprompt_overlays` delegates.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_tribe_prompts.py` (new) — digest,
  grouping, LRU.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_tribe_aggregation.py` — `"prompts"`
  section wiring.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe_prompts.py` (new) —
  renderer.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py` — required sections,
  placement, and `unit_numbers`.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe_common.py` —
  `SECTIONS.prompts` and heading `summary`.
- `docs/ace.md`.
- Tests and goldens listed below.

## Testing

Unit tests (new `tests/ace/tui/widgets/test_agent_tribe_prompts.py` and
`tests/ace/tui/widgets/test_agent_display_tribe_prompts.py`, plus updates to the
existing tribe section and aggregation tests):

- `split_prompt_preamble`: frontmatter, stacked directives, VCS tag between directive
  runs, a body that is only an xprompt, a directives-only prompt, and
  `%xprompts_enabled:false` followed by a Markdown heading. The `prompt_snippet` outputs
  are unchanged.
- `xprompt_overlay_spans` matches what `apply_xprompt_overlays` stylizes: replay the
  spans on a `Text` and compare with a directly overlaid `Text`.
- Digest: joins a hard-wrapped first paragraph and truncates at 120 characters on a word
  boundary; uses the xprompt invocation (with args) as the headline; strips heading
  markers; directive-argument hex colors never become chips; chips already in the
  headline are dropped; the fallback digest is used when a helper raises; the LRU hit
  path skips tokenization (monkeypatch the span function to count calls).
- Grouping: identical bodies with different preambles coalesce in first-appearance
  order; the same body in different projects does not; `group_key` stays stable across
  rebuilds; monitors, gates, and proc shells are skipped; workflow step children use
  only their step prompt; `AGENT PROMPT` is used only as a fallback; `multi_project` is
  set correctly.
- Aggregation: seeded `raw_xprompt.md` artifacts across mixed units (a clan, a family,
  standalone agents) produce prompt groups; cached clan snapshots are reused without
  duplicate loads; a zero-prompt tribe is known-empty;
  `tribe_enrichment_sections_for_fold_state` now always includes `"prompts"` (update the
  existing exact-set assertions).
- Renderer: the four section levels (entry counts and tails; level 1 omits unit-root
  labels and tags; level 2 adds labels and tags; level 3 previews and truncation tail;
  level 4 full body, launch line, and the 500-line safety cap); chips use the roster
  jump-map digits, cap at 3 plus `+K`, and fall back to `•`; `×N` and the shared-by
  list; a per-entry override (`tribe:prompt:<key>`) opens one entry while its siblings
  stay at the section level; entry anchors are fold-only (not `Ctrl+J`/`Ctrl+K` stops)
  while the `PROMPTS` heading is a stop; placement is after `TRIBE MEMBERS` and before
  `ERRORS`; the section is absent and the scanning tail shows while not loaded; the
  section is absent when loaded and empty.
- Perf guard: render level 1 with 200 groups and assert that at most 8 entry lines
  appear and that the render never tokenizes (monkeypatch `xprompt_inspect.tokenize` to
  raise during `build_tribe_detail_text`).

Visual goldens (new
`tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_prompts.py`, which keeps the
existing 525-line tribe visual module under the size limit). Seed `raw_xprompt.md` into
tmp artifact dirs via `artifacts_dir` on fixture agents, following the seeding pattern
in `tests/ace/tui/visual/_ace_agents_png_snapshot_family_panel_fixtures.py`. Use a tribe
containing a family (distinct `--plan`/`--code` prompts), a clan with two
`#bd/work_phase_bead:<id>` phase workers, two standalone agents with identical bodies
but different preambles, and a monitor. Focus the panel, wait for `PROMPTS` in the SVG,
and move the `PROMPTS` heading to the viewport top with `Ctrl+J`. Capture
`agents_tribe_panel_prompts_glance_120x40` (level 1) and
`agents_tribe_panel_prompts_inspect_120x40` (level 3). The existing
`agents_tribe_panel_level_{1..4}_120x40` goldens must stay byte-identical, because their
fixtures have no prompt artifacts and must render exactly as today.

Verification: run `just fix` then `sase tool run check`. Then run
`just fix-tui-screenshots` with targeted selectors for the tribe visual modules through
`/sase_monitor`, and inspect every created golden and the report before finalizing: the
two new goldens are created, and the existing tribe goldens are unchanged. Also check
the result visually with a live `sase screenshot` of a real tribe panel at levels 1 and
3, confirming the section reads cleanly in the default layout.
