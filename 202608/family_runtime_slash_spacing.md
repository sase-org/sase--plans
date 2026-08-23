---
tier: tale
title: Tighten the family runtime separator to a bare slash
goal: "The active agent-family runtime suffix renders the current shell runtime and the
  family total joined by a bare `/` with no surrounding spaces, so the pair reads as one
  compact value instead of two loosely separated ones.

  "
size: small
proposed_by: bbugyi200.athena.0bo.f0
---

# Plan: Tighten the family runtime separator to a bare slash

## Context

Commit `184fa9aed` ("feat(ace): show current family shell runtime") made an active
sequential-family container row render two elapsed values in its runtime suffix: the
currently running concrete shell's runtime, then the family's aggregate runtime. It
joined them with `" / "` — a slash padded by a space on each side.

In review the padded separator reads as two independent columns rather than one "current
of total" reading, and it costs two extra cells in the right-aligned suffix slot on
every active family row. The requested change is purely presentational: drop the
surrounding spaces so the suffix renders `1m05s/3m05s` instead of `1m05s / 3m05s`.

Nothing about which shell is selected, how either duration is computed, when the
two-value form appears, or how rows are patched and cached changes. The separator is a
single literal in one renderer, so the work is the literal plus the assertions,
documentation, and PNG goldens that pinned the old spacing.

## User-visible contract

- On an active sequential-family container row, the live suffix renders as
  `🏃‍♂️ <current-shell-runtime>/<family-total-runtime>` — no space before or after the
  slash.
- Both duration values, their styles, the `🏃‍♂️` marker, and the marker's trailing space
  are unchanged. Only the separator between the two durations changes.
- The two-value form still appears under exactly the same conditions as today: a
  sequential family with a concrete in-flight agent or monitor shell that has a usable
  runtime. Every case that renders a single value today still renders exactly one value
  with no slash.
- Row width shrinks by two cells on affected rows. Right-edge alignment continues to be
  derived from the rendered suffix width, so no padding constant needs adjusting.
- Standalone agent rows, workflow-step rows, clan containers, parallel families,
  completed-family timestamp suffixes, unread markers, user-paused markers, and
  file-change pencils are untouched.

## Implementation

### 1. Change the separator literal

In `src/sase/ace/tui/widgets/_agent_list_render_layout.py`, inside
`build_runtime_suffix()`, the current-shell branch appends the left value and then
`suffix.append(" / ")`. Change that appended string to `"/"`.

Leave the append unstyled, exactly as it is today. The separator inherits the `Text`
default rather than `_RUNTIME_ELAPSED_STYLE`, which keeps the two bold duration values
as the salient tokens and the joiner visually subordinate. Do **not** take this change
as an opening to introduce a dedicated separator style constant, change the elapsed
style, or restyle the marker — that is a different decision than the one being made
here, and widening the diff would force a second round of golden regeneration.

No other production file changes. In particular:

- `current_family_shell_row()` and `compute_leaf_row_runtime()` are untouched; shell
  selection and duration computation are unaffected by spacing.
- `_agent_list_render_cache.py` is untouched. The render-cache key already covers the
  inputs that can change the suffix; the separator is a constant, not cache state.
- `AgentList.patch_active_runtime_rows()` and `patch_agent_row()` are untouched. The
  tick path re-renders the whole row through `build_runtime_suffix()` rather than
  splicing text, so it picks up the new separator with no changes.

### 2. Update the assertions that pinned the padded separator

These are the checked-in expectations that encode `" / "`. Update each to the unpadded
form:

- `tests/ace/tui/widgets/test_agent_list_runtime_rendering.py`
  - the promoted-root case: `"🏃‍♂️ 3m05s / 3m05s"` → `"🏃‍♂️ 3m05s/3m05s"`
  - the continuation case: `"🏃‍♂️ 1m05s / 3m05s"` → `"🏃‍♂️ 1m05s/3m05s"`
  - the nested-monitor case: `"🏃‍♂️ 2m / 2m"` → `"🏃‍♂️ 2m/2m"`
  - the completed-family case asserts `" / " not in suffix.plain`; tighten it to
    `"/" not in suffix.plain`. This is a strictly stronger negative check and is safe:
    neither `format_compact_duration()` nor the completed-row timestamp formats
    (`%H:%M:%S`, `%b %-d`, `%b %-d '%y`) can emit a slash, so a bare `/` in a suffix can
    only come from the two-value family form.
