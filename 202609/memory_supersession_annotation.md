---
tier: tale
title: Annotate superseded memory strands instead of hiding them
goal:
  A memory strand can declare in `metadata` that it was superseded, that fact renders as
  a marker on the strand's roster bullet and on every `sase memory read`/`show` of it, a
  validator warns when the declaration and the strand body disagree, and the `decisions`
  corpus's one real supersession is finally marked.
size: medium
proposed_by: bbugyi200.apollo.2
create_time: 2026-09-01 17:20:17
status: wip
---

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                      | Why                                                               |
| ------------ | ----------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| derives-from | [research:202608/superseding_memory_strands/superseding_memory_strands.md][1] | The research report whose §7 recommendation this plan implements. |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202608/superseding_memory_strands/superseding_memory_strands.md

<!-- sase:links:end -->

# Plan: Annotate Superseded Memory Strands

## Why

Research report `superseding_memory_strands.md` (§7) evaluated a proposal to let one
memory strand supersede another — rendering the superseded strand on read, and dropping
it from the descriptor roster and therefore from every generated agent instruction file.
Its recommendation, which this plan implements:

- **Build annotation, and put it in the roster.** The report measured the audited read
  log: nine reads total across the entire `decisions` web ever, and four of its eight
  records never read at all — while every agent receives the roster in `AGENTS.md` on
  every turn. The roster is the only surface with reach, so that is where a supersession
  signal has to land.
- **Do not build hiding.** Hiding contradicts the hub-note model, ADR practice (`-s` in
  `npryce/adr-tools` annotates both records, it does not remove either), and the
  `decisions` descriptor's own written policy that a superseded record is "marked
  superseded in prose, never edited in place". It is also a one-way door on the audit
  trail, where annotation is not, and the whole token argument is noise: the `decisions`
  roster is ~43 tokens per record inside a ~3,000-token `CLAUDE.md`.
- **Do not add a typed `supersedes:` frontmatter edge.** A strand's `metadata` is
  already a free-form mapping, so a scoped supersession is expressible today with no
  parser change — and a new parallel link syntax is exactly what
  `decisions:memory-links-are-authored` reopens for, not something to add speculatively.

## What Already Landed — Do Not Redo It

The report was written against `fdb962c13` on 2026-08-30. Two of its five steps have
since shipped; this plan covers only what is left. Verified against `3e70017fd`:

- **Report Step 1 (unbreak links in `decisions`) is done.** Commit `70dd1da61` dropped
  the `closure: none` key from `sase/memory/decisions.md`, so the web now takes the
  permissive `link_reference: explicit` default.
  `sase memory show decisions:gates-never-block` resolves its authored
  `![[decisions/single-turn-agents]]` link. Do not touch that frontmatter.
- **Report Step 5's link documentation is done.** Commits `0860fcb20` and `4509c9d67`
  gave `docs/memory.md` a full `## Memory Links` section, `sase/memory/README.md` (and
  its template) a `link_reference` / `link_rendering` schema entry, and
  `src/sase/xprompts/skills/sase_memory_write.md` a `## Links` section. Only the
  _supersession authoring_ half of Step 5 is still missing.
- **The corpus grew a second partial supersession.**
  `decisions:memory-links-are-authored` (accepted 2026-08-30) states in its own body
  that it "supersedes the 'not a new, parallel link syntax' clause of
  `[[decisions/memory-webs]]`". So `memory-webs` is now partially superseded by _two_
  records with _different_ scopes. The data model below must handle more than one
  successor per strand; the report's single-successor example is out of date.

Everything else the report recommends against building (roster suppression, a typed
`supersedes:` edge, coupling `sase memory init` to the artifact link store) stays
unbuilt. See Non-Goals.

## Design

### Frontmatter Shape — No Parser Change

`MemoryStrand.metadata` is already `dict[str, Any]`, parsed by `_metadata()` in
`src/sase/memory/web/frontmatter.py` which validates only that the value is a mapping,
and round-tripped unchanged by `render_strand_frontmatter`. Supersession is declared
inside it:

```yaml
metadata:
  status: superseded-in-part
  decided: 2026-08-24
  superseded_by:
    - decisions/webs-render-in-their-own-section
    - decisions/memory-links-are-authored
```

