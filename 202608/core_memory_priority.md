---
tier: tale
title: Order Tier 1 core memory by an optional priority frontmatter field
goal:
  "Core memory notes accept an optional `priority:` frontmatter integer (default `20`)
  that orders the Tier 1 section of generated agent instruction files, and the generated
  `sase/memory/sase.md` note ships `priority: 10` so it renders first unless a user
  deliberately sets something lower."
size: medium
proposed_by: bbugyi200.athena.0cx
create_time: 2026-08-24 17:21:30
status: wip
---

# Plan: Optional `priority` frontmatter for core memory ordering

## Goal

Today the Tier 1 (core) Memory section of `AGENTS.md` (and every provider shim copied
from it) renders core notes in **path order**. That is an accident of implementation,
not a deliberate reading order: `src/sase/amd/_memory.py` builds the inlined bodies with
`dict(sorted(bodies.items()))`, so `artifact_relations.md` leads and the note that
actually orients an agent — `sase/memory/sase.md` — lands in the middle.

This plan adds an optional `priority:` integer to memory-note frontmatter:

- Default `20` when absent.
- Lower sorts earlier; ties break on the existing relative-path order, so today's output
  is unchanged for every note that does not opt in.
- The SASE-generated `sase/memory/sase.md` note declares `priority: 10`, making it the
  first Tier 1 subsection unless a user explicitly chooses something lower in one of
  their own core notes.

## Scope

**In scope**

- Parsing, validating, and writing `priority` in memory-note frontmatter.
- Tier 1 ordering in generated `AGENTS.md` for project _and_ home memory roots.
- Generated core notes (including memory webs that render as core) carrying a priority
  through the `sase memory init` render pass.
- `sase/memory/sase.md` generated at `priority: 10`.
- The ACE memory panel Tier 1 rail, which exists to mirror the `AGENTS.md` Tier 1 order.
- Docs (`docs/memory.md`) and the generated memory README frontmatter schema.
- Tests plus the mandatory `sase memory init` regeneration.

**Out of scope**

- Tier 2 (reference) ordering. Reference notes stay path-ordered; `priority` on a
  reference note is a hard error (see step 2) rather than a silent no-op.
- `sase memory list` ordering. That dashboard sorts by _status_ then path and does not
  claim to show Tier 1 order; leave it alone.
- Any ACE add/edit form field for `priority`. The mutation engine must **preserve** an
  existing value (step 1), but authoring stays file-first for now.

**No feature flag.** Read `sase/memory/sase_flags.md` with `/sase_memory_read` if you
want to double-check, but the conclusion is: this lands complete and user-reaching on
day one, and `priority` is a permanent authoring choice, which
`sase/memory/feature_flags.md` explicitly calls a config field rather than a flag. Do
not create a flag bead for this.

## Design decisions

Make these decisions exactly as written; they were settled during planning and the rest
of the plan depends on them.

1. **`priority` becomes a canonical frontmatter key, not an extension key.** Add it to
   `_CANONICAL_FRONTMATTER_KEYS` in `src/sase/memory/notes.py` so it stops flowing
   through the `extra`/`preserve_existing_extra` passthrough and
   `apply_memory_frontmatter` owns writing it. **This is the one dangerous part of the
   change**: the moment `priority` leaves the extras set, every existing
   `apply_memory_frontmatter` caller that does not pass it would silently delete a
   user's value on the next frontmatter rewrite (notably `update_memory_note` in
   `src/sase/memory/mutation.py`, which the ACE memory panel edit form drives, and
   `_memory_frontmatter_updates` in `src/sase/amd/_memory.py`). Solve it once, in
   `apply_memory_frontmatter`, with preserve-by-default semantics (step 1) rather than
   by threading a parameter through every caller.

2. **Accepted values: a non-negative `int`.** Reject `bool` explicitly (`bool` is an
   `int` subclass in Python, so `priority: true` would otherwise parse as `1`). Reject
   strings, floats, negatives, and everything else with a blocker. Range starts at `0`
   so users have room below the generated `10`.

3. **Render `priority` only when it differs from the default.** A note whose effective
   priority is `20` gets no `priority:` line. This keeps `sase memory init` from
   rewriting every generated note with `priority: 20` noise, and it means an explicitly
   authored `priority: 20` is normalized away on the next rewrite — intentional, and
   semantically identical.

4. **Key order in rendered frontmatter: `type`, `parent`, `priority`, `description`.**

