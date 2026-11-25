---
tier: epic
title:
  On-demand memory web rosters — read the decisions roster when planning, not on every
  turn
goal:
  "Any memory web can declare `roster_loading: on_demand`. Agent instruction bundles and
  the committed AGENTS.md then carry a short generated pointer instead of the full
  roster. The pointer names the exact audited command and lists who MUST run it:
  planners and researchers, plus anyone changing governed behavior without a plan. The
  command is `sase memory read roster:<web>`, it batches with strand reads, and the TUI
  shows it in the agent MEMORY lane and on the Memory panel's web card. A bare read of
  an on-demand web can no longer dump every strand body by accident. The decisions webs
  in sase and bob-cli adopt it, so the decisions section drops from about 8.4 KB to
  about 0.8 KB in sase and from about 25.6 KB to about 1 KB in bob-cli. bob-cli's Codex
  bundle goes from over its cap to roughly 19 KB of headroom."
decisions:
  sase_decisions_web:
    ask:
      Should sase's decisions web load on demand, add a rosters-load-on-demand record,
      and mark the old record superseded?
    memory:
      - decisions
      - decisions:rosters-load-on-demand
      - decisions:webs-render-in-their-own-section
    default: true
    requested:
      As a part of this change, we should update the decisions memories appropriately
      for both of these projects (sase and bob-cli).
    answer: true
  bob_cli_decisions:
    ask:
      Should bob-cli's decisions web load on demand, with a rewritten preamble and a
      planning-time policy record?
    default: true
    why:
      Requested for both projects; bob-cli's Codex bundle is already 4.5 KB over its cap
    answer: true
  glossary_memory_web:
    ask:
      Should the memory-web glossary strand stop saying every descriptor carries its
      roster into instructions?
    memory:
      - glossary:memory-web
    default: true
    corrects:
      so its body always renders in the generated document's Memory Webs section and
      carries a roster naming every strand
    answer: true
  bob_cli_gotchas:
    ask:
      Should bob-cli get a tiny core gotchas note so its release-train rule stays
      visible on every turn?
    default: false
    why:
      New always-loaded guidance the prompt did not ask for; turn it on to keep that
      rule ambient
    answer: true