- `status` recognizes exactly two supersession values, `superseded` (whole) and
  `superseded-in-part` (partial). Every other value — including today's ubiquitous
  `accepted` — is left completely alone: no marker, no warning, no error. `metadata` is
  free-form and other webs must stay unaffected.
- `superseded_by` accepts a single non-empty string or a list of them, each an ordinary
  memory link target in any form `[[...]]` accepts (`web:keyword`, `web/slug`,
  `note.md`, or a bare token).
- **Per-successor scope lives in the strand's body prose, not in frontmatter.** Partial
  supersession is the only kind this corpus has produced, and the scope of each
  retirement is a sentence, not a slug — it belongs next to the passage it retires,
  carried by an authored `[[...]]` link. Keeping it out of frontmatter is what lets
  `superseded_by` stay a flat list of targets and keeps the roster marker to a few
  tokens.

### Marker Text

Roster, `roster: list` (the `decisions` shape) — the marker is inserted between the slug
and the summary, and the whole bullet keeps going through `wrap_markdown`:

```
- **Memory Webs** (`memory-webs`) - *[partly superseded by `webs-render-in-their-own-section`,
  `memory-links-are-authored`]* A keyed memory collection is a flat descriptor note plus
  a sibling strand directory, addressed web:keyword.
```

- Whole supersession reads `*[superseded by ...]*`; partial reads
  `*[partly superseded by ...]*`.
- Successor addresses display with a leading `<own-web>/` stripped, because inside the
  `decisions` roster the `decisions/` prefix is pure noise. Any other form displays
  verbatim.

Roster, `roster: inline` (the `glossary` shape) — that roster is a semicolon-joined term
list with no summary column, so the marker is a bare suffix with no successor list:
`Memory Webs (memory strands, web-backed memory) [partly superseded]`. Nothing in the
corpus uses it today; this exists so an inline-roster web is not a silent hole, and it
costs zero tokens until someone marks a strand in such a web.

Read/show — a line right after the node's provenance line, in the rendered body (not
stderr; `sase artifact read` prints its supersession warning to stderr, which an agent
capturing stdout never sees — mirror the behavior, not the stream):

- Markdown:
  `> **Partly superseded** by `decisions/webs-render-in-their-own-section`, `decisions/memory-links-are-authored`.`
- Rich: the same sentence as a `bold yellow` line.
- JSON: a `supersession` key on the node payload —
  `{"status": ..., "partial": bool, "superseded_by": [...]}` — or `null`.

Successor addresses render verbatim here (no prefix stripping): a read can be part of a
mixed batch, so the fully qualified address is the useful one. The successor's label and
summary are not repeated, because the body's authored `[[...]]` link already puts them
in the `## Linked References` section.

### Validation

Fold the checks into the existing `validate_memory_webs` warning path rather than adding
a standalone `sase doctor` check. That path already reaches both surfaces that matter:
`sase doctor`'s `config.memory_webs` check and `sase memory init`'s plan warnings. It is
strictly more coverage than a doctor-only check for less code.

**Warnings, never blockers.** Blockers fail `sase memory init` for the whole repo, which
is far too harsh for an authoring convention; unresolved authored links already set the
warning precedent, and `run_init_memory` returns non-zero only on blockers. The rules:

1. `superseded_by` names a target that does not resolve.
2. `status` is a supersession value but `superseded_by` is missing or empty.
3. `superseded_by` is present but `status` is not a supersession value.
4. `superseded_by` is not a string or a list of non-empty strings.
5. The strand body authors no `[[...]]` link resolving to a named successor. Skip this
   rule when the strand's effective `link_reference` is `none`, where authored links are
   deliberately off.

Rule 5 is the enforceable half of the whole proposal — prose cross-references cannot be
validated, and prose is exactly what failed silently on this corpus's first and only
supersession. Compare resolved identity (web slug plus strand slug), not raw strings, so
a successor linked by keyword or alias satisfies the rule.

## Steps

Steps 1–3 are the mechanism; Steps 4–5 are the memory and documentation edits that use
it. Do them in this order: Step 4 depends on Step 1 existing, and Step 6 must come last.

### 1. Parse supersession metadata

New module `src/sase/memory/web/supersession.py`. It must import only from `.models`
(and stdlib) so `roster.py` can use it without an import cycle — `link_resolve` is
already lazily imported inside `validation.py` for this reason.

