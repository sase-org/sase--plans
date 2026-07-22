---
tier: tale
title: Fix sase ace crash when killing-and-editing a clan member
goal: Killing-and-editing a clan member (e.g. sase-8k.2) relaunches it with forced
  name reuse of its full hood name instead of crashing sase ace.
create_time: 2026-07-22 12:27:03
status: wip
---

- **PROMPT:** [202607/prompts/clan_member_kill_and_edit_crash.md](prompts/clan_member_kill_and_edit_crash.md)

# Fix `sase ace` crash when killing-and-editing a clan member

## Summary

Killing-and-editing a clan member (e.g. `sase-8k.2` in the `sase-8k` clan) crashes `sase ace` with:

```
ValueError: Cannot relaunch agent 'sase-8k' as a member of clan 'sase-8k':
expected the 'sase-8k.<suffix>' hood.
```

The kill-and-edit code path incorrectly normalizes the clan member's name `sase-8k.2` down to the bare clan name
`sase-8k` before forcing name reuse. `sase-8k` is not a valid member of its own hood, so the downstream clan-member
rewrite rejects it and the TUI crashes.

## Root cause

The crash originates in `prepare_kill_and_edit_prompt`
(`src/sase/ace/tui/actions/agent_workflow/_entry_name_prompts.py`):

```python
def prepare_kill_and_edit_prompt(
    raw_prompt: str,
    agent_name: str | None,
) -> str:
    from sase.plan_chain import agent_family_base

    replacement_name = agent_family_base(agent_name) or agent_name
    return force_name_reuse_in_prompt(raw_prompt, replacement_name=replacement_name)
```

`agent_family_base(agent_name)` is meant to normalize an **agent-family phase** member (which uses the canonical `--`
separator, e.g. `sase-8a.3--plan`) down to its family base so kill-and-edit reuses the base name. This normalization was
added in commit `48abe26ea` ("fix(agent): retry family phases from base names").

However, `agent_family_base` (via `_split_agent_family_name` in `src/sase/plan_chain.py`) also treats the **`.`
separator** as a _legacy_ family-phase separator. Its separator list is:

```python
separators = [AGENT_FAMILY_SEPARATOR, "."]   # AGENT_FAMILY_SEPARATOR == "--"
```

A clan member's hood suffix looks exactly like a legacy dotted family-phase suffix. For `sase-8k.2`:

- `rpartition("--")` finds no `--`, so the `--` branch is skipped.
- `rpartition(".")` splits into `("sase-8k", ".", "2")`.
- `canonical_plan_chain_suffix(".2")` returns `"--2"` (a valid plan-chain root/phase suffix), so the split succeeds.
- `agent_family_base("sase-8k.2")` therefore returns `"sase-8k"`.

Clan members (`<clan>.<suffix>`) and legacy dotted family members (`<family>.<suffix>`) are **indistinguishable by name
alone**; only context disambiguates them. So the clan member `sase-8k.2` is misread as a family phase whose base is
`sase-8k`.

The wrong `replacement_name = "sase-8k"` then flows into `force_name_reuse_in_prompt`. Because the stored prompt
declares clan membership via the `%id` join form (`%id(2, clan=sase-8k, bead=sase-8k.2)`), `force_name_reuse_in_prompt`
routes to `rewrite_prompt_clan_member_name(prompt, "sase-8k", current_agent_name="sase-8k", force_reuse=True)`. That
function validates the reuse name against the clan hood:

```python
prefix = f"{clan_name}."          # "sase-8k."
if not agent_name.startswith(prefix) or not agent_name.removeprefix(prefix):
    raise ValueError(...)         # "sase-8k" does not start with "sase-8k." -> crash
```

`"sase-8k"` is not within the `"sase-8k."` hood, so it raises `ValueError` and `sase ace` crashes.

### Reproduction (confirmed)

```python
from sase.ace.tui.actions.agent_workflow._entry_name_prompts import prepare_kill_and_edit_prompt

raw = "#gh:gh_sase-org__sase\n%id(2, clan=sase-8k, bead=sase-8k.2)\n%model:@small_phase_worker\n%auto"
prepare_kill_and_edit_prompt(raw, "sase-8k.2")
# -> ValueError: Cannot relaunch agent 'sase-8k' as a member of clan 'sase-8k': ...
```

## The fix

Restrict the family-base normalization in `prepare_kill_and_edit_prompt` to **canonical `--` family phases** — the only
names commit `48abe26ea` intended to normalize. A clan member lives in a `<clan>.<suffix>` hood; its dotted suffix is
not a family phase, so its full name must be preserved so that forced reuse targets the actual agent.

Edit `prepare_kill_and_edit_prompt` in `src/sase/ace/tui/actions/agent_workflow/_entry_name_prompts.py` to only apply
`agent_family_base` when the name contains the canonical family separator (`AGENT_FAMILY_SEPARATOR == "--"`):