phases:
  - id: roster-axis
    title: The roster_loading axis and the generated on-demand pointer
    depends_on: []
    size: medium
    description:
      "roster-axis: parse and validate `roster_loading: always | on_demand` and
      `roster_read_when:` on web descriptors, reserve the `roster` web slug, and render
      on-demand webs into AGENTS.md and instruction bundles with a generated pointer in
      place of the managed roster. The on-disk descriptor keeps its full roster. Also
      remove on-demand webs from the compaction ladder, update the Memory Webs intro and
      SASE Memory template text, add a Roster column to `sase memory web list`, and
      update the reference docs."
  - id: roster-read
    title: The audited roster:<web> read, the bare-web guard, and skill text
    depends_on:
      - roster-axis
    size: medium
    description:
      "roster-read: add the `roster:<web>` selector to `sase memory read` and `sase
      memory show`. It renders in Markdown, rich, and JSON, and each read records one
      schema-v3 audit event before printing. Add the CLI-only bare-web guard with
      `-a/--all-strands`. Update the descriptor-read error text and the report's
      reproduced command. Update the `/sase_memory_read` skill source and add one
      project-agnostic sentence to `/sase_plan`. Update docs and regenerate CLI help."
  - id: tui-surfaces
    title: TUI surfaces for on-demand rosters and roster reads
    depends_on:
      - roster-axis
      - roster-read
    size: small
    description:
      "tui-surfaces: show roster reads correctly in the agent MEMORY lane, including the
      report hint and the version chip. Give the Memory panel's on-demand webs an ON
      DEMAND badge, a `Roster` row, a `Read when` row, a `Last roster read` row, and a
      rail-row suffix. Update targeted visual goldens if they cover these surfaces."
  - id: adopt-sase
    title: sase's decisions web adopts the on-demand roster
    depends_on:
      - roster-axis
      - roster-read
    size: small
    description:
      "adopt-sase: flip `sase/memory/decisions.md` to on-demand with its read triggers.
      Add the `rosters-load-on-demand` record, mark `webs-render-in-their-own-section`
      superseded in part, and correct the memory-web glossary strand, each as its memory
      decision allows. Run `sase memory init` and `just fix`, verify the bundle shrank,
      and deploy the updated skills from the landed tree."
  - id: adopt-bob-cli
    title: bob-cli's decisions web adopts the on-demand roster
    depends_on:
      - roster-axis
      - roster-read
    size: small
    description:
      "adopt-bob-cli: in the bob-cli checkout opened with `/sase_repo`, flip
      `sase/memory/decisions.md` to on-demand. Rewrite its preamble around planning-time
      reads and the governed domains, and add the policy record and the gotchas note as
      the memory decisions allow. Regenerate AGENTS.md/CLAUDE.md with a sase build that
      understands the new key, and verify the Codex bundle fits with no compaction."
proposed_by: bbugyi200.athena.1g7
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 15:44:35
status: wip
---

- **PROMPT:**
  [prompts/202611/on_demand_web_rosters.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/on_demand_web_rosters.md)

# On-demand memory web rosters

## Why

Every SASE agent turn pays for the whole `decisions` roster, even though most agents
never read a decision record. The consolidated research
(`research:202611/on_demand_web_rosters/on_demand_web_rosters.md`) measured this at
`df9ce4ef55`:

| Measure                                 | sase                   | bob-cli                                     |
| --------------------------------------- | ---------------------- | ------------------------------------------- |
| Decision strands                        | 31                     | 77 (about 30 new each month)                |
| `proj.web.decisions` in the root bundle | 8,408 B (~2.1k tokens) | 25,641 B (~6.4k tokens)                     |
| Share of the root bundle                | 38%                    | 69%                                         |
| Codex bundle against its 32,768 B cap   | fits                   | **4.5 KB over**; compaction runs every turn |

Researchers, not planners, are the heaviest readers of decision records (78% of decision
reads in sase). The bare read `sase memory read decisions` prints every strand body
(about 69 KB in sase, 142–205 KB in bob-cli). Agents have already run it by accident
while looking for an index.

This epic adds one generic mechanism: any web can opt into on-demand roster loading. No
code may special-case `decisions`. Both projects' decisions webs then adopt it.

## Governing decisions

Each phase works within these records. Read the ones that govern your phase with
`sase memory read decisions:<keyword> -r "<why>"`.

- `decisions:webs-render-in-their-own-section`: its reopen clause asks for exactly this.
  It wants "a new, explicitly scoped opt-out mechanism designed for that purpose — not a
  revival of `type:` on descriptors". The new axis is that mechanism, and the
  descriptor's authored body still renders in the Memory Webs section.
- `decisions:readable-agents-md`: the committed AGENTS.md is "the full runtime
  projection". When bundles carry the pointer, AGENTS.md carries the same pointer. The
  on-disk descriptor keeps the full roster for human readers.
- `decisions:instructions-per-invocation`: bundle composition stays in Python. A
  per-invocation bundle is frozen at spawn, so the host cannot know whether an agent
  will plan. That is why the pointer is standing text and not an audience-specific
  bundle.
- `decisions:corpus-before-mechanism`: two real catalogs (31 and 77 records) that a bare
  read cannot serve justify the mechanism. Nothing speculative ships in v1 (see
  Deferred).
- `decisions:rust-core-required`: the design needs **no sase-core change**. A roster
  read is a schema-v3 `kind="web"` event, which the Rust audit reader (`audit_query.rs`)
  already accepts. Do not add a new audit kind or schema version; the reader silently
  drops unknown ones.
- `decisions:check-full-is-explicit` and `decisions:guarded-recipes`: verify with
  `sase tool run check`, never `just check-full`.

No feature flag. `roster_loading` is a permanent per-web config field that users choose,
and the bare-web guard only affects webs that opt in. Rollback is reverting the two
descriptors' frontmatter (see `sase_flags.md`).

## Design at a glance

### 1. Descriptor frontmatter: the axis and its readers

```yaml
web: true
description: >-
  Architectural decision records — accepted choices, their rejected alternatives, and
  what would reopen them.
roster: list
roster_label: DECISIONS
strand_noun: decision
roster_loading: on_demand # new: always (default) | on_demand
roster_read_when: # new: when agents MUST read the roster
  - before writing or revising a SASE plan (any agent using `/sase_plan`); cite each
    decision that governs the plan as `decisions:<keyword>`
  - when performing research, to find decisions related to the question before
    recommending anything
  - before changing behavior a decision may govern when no plan already cites it
