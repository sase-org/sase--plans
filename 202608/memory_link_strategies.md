---
status: done
tier: epic
title: Memory link reference and rendering strategies
goal: "Memory notes, web descriptors, and strands declare how links to other memory
  files are detected and rendered, `[[target]]` / `![[target]]` links resolve and render
  in `sase memory show`/`read`, and the existing corpus links itself.

  "
phases:
  - id: schema
    title: Link strategy frontmatter
    depends_on: []
    size: medium
    description:
      "schema: add `link_reference` and `link_rendering` frontmatter to flat notes, web
      descriptors, and strands, with strand-over-web-over-default precedence,
      validation, and `closure:` accepted as a legacy alias."
  - id: scan
    title: Link scanner and target resolver
    depends_on: []
    size: medium
    description:
      "scan: add the `[[target]]` / `![[target]]` body scanner that skips code zones,
      plus a resolver that maps a raw target onto a flat note, web descriptor, or
      strand."
  - id: closure
    title: Links in the closure walk
    depends_on:
      - schema
      - scan
    size: medium
    description:
      "closure: classify each detected link as an inline or reference edge from the
      effective strategies, feed inline edges into the existing BFS through
      `precomputed_spans`, add cross-unit inline targets as extra roots, and make ACE
      strand relations follow the same rules."
  - id: render
    title: Linked References output
    depends_on:
      - closure
    size: medium
    description:
      "render: emit a numbered `## Linked References` section for every note and web
      unit across the markdown, rich, and json formats, and carry link data in the JSON
      payloads."
  - id: migrate
    title: Declare existing web strategies
    depends_on:
      - render
    size: small
    description:
      "migrate: state `glossary`'s implicit/inline strategies explicitly, drop the
      legacy `closure:` key from in-repo descriptors, and report unresolved links as
      doctor and init warnings."
  - id: taskgen
    title: Generated task-type strand links
    depends_on:
      - render
    size: small
    description:
      "taskgen: make the generated task-type strands emit a Related Task Types section
      linking every other catalog type their own prose names."
  - id: content
    title: Link the existing corpus
    depends_on:
      - migrate
    size: medium
    description:
      'content: add links across the hand-authored notes and strands that already
      cross-reference each other in prose, and record the decision that supersedes the
      "not a new, parallel link syntax" clause of the memory-webs record.'
  - id: docs
    title: Skill and documentation updates
    depends_on:
      - migrate
    size: small
    description:
      "docs: teach `/sase_memory_write` to author links, and document the two strategies
      in `docs/memory.md` and the generated memory README."
proposed_by: bbugyi200.athena.sase-vk.land.w1.w0
bead_id: sase-vw
create_time: 2026-09-09 19:50:51
---

- **PROMPT:**
  [prompts/202608/memory_link_strategies.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/memory_link_strategies.md)
