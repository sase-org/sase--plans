---
create_time: 2026-07-08 14:44:44
status: wip
prompt: 202607/prompts/agent_reply_subsection_id.md
tier: tale
---
# Plan: Show the family `<id>` in generic "AGENT" sub-sections of the AGENT REPLY panel

## Problem

On the "Agents" tab, when you select a family root that has follow-up members, the metadata panel renders a consolidated
**AGENT REPLY** section. Each member (the main agent plus every follow-up) gets its own **phase divider** sub-section
header, e.g.:

```
──────────────────────────────────────────────────
AGENT REPLY

─── PLANNER ─── 14:23:45 ──────────────────────────
...planner reply...
─── CODER ─── 14:31:02 ────────────────────────────
...coder reply...
```

Plan-chain roles get descriptive sub-section names (`PLANNER`, `CODER`, `QUESTIONS`, `EPIC`, `LEGEND`, `COMMIT`,
`PLANNER (round N)`). But **custom family members** — created with the two-argument naming directive `%n(foo, bar)` /
`%name(foo, bar)` — all collapse to the same generic label:

```
─── AGENT ─── 14:23:45 ────────────────────────────
─── AGENT ─── 14:31:02 ────────────────────────────
```

When a family has multiple custom members (`foo--bar`, `foo--baz`, …) their sub-sections are **indistinguishable**. You
cannot tell which reply belongs to which member, even though the row list clearly shows `foo--bar` vs `foo--baz`.

The desired behavior: the generic "AGENT" sub-section should always carry the member's **`<id>`** — the family suffix
token that distinguishes it from its neighbors — so the dividers read `AGENT (bar)`, `AGENT (baz)`, etc., mirroring the
descriptive naming the plan-chain phases already enjoy.

## Where the label comes from