```

`roster_loading` is a loading axis. It is independent of the existing
`roster: inline | list` style axis. `roster_read_when` is the lead's refinement of the
research spec. Each web declares its own read triggers as data, and SASE renders them
into the pointer. This keeps the mechanism generic: a future on-demand web, such as a
runbook catalog, can name different triggers without touching code. The pointer also
always has the same shape and an exact, copy-pasteable command. The research-agent
requirement from the prompt is the second trigger in both projects' descriptors.

### 2. The generated pointer (what every agent sees)

For an on-demand web, the instruction projection (bundle **and** committed AGENTS.md)
replaces the whole managed roster region with this generated pointer, wrapped to the
Markdown print width:

```markdown
**Roster on demand.** This web's 31 decisions are not listed here. You MUST read the
DECISIONS roster with `sase memory read roster:decisions -r "<why>"`:

- before writing or revising a SASE plan (any agent using `/sase_plan`); cite each
  decision that governs the plan as `decisions:<keyword>`
- when performing research, to find decisions related to the question before
  recommending anything
- before changing behavior a decision may govern when no plan already cites it

Read one decision with `sase memory read decisions:<keyword> -r "<why>"`.
```

With no `roster_read_when`, the middle paragraph becomes: "Read the DECISIONS roster
with `sase memory read roster:decisions -r "<why>"` when you need one." The bullet list
is then omitted. The count uses `strand_noun`, singular for one (`1 decision`,
`0 decisions`). The label is `roster_label`. The slug is the web slug. No string in the
code mentions `decisions`. The authored preamble above the region still renders
unchanged.

### 3. The audited roster read

```bash
sase memory read roster:decisions -r "Planning X; need governing decisions"
sase memory read roster:decisions decisions:rust-core-required -r "…"   # batches
sase memory show roster:decisions                                         # unaudited twin
```

Markdown output (the default; agents see this):

```text

---------- MEMORY ROSTER: decisions

*31 decisions — Architectural decision records — accepted choices, their rejected
alternatives, and what would reopen them.*

1. **A Gate Never Blocks An Agent** (`gates-never-block`)
   - _[partly superseded by `handoffs-end-through-host-completion`]_ Creating a gate …
…

Read one decision with `sase memory read decisions:<keyword> -r "<why>"`.
```

The roster body is byte-for-byte `render_strand_roster(web)` for the live,
project-over-home merged web. It includes summaries and supersession markers, and it
never uses the possibly stale on-disk region. Strand bodies and link expansion never
appear. Loading is a prompt policy, not an access rule, so `roster:<web>` works for
`always` webs too (`roster:glossary` prints the inline glossary roster).

### 4. The bare-web guard

`sase memory read decisions` (a bare web selector) on an on-demand web without
`-a/--all-strands` is refused before anything is printed or logged:

```text
sase memory read: decisions loads its roster on demand, so a bare `decisions` read would print all 31 decision bodies.
  roster:     sase memory read roster:decisions -r "<why>"
  one entry:  sase memory read decisions:<keyword> -r "<why>"
  every body: add -a/--all-strands