- **BEAD:**
  [sase-vw](https://github.com/sase-org/sase--beads/blob/main/pages/sase-vw/README.md)

# Plan: Memory link reference and rendering strategies

## Problem

`sase/memory/decisions/gates-never-block.md` already contains
`![[decisions/single-turn-agents]]`, and `sase memory show decisions:gates-never-block`
prints that token verbatim: nothing detects it, resolves it, or renders the linked
strand. The only working link mechanism today is the `glossary` web's
`closure: mentions` phrase matcher, which is implicit, un-authorable per link, and
hard-wired to inline rendering. Every other memory file — flat reference notes, the
`decisions` web, the generated `task_types` web — can only cross-reference in prose that
a reader has to follow by hand.

This epic introduces two frontmatter axes that every memory file can declare, an
authored `[[...]]` link syntax, and the rendering that makes both useful in
`sase memory show` and `sase memory read`.

## Decisions and assumptions

These are the choices a phase worker should implement rather than re-litigate. Each one
was a real fork; the reasoning is recorded here so phases stay consistent.

**This epic contradicts an accepted decision record, and phase `content` resolves
that.** `sase memory read decisions:memory-webs` says, under "Reopens when": _"A web's
strand count or supersession rate outgrows prose cross-references, at which point the
existing `supersedes` / `superseded-by` artifact relations are the adopted mechanism —
not a new, parallel link syntax."_ This epic builds exactly that parallel link syntax.
It also satisfies the reopen condition of `decisions:corpus-before-mechanism` — the
corpus (55+ strands across three webs plus 13 flat notes, with an already-authored
`![[...]]` link that does nothing) now exists and demonstrably needs a mechanism plain
audited reads cannot serve. The user asked for this feature directly, so the work
proceeds; phase `content` writes the superseding record rather than leaving the tree in
silent conflict with its own decisions web.

**Field names.** `link_reference: explicit | implicit | none` (default `explicit`) and
`link_rendering: reference | inline` (default `reference`). These mirror the user's own
words. The value `reference` on the rendering axis and the field name `link_reference`
on the detection axis are close together; keep the docs and help text explicit about
which is which rather than renaming either.

**Precedence.** Strand frontmatter wins over its web descriptor, which wins over the
built-in default. A flat note uses its own frontmatter or the default. Precedence is
resolved once, at parse time, into effective values carried on the model — no caller
should re-derive it.

**`explicit` links are always honored.** `link_reference: implicit` means _implicit
mentions in addition to_ authored links, not instead of them. `link_reference: none`
disables both, so `[[...]]` renders verbatim with no Linked References section — that is
the escape hatch for a note whose body legitimately discusses the syntax.

**Legacy `closure:` is an alias, not a flagged branch.** `closure: mentions` maps to
`link_reference: implicit`, `closure: none` maps to `link_reference: none`. Declaring
both `closure:` and `link_reference:` on one descriptor is a validation blocker. No
feature flag: this is a frontmatter alias in the same class as the still-accepted
`type: short` / `type: long` note types, which never carried one. Home-scope and other
projects' descriptors keep working unchanged.

**Inline edges consume closure depth; reference edges terminate the walk.** `-d/--depth`
keeps its current meaning. `-d 0` prints only the requested units and lists every link
as a reference. This is what keeps output bounded when a whole web is requested.

**No `[[target|label]]` display-text form in v1.** Out of scope; add it only if authored
content asks for it.

**The scanner stays in Python.** `sase memory read decisions:rust-core-required` and
`sase/memory/rust_core_backend_boundary.md` put shared backend behavior in
`../sase-core`, and this is a judgment call worth naming: the entire memory-web
subsystem — discovery, frontmatter parsing, the closure BFS in
`src/sase/memory/web/resolution.py`, validation, roster rendering — already lives in
Python in this repo, and both consumers (the CLI and the ACE memory panel) share it
there. The code-zone primitive the scanner depends on is already Rust-owned and reached
through `sase.xprompt._literal_zones.code_literal_ranges`, so the boundary is respected
where it actually exists. Moving the memory-web domain layer into `sase-core` is a
separate migration, not this epic.

## Link strategy frontmatter

Add the two keys to all three parse sites and carry effective values on the models.

`src/sase/memory/web/models.py`

- Add `MemoryLinkReference = Literal["explicit", "implicit", "none"]` and
  `MemoryLinkRendering = Literal["reference", "inline"]`.
- Add `link_reference` and `link_rendering` fields to both `MemoryWeb` and
  `MemoryStrand`. On `MemoryStrand` these are the _effective_ values already resolved
  against the owning descriptor, so `closure.py` and the renderers never walk back up.
- Keep `WebClosureMode` and `MemoryWeb.closure` only if something still reads them after
  phase `closure` lands; prefer deleting both, since `symvision` will otherwise report
  them unused.

`src/sase/memory/web/frontmatter.py`

- Validate the two keys in `parse_web_descriptor` next to the existing `roster` and
  `closure` checks, with the same `(None, error)` failure shape:
  `link_reference must be explicit, implicit, or none` and
  `link_rendering must be reference or inline`.
- Accept legacy `closure:` and map it as described above. Error when a descriptor
  declares both `closure:` and `link_reference:`.
- Parse per-strand overrides in `parse_memory_strand`. `parse_memory_strand` does not
  currently receive the owning `MemoryWeb`, so either thread the descriptor's effective
  values in as keyword arguments from `discovery.py` and `generated.py`, or resolve
  precedence in one place immediately after strand parsing — pick one and use it for
  both file-backed and generated webs.
- `render_strand_frontmatter` must be able to emit the keys so ACE panel writes and
  generated strands round-trip.

`src/sase/memory/notes.py`

- Add `link_reference` and `link_rendering` to `MemoryNote` and to
  `_CANONICAL_FRONTMATTER_KEYS`, so `apply_memory_frontmatter` stops treating them as
  arbitrary passthrough extras.
- Invalid values fall back to the default rather than failing the parse; flat-note
  parsing is not fail-closed today and this epic should not change that. Report the
  invalid value through the validation surfaces in phase `migrate` instead.

`src/sase/memory/web/validation.py`

- Nothing new here yet beyond the parse errors surfacing through discovery issues;
  unresolved-link reporting arrives in phase `migrate`.

Tests: extend `tests/memory/test_memory_web.py` for descriptor and strand parsing,
precedence, both error messages, and the legacy alias in both directions; extend
`tests/test_memory_notes.py` for flat-note parsing and frontmatter round-tripping.

## Link scanner and target resolver

Two new focused modules under `src/sase/memory/`.

`src/sase/memory/links.py` — the scanner.

- `scan_memory_links(body: str) -> tuple[MemoryLink, ...]` where `MemoryLink` carries
  `raw`, `target`, `inline: bool`, and the character span.
- `![[target]]` sets `inline=True`; `[[target]]` sets `inline=False`.
- Skip every range returned by `sase.xprompt._literal_zones.code_literal_ranges(body)`
  so fenced blocks and inline code never produce links. This is not hypothetical:
  `sase/memory/xprompts.md` line 17 contains `` `[[ ... ]]` `` inside inline code and
  must yield no link.
- Scan the body only. Frontmatter is never scanned.
- Preserve document order and deduplicate by `(target, inline)`, keeping the first span.

`src/sase/memory/link_resolve.py` — the resolver.

Resolve one raw target against the scoped universe (the flat notes plus scope-merged
webs the caller already has), in this order:

1. `<web>:<keyword>` — reuse `resolve_memory_strand`, which already accepts slug,
   keyword, alias, and unambiguous prefix.
2. `<web>/<slug>` — the form already authored in
   `sase/memory/decisions/gates-never-block.md`.
3. `<note>.md` — a flat note.
4. Bare token — the source strand's own web first, then a flat note stem (`symvision` →
   `symvision.md`), then a web slug (`glossary` → that web's descriptor note).

Return a typed result that distinguishes a resolved flat note, a resolved strand, a
resolved descriptor, and an unresolved target carrying near-miss candidates. Drop
self-links. An unresolved target is never an exception on the read path: it is data the
renderer prints and the validators warn about.

Tests: a new `tests/memory/test_memory_links.py` covering each target form, the `!`
prefix, code-zone suppression (including the real `xprompts.md` case), self-link
dropping, dedupe, and unresolved targets with and without candidates.

## Links in the closure walk

`src/sase/memory/web/closure.py` is the integration point, and
`resolve_glossary_closure` already accepts
`precomputed_spans: Mapping[int, Sequence[GlossarySpan]]`. That is the injection point:
no Rust change and no change to the BFS itself.

- Classify each scanned link on a strand: `inline=True` (from `!`) or the strand's
  effective `link_rendering` decides. Inline links become synthetic `GlossarySpan`
  entries pointing at the target entry index; reference links are collected separately
  and returned alongside the closure.
- When the effective `link_reference` is `implicit`, keep compiling the phrase matcher
  as `build_strand_mention_catalog` does today for `closure == "mentions"`, and merge
  its spans with the synthetic link spans. When it is `explicit`, compile no matcher and
  pass only synthetic spans. When it is `none`, pass neither.
- Provenance: an inline link's node should carry a referrer that reads as a link, not a
  phrase mention, so `selector_render` can print `linked from <source>` instead of
  `mentioned as "<text>" in <term>`. Extend `_GlossaryReferrer` or the
  `MemoryWebReadNode.referrer` tuple rather than faking a matched phrase.

`src/sase/memory/selector.py`

- `_resolve_web_sections` grows a second pass: after each web's closure resolves,
  collect inline link targets that live outside that web (another web, or a flat note)
  and add them as extra roots to the owning unit, creating a section or note unit if the
  batch did not already have one. Mark them `origin="related"` with the source as
  referrer. Cross-web inline links must work — `decisions` strands linking `glossary`
  strands is the obvious near-term case.
- Collect each unit's _reference_ links onto the resolved batch so phase `render` has
  them: extend `MemoryWebReadSection` and `ResolvedMemoryNote` (or wrap the latter) with
  a resolved-links tuple.
- Flat notes get link handling too, not just strands: `_resolve_note_selector` scans the
  note body and resolves against the same universe.

`src/sase/ace/tui/memory_panel_catalog.py`

- Line 457 gates the phrase catalog on `web.closure == "mentions"`; switch it to the
  effective `link_reference`, and make `strand_mention_relations` include explicit link
  edges so the panel's relations match what the CLI prints.

Tests: extend `tests/memory/test_memory_selector.py` for edge classification, depth
interaction (`-d 0`, `-d 1`, truncation), cross-web inline roots, and
`link_reference: none`; extend `tests/ace/tui/test_memory_panel_catalog.py` and
`tests/ace/tui/modals/memory_panel_test_helpers.py` for the strategy rename.

## Linked References output

`src/sase/memory/render.py` (single-note path), `src/sase/memory/selector_render.py`
(batch and web path).

Markdown — appended at the bottom of each unit, after `## Children` when that section is
present:

```markdown
## Linked References

The below memory files are linked from this one. Read one with your `/sase_memory_read`
skill; do not open the file directly.

### 1. `decisions:two-speed-verification`

**Verification Is Two-Speed** — just check is the agent default and just check-full
gates landing, because host capacity is the constraint, not test speed.
```

- Numbered `###` subsections, taking their shape from
  `sase.memory.notes.render_long_memory_sections`, which renders the same idea for the
  Reference Memory section of `AGENTS.md`. The numbering is explicit here because
  `sase/amd/_section_numbers.py` does not run on this path.
- The heading is the selector an agent would pass to `sase memory read`. The body is the
  target's display label plus its `summary` (strand) or `description` (flat note).
- A target that is always-loaded context — a `type: core` note or a web descriptor —
  renders with a trailing marker such as
  `(always-loaded core memory — already in your context)` and no read suggestion,
  because `sase memory read` refuses those.
- Unresolved targets render last under an `Unresolved:` line listing the raw tokens.
- A web section emits one aggregated block covering every rendered node, deduped, and
  excluding any target already rendered inline in that same section.
- Emit nothing when a unit has no reference links.

Rich: a `Linked References` block built like the existing `_build_children_block`,
numbered, using `PATH_COLOR` for the selector and dim for the summary.

JSON: add `"linked_references"` to the note and web-section payloads, and a `"links"`
list to `_node_json` / `_note_json` with `target`, `address`, `kind`
(`"inline"`/`"reference"`), `resolved`, `label`, and `summary`. Keep the payload
sorted-key stable — these are golden-compared.

Two invariants to preserve: a single-note batch still renders byte-identically between
`show` and `read`, and `build_memory_read_event_for_view` keeps counting only bytes
actually printed as content — inline-expanded strands already count as nodes; reference
entries are listings, not reads, and must not inflate `byte_count`.

Tests: extend `tests/memory/test_memory_selector_render.py` for all three formats and
both unit kinds; extend `tests/main/test_memory_read_selectors.py` for the end-to-end
CLI output; add the concrete acceptance case —
`sase memory show decisions:gates-never-block` renders `decisions/single-turn-agents`
inline at the bottom, exactly as `sase memory show glossary:<term>` does today.

## Declare existing web strategies

Memory-file edits in this phase are authorized by this plan; use `/sase_memory_write`
and run `sase memory init` afterward.

- `sase/memory/glossary.md`: replace `closure: mentions` with `link_reference: implicit`
  and `link_rendering: inline`. Behavior must be unchanged — diff
  `sase memory show glossary -f json` before and after.
- `sase/memory/decisions.md`: drop `closure: none` so the defaults (`explicit` /
  `reference`) apply. This is the change that makes the existing
  `![[decisions/single-turn-agents]]` link resolve.
- `sase/memory/task_types.md`: leave it on the defaults; add the keys explicitly only if
  the generated descriptor template already spells out its other web keys.
- `src/sase/memory/web/validation.py`: report unresolved explicit links as
  `MemoryWebValidationReport.warnings`, not blockers — a home-scope or other-project
  note with a stale link must stay readable.
- `src/sase/doctor/checks_config_memory_webs.py`: surface those warnings, and add
  equivalent reporting for flat notes with unresolved links and for flat notes carrying
  an invalid `link_reference` / `link_rendering` value.

Tests: update every fixture that writes `closure:` —
`tests/memory/test_memory_selector.py`, `tests/memory/test_memory_selector_render.py`,
`tests/main/test_memory_read_selectors.py`, `tests/test_memory_read_report.py`,
`tests/ace/tui/test_memory_panel_catalog.py`,
`tests/ace/tui/modals/memory_panel_test_helpers.py` — keeping at least one fixture on
the legacy key to prove the alias still works. Add doctor coverage for the new warnings.

## Generated task-type strand links

`src/sase/main/init_memory/root_rendering_task_types.py` renders each strand body from
the live catalog. Today `bug.md` ends its "When To Use" with _"Do not use this for a
flake, a confirmed CI failure, or a GitHub-mirrored bug"_ and links to nothing; `ci.md`
says _"Use flake instead…"_; `flake.md` says _"Use ci instead…"_.

Add a `## Related Task Types` section to `_render_task_type_strand_body`, listing
`[[task_types/<slug>]]` for every other catalog type whose slug or label appears in this
type's own `summary`, `when_to_use`, or `create_refusal` text. Match case-insensitively
on whole words, exclude the type itself, and order by slug so the output is
deterministic.

Do not put link syntax into the task-type specs in `src/sase/task_types/_builtin.py`:
`when_to_use` is also printed by `sase bead task-type show`, the `/sase_new_task` gate
presentation, and the bead body templates, none of which understand `[[...]]`. Deriving
the links at strand-render time keeps the wiki syntax inside memory. The considered
alternative — a new `related:` key on the task-type spec — was rejected for this epic
because it changes the plugin-facing spec schema, the committed `sase/task_types.json`
snapshot, and every type's digest for a relationship the existing prose already states.

Because the strand bodies change, the digests and the generated
`sase/memory/task_types/` files change; run `sase memory init` and commit the
regenerated strands.

Tests: extend the task-type generation tests (`tests/memory/test_mutation_generated.py`
and the init-memory task-type tests) to assert the Related section content, its absence
when nothing matches, and determinism.

## Link the existing corpus

Memory-file edits here are authorized by this plan; route them through
`/sase_memory_write` and finish with `sase memory init`.

Add links where the corpus already cross-references itself in prose. Known cases found
while planning, and the audit should not stop at them:

- `sase/memory/lint_and_test.md` names `decisions:two-speed-verification` and
  `symvision.md` in prose — link both.
- `sase/memory/sase_beads.md` points at `task_types:<slug>` — link the `task_types` web.
- `sase/memory/decisions/memory-webs.md` names `corpus-before-mechanism` in prose and
  describes the same shape as `glossary:memory-web` and `glossary:memory-strand`.
- `sase/memory/decisions/host-owned-completion.md`,
  `sase/memory/decisions/gates-never-block.md`, and
  `sase/memory/decisions/single-turn-agents.md` form a cluster that currently links only
  one way.
- `sase/memory/sase.md` is generated — if it should link, change its template in
  `src/sase/main/init_memory/templates/memory-sase.template.md`, not the note.

Then write the decision record this epic owes, as
`sase/memory/decisions/memory-links-are-authored.md` (choose the final slug and keyword
when authoring):

- **Claim.** A memory file declares how its links are detected and rendered, and authors
  links inline as `[[target]]` / `![[target]]`.
- **Why.** Cover the alternatives honestly: staying on implicit phrase matching (works
  only for a glossary-shaped corpus, cannot be authored per link, cannot be scoped to
  one target), and the `supersedes` / `superseded-by` artifact relations that
  `decisions:memory-webs` named as the adopted mechanism (they are out-of-band typed
  relations between artifacts, recorded in an index rather than authored in the prose
  that motivates them, and cannot express "render this target's body at the bottom of
  this read").
- **Cost.** A second link vocabulary in the tree; unresolved links that only warn; two
  more frontmatter keys on every memory kind.
- **Reopens when.** State the condition honestly.
- Record in prose that this supersedes the "not a new, parallel link syntax" clause of
  `decisions:memory-webs` and that it satisfies the reopen condition of
  `decisions:corpus-before-mechanism`. Per the `decisions` web's own rule, records are
  immutable — write the supersession into the new record and leave both existing records
  untouched.

Optionally add a `glossary` strand for the new vocabulary (memory link, link reference
strategy, link rendering strategy) if the terms end up used across notes.

Verification for this phase is content-level: `sase doctor` reports no memory-web
blockers and no unresolved-link warnings, and `sase memory show` on each edited file
renders the intended Linked References or inline expansion.

## Skill and documentation updates

`src/sase/xprompts/skills/sase_memory_write.md` — the agent-facing contract. Add a short
section covering: the two frontmatter keys and their defaults; `[[target]]` for a listed
reference and `![[target]]` to force inline; the four target forms; that links are
ignored inside code fences and inline code; and the expectation that a new or edited
note links the memory it already names in prose. Keep it tight — this skill is read on
every memory write, and the file is currently 49 lines.

Note the skill-source contract from `sase memory read generated_skills.md`: edit only
the source template here, do not touch deployed chezmoi `SKILL.md` files, and leave
`sase skill init --force` for a clean, landed tree.

`docs/memory.md` — add a "Memory Links" section after "Memory Webs" describing both
axes, the syntax, resolution order, depth interaction, and the `Linked References`
output. Update the "Show a Note" section, which currently promises only frontmatter
stripping and a `## Children` section.

`src/sase/main/init_memory/templates/memory-README.template.md` — extend the
"Frontmatter Schema" list with the two keys and the "Linking" list with the `[[...]]`
forms, then regenerate `sase/memory/README.md` via `sase memory init`.

`src/sase/main/parser_memory.py` — the `read` and `show` descriptions mention
mention-closure behavior in the `-d/--depth` help; reword for links. No new CLI options:
per `sase memory read cli_rules.md`, options are optional modifiers, and the strategies
are file-declared, not per-invocation.

## Verification

Every phase runs `just check` before it reports done, per
`sase memory read lint_and_test.md`; run `just install` first in a fresh workspace
clone. The `land` step for the combined tree runs `just check-full` through
`/sase_monitor` with the `TESTING` / `TESTED` status pair — it outruns a single turn.

Phase-independent gates worth naming because they will bite:

- `symvision` flags newly unused symbols. Deleting `WebClosureMode` and
  `MemoryWeb.closure` in phase `closure` is likely required rather than optional.
- `toobig` caps files at 1000 lines; `src/sase/memory/notes.py` (586) and
  `src/sase/memory/web/frontmatter.py` (388) have room, but keep the scanner and
  resolver as their own modules rather than growing either.
- `just fmt` / `fmt-md-check` — the frontmatter renderer in
  `sase.memory.notes._prettier_stable_frontmatter` must keep the new keys
  Prettier-stable.
- Memory content changes require `sase memory init`, which regenerates `AGENTS.md`, the
  provider shims, and `sase/memory/README.md`.

The end-to-end acceptance check for the whole epic:

```bash
sase memory show decisions:gates-never-block    # single-turn-agents rendered inline at the bottom
sase memory show lint_and_test.md               # Linked References lists two-speed-verification and symvision.md
sase memory show glossary:stitch -f json        # byte-identical to the pre-epic output
sase memory show task_types:bug                 # Related Task Types links ci and flake
```
