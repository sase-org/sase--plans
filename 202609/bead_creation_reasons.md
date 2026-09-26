---
tier: epic
title: Make agent-filed beads explain themselves in Context
goal:
  Every new bead records why it was created, and agent-created beads are recognizable
  and informative in the Context card.
phases:
  - id: durable_reason
    title: Persist and index the bead creation reason in sase-core
    depends_on: []
    size: medium
    description:
      "durable_reason: add a validated, immutable creation reason to bead creation,
      storage, events, and the touch index while keeping historical beads readable."
  - id: creation_flows
    title: Require reasons in user creation flows and supply them in generated flows
    depends_on:
      - durable_reason
    size: medium
    description:
      "creation_flows: wire the reason through Python, require it in CLI and TUI
      creation, update generated creators and guidance, and ratchet the core revision."
  - id: context_presentation
    title: Give created and assigned beads distinct, polished Context treatments
    depends_on:
      - creation_flows
    size: medium
    description:
      "context_presentation: show a prominent created state and its reason, clarify
      assignment-only rows, expose reasons in bead details, and verify responsive visual
      output."
proposed_by: bbugyi200.athena.0sv
create_time: 2026-09-26 11:35:46
status: wip
---

- **PROMPT:**
  [prompts/202609/bead_creation_reasons.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/bead_creation_reasons.md)

# Problem and intended behavior

The Agents Context card currently merges event-indexed bead touches with the agent's
assigned bead IDs. Its `own` chip means _assigned to the agent_, not _created by the
agent_. A newly assigned bead can therefore appear as `sase-1ao · own` with no title or
reason; a real agent creation appears through a `created` event, but its reason is not
stored. The existing one-line fallback also replaces a title with a read or close
reason, so adding a reason without redesigning the row would keep important context
hidden.

Make `creation_reason` a separate, immutable explanation of **why this bead was filed**,
distinct from its title (what it is) and description (scope/evidence). Example:
`sase bead create -T 'task(bug)' -t 'Fix retry race' -z medium -w 'A second agent reproduced dropped retries after the queue change' -d @/tmp/diagnosis.md`.
`-w/--reason` is required for every direct `sase bead create` invocation; `@<path>`
works as it does for descriptions. Reject whitespace-only reasons before opening a
mutation, with a clear error and help example. Preserve the historical empty-reason
state while reading pre-feature beads and when an older released client calls the
backward-compatible core wire without the new field; do not fabricate one from title or
description.

Visually, distinguish a bead the displayed agent **created** from one it was merely
**assigned**. A non-closed created row gets a warm amber `▐CREATED▌` badge and the
existing `✚` glyph; a standing close retains its green/grey `▐CLOSED▌` priority and gets
a compact `created` chip. Rename the ambiguous `own` presentation to `assigned` for
assignment-only rows and show the already-loaded bead title when available. For a
created row, show a bounded title line and a clearly labeled `why: …` line; keep close
reasons and note previews available without conflating them with the filing reason. Use
the existing width-aware wrapping and hint target, and let legacy rows fall back
gracefully. The badge describes a durable event, so it never says `NEW` after the bead
ages.

## Phase 1: durable_reason

In the linked `sase-core` checkout, extend `BeadCreateRequestWire` and `IssueWire` with
`creation_reason`. The new request field must be optional/defaulted for released Python
clients using the same core minor line; reject a reason explicitly supplied as blank or
over a sensible length bound, while the current SASE entry points supply a nonblank one.
Keep `IssueWire` deserialization defaulted so old event streams, JSONL projections, and
fixtures remain readable; do not add the field to update requests. Persist it in the
`issue_created` payload and normal projections, and prove it survives event replay,
JSONL round trips, and the PyO3 `bead_create` binding. Update internal Rust creation
paths that represent new user-facing flows to pass meaningful reasons; generated
examples may use deterministic purpose statements, but must not substitute a bead title
for a reason.

Extend `bead/touch_index.rs` to carry the creation reason from the original
`issue_created` event to the derived touch rows. Do not attribute a `created` verb to
another actor or infer one from `own`/assignment. Bound the indexed preview while
retaining the full stored reason for detail views. Bump
`BEAD_TOUCH_INDEX_WIRE_SCHEMA_VERSION` so old caches rebuild from streams; test refresh,
cache miss/rebuild, old reasonless streams, actor attribution, and created-plus-closed
rows. Run focused core/binding tests and the linked repo's guarded
`sase tool run check`.

## Phase 2: creation_flows

In `sase`, add `creation_reason` to the Python `Issue` model, Rust wire conversion,
project and mutation facades, and direct creation outputs. Add alphabetically positioned
`-w/--reason` to `sase bead create`, with excellent help, `@<path>` support, and a
required-value check before mutation. Update CLI tests for missing, blank, too-long,
file-backed, and valid reasons, plus persistence/readback. Reuse the same validation
semantics in the task-creation TUI modal: a labeled required reason field, focused
error, scrollable layout, and pass-through to the worker. Keep `description` optional
and separate.

Supply explicit purpose-specific reasons to the plan/epic and phase generator and to
feature-flag bead creation; audit every `project.create`/`bead_create` caller. Update
examples and documentation, including `src/sase/xprompts/skills/sase_new_task.md`, the
bead onboard/help examples, and the generated bead-memory template/canonical note as
part of the requested CLI contract change. Follow `/sase_memory_write` when modifying
memory and regenerate its instruction outputs; preview skill generation but deploy only
from a landed source revision. Move `sase-core-revision.txt` past the core commit before
relying on the new binding. Verify CLI parsing, generated bead flows, the TUI modal, and
`sase tool run check`.

## Phase 3: context_presentation

Project the indexed reason through `BeadTouch` and `BeadTouchEntry` without disk I/O in
rendering. Preserve exact creator attribution through the existing event index and
agent-session matcher. In `_agent_bead_touches.py`, render the amber created treatment,
the separate title and `why:` lines, and a calmer assigned-only treatment. A row with
both created and closed remains visibly closed and still exposes its creation reason;
read/close reasons retain explicit labels rather than replacing the title. Preserve
chronological order, five-row cap, numbered hints, narrow/split layouts, bounded lines,
safe text rendering, and legacy empty-reason fallbacks. For an assigned-only row, use an
already-resolved matching bead summary for its title when available, with no synchronous
bead lookup on the UI thread. Do not claim it was created by that agent.

Show `Creation reason` in `sase bead read/show` and the Beads detail pane so the compact
card can link to the full text. Test rich style spans and plain text for created,
assigned-only, created-plus-closed, old reasonless, long/control-character, and
session-member attribution cases. Add or update focused ACE visual snapshots for wide
and narrow cards, run the targeted `just fix-tui-screenshots` recipe, inspect the report
and changed PNGs, and capture a live TUI view if fixture output cannot establish the
visible result. Finish with `sase tool run check`; do not run `just check-full` without
an explicit request.

# Acceptance criteria

- A new direct CLI or TUI bead in current SASE cannot be created without a substantive
  reason, and all current automatic bead creators provide one. Older clients remain
  compatible with the defaulted core wire.
- The reason is durable, immutable through normal updates, preserved through core/Python
  round trips, and visible in bead detail. Historical beads with no reason still load.
- The Context card clearly distinguishes `created` from `assigned`, shows the filing
  reason and title for an agent-created bead, and never manufactures creator credit for
  an assigned bead.
- The card remains readable and responsive in narrow and split layouts, with no
  store/index read added to rendering; visual goldens and the repository checks pass.