```

`always` webs such as `glossary` are unaffected. The guard lives **only** in the `read`
and `show` CLI resolution path. `resolve_memory_selector_batch` stays permissive by
default because three non-CLI callers must keep resolving bare webs:

- plan memory decisions (`src/sase/sdd/_plan_decisions_shared.py`); plans routinely
  declare `memory: [decisions, …]`
- TUI memory-read reports re-resolving a historical bare read
  (`src/sase/memory/memory_read_report.py`)
- the Memory panel's audited strand preview
  (`src/sase/ace/tui/modals/memory_panel_load.py`)

### 5. In the TUI

- **Agent MEMORY lane:** a roster read is one row, `roster:decisions`, with its reason
  and a numbered hint. The hint opens the generated memory-read report, which re-renders
  the roster section. A version chip pins the descriptor blob the roster was rendered
  from (see the audit contract).
- **Memory panel:** the web rail row reads `decisions  31 decisions · on demand`. The
  web card shows an `ON DEMAND` badge and these rows: `Roster: list · on demand`,
  `Read when` (one `•` line per trigger), and
  `Last roster read: <agent> · <reason> · <age>`.

## Cross-phase contracts

Later phases rely on these names and strings. Do not rename them without updating every
phase that uses them.

- **Model:** `WebRosterLoading = Literal["always", "on_demand"]` in
  `src/sase/memory/web/models.py`. `MemoryWeb` gains
  `roster_loading: WebRosterLoading = "always"` and
  `roster_read_when: tuple[str, ...] = ()`, each item whitespace-normalized. Generated
  webs (`task_types`) keep the defaults.
- **Parse errors** (`src/sase/memory/web/frontmatter.py`, fail closed like the existing
  `roster must be inline or list`):
  - `roster_loading must be always or on_demand`
  - `roster_read_when must be a list of non-empty strings`
- **Validation** (`src/sase/memory/web/validation.py`):
  - Blockers:
    - an on-demand web with no `description`
    - an on-demand web whose body is empty outside the managed region (the pointer needs
      authored prose to anchor to)
  - Warnings:
    - on-demand with no `roster_read_when` (agents will not know when to read it)
    - `roster_read_when` on an `always` web (it has no effect)
    - `roster: inline` with `on_demand` (the pointer can be larger than the inline
      roster it replaces)
  - `roster` joins `_STATIC_RESERVED_WEB_NAMES`.
- **Projection helper:**
  `render_web_projection_body(web, *, compaction=None) -> tuple[str | None, str | None]`
  in `src/sase/memory/web/roster.py`. For `always` webs it returns exactly today's
  `strip_managed_roster_markers(render_web_body_with_roster(...))` bytes. For
  `on_demand` webs it replaces the whole marker-delimited region with
  `render_on_demand_roster_pointer(web)`, or appends the pointer where the roster would
  go if the markers are absent. Note that `strip_managed_roster_markers` removes only
  the marker lines, not the roster. Descriptor writers
  (`render_web_descriptor_with_roster`) are unchanged, so the on-disk region stays full
  and `sase memory init` stays idempotent.
- **Selector:** `RosterSelector(raw, web_slug)` in `src/sase/memory/selector_models.py`.
  `classify_selector` matches the `roster:` prefix before the strand branch. `roster:`
  with an empty web part is `invalid_syntax`, and an unknown web fails the whole batch
  closed, like an unknown strand web. The batch model gains a `rosters` tuple and
  `has_roster_selector`. A roster-only batch has `kind == "web"`. Batch units gain
  `kind="roster"`. Duplicate roster selectors collapse to one section.
- **Audit event** (schema v3, no sase-core change): `kind="web"`. `selectors` holds the
  raw selectors, as today. `resolved_targets` gains `roster:<slug>`, so a roster-only
  read has `canonical_path == "roster:decisions"`. `included_targets` gains nothing.
  `byte_count` adds the UTF-8 bytes of the rendered roster body. For a file-backed web,
  `included_blob_oids` gains one pair: the descriptor's history target and the git blob
  OID of the descriptor's raw bytes. That blob carries the full managed roster on disk,
  so the existing version-chip and pager machinery can pin it. Use the target form that
  `core_selector_for_target` translates to the descriptor's file history. Never key a
  blob pair as `roster:<slug>`, because `version_for_blob` would then report a bogus
  "uncommitted" chip. Generated webs add no blob pair. The event is appended before
  anything prints, exactly like other reads.
- **Guard:** `MemorySelectorError` with reason `on_demand_web`, raised from the
  `read`/`show` CLI path when `-a/--all-strands` is absent. For `show`, the hint
  commands say `sase memory show` and carry no `-r`.
- **Pointer and footer wording:** sections 2 and 3 above are the contract. Tests assert
  them exactly.

## Phase `roster-axis` — The roster_loading axis and the generated on-demand pointer

Implement the model, parse, validation, reserved-slug, and projection contracts above.

1. `src/sase/memory/web/models.py` and `frontmatter.py`: add the two fields and parse
   them next to `roster` in `parse_web_descriptor`. Unknown frontmatter keys stay
   ignored, so an older sase reads an on-demand descriptor as `always`; that is the
   intended version-skew behavior.
2. `src/sase/memory/web/validation.py`: add the blockers and warnings, and reserve
   `roster`.
3. `src/sase/memory/web/roster.py`: add `render_on_demand_roster_pointer(web)` and
   `render_web_projection_body(...)`, and export them from `sase.memory.web`.
4. `src/sase/main/init_memory/root_planning.py` `memory_web_root_plan`: build
   `web_note_bodies` from `render_web_projection_body(web)`. This single change feeds
   both the committed AGENTS.md (`sase memory init`) and per-invocation bundles
   (`src/sase/instructions/compile.py`), and helper bundles inherit it.
5. `src/sase/instructions/compaction.py` and `compile.py`: `_web_section_text` renders
   through `render_web_projection_body`. `CompactableWeb` candidates are only
   `roster == "list"` **and** `roster_loading == "always"`. On-demand webs have nothing
   left to compact, and the ladder still runs on the remaining list webs. Do not touch
   the instruction-manifest wire; the omission is authored content, not a compaction
   step.
6. Intro text: `_WEB_MEMORY_INTRO` is defined in both `src/sase/amd/memory_units.py` and
   `src/sase/amd/_memory.py`. Update whichever copies are live and leave the first
   sentence ("Each memory web below is a keyed collection.") byte-identical, because
   `_WEB_MEMORY_INTRO_FIRST_SENTENCE` detects the section with it. The second sentence
   should say that a descriptor is always loaded with its roster unless the descriptor
   says the roster loads on demand, and that such rosters are read as `roster:<web>`.
   Keep it short.
7. `src/sase/main/init_memory/templates/memory-sase.template.md` (the generated SASE
   Memory core text, every project): reword the "Memory webs" bullet. A descriptor is
   always inlined, its roster is too unless the web loads its roster on demand (read
   with `sase memory read roster:<web>`), and a strand body never is. This regenerates
   `sase/memory/sase.md` and AGENTS.md. `sase/memory/sase.md` is generated, so change
   only the template, never the note.
8. `sase memory web list`: replace the stale "rendering (core/reference)" description in
   `src/sase/main/parser_memory.py`. Add a `Roster` column (`list`, `inline`,
   `list · on demand`) to the table. Add `roster` and `roster_loading` to its JSON
   output, with existing keys unchanged.
9. Docs, routed per `docs_writing.md`:
   - `docs/reference/memory/memory.md`: the two keys, the pointer, and the
     on-disk-versus-projection split.
   - `docs/reference/agents/memory-initialization.md`: it currently documents the inline
     behavior verbatim.
   - `docs/reference/memory/instruction-bundles.md` and
     `docs/contribute/instruction-bundle-internals.md`: standing omission versus cap
     compaction.
10. Run `sase memory init` so the sase repo's own AGENTS.md/CLAUDE.md pick up the new
    intro and template text. No web in this repo is on-demand yet.

**Tests.** Use an arbitrarily named fixture web, never `decisions`, to prove the
mechanism is generic:

- accept and reject values for both keys; each validation blocker and warning; the
  reserved slug
- the projection: the bundle and AGENTS.md contain the preamble plus the pointer and
  zero roster entries, while the on-disk descriptor keeps its full region
- the marker-absent case
- `init` is idempotent, and `init --check` is clean
- an absent key and an explicit `always` reproduce today's output byte-for-byte
- the compaction ladder skips on-demand webs but still compacts a co-resident `always`
  list web
- the pointer's exact text, singular and plural, with and without triggers

## Phase `roster-read` — The audited roster:<web> read, the bare-web guard, and skill text

Implement the selector, audit, and guard contracts above.

1. Selector and resolution:
   - `src/sase/memory/selector_models.py`, `selector.py`, and `selector_web.py`:
     classify, resolve through the same project-over-home scoped web discovery that
     strand reads use, and produce roster sections and render units.
   - `src/sase/memory/selector_postprocess.py`: roster units keep the user's selector
     order.
2. Rendering (`src/sase/memory/selector_render.py`):
   - **Markdown:** the `---------- MEMORY ROSTER: <slug>` header via
     `_markdown_section_header` (extend its kind literal), then the italic
     count-and-description line, then `render_strand_roster(web)`, then the footer from
     section 3.
   - **Rich:** a header grid matching `_web_header`, with `MEMORY ROSTER` and the slug
     on the left and `31 decisions · on demand` (or `· always`) dim on the right; the
     dim description; the roster rendered as Markdown; the dim footer.
   - **JSON:** a new top-level `rosters` list. Each entry has `web`, `label`,
     `strand_noun`, `count`, `roster`, `roster_loading`, `read_when`, and `entries`
     (`number`, `keyword`, `slug`, `summary`, `supersession`), plus `markdown` (the
     exact roster body). Existing keys are unchanged.
   - `memory_selector_batch_file_paths` returns the descriptor path for a roster
     section.
3. Audit (`src/sase/memory/cli_read.py` `build_memory_read_event_for_view` /
   `_batch_targets` / `_batch_blob_oids`): emit the event in the contract shape. Add a
   test that round-trips a roster event through the Rust-backed `query_memory_audit`
   (`src/sase/memory/_read_log_query_facade.py`) and asserts that `kind`, `selectors`,
   `resolved_targets`, and `included_blob_oids` survive.
4. Guard and flag:
   - Add `-a/--all-strands` to `_add_memory_view_arguments` in
     `src/sase/main/parser_memory.py`, keeping options sorted (`-a`, `-d`, `-f`, `-p`,
     `-r`). Help text: "Let a bare web selector print every strand body, even for a web
     whose roster loads on demand".
   - Enforce the guard in `resolve_memory_view` (`src/sase/memory/cli_show.py`) through
     an explicit keyword such as `refuse_on_demand_bare_webs=True` on
     `resolve_memory_selector_batch`, which defaults to `False`. That keeps the report,
     plan-decision, and Memory panel callers permissive.
   - Update the `selectors` help and epilog examples to include `roster:<web>`.
5. Error text: the descriptor-path refusal in `src/sase/memory/_read_log_paths.py` names
   `sase memory read roster:<slug>` alongside the strand form. Keep "always-loaded" only
   for `always` webs.
6. Reports (`src/sase/memory/memory_read_report.py`): roster sections render through the
   shared Markdown path. `_reproduced_command` adds `-a` whenever a selector is a bare
   web, so a historical bare read of a now-on-demand web still reproduces.
7. `sase memory web show`: its description points agents at
   `sase memory read roster:<web>` as the audited catalog read, and its title line notes
   `roster on demand` for on-demand webs.
8. Skill sources (`generated_skills.md` applies; do not deploy from this phase):
   - `src/sase/macros/skills/sase_memory_read.md`:
     - the fourth selector shape, `roster:<web>` (titles and summaries, no bodies)
     - on-demand webs: what the pointer means, the guard, and `-a`
     - the batching tip `sase memory read roster:<web> <web>:<keyword> -r …`
     - fix the "always inlines" sentence so it matches the new axis
   - `src/sase/macros/skills/sase_plan.md` step 1: add one project-agnostic sentence
     after the `sase_sizes.md` command. If your instructions' Memory Webs section says a
     web's roster MUST be read before writing a plan, add those `roster:<web>` selectors
     to that same `sase memory read` command, and follow any citation rule the pointer
     states. The sentence must never name `decisions`. For projects without on-demand
     webs it is a no-op. This is the requirement adjustment the research recommended and
     the user accepted.
   - Preview with `sase skill init --diff`.
9. Docs: the read and guard sections of `docs/reference/memory/memory.md`. Regenerate
   CLI help with `just fix`.

**Tests:**

- classification of `roster:decisions` versus `glossary:stitch`, a bare web, `.md`, and
  `roster:`
- Markdown output equals header plus intro plus `render_strand_roster` plus footer, with
  no strand bodies and no `## Linked References`
