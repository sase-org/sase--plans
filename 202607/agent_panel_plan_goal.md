---
tier: tale
goal: 'The agent metadata panel on the `sase ace` Agents tab shows the plan''s `goal`
  frontmatter as a prominent, beautifully rendered field whenever the selected agent
  row (root or child) is associated with a plan — resolved reliably, read off the
  event loop, and gracefully absent when no plan or no goal exists.

  '
create_time: 2026-07-15 08:13:15
status: done
prompt: 202607/prompts/agent_panel_plan_goal.md
---

# Plan: Show the plan `goal` in the Agents-tab metadata panel

## Context

Since the sase-61 epic, SASE plan files carry a structured `goal` string in their YAML frontmatter (authored per the
`/sase_plan` skill; required for both `tale` and `epic` tiers). The `goal` is the single best one-line answer to "what
outcome is this work driving toward." Today that intent is invisible in the TUI: when you highlight an agent on the
**Agents** tab of `sase ace`, the metadata panel shows identity/lifecycle fields (Name, Bead, ChangeSpec, Model,
Timestamps, …) but never the goal of the plan the agent is executing or drafting.

This plan adds a **`Goal:`** field to that metadata panel, shown whenever the selected agent row — root or child — is
associated with a plan file that has a `goal`. It must be **intuitive** (reads naturally next to the existing
work-identity fields), **reliable** (correctly resolves the agent's plan across the several ways agents link to plans,
and degrades silently when there is no plan/goal), and **beautiful** (a distinct, prominent, well-wrapped treatment
consistent with the panel's visual language).

### Where the pieces already live (grounding for the implementer)

- **Metadata panel rendering** is a single imperative Rich `Text` built by `build_header_text(...)` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`. Every field is a `label`/`value`/`\n` triple; labels
  use the accent style `"bold #87D7FF"`. The `Bead:` field there already renders through the cached bead pattern.
- **Expensive, disk-derived header fields** are precomputed off the Textual event loop by
  `build_detail_header_summary(agent)` in `.../prompt_panel/_agent_display_header_summary.py`, stored in the
  `DetailHeaderSummary` dataclass (`.../prompt_panel/_agent_display_state.py`), cached per `agent.identity` with a TTL,
  and consumed by `build_header_text`. The worker that runs it (`start_agent_detail_header_enrichment` in
  `.../prompt_panel/_agent_display_async.py`) already runs `thread=True` and re-renders on completion. The cheap j/k
  fast path renders with no summary, so summary-sourced fields are simply skipped until the debounced full render lands
  — exactly the behavior we want for a disk-derived field.
- **Lenient, best-effort plan frontmatter reads** are already done in Python by `src/sase/sdd/plan_tiers.py`
  (`read_plan_tier`, `classify_plan_file`) on top of `src/sase/sdd/frontmatter.py::parse_frontmatter`.
  Strict/authoritative plan schema validation lives in the Rust core binding (`sase.sdd.plan_validate.validate_plan`),
  but that path returns `plan=None` on _any_ validation failure, so it is the wrong tool for a tolerant display read
  (older/mid-edit plans would show nothing).
- **Agent → plan-file association** already has a canonical priority resolution in
  `src/sase/agent/_family_attach_candidates.py::family_sase_plan()` (`sdd_plan_path` → `plan_path` → `plan_path.json`
  marker → done marker). Epic/phase agents launched by `sase bead work` do **not** carry a direct plan path; they carry
  `epic_bead_id` / `phase_bead_id`, and the plan file is reachable via the bead's `design` field
  (`src/sase/bead/epic_from_plan.py` sets `design = plan_ref` on the epic; phases inherit through `parent_id`). Bead
  lookup for an agent already exists in `src/sase/ace/tui/models/agent_bead.py` (`derive_agent_bead_id_from_name`,
  `BeadIssueLookupSession`, TTL cache), and `Issue.design` (`src/sase/bead/model.py`) holds the plan path.

## Goals / non-goals

**Goals**

- Show the associated plan's `goal` in the Agents-tab metadata panel for the selected root or child agent.
- Resolve the association reliably across the real ways agents link to plans: direct plan-path metadata (planning /
  plan-chain agents) and bead-derived paths (epic and phase work agents).
- Keep the Agents tab responsive: all disk / bead-store work stays off the event loop and is cached; the hot j/k path is
  untouched.
- Degrade silently and correctly: no plan, missing/deleted plan file, no `goal` key, empty goal, or unparseable
  frontmatter all render _nothing_ (no error, no empty `Goal:` label).

**Non-goals**

- No Rust / `sase-core` changes. This is a tolerant _display_ read that mirrors the existing Python-side lenient tier
  reads; it must not depend on strict validation. (This also sidesteps the fact that the sase-61 `plan_validate` Rust
  source is not checked out in a locally editable worktree.)
- No new keymap, no interactive editing of the goal, no changes to how goals are authored or validated.
- Not surfacing the goal on every agent _row_ in the list — only in the detail panel for the selected agent (matching
  the request and keeping row rendering cheap).

## Design

### 1. A tolerant, reusable goal reader (Python, no Rust)

Add a best-effort goal reader alongside the existing tier readers, following the exact shape of `read_plan_tier` /
`read_plan_tier_from_content` in `src/sase/sdd/plan_tiers.py`:

- `read_plan_goal_from_content(content: str) -> str | None`
- `read_plan_goal(path: Path) -> str | None`

Behavior:

- Parse frontmatter best-effort (reuse the module's existing parser); return `None` on any parse error, missing file,
  non-mapping frontmatter, or missing/blank `goal`.
- Accept only a string `goal`; coerce nothing exotic.
- **Normalize whitespace for display**: collapse runs of whitespace (including the internal and trailing newlines that
  YAML folded/block scalars produce) into single spaces and strip. A blank result returns `None`. This is what makes
  multi-line authored goals render as one clean sentence.

This keeps the tolerant-read policy in one place and stays consistent with how `tier` is already read for
display/classification.

### 2. Reliable agent → plan-goal resolution (off the event loop)

Add a small resolver module for the TUI, mirroring the structure and caching discipline of
`src/sase/ace/tui/models/agent_bead.py` (a sibling `agent_plan_goal.py` is a natural home). It exposes:

- `resolve_agent_plan_goal(agent, *, lookup_session=None) -> str | None` — the off-loop resolver (safe to call only from
  a worker thread), and
- a module-level, bounded cache so repeated TTL-driven summary rebuilds for the same plan file do not re-read/parse it.

**Association priority (first hit wins):**

1. **Direct plan-path metadata on the agent.** Resolve the plan path the same way `family_sase_plan()` does, from fields
   the TUI loaders already have access to (`sdd_plan_path` / `plan_path` / the `plan_path.json` marker / the done
   marker). Introduce a single clean `Agent.plan_path: str | None` field (`src/sase/ace/tui/models/agent.py`) populated
   by the loaders from those record fields, rather than overloading the multi-purpose `extra_files`. This is the
   authoritative "the plan this agent authored/approved/executes directly" signal.
2. **Bead-derived plan path.** If there is no direct path but the agent maps to a bead (reuse
   `derive_agent_bead_id_from_name`, or the `epic_bead_id` / `phase_bead_id` meta), look up the `Issue` via a
   `BeadIssueLookupSession`. Use the epic bead's `design`; for a phase bead (no own `design`), walk to its parent and
   use the parent's `design`. This is what covers the sase-61 epic/phase execution agents — the primary audience.

Resolve the stored `design`/metadata plan reference to an absolute, readable path (reuse the existing storage-path
resolution used elsewhere for bead `design` / SDD plan references; the plans root is discoverable and the loaders
already resolve similar plan paths for the file panel). Then read the goal via the reader from section 1.

**Caching:** key the goal cache by the resolved plan path and invalidate on file mtime (per the TUI perf rule "cache
disk reads keyed by mtime"), with a bounded LRU and a short TTL for the "no plan / unresolved" negative result, matching
the `_BeadDisplayCache` design. This guarantees at most one read+parse per plan file per change, even as the detail
summary is rebuilt across selection churn.

### 3. Wire the goal into the detail-header summary

- Add `plan_goal: str | None` to `DetailHeaderSummary` (`.../prompt_panel/_agent_display_state.py`).
- In `build_detail_header_summary(agent)` (`.../prompt_panel/_agent_display_header_summary.py`) — which already runs in
  a worker thread — call `resolve_agent_plan_goal(agent)` and set `plan_goal`. Reuse a single `BeadIssueLookupSession`
  for the call so bead-derived resolution shares one store session, consistent with the existing bead pattern.
- No change to the cheap path: when there is no summary yet, the field is simply absent, and the debounced full render
  fills it in — identical to how `bead_display`, `xprompts_used`, etc. behave today.

### 4. Beautiful, intuitive rendering

Render a `Goal:` field in `build_header_text` (`.../prompt_panel/_agent_display_header.py`), gated on
`summary is not None and summary.plan_goal`:

- **Placement:** immediately after the `Bead:` field (falling back to just after `Name:` when no bead is shown). The
  bead line answers _what_ the work item is; the goal answers _what outcome it drives_ — they belong together at the top
  of the panel where intent is most useful.
- **Prominence & style:** a distinct, panel-consistent treatment. Use the standard label accent for `Goal:` and render
  the value in a differentiated accent (italic, in a warm readable tone that reads as "the why"), so it stands out from
  the neutral identity values without clashing with the existing palette. Prefer a deterministic geometric marker over
  emoji if a leading glyph is used, to keep the PNG visual snapshots stable and fontconfig-independent.
- **Wrapping & length:** the value is already whitespace-normalized to one logical line; soft-truncate at a word
  boundary to a sensible cap (roughly two lines' worth of characters) with a trailing `…`, and let the panel wrap it.
  The full goal is never hidden from the user: the plan file itself is already reachable in the detail file panel, so
  the header stays scannable while the complete text remains one keystroke away. Truncation is character/word-boundary
  based (not line-count based) to stay deterministic for snapshot tests.

### 5. Reliability / edge cases (all render nothing, never error)

- Agent with no associated plan (e.g. generic/utility agents).
- Plan path resolves but the file is missing/deleted or unreadable.
- Frontmatter present but no `goal`, blank `goal`, or non-string `goal`.
- Unparseable / malformed frontmatter (tolerated, like tier reads).
- Phase bead whose parent/epic is missing or whose `design` is empty.
- Goal identical to the bead title/description (still show it — the plan file is the source of truth and stays fresh if
  edited, unlike the bead's creation-time copy).
- Selection churn / stale results: rely on the existing `is_current(...)` generation guard in the summary worker so a
  late goal resolution never paints onto a different selected agent.

## Testing

- **Unit — goal reader** (`read_plan_goal` / `read_plan_goal_from_content`): present single-line goal; folded/block
  multi-line goal normalized to one line; missing `goal`; blank/whitespace `goal`; non-string `goal`; no frontmatter;
  malformed YAML; missing file.
- **Unit — resolver** (`resolve_agent_plan_goal`): direct `sdd_plan_path`/`plan_path` agent; epic-bead agent via
  `design`; phase-bead agent via parent `design`; agent with neither (→ `None`); missing plan file (→ `None`); cache hit
  does not re-read after first resolution; mtime change invalidates.
- **Rendering** (header build): `Goal:` appears with a resolvable goal; absent when `plan_goal` is `None`; absent on the
  cheap (no-summary) path; long goal is truncated with `…`; multi-line authored goal renders on one normalized line.
- **Visual** — add/refresh an ACE PNG snapshot (`tests/ace/tui/visual/snapshots/png/`) exercising an agent whose plan
  has a goal, to lock in the new field's appearance; run `just test-visual` and accept the new golden with
  `--sase-update-visual-snapshots`.
- Full `just check` (install first, since workspaces are ephemeral).

## Documentation

- Update the Agents-tab metadata-field description in `docs/ace.md` to mention the new `Goal:` field.
- No keymap and no `sase ace` _option_ is added, so the `?` help popup, the keybinding footer, and the help-modal width
  rules are not affected — but confirm this during implementation against `src/sase/ace/CLAUDE.md`.

## Risks & mitigations

- **Event-loop stalls from disk/bead I/O.** Mitigated by doing all resolution inside the already-off-loop
  `build_detail_header_summary` worker, plus an mtime-keyed bounded cache; the hot j/k path and per-row rendering are
  untouched.
- **Wrong plan associated with an agent.** Mitigated by an explicit, tested priority chain that mirrors the established
  `family_sase_plan()` ordering and the existing bead-resolution machinery, rather than a new ad hoc heuristic.
- **Over-strict reads hiding valid goals.** Mitigated by using the tolerant Python reader (not strict Rust validation),
  consistent with existing lenient tier reads.
- **Snapshot flakiness.** Mitigated by deterministic, character-based truncation, whitespace normalization, and avoiding
  emoji in the rendered field.
- **Boundary discipline.** This is presentation-only glue plus a tolerant frontmatter read that matches existing
  Python-side precedent, so it does not cross into `sase-core`; if a future frontend needs the authoritative validated
  goal, that remains the strict `plan_validate` binding.