The single source of the sub-section label is `get_phase_label(agent)` in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_content.py`. It maps an agent's `role_suffix` /
`agent_family_role` to a display string, returning the descriptive plan-chain labels and falling back to the literal
`"AGENT"` for anything unrecognized.

It is consumed by the two consolidated-reply render paths, each of which iterates the main agent plus every
`followup_agents` entry and passes the label to `render_phase_divider(label, start_time)`:

- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_render.py` — normal render (`_update_display_impl`).
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_hints.py` — file-path-hint render (`update_display_with_hints`).

Phase dividers appear **only** when `agent.followup_agents` is non-empty — i.e. a family with ≥2 members — and every
follow-up is an `is_family_member_child` row. So every generic-"AGENT" divider belongs to a family member whose
distinguishing `<id>` is always derivable from its family suffix.

## The `<id>`

The `<id>` is the **family suffix token** — the part of the member's name after the `--` separator (equivalently,
`role_suffix` with its leading separator stripped):

| Member launched via         | `role_suffix` | `agent_family_role` | `<id>`     |
| --------------------------- | ------------- | ------------------- | ---------- |
| `%n(foo, bar)`              | `--bar`       | `bar`               | `bar`      |
| `%n(foo, reviewer)`         | `--reviewer`  | `reviewer`          | `reviewer` |
| bare root promoted to `--0` | `--0`         | `root`              | `0`        |

For a custom `%n(foo, bar)` member, `agent_family_role_for_suffix` returns no recognized plan-chain role, so
`get_phase_label` hits the `"AGENT"` fallback today.

### Interaction with the just-shipped bare-family-root `--0` feature

The recently merged change (commit `3ffa4c7`) gives a bare family root (`foo`) the reserved `--0` display slot once a
sibling is dynamically attached, so the family renders as `foo--0` + `foo--bar`. That promoted root is set in memory to
`role_suffix="--0"`, `agent_family_role="root"`, `plan_chain_root=False`.

There is a pre-existing wart here: the numeric tokens `0`/`1` classify as the "root question" continuation sequence
inside `plan_chain._parse_plan_chain_suffix`, so `get_phase_label` currently labels the promoted root **`QUESTIONS`** —
even though it is a plain custom agent, not a question member. For a coherent custom family, the root divider should
read `AGENT (0)` to match its `foo--0` row name and its `AGENT (bar)` sibling. This plan fixes that as part of the same
visual story.

Crucially, the promoted bare root is distinguishable from a genuine plan-chain workflow root: plan-chain roots always
carry `plan_chain_root=True` (and a `--plan`/`--code`/… phase suffix), while the promoted bare root has
`plan_chain_root=False`. And genuine question members carry `agent_family_role="q"`, never `"root"`. So the guard
`agent_family_role == "root" and not plan_chain_root` isolates exactly the promoted-root case without touching questions
or plan-chain rendering.

## Recommended approach — enrich the display fallback (presentation-only)

Keep the change in the TUI display layer plus one small, well-named suffix helper. No on-disk mutation, no change to
name resolution / `%wait` / registry, and no change to the shared suffix classifier `_parse_plan_chain_suffix` (whose
broad blast radius across the questions/feedback flows we deliberately avoid perturbing).

### 1. A single source of truth for the id token

Add a small public helper next to the other family-suffix helpers in `src/sase/plan_chain.py` (e.g.
`agent_family_suffix_token`) that returns the bare token for a family suffix (`--bar` → `bar`, `--0` → `0`, legacy
`.xyz`/`-2` → `xyz`/`2`), or `None` for an empty / separator-only suffix. This is pure, trivial string logic that
belongs beside `_agent_family_suffix` / `agent_family_base`; it is presentation-agnostic and not Rust-core domain
behavior.

### 2. Teach `get_phase_label` to emit the id

Rework the fallback so that, instead of a bare `"AGENT"`, it returns `f"AGENT ({token})"` when a token is derivable:

- Add an `is_promoted_root = agent.agent_family_role == "root" and not agent.plan_chain_root` guard on the `role == "q"`
  branch, so the promoted bare root falls through the plan-chain branches to the id fallback (yielding `AGENT (0)`)
  instead of being mislabeled `QUESTIONS`. All other branches are unaffected: a promoted root's numeric suffix never
  matches the phase-label or feedback-round branches.
- Keep every existing plan-chain label exactly as-is (`PLANNER`, `CODER`, `QUESTIONS`, `EPIC`, `LEGEND`, `COMMIT`,
  `PLANNER (round N)`).
- At the end, return `f"AGENT ({token})"` if `agent_family_suffix_token(agent.role_suffix)` yields a token, else the
  bare `"AGENT"` (the no-suffix case — e.g. a root left bare because an explicit `foo--0` sibling suppressed promotion).

`render_phase_divider` is untouched: it renders the returned string uniformly, so the label needs no new plumbing.

### 3. (Secondary, for cross-panel consistency) the context-members compact label

`src/sase/ace/tui/agent_context_members.py::_compact_role_label` powers the compact family labels in the context/tools
side panels. It already surfaces the custom token for `%n(foo, bar)` members (via its `agent_family_role` fallback →
`bar`), but it inherits the same promoted-root wart and shows `q`. Add the same
`agent_family_role == "root" and not plan_chain_root` early return there (reusing `agent_family_suffix_token`) so the
side panel shows `0` for the promoted root, keeping both panels telling the same story. This is a clearly-scoped
secondary touch; the primary feature is self-contained in items 1–2 and this can be dropped without affecting it.

## Design rationale — intuitive, reliable, beautiful

- **Intuitive.** The `<id>` shown in the divider is exactly the suffix a user sees in the row name (`foo--bar` → the
  divider says `AGENT (bar)`; `foo--0` → `AGENT (0)`). Row list, AGENT REPLY dividers, and the context-members panel all
  agree on the same identity.
- **Reliable.** Presentation-only; no disk writes, no reference/registry impact, and the shared suffix classifier is
  left untouched so the questions/feedback/plan flows keep their exact behavior. The promoted-root guard keys off
  `plan_chain_root`, which unambiguously separates the (in-memory) bare-root promotion from real plan-chain workflow
  roots.
- **Beautiful.** `AGENT (bar)` mirrors the existing `PLANNER (round N)` parenthetical convention — same uniform
  `bold #AF87FF` divider styling, `AGENT` kept uppercase like the other role labels, and the token kept in its own case
  (lowercase, as `(round 2)` already is). The result reads as one consistent family of sub-section headers rather than a
  bolted-on annotation.

### Format decision