- an empty web gives a labeled empty roster, not an unknown-web error
- scope merging matches strand reads
- a mixed batch (`roster:` plus strand plus note) gives one event in selector order
- `show` logs nothing
- the guard fires only for on-demand webs; `-a` bypasses it; nothing is printed or
  logged when it fires
- a plan memory decision naming a bare on-demand web still validates
- the report re-resolves a bare on-demand read
- the JSON shape

## Phase `tui-surfaces` — TUI surfaces for on-demand rosters and roster reads

Read `tui.md` and `tui_perf.md` first. Every new value comes from data the snapshot or
the event already loads on worker threads. Add no new I/O on the UI thread.

1. Agent MEMORY lane (`src/sase/ace/tui/widgets/prompt_panel/_agent_memory_reads.py`,
   `_agent_memory_versions.py`):
   - Confirm a roster-only event renders as `roster:decisions` with its reason and a
     report hint. `_display_selector` already joins selectors when `resolved_path` is
     empty.
   - Confirm that opening the report writes no audit event.
   - Confirm `resolve_event_chips` yields a real version chip from the descriptor blob
     pair, never "uncommitted" for an unchanged committed descriptor.
   - Mixed batches keep today's aggregate chip.
2. Memory panel:
   - `src/sase/ace/tui/modals/memory_panel_rendering_rail.py`: append `· on demand` dim
     to an on-demand web's rail row.
   - `src/sase/ace/tui/modals/memory_panel_web_rendering.py`: add an `ON DEMAND` badge
     after `WEB`, the `Roster` row (`list · on demand`), a `Read when` row (one `•` line
     per trigger, omitted when empty), and `Last roster read` from
     `snapshot.read_summaries.get(f"roster:{web.slug}")`. That key works because
     `summarize_memory_reads_by_path` groups by `resolved_targets`.