- A frozen dataclass carrying the recognized status, a `partial: bool`, and
  `superseded_by: tuple[str, ...]`.
- A parser taking a `MemoryStrand` and returning that dataclass or `None`, tolerant of
  every malformed shape rule 4 covers (never raise; return `None` and let validation
  produce the warning).
- A small helper that formats the marker phrase, taking the owning web slug so it can
  strip a same-web prefix.

Export the new names from `src/sase/memory/web/__init__.py` alongside the other web
helpers.

### 2. Render the marker in the roster

`src/sase/memory/web/roster.py`, `render_strand_roster` — both branches:

- `roster == "list"`: insert the marker between the slug and the summary as shown above,
  before wrapping.
- `roster == "inline"`: append the bare `[superseded]` / `[partly superseded]` suffix in
  `_inline_entry`.

`ordered_web_strands` and the set of rendered strands are untouched: **annotation never
hides**. The roster region is a managed region, so this changes committed descriptor
content — handled in Step 6.

### 3. Render the marker on read/show, and validate

- `src/sase/memory/selector_render.py`: emit the status line in `_web_section_markdown`
  (after the `_provenance_label` line), in `_node_blocks` (rich), and as a
  `supersession` key in `_node_json`.
- `src/sase/memory/web/validation.py`: implement rules 1–5.
  `_unresolved_web_link_warnings` already builds the `notes` and `scoped_webs` a
  resolver needs and already walks every strand of every web — extend that walk rather
  than duplicating the discovery. Reuse `resolve_memory_link_target` and
  `scan_memory_links`.

Warning strings follow the existing `f"{strand.path}: ..."` shape so they render
correctly through both `_prefixed(...)` in the doctor check and the init plan warnings.

### 4. Mark `decisions:memory-webs`

`sase/memory/decisions/memory-webs.md`. This record is partially superseded twice.

Frontmatter — change `status: accepted` to `status: superseded-in-part`, keep `decided`
as it is, and add the `superseded_by` list from the Design section above. Adding the
status mark _is_ the ADR-sanctioned edit to an accepted record; the descriptor's "never
edited in place" rule governs the argument, not the status field.

Body — add two scoped marks. Do not reword, delete, or soften the retired passages
themselves; a record is immutable once accepted.

- After the Claim paragraph:

  > _Partially superseded:_ the sentence above on core/reference rendering is retired by
  > [[decisions/webs-render-in-their-own-section]].

- After the Reopens-when paragraph:

  > _Partially superseded:_ the "not a new, parallel link syntax" clause above is
  > retired by [[decisions/memory-links-are-authored]].

Use `[[...]]`, not `![[...]]`: a reference gives the pointer, its label, and its summary
without transcluding a full record the reader may not want. This is the first actual use
of the convention the descriptor has always claimed, and it is what makes Step 3's rule
5 pass.

Also update one clause in `sase/memory/decisions.md`'s descriptor prose — "the old one
is marked superseded in prose" — to name the mechanism that now exists: a
`metadata.status` plus `superseded_by` mark and a `[[...]]` back-link on the older
record. This note is always-loaded, so keep it to a clause, not a paragraph.

### 5. Document supersession authoring

- `src/sase/xprompts/skills/sase_memory_write.md`: a short `## Supersession` section —
  mark the _older_ record with `metadata.status` plus `superseded_by`, add a `[[...]]`
  back-link stating what was retired, never delete or reword an accepted body beyond
  adding the mark, and remember that marking a record changes always-loaded agent
  instructions. Do **not** run `sase skill init` from the workspace; per
  `sase/memory/generated_skills.md` a chezmoi deploy is refused from an unlanded tree
  and reverts other agents' deployments. The template edit lands; deployment is a
  separate post-land step.
- `docs/memory.md`: extend `## Memory Webs` (or the `## Memory Links` section) with the
  supersession metadata shape, the two recognized `status` values, where the marker
  renders, and the fact that the checks are warnings.
- `src/sase/main/init_memory/templates/memory-README.template.md`: add a `metadata`
  supersession line to the `### Frontmatter Schema` list. Editing the template is
  required — `sase/memory/README.md` is generated from it and hand edits are
  overwritten.

### 6. Regenerate and verify