```python
def prepare_kill_and_edit_prompt(
    raw_prompt: str,
    agent_name: str | None,
) -> str:
    """Return the exact editable prompt for a kill-and-edit relaunch.

    Both the focused-agent and marked-agent routes use this boundary so name
    aliases, templates, and already-forced directives cannot drift between
    the single- and multi-pane workflows.
    """
    from sase.plan_chain import AGENT_FAMILY_SEPARATOR, agent_family_base

    # Only canonical ``--`` family phases normalize to their family base. A
    # clan member lives in a ``<clan>.<suffix>`` hood, and its dotted suffix is
    # NOT a family phase: ``agent_family_base`` would misread it as one (e.g.
    # ``sase-8k.2`` -> ``sase-8k``), forcing reuse of the bare clan name and
    # crashing ``rewrite_prompt_clan_member_name``. Preserving the full clan
    # member name lets forced reuse target the actual agent.
    replacement_name = agent_name
    if agent_name and AGENT_FAMILY_SEPARATOR in agent_name:
        replacement_name = agent_family_base(agent_name) or agent_name
    return force_name_reuse_in_prompt(raw_prompt, replacement_name=replacement_name)
```

With this change:

- Clan member `sase-8k.2` (no `--`) keeps its full name; `force_name_reuse_in_prompt` produces
  `%id(!2, clan=sase-8k, bead=sase-8k.2)` (correct forced reuse of `sase-8k.2`) — no crash.
- Canonical family phase `sase-8a.3--plan` (has `--`) still normalizes to base `sase-8a.3` — unchanged behavior.
- Family phase within a clan `sase-8k.2--plan` (has `--`) still normalizes to base `sase-8k.2` — unchanged behavior.

### Why the fix is localized to the kill-and-edit path

The crash is unique to the **forced-reuse** path. `prepare_kill_and_edit_prompt` passes the normalized base as the
**exact** name to reuse, and `rewrite_prompt_clan_member_name` requires an exact-name member of the hood, so the bare
clan name is rejected.

`agent_family_base` must **not** be changed globally. Two sibling relaunch paths deliberately rely on its `.`-splitting
to derive the clan base when allocating a **new** hood-sibling retry name, and they neither crash nor want different
behavior:

- `_retry_edit_agent` (`src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py`): for `sase-8k.2` it computes
  `agent_family_base("sase-8k.2") == "sase-8k"`, then `allocate_retry_name("sase-8k")` yields a fresh hood sibling like
  `sase-8k.r0`. Allocating from the full name would instead nest a child hood (`sase-8k.2.r0`), which is wrong.
- `retry_mobile_agent` (`src/sase/integrations/_mobile_agent_lifecycle.py`): same allocate-a-new-name pattern.

Because those paths allocate fresh names (not forced reuse) and depend on the current behavior, the fix is scoped to
`prepare_kill_and_edit_prompt`, which both kill-and-edit routes (focused and marked/bulk) funnel through:

- Focused route: `_entry_relaunch.py` `_kill_and_edit_agent` -> `prepare_kill_and_edit_prompt`.
- Marked/bulk route: `src/sase/ace/tui/actions/agents/_marking_kill.py` -> `prepare_kill_and_edit_prompt`.

## Tests

Add regression coverage to `tests/ace/tui/test_retry_edit_agent_name.py`.

1. Extend the existing `test_prepare_kill_and_edit_prompt_contract` parametrization with a clan-member case (and keep
   the family-within-clan case to lock in that `--` phases still normalize):

   ```python
   (
       "%id(2, clan=sase-8k, bead=sase-8k.2)\nDo work",
       "sase-8k.2",
       "%id(!2, clan=sase-8k, bead=sase-8k.2)\nDo work",
   ),
   (
       "%id(2, clan=sase-8k, bead=sase-8k.2)\nDo work",
       "sase-8k.2--plan",
       "%id(!2, clan=sase-8k, bead=sase-8k.2)\nDo work",
   ),
   ```

2. Add a focused-route integration test alongside `test_kill_and_edit_family_phase_forces_base_name_reuse` that drives
   `_kill_and_edit_agent` for a clan member and asserts the relaunch preserves clan hood membership (mirroring the
   `_App`/`_Agent` harness already used in that file):

   ```python
   def test_kill_and_edit_clan_member_preserves_hood() -> None:
       app = _App(
           _Agent(
               "%id(2, clan=sase-8k, bead=sase-8k.2)\nDo work",
               agent_name="sase-8k.2",
           )
       )

       app._kill_and_edit_agent()

       assert app.launched == (
           "%id(!2, clan=sase-8k, bead=sase-8k.2)\nDo work",
           "/tmp/proj/proj.sase",
           "branch",
           False,
       )
   ```

   Match the exact `_App`/`_Agent` construction and assertion tuple shape used by the neighboring tests in the file; the
   strings above reflect the confirmed output.

## Validation

From a clean workspace checkout of this repo:

```bash
just install     # ephemeral workspaces need deps installed before checks
just check       # ruff + mypy + fast pytest
```

Also run the focused test module to confirm the regression is covered:

```bash
.venv/bin/python -m pytest tests/ace/tui/test_retry_edit_agent_name.py -q
```

All existing cases in `test_retry_edit_agent_name.py` (40 at time of writing) must continue to pass, plus the new
clan-member cases.

## Files touched

- `src/sase/ace/tui/actions/agent_workflow/_entry_name_prompts.py` — guard family-base normalization behind the
  canonical `--` separator in `prepare_kill_and_edit_prompt`.
- `tests/ace/tui/test_retry_edit_agent_name.py` — add clan-member kill-and-edit regression cases.

## Out of scope

- Changing `agent_family_base` / `_split_agent_family_name` semantics or the `.`-separator handling in
  `src/sase/plan_chain.py` (sibling retry/mobile paths depend on it, as described above).
- The retry-edit (`_retry_edit_agent`) and mobile retry (`retry_mobile_agent`) paths, which allocate new hood-sibling
  names and do not exhibit the crash.