3. Docs: if a `docs/reference/tui/` page documents the Memory panel card rows or the
   MEMORY lane, add the new rows in a sentence or two.
4. Visual snapshots: if PNG goldens cover the Memory panel web card or rail, run a
   targeted `just fix-tui-screenshots -- <selectors>` and inspect every changed golden
   as `lint_and_test.md` requires. If none cover them, add a focused text-render test
   instead of a new golden.

**Tests:** the lane row text and hint registration for a roster event; the chip
behavior; the card badge and rows for on-demand and `always` webs; and the `always` card
unchanged byte-for-byte.

## Phase `adopt-sase` — sase's decisions web adopts the on-demand roster

Use `/sase_memory_write` and stay inside the accepted memory decisions. Then:

1. `sase/memory/decisions.md` (`sase_decisions_web`):
   - Add `roster_loading: on_demand` and the three `roster_read_when` triggers from
     design section 1.
   - Keep the managed region.
   - Rewrite only the preamble's last sentence, so it no longer duplicates the pointer's
     strand-read line and stays projection-neutral: never "below" or "omitted here",
     because the descriptor renders both on disk and in projections.
2. New strand `sase/memory/decisions/rosters-load-on-demand.md` (`sase_decisions_web`):
   - Keyword "Web Rosters Can Load On Demand", plus a summary.
   - **Claim:** the axis, the generated pointer with web-declared triggers, the
     `roster:<web>` read, and the CLI-only bare-web guard. The descriptor preamble stays
     always loaded, AGENTS.md mirrors the bundle, and the on-disk descriptor keeps the
     full roster.
   - **Why:** the measurements, plus the fact that frozen per-invocation bundles make
     planner-scoped compilation impossible.
   - **Rejected alternatives:** the research table (rely on compaction, a titles-only
     roster, reviving `type:`, `when:` audiences, a flat reference note, semantic
     search), plus the four rejected command shapes.
   - **Cost:**
     - ambient discovery is lost for unplanned work
     - compliance is social in v1
     - `roster` is a reserved web slug
     - one more projection divergence
   - **Reopens when:**
     - planner compliance measured from `memory_reads.jsonl` stays poor; then add the
       deferred `sase plan validate` check or a titles-only `compact` mode
     - the host gains a reliable fact at spawn that an agent will plan
   - Cite the research ref as evidence, and link
     `[[decisions/webs-render-in-their-own-section]]`.