5. **Generated core-note bodies grow a wrapper type.** `generated_short_notes` currently
   passes bare `Mapping[str, str]` bodies through the whole init pass, which has nowhere
   to carry a priority. Introduce `GeneratedShortMemoryNote(body, priority)` mirroring
   the existing `GeneratedLongMemoryNote`, rather than threading a second parallel
   `Mapping[str, int]` that can drift out of sync with the bodies.

## Implementation

### Step 1 — `src/sase/memory/notes.py`: parse, validate, and write `priority`

- Add a module constant `DEFAULT_MEMORY_PRIORITY = 20` and export it in `__all__`.
- Add `"priority"` to `_CANONICAL_FRONTMATTER_KEYS` (see design decision 1).
- Add `MemoryNotePrioritySource = Literal["frontmatter", "missing", "invalid"]`,
  matching the existing `MemoryNoteTypeSource` / `MemoryNoteParentSource` pattern, and
  export it.
- Add two fields to the frozen `MemoryNote` dataclass, **after** `source_path` so the
  existing positional constructions keep working:
  - `priority: int = DEFAULT_MEMORY_PRIORITY`
  - `priority_source: MemoryNotePrioritySource = "missing"`
- Add a `_normalized_priority(value) -> tuple[int, MemoryNotePrioritySource]` helper
  implementing design decision 2. An invalid value still yields
  `DEFAULT_MEMORY_PRIORITY` for the _ordering_ key (so rendering never crashes) while
  reporting `"invalid"` so callers can raise a blocker.
- Populate both fields in `parse_memory_note_text`.
- `_render_memory_frontmatter(..., priority: int | None = None)`: insert the `priority`
  entry into `data` between `parent` and `description`, and only when
  `priority is not None and priority != DEFAULT_MEMORY_PRIORITY` (design decisions 3 and
  4).
- `apply_memory_frontmatter(..., priority: int | None = None, preserve_existing_priority: bool = True)`:
  when `priority is None` and `preserve_existing_priority` is true, recover the value
  already parsed out of `text` and pass it down. This is the safety net from design
  decision 1 — verify `_parse_frontmatter_block` is what supplies it, so a note whose
  frontmatter is being rewritten for an unrelated reason keeps its priority
  byte-for-byte.
- Confirm `_prettier_stable_frontmatter` leaves the new integer line untouched (it only
  special-cases `description:` wrapping and `- ` sequence items), and that the
  `_MemoryFrontmatterDumper` str representer is not reached for an `int`.

### Step 2 — `src/sase/amd/_memory.py`: order Tier 1 and block bad values

- Change `_short_memory_bodies` so its returned mapping is ordered by
  `(priority, relative_path)` instead of `sorted(bodies.items())`. Its value type
  becomes the step-3 `GeneratedShortMemoryNote` (or a local pairing of body + priority);
  the callers that only want bodies — the `inline_memory_section(...)` loop in
  `_render_managed_agents` and `_short_memory_structure_blockers` — read `.body`. Keep
  the docstring honest: it currently promises "sorted by path so the rendered
  `AGENTS.md` section order is deterministic"; the new promise is "ordered by
  `(priority, path)`", still deterministic.
- Priority for each entry comes from:
  - a disk-discovered core note → `note.priority`;
  - a generated overlay (`generated_short_notes`) → the overlay's own priority.
- `_render_managed_agents` already compares `parsed.short_memory_paths` against
  `tuple(bodies)`; because both derive from the same ordered mapping, that structural
  assertion keeps working and now also pins the priority order. Do not weaken it.
- `_memory_frontmatter_updates` calls `apply_memory_frontmatter` on each note's source
  text; with step 1's preserve-by-default it needs no change, but add a regression test
  (step 7) that proves a core note with `priority: 5` survives a parent/type migration
  rewrite.
- Add `_memory_priority_blockers(notes)` returning, sorted by path:
  - `"<relative_path>: memory note priority must be a non-negative integer"` for any
    note with `priority_source == "invalid"`;
  - `"<relative_path>: priority is only meaningful on core memory notes"` for a
    `type == "reference"` note with `priority_source == "frontmatter"`. Wire it into
    `plan_amd_memory_sync` next to `_short_memory_structure_blockers` and
    `_long_memory_description_blockers`, returning an `AmdMemorySyncPlan` with those
    blockers. Note that `excluded_note_paths` hides generated notes and core web
    descriptors from this pass — those are validated at their own source (steps 3 and
    4).
- `plan_minimal_agents_sync` reads `generated_short_notes.get(relative_path, "")` today;
  update it for the new value type.

