---
tier: tale
title: Reword two Launch Control setting labels
goal:
  Launch Control's default-model row reads "default model" and its runner-limit row
  reads "max runners", with every dependent string, test, and PNG golden updated to
  match.
size: small
proposed_by: bbugyi200.athena.05v
create_time: 2026-08-18 07:33:51
status: wip
---

# Reword two Launch Control setting labels

Rename two scalar-setting row labels in the ACE Launch Control panel (leader `,m`):

| Row                          | Current label    | New label       |
| ---------------------------- | ---------------- | --------------- |
| `llm_provider.default_model` | `launch model`   | `default model` |
| `max_running_agents`         | `running agents` | `max runners`   |

Both new labels are strictly better than what they replace:

- `default model` matches the config path it edits (`llm_provider.default_model`) and
  its own description strip ("Used when a launch has no explicit %model directive."), so
  the row, the path shown beneath it, and the concept now share one word.
- `max runners` says the value is a _cap_, which `running agents` did not. The old label
  read like a live count of currently-running agents — which is exactly what the ACE
  header (`79 [7/10 running ...]`) actually shows — so the row invited misreading a
  configured limit as a status readout. "Runners" also matches the vocabulary already
  used everywhere else in this feature: `%wait(runners=N)`, `runner_limit_override`,
  `RunnerLimitSettingRow`, `setting:runner_limit`.

The other four rows in the section (`epic lander`, `big epic lander`,
`big epic starts at`, `default effort`) are unchanged.

## Context

Launch Control's "Launch settings" section is built from four row dataclasses in
`src/sase/ace/tui/modals/models_panel_rows.py`. Each row carries a `label` field, and
rendering pads that label into a 22-column name cell (`_NAME_CELL` in
`models_panel_rendering_layout.py`). Both new labels are shorter than the cell, so no
layout, truncation, or column-width change is involved.

Two facts make this bigger than a two-string edit:

1. **`label` is not display-only.** It flows into the alias-history modal title, the
   edit/reset preview modal's target label, temporary-override toasts, and — via
   `build_model_setting_commit_offer` — the git commit subject written when the user
   edits a launch model setting from the panel. Those follow the rename automatically
   and correctly; see "Expected downstream text changes" below.
2. **One user-facing string duplicates a label instead of reading it.**
   `models_panel_history.py` hardcodes
   `"running agents is not an alias; it has no run history."` rather than interpolating
   `row.label`. Renaming the row without touching that string would leave a warning
   naming a row the panel no longer has. Step 2 removes the duplication so this class of
   drift cannot recur.

Jump hints (`'`) key off `row_id`, not `label`, and row order is a fixed tuple, so
neither is affected.

## Scope

**In scope**

- The two label strings.
- The hardcoded warning strings that duplicate scalar-row labels.
- Tests and PNG goldens that assert or render those labels.

**Out of scope** — do not change these, and do not "fix" them opportunistically:

- The runner-limit modal chrome: the `Max Running Agents` action-modal title, the
  `Edit Max Running Agents` preview title, the `"max running agents"` preview target
  label, `chore: update max running agents`, and the `"Enter a running-agent limit."` /
  `"The running-agent limit must be at least 1."` parse errors. These name the config
  field `max_running_agents`, which is not being renamed, and "Max Running Agents" is
  already consistent in meaning with "max runners".