3. `sase/memory/decisions/webs-render-in-their-own-section.md` (`sase_decisions_web`):
   set `metadata.status: superseded-in-part` and `superseded_by` to the new record. Add
   a back-link blockquote that states what is retired: the Cost sentence "there is no
   longer a way to make one read-on-demand", for rosters only. Edit nothing else.
4. `sase/memory/glossary/memory-web.md` (`glossary_memory_web`): the descriptor's
   authored body always renders in the Memory Webs section; its roster renders there
   only when `roster_loading` is `always`, the default; otherwise a generated pointer
   names `sase memory read roster:<web>`.
5. Run `sase memory init`, then `just fix`. That regenerates the design-decisions and
   glossary docs pages and CLI help.
6. Verify:
   - AGENTS.md and CLAUDE.md show the pointer under `### Decisions (decisions)` with
     zero roster entries.
   - `sase instructions render` shows `proj.web.decisions` under about 1 KB and no
     longer the largest section.
   - `sase memory read roster:decisions -r …` from this workspace prints all strands.
   - The Codex and other provider size or parity tests pass. Update any test that pinned
     the old section sizes, and say so.
7. Deploy skills per `generated_skills.md`: `roster-read`'s skill sources must already
   be on the canonical branch. Run `sase skill init --diff`, then
   `sase skill init --force` from the clean tree (`chezmoi apply` if it was skipped). If
   the provenance or source guard refuses, do not bypass it. Record a
   `PROPOSED FOLLOW-UP:` naming the deploy command.