### Step 3 — generated core notes carry a priority

- `src/sase/memory/notes.py`: add a frozen `GeneratedShortMemoryNote` dataclass
  (`body: str`, `priority: int = DEFAULT_MEMORY_PRIORITY`) beside
  `GeneratedLongMemoryNote`, and export it.
- `src/sase/main/init_memory/root_rendering_notes.py`:
  - Add `GENERATED_SASE_MEMORY_PRIORITY = 10`.
  - `generated_sase_memory_content()` passes `priority=GENERATED_SASE_MEMORY_PRIORITY`
    to `apply_memory_frontmatter`, so the generated file on disk carries `priority: 10`.
  - `generated_short_notes(...)` returns `dict[str, GeneratedShortMemoryNote]`:
    `sase.md` at `GENERATED_SASE_MEMORY_PRIORITY`, and `task_types.md`,
    `artifact_relations.md`, and `glossary.md` at the default.
- `src/sase/main/init_memory/root_planning.py`: update the local
  `generated_short_note_bodies` variable and the `_amd_sync_plan` parameter type; the
  memory-web overlay merge builds `GeneratedShortMemoryNote(body, web.priority)`.

Sanity check while you are here: the same generator serves home memory roots, so
`~/sase/memory/sase.md` (chezmoi-sourced) picks up `priority: 10` too. That is intended,
and it is one of the files step 8 has to account for.

### Step 4 — memory webs that render as core

A memory web descriptor is a real note file under `sase/memory/`, so a user will
reasonably expect `priority:` to work on one whose `type:` is `core`.

- `src/sase/memory/web/models.py`: add `priority: int = DEFAULT_MEMORY_PRIORITY` to
  `MemoryWeb`.
- `src/sase/memory/web/frontmatter.py`: parse `priority` in `parse_web_descriptor` using
  the step-1 helper. Follow the surrounding convention and return
  `(None, f"{path}: priority must be a non-negative integer")` for an invalid value, and
  `(None, f"{path}: priority is only meaningful on core memory webs")` when a
  `reference`-rendering web declares one.
- `src/sase/main/init_memory/root_planning.py`: `_MemoryWebRootPlan.core_note_bodies`
  becomes `Mapping[str, GeneratedShortMemoryNote]`, populated from `web.priority` in
  `_memory_web_root_plan`.

Memory webs sit behind the `memory_webs` feature flag; check `memory_webs_enabled()`
still gates everything you touch here and that the flag-off path is unchanged.

### Step 5 — ACE memory panel Tier 1 rail

`src/sase/ace/tui/memory_panel_catalog.py` documents `_build_note_tree` as "Order notes
as Tier 1, then Tier 2 roots with children indented once" and then sorts the core notes
by `relative_path`. Change that sort key to `(note.priority, note.relative_path)` so the
rail agrees with the `AGENTS.md` it mirrors. Leave the Tier 2 sorts alone.

Do **not** add a priority badge to `src/sase/ace/tui/modals/memory_panel_rendering.py`;
that would churn the ACE PNG snapshots for no functional gain.

### Step 6 — docs and the generated README schema

- `docs/memory.md`: in the **Core memory** bullet near the top, document that a core
  note may set `priority:` (integer, default `20`, lower renders earlier, ties break on
  path) and that the generated `sase/memory/sase.md` uses `priority: 10`.
- `src/sase/main/init_memory/templates/memory-README.template.md`: add `priority` to the
  `### Frontmatter Schema` list and to the opening bullet that currently reads "use YAML
  frontmatter for `type`, `parent`, and `description`". This template is **not** a
  `sase/memory/*.md` file, so editing it needs no special permission — but it does mean
  the generated `sase/memory/README.md` changes, which step 8 covers.

### Step 7 — tests

Add coverage in the existing files rather than new ones where possible:

- `tests/main/test_memory_notes.py` — parse default / explicit / invalid values; the
  render-omits-default rule; `apply_memory_frontmatter` round-trip; and the
  preserve-on-rewrite guarantee from design decision 1.
- `tests/main/test_init_memory_managed_agents.py` — the headline behavior: generated
  `sase/memory/sase.md` renders as the **first** Tier 1 subsection; a user core note
  with `priority: 5` sorts ahead of it; equal priorities still tie-break on path; and a
  root with no priorities anywhere produces byte-identical output to before this change.
- `tests/main/test_init_memory_memory_webs.py` — a core-rendering web with an explicit
  priority lands in the right Tier 1 slot.