Run `sase memory init` **last**, after Steps 1–5, so the roster marker and the
descriptor prose land together. This regenerates `sase/memory/decisions.md`'s managed
roster region, `sase/memory/README.md`, `AGENTS.md`, and the provider shims.

**Hazard: `sase memory init` writes the home root too.** It has no project-only scope,
and this machine's home/chezmoi root already carries unrelated pre-existing drift
(`~/sase/memory/sase.md`, its README, and the home `AGENTS.md`/`CLAUDE.md`/`GEMINI.md`/
`QWEN.md`/`OPENCODE.md` shims). Before changing anything, capture
`sase memory init --check --diff` as a baseline. Afterwards, any home-root change beyond
that baseline is unexpected and must be investigated. If the chezmoi repo ends up dirty,
open it through `/sase_repo`, keep the pre-existing drift out of this change's story,
and state plainly in the final report what was written there and why.

## Non-Goals

Each of these is a decision the report reached on evidence, not an omission. Do not
"finish the job" by adding them.

- **Roster suppression, or hiding a strand from generated agent instructions.** The
  explicit recommendation against. It creates a "present but invisible" state every one
  of the web validator's rules would have to reason about, makes bare-web reads
  inconsistent with the roster, and destroys information on the only surface with reach.
- **A typed `supersedes:` / `superseded_by:` top-level frontmatter edge.** `metadata`
  already carries it with no parser change, and a new parallel link syntax is the reopen
  condition of `decisions:memory-links-are-authored`, not a free move.
- **A forward `supersedes` mark on the newer records.**
  `webs-render-in-their-own-section` and `memory-links-are-authored` already state what
  they retire, in prose, with authored links. The pointer that was missing runs
  backward, and only backward.
- **Any coupling of `sase memory init` to the artifact link store.** Artifact-link truth
  lives in per-artifact JSON under document sidecars with a machine-local aggregate;
  `sase memory init` renders deterministically from committed memory files. Coupling
  them would make the same commit render different instructions depending on which
  sidecars are present.
- **The `sase memory web show` index table.** Out of scope. The report justified the
  roster (measured reach) and read/show (correctness); the index has neither claim on
  it.
- **Any Rust core change.** The memory-web subsystem — discovery, frontmatter, roster,
  links, rendering — is entirely Python-resident; `sase_core` carries only
  `glossary.rs`, the catalog/validation facade this work does not touch. A Python helper
  shared by the CLI renderers is the right home, and the ACE Memory panel already
  renders `strand.metadata` generically, so it displays the new keys with no change.

## Tests

- `tests/memory/test_memory_web.py` — parser: every recognized and malformed `metadata`
  shape; roster marker for `roster: list` and `roster: inline`; a strand with
  `status: accepted` and one with no `metadata` render byte-identically to today; a
  superseded strand still appears in the roster.
- `tests/memory/test_memory_web.py` (or a sibling) — validation rules 1–5, including the
  `link_reference: none` skip for rule 5, asserting they arrive as `warnings` and never
  as `blockers`.
- `tests/memory/test_memory_selector_render.py` — the status line in markdown and rich,
  and the `supersession` key in JSON, plus a nodes-without-supersession case.
- `tests/doctor/test_config_memory_webs.py` — a supersession warning reaches the
  `config.memory_webs` check as `WARN` with the message in `details`.
- Update any generated-instruction golden that pins the `decisions` roster text
  (`tests/main/test_init_memory_memory_webs.py` and neighbors under `tests/main/` are
  the likely ones; search for the roster payload rather than assuming the file).

## Verification

- `just install` first — this is an ephemeral workspace clone with a possibly stale
  virtualenv — then `just check`.
- `sase memory init --check` must be clean for the project root after Step 6.
- `sase doctor` must report `config.memory_webs` as `OK`, not `WARN`. If it warns, the
  Step 4 marks and the Step 3 rules disagree with each other and one of them is wrong.
- `sase memory read decisions:memory-webs -r "<why>"` shows the partial-supersession
  line and both back-links under `## Linked References`.
- The `decisions` roster bullet for `memory-webs` in the generated `AGENTS.md` and
  `CLAUDE.md` carries the marker, and all eight records are still listed.
- Hand `just check-full` to `/sase_monitor` before landing; it routinely outruns a turn
  and must never be run inline.