`AGENT (<id>)` is chosen for consistency with `PLANNER (round N)`. Alternatives considered and rejected for the default:
bare `AGENT <id>` (less structured, easy to misread against the divider rule), `AGENT · <id>` / `AGENT [<id>]` (novel
punctuation with no precedent in the panel), and a distinctly-styled token run (would require threading a structured
label through `render_phase_divider` and would diverge from the uniform `(round N)` treatment). The implementer may
revisit if desired, but the parenthetical uniform-style form is the recommended baseline.

### Reserved-token edge cases (documented, expected)

If a user names a member with a reserved role word — `%n(foo, plan)`, `%n(foo, code)`, `%n(foo, q)`, etc. — that member
already renders as the corresponding phase label (`PLANNER`, `CODER`, `QUESTIONS`) rather than `AGENT (...)`. This is
pre-existing suffix behavior and is left as-is. An explicit `%n(foo, 0)` member (rare; `--0` is a reserved slot the
auto-allocator skips) classifies as a question member and is likewise out of scope.

## Testing

- **Unit — `get_phase_label`** (`tests/ace/tui/widgets/test_agent_display_phase_labels.py`):
  - Custom member `role_suffix="--bar", agent_family_role="bar"` ⇒ `AGENT (bar)`.
  - Custom named member `role_suffix="--reviewer", agent_family_role="reviewer"` ⇒ `AGENT (reviewer)`.
  - Promoted bare root `role_suffix="--0", agent_family_role="root", plan_chain_root=False` ⇒ `AGENT (0)` (regression
    guard against the old `QUESTIONS`); likewise `--1` ⇒ `AGENT (1)`.
  - Plan-chain workflow root `role_suffix="--plan", agent_family_role="root", plan_chain_root=True` ⇒ `PLANNER`
    (unchanged).
  - Genuine question member `role_suffix="--0", agent_family_role="q"` ⇒ `QUESTIONS` (unchanged).
  - `role_suffix=None` ⇒ `AGENT` (unchanged bare fallback).
  - Update `test_unknown_suffix` (`role_suffix=".xyz"`): now `AGENT (xyz)` — an intended behavior change.
  - All existing plan-chain and feedback-round assertions must still pass unchanged.
- **Unit — `agent_family_suffix_token`** (new cases, e.g. in `tests/test_plan_chain*.py` or a dedicated module):
  `--bar`→`bar`, `--0`→`0`, `--reviewer`→`reviewer`, `.xyz`→`xyz`, `-2`→`2`, `None`→`None`, `""`→`None`, `"--"`→`None`.
- **Render integration**: build an agent with `followup_agents` for a custom family (root `foo--0` + `foo--bar`) and
  render through the test double (`FakePromptPanel` in `tests/ace/tui/widgets/_agent_display_helpers.py`), asserting
  both the normal and the hints render paths emit `AGENT (0)` and `AGENT (bar)` in the AGENT REPLY section.
- **Context-members (only if item 3 is included)**: assert the promoted root's compact label becomes `0` (was `q`); a
  custom `%n(foo, bar)` member still yields `bar`.
- **Visual/PNG snapshots**: run `just test-visual`. No current golden is expected to exercise a custom-family AGENT
  REPLY divider, so this should stay green. Add a new golden only if an existing fixture naturally renders a
  multi-member custom family; otherwise the divider text is fully asserted at the render-integration level and a
  gratuitous snapshot is not warranted.
- Run `just check` before completion (per repo policy), preceded by `just install`.

## Risks / open questions

- **Scope of the promoted-root fix.** Fixing the `QUESTIONS`→`AGENT (0)` mislabel is in-scope because it is the exact
  scenario the new `--0` slotting created and it would otherwise look broken next to the `AGENT (bar)` sibling. It is
  confined to the display layer via the `plan_chain_root` guard; the shared classifier is not modified.
- **Format bikeshed.** `AGENT (<id>)` is the recommended baseline; final punctuation/casing is easy to adjust and worth
  a quick confirmation.
- **Secondary panel coupling.** Including item 3 keeps the context-members panel consistent but widens the change
  slightly; it is cleanly separable from the core feature.

## Out of scope

- Changing how `%n(parent, suffix)` resolves or allocates suffixes, or how references (`%wait`/`@`) resolve.
- Relabeling recognized plan-chain phases or reserved-token members (`plan`/`code`/`q`/…).
- Any Rust-core change: the label is presentation-only and the suffix token helper is pure Python beside its existing
  siblings in `plan_chain.py`.