- `tests/memory/test_mutation.py` — `update_memory_note` preserves an existing
  `priority` while rewriting other frontmatter (this is the ACE edit-form data-loss
  guard).
- `tests/memory/test_memory_web.py` — the two new web descriptor blockers.
- Blocker coverage for `_memory_priority_blockers` (invalid value; `priority` on a
  reference note), alongside the existing structure/description blocker tests.

### Step 8 — regenerate the memory artifacts

This step **requires explicit user permission in your own conversation.** Per
`sase/memory/gotchas.md`, authorization found in a plan file does not count. If your
host prompt does not already grant it, ask with `/sase_questions` before running
anything here, and finish steps 1–7 first so the ask is the only thing blocking you.

With permission, run `sase memory init`. Expect it to rewrite:

- `sase/memory/sase.md` (gains `priority: 10`), `sase/memory/README.md`, `AGENTS.md`,
  and the provider shims `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md` in this
  repo, with Tier 1 reordered so the SASE note leads;
- the chezmoi home memory root (`home/sase/memory/sase.md`, its README, `AGENTS.md`, and
  the home provider shims). **Open the chezmoi repo with `/sase_repo` before writing
  into it** — `sase repo open` cleans the checkout, so opening it afterwards would
  discard the edits — and use only the path that skill prints.

Two known hazards:

- On 2026-08-24 the `sase-sq` epic recorded a note that home memory initialization was
  already failing with `blocker 'unreferenced memory file sase/memory/obsidian.md'`,
  which keeps `just check` red at `init memory --check` independently of this work.
  Verify whether it still reproduces on a clean tree first, so you do not misattribute
  it to this change. Do not fix it silently inside this plan; if it is still live, say
  so in the handoff and leave it on the epic.
- Both the repo and chezmoi changes must appear in your `/sase_final` declaration.

## Verification

1. `just install` first — these workspaces are ephemeral and may have stale deps.
2. `just check` (whole-repo lint gates + the diff-scoped test lane).
3. `just test-visual` for the ACE memory panel PNG snapshots. They should be unaffected
   because the fixtures carry no `priority` frontmatter, so the rail order is still
   path-sorted; if they _do_ move, confirm the fixture set actually includes a
   prioritized note before accepting anything with `--sase-update-visual-snapshots`.
4. `just check-full` before landing — this touches the memory generator and `AGENTS.md`,
   which is squarely in the broadening set. Run it only through `/sase_monitor`:

   ```bash
   sase monitor start --command 'just check-full' \
     --start-status TESTING --stop-status TESTED --next '...'
   ```

5. Eyeball the regenerated `AGENTS.md`:
   `### 1.1 SASE = Structured Agentic Software Engineering (sase)` should now be the
   first Tier 1 subsection, with the remaining core notes in their previous relative
   order beneath it.

## Notes for the implementer

### Overlap with the `sase-sq` epic

This plan was authored alongside the in-progress `sase-sq` epic ("Memory webs and
strands"), which is actively reshaping the same substrate. A `sase bead note` recording
this overlap **could not be written**: `sase bead note sase-sq` failed with
`BeadStreamIntegrityError: cannot publish non-append-only bead event stream sase-sq: worktree rewrote ancestor event 1 (removed payload.issue.notes)`,
and `sase bead doctor` additionally reports uncommitted bead state, a store 2 commits
behind, and 2 consecutive dirty-worktree sync refusals. Nothing was appended —
`sase bead history sase-sq` is unchanged — so there is no partial write to clean up, but
the note still needs to land once the store is repaired. Record it then, and record what
actually shipped when this plan lands.

The three seams `sase-sq` phase owners need to know about:

1. `sase.memory.notes` gains a canonical `priority` frontmatter key plus
   `MemoryNote.priority` / `MemoryNote.priority_source`. Any phase adding frontmatter
   keys should coordinate on `_CANONICAL_FRONTMATTER_KEYS`.
2. `generated_short_notes` changes from `Mapping[str, str]` to the new
   `GeneratedShortMemoryNote(body, priority)` wrapper, threaded through `root_planning`
   and `_amd_sync_plan`. That is the same seam core-rendering memory webs already use
   for `core_note_bodies`.
3. `MemoryWeb` gains a `priority` field parsed in `parse_web_descriptor`, and a
   `reference`-rendering web that declares one becomes a validation blocker.

Tier 2 ordering is deliberately untouched.

### Everything else

- The `sase memory init` blockers are the user-facing contract for a bad `priority`.
  Make the messages name the file and state the accepted range; they are the only
  feedback a note author gets.