- The row description strips in `models_panel_rendering_descriptions.py` ("Maximum
  number of agents admitted to run at once.", "Temporary maximum running-agent limit.").
  Both remain accurate.
- Every other "running agents" string in ACE (the `@@@` / `!@` query tokens, the kill
  modal, `sase agent --list`). Those really do mean live agents.
- The config keys themselves (`llm_provider.default_model`, `max_running_agents`),
  module names, and dataclass names.

## Implementation

### Step 1 — Rename the two labels

In `src/sase/ace/tui/modals/models_panel_rows.py`:

- Line 78: `RunnerLimitSettingRow.label` default `"running agents"` → `"max runners"`.
- Line 104: the `_LAUNCH_SETTING_ORDER` entry `(DEFAULT_MODEL_FIELD, "launch model")` →
  `(DEFAULT_MODEL_FIELD, "default model")`.

Leave the `RunnerLimitSettingRow` docstring ("Captured global running-agent limit row")
alone — it describes the config field, not the label.

### Step 2 — Stop duplicating labels in the history warning

`ModelsPanelHistoryMixin._alias_history_entry_request` in
`src/sase/ace/tui/modals/models_panel_history.py` has three near-identical branches for
`DefaultEffortSettingRow`, `RunnerLimitSettingRow`, and
`BigEpicPhaseThresholdSettingRow`, each hardcoding a label that also exists on the row.
Collapse them into one branch that reads `row.label`:

```python
if isinstance(
    row,
    DefaultEffortSettingRow | RunnerLimitSettingRow | BigEpicPhaseThresholdSettingRow,
):
    self.notify(
        f"{row.label} is not an alias; it has no run history.",
        severity="warning",
    )
    return None
```

The rendered text for the two untouched rows (`default effort`, `big epic starts at`) is
byte-identical to today's, and the runner-limit row now says "max runners is not an
alias; it has no run history." Keep the `# type: ignore[attr-defined]` comment on
`self.notify`, matching the surrounding calls.

Verify the three row classes are still all imported in that module after the collapse
(they are used in the `isinstance` check), and that no import becomes unused —
`just lint` will catch it either way.

### Step 3 — Update the assertions that pin the old labels

These are the only test occurrences; `grep -rn 'launch model\|running agents' tests/`
returns others, but every remaining hit is either an unrelated docstring about live
agents or the `@large` alias-description fixture string `"Default launch model."`, which
belongs to a different feature and must not be touched.

- `tests/test_models_panel_alias_rendering.py:313` — `label="launch model"` constructor
  arg; `:334` — `assert "launch model" in line`; `:363` —
  `assert "running agents" in runner_line`.
- `tests/test_models_panel_bucket_navigation.py:57` —
  `_launch_setting_row(DEFAULT_MODEL_FIELD, "launch model")`.
- `tests/test_models_panel_history.py:144`, `:165`, `:236`, `:261` — the
  `"launch model"` fixture label; `:285` — the parametrize case
  `("setting:runner_limit", "running agents")` →
  `("setting:runner_limit", "max runners")`.
- `tests/_models_panel_provider_routing_helpers.py:77` —
  `(DEFAULT_MODEL_FIELD, "launch model")`.
- `tests/test_models_panel_runner_limit.py:129` —
  `assert "running agents" in runner_row`. Leave `:146`
  (`"Already-running agents continue"`) alone; that is out-of-scope modal copy.

Do **not** touch `tests/test_models_panel_runner_limit.py`'s other assertions on the
runner-limit modal, nor `tests/ace/tui/test_model_completion_panel_titles.py`,
`tests/test_models_panel_display.py:22`, or
`tests/test_xprompt_model_completion_aliases.py` — those assert the alias description
`"Default launch model."`, which is fixture data for a different surface.

### Step 4 — Regenerate the affected PNG goldens

The 48 `models_panel_*` goldens in `tests/ace/tui/visual/snapshots/png/` include every
view that renders the Launch settings section, so a subset of them will change.

Run the scoped visual lane first, without updating, to learn exactly which goldens move:

```bash
just test-visual -- -k models_panel
```

Inspect the failures' artifacts under `.pytest_cache/sase-visual/` and confirm each diff
is confined to the two renamed label cells and nothing else — no value-column shift, no
row reordering, no unrelated pixel drift. Then accept them:

```bash
just test-visual -- -k models_panel --sase-update-visual-snapshots
```

Use the `-k`-scoped form rather than bare `just update-visual-snapshots`; the latter
regenerates the whole 477-golden corpus and would silently absorb unrelated drift.

Goldens whose modals are constructed directly in the test (e.g.
`models_panel_edit_preview_120x40`, `models_panel_runner_limit_edit_preview_120x40`,
`models_panel_alias_history_*`) pass their own labels and should **not** change; if one
of them does change, stop and investigate rather than accepting it.

Then re-run the scoped lane clean to confirm it is green, and finally run the full
visual suite once (`just test-visual`) to prove nothing outside `models_panel` moved.

## Expected downstream text changes

All of these follow from Step 1 and are intended. Confirm each looks right rather than
treating any as an unintended regression:

- Alias-history modal title for the default-model row: `@large · launch model` →
  `@large · default model`.
- History warning when the default-model row points at a concrete model:
  `launch model is a concrete model, not an alias; ...` → `default model is ...`.
- History warning on the runner-limit row (Step 2):
  `running agents is not an alias; it has no run history.` →
  `max runners is not an alias; it has no run history.`
- Edit / reset preview modal target label for the default-model row
  (`Model Setting · launch model` → `· default model`).
- Temporary-override toasts naming the default-model row.
- Git commit subject offered after a persistent default-model edit:
  `chore: Update model setting launch model` →
  `chore: Update model setting default model` (from `build_model_setting_commit_offer`).
  This is an improvement: the subject now names the config path being written.

## Verification

1. `just install` first — this workspace may be stale.
2. `just check` for the lint gates and the diff-scoped test lane.
3. The scoped and full visual runs from Step 4 (`just test-visual` is excluded from both
   `just check` and `just check-full`, so it must be run explicitly).
4. `just check-full` before landing, via `/sase_monitor` — it outruns a single turn.
5. Confirm no stale label text remains:

   ```bash
   grep -rn '"launch model"\|"running agents"' src/ tests/
   ```

   This should return nothing. A broader `grep -rn 'launch model\|running agents'` will
   still return the out-of-scope hits catalogued above; that is expected.

## Done when

- Launch Control's first setting row reads `default model` and its sixth reads
  `max runners`, with the other four rows unchanged.
- No user-facing string in the Launch Control panel or its modals still names either row
  by its old label.
- The scalar-row history warning derives its text from `row.label`.
- `just check`, `just check-full`, and `just test-visual` are all green.