- `tests/ace/tui/widgets/test_agent_list_runtime_patching.py` — the tick test's
  before/after expectations (`"🏃‍♂️ 1m59s / 3m59s"`, `"🏃‍♂️ 2m / 4m"`) and the
  collapsed-vs-expanded parity test's two `"🏃‍♂️ 1m05s / 3m05s"` expectations.
- `tests/ace/tui/widgets/test_agent_render_cache.py` — the four expectations in the two
  cache-invalidation tests (`"🏃‍♂️ 5m / 5m"`, `"🏃‍♂️ 1m / 4m"`, `"🏃‍♂️ 5m / 5m"`,
  `"🏃‍♂️ 3m / 3m"`).

Do not weaken any of these to substring or regex checks. Full-string `.plain`
comparisons are what make the spacing itself testable; that is precisely the property
under change here.

Before editing, re-derive this list from the tree rather than trusting it verbatim —
`grep -rn ' / ' tests/ace/tui/` narrowed to the runtime suites — in case other
assertions land between this plan and its implementation.

The visual test in `tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py`
asserts the two duration tokens (`1m05s`, `3m05s`) individually against the exported
SVG, never the joined string, so it needs no edit. Confirm that during verification
rather than assuming it: the durations are styled and the separator is not, so they
remain distinct SVG text runs.

### 3. Regenerate the two family-runtime PNG goldens

The suffix is right-aligned, so removing two cells shifts the rendered row. Both goldens
added by `184fa9aed` must be regenerated:

- `tests/ace/tui/visual/snapshots/png/agents_running_family_runtime_collapsed_120x40.png`
- `tests/ace/tui/visual/snapshots/png/agents_running_family_runtime_expanded_120x40.png`

Regenerate with the repo's own update flag, then re-run the same node without the flag
to confirm exact comparison passes:

```bash
just install   # ephemeral workspace: required before any other recipe
uv run pytest tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py \
  -m visual -k running_family_current_runtime --sase-update-visual-snapshots
uv run pytest tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py \
  -m visual -k running_family_current_runtime
```

Note the two gotchas the previous implementation hit: visual tests are excluded by the
default marker expression, so `-m visual` is mandatory, and the fixture pins the clock
through the `agent_time` module, so nothing in the render path may import `local_now`
directly.

Regenerate **only** these two goldens. `just test-visual` currently fails broadly on
this machine (~364 unrelated mismatches, tracked as standing backlog on task `sase-r5`);
that pre-existing drift is out of scope and must not be "fixed" by mass-updating
goldens. Confirm the two targeted goldens pass exact comparison and leave every other
snapshot file untouched — `git status` at the end must show no PNG changes beyond these
two.

### 4. Update the documentation

Two prose lines spell the format out with spaces; both must match the new rendering:

- `docs/ace.md` — the runtime-suffix section:
  `🏃‍♂️ <current-shell-runtime> / <family-total-runtime>` →
  `🏃‍♂️ <current-shell-runtime>/<family-total-runtime>`.
- `docs/agent_families.md` — the family-runtime paragraph: the same substitution.

Keep the surrounding explanatory sentences (which side is the concrete shell, which is
the aggregate) as-is; only the format literal changes. Respect the existing wrap width
when the shorter literal changes where the line breaks.

No `sase/memory/` file documents this format, so no memory edit — and therefore no
`sase memory init` — is in scope.

## Verification

- `uv run pytest tests/ace/tui/widgets/test_agent_list_runtime_rendering.py tests/ace/tui/widgets/test_agent_list_runtime_patching.py tests/ace/tui/widgets/test_agent_render_cache.py tests/ace/tui/widgets/test_agent_list_runtime_compute.py tests/ace/tui/models/test_agent_family_members.py`
  — all pass.
- The two targeted visual nodes pass exact PNG comparison after regeneration.
- `just check` passes. If its scoped lane escalates to the full suite, let it finish
  rather than substituting a narrower run.
- `git status` shows only: the one renderer literal, the three test files, the two
  documentation files, and the two regenerated PNG goldens.

## Risks and non-goals

- **Only real risk**: an assertion pinning `" / "` that this plan did not enumerate, in
  a suite the scoped test lane does not select. Step 2's re-grep is the mitigation, and
  `just check`'s escalation to the full suite is the backstop.
- **Non-goal**: restyling the separator, the durations, or the `🏃‍♂️` marker.
- **Non-goal**: changing which shell is chosen as "current", how either duration is
  computed, or when the two-value form appears.
- **Non-goal**: the broad `just test-visual` PNG drift already tracked by `sase-r5`.