If a memory decision above is off, skip that step. Treat a declined change as
`sase_memory_write.md` directs.

## Phase `adopt-bob-cli` — bob-cli's decisions web adopts the on-demand roster

1. Open the repo with `/sase_repo`
   (`sase repo open bob-cli -r "Adopt on-demand decisions roster"`). Read its AGENTS.md
   and follow its conventions and verification recipe.
2. Pick the sase build. The installed `sase` is an editable install of the primary sase
   checkout. If `sase memory read roster:decisions -r …` succeeds from the sase
   workspace, the installed build is current; use it. Otherwise run the workspace build
   (`<sase workspace>/.venv/bin/sase`, after `just install-venv` if needed) for every
   `sase memory init` and `sase instructions render` in bob-cli, and say so in your
   final summary. An older sase ignores the new keys and would silently re-expand the
   roster.
3. bob-cli `sase/memory/decisions.md` (`bob_cli_decisions`):
   - Add `roster_loading: on_demand` and `roster_read_when` triggers:
     - before writing or revising a SASE plan (any agent using `/sase_plan`); cite each
       decision that governs the plan as `decisions:<keyword>`
     - when performing research, to find decisions related to the question before
       recommending anything
     - before changing governed behavior when no plan cites the decision: capture,
       walk/review, tasks/lanes/decay, gcal/meetings, refs, themes/relations, releases,
       or Mac Capture
   - Rewrite the preamble so it no longer claims "Each roster summary is a rule to
     follow as written". Records are rules once read, planners cite them, and direct
     changes in the governed domains consult the roster first. Keep the immutability,
     evidence, and "ask Bryan" rules.
4. New bob-cli record (`bob_cli_decisions`): for example
   `decision-rules-are-read-at-planning-time`. Follow bob-cli's record shape ("where it
   applies", claim, why, cost, reopen). Cite the research ref as evidence. Also cite
   sase's `rosters-load-on-demand` record if `adopt-sase` has already landed; the two
   phases run in parallel. Reopens when unplanned changes measurably contradict records.
5. If `bob_cli_gotchas` is on: add a tiny `type: core` note named `gotchas` to bob-cli's
   memory root (about 80 tokens), carrying only the release-train rule, and link the
   governing record.
6. Run `sase memory init` with the chosen build. Verify:
   - AGENTS.md and CLAUDE.md carry the pointer and no roster entries.
   - `sase instructions render` for bob-cli: the decisions section is about 1 KB, and
     the Codex bundle fits its 32,768 B cap with no `bundle.compaction` step. Report the
     before and after bytes.
   - bob-cli's own check recipe passes.

## Deferred (not in this epic; each has a trigger)

- A `sase plan validate` check that a required roster was read in the same run, with a
  `roster_required_for:` vocabulary: when measured planner compliance falls below ~90%.
- First-class `kind="roster"` audit events (schema v4, sase-core first): when
  roster-typed queries are wanted.
- `roster_loading: compact` (a standing titles-only rung): when on-demand compliance is
  poor in bob-cli but full omission is too aggressive.
- Audited pattern-filtered roster reads: when governed-domain coders demonstrably need
  them.
- Stripping the on-disk managed region: when descriptor merge conflicts measurably slow
  bob-cli landings.
- Research-macro opt-in (`#research*` prompts) in `sase-research-artifacts`: when
  researcher roster reads drop after rollout. The pointer's research trigger covers
  research agents in v1.

## Verification (every code phase)

- `sase tool run check` in the sase workspace (run `just fix` first). Never run
  `just check-full` unless explicitly told.
- Phases that change generated instructions run `sase memory init` and commit the
  regenerated AGENTS.md and CLAUDE.md. CI's `just validate` drift gate uses the
  workspace code.
- Post-rollout (no phase owns this): watch `memory_reads.jsonl` for planner runs with no
  `roster:` read before `sase plan propose`. That is the evidence trigger for the first
  deferred item.
