---
tier: tale
title: Force `,x` name reuse only when the prompt declared `%id`
goal:
  ACE `,x` seeds a prompt that never declared `%id` verbatim — no injected
  `%id:!<auto-id>` and no "rewrite has no %id identity" refusal — while named, clan, and
  family-member relaunches keep their current identity rewrites.
size: medium
proposed_by: bbugyi200.athena.071
create_time: 2026-08-18 18:53:39
status: wip
---

# Force `,x` name reuse only when the prompt declared `%id`

## Problem

ACE Agents-tab `,x` (leader-mode `kill_and_edit`) is broken for agents whose prompt
never named them. It fails in two ways:

1. **Hard refusal.** When the focused row has no `agent_name` yet, `,x` raises
   `KillAndEditPromptError` and nothing is killed or seeded. Observed twice in
   `~/.sase/logs/tui.log` (2026-08-18 18:34:39 and 18:34:44):

   ```
   sase.agent.relaunch_prompt.KillAndEditPromptError: Cannot relaunch '(unnamed)':
   rewrite has no %id identity (rewrite produced '#gh:gh_sase-org__sase Did we ever
   fix the issue where `@/path/to/file` references in bead notes / descriptions were
   not…')
   ```

   The agent was `06y`
   (`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/18/20260818183434/`).
   Its `raw_xprompt.md` is a single line with no `%id`; its `agent_meta.json` name `06y`
   is an auto-allocated id. Both presses landed 5 s and 10 s after launch, before ACE's
   meta enrichment had put a name on the row, so `agent_name` was `None`.

2. **Silent forced reuse of an auto-allocated id.** Once the row _does_ carry
   `agent_name`, `,x` rewrites the prompt to `%id:!06y\n<original>`, forcing the
   relaunch to wipe and retake an id the user never asked for.

Confirmed against the real artifacts on this machine:

```
prepare_kill_and_edit_prompt(raw, "06y") -> '%id:!06y\n#gh:gh_sase-org__sase Did we…'
prepare_kill_and_edit_prompt(raw, None)  -> KillAndEditPromptError: rewrite has no %id identity
```

The intended rule is already documented. `docs/ace.md` (the `,x` rewrite table,
~line 2052) lists `no %id directive | unchanged — a fresh name is used`, and the
paragraph under it says "a prompt that never named its agent has nothing to reuse, so it
simply relaunches under a newly allocated name". Behavior drifted away from the docs;
this plan puts it back.

## How we got here (commits from the last week)

Reviewed every commit touching `,x` / kill-and-edit / forced name reuse:

- `dc4ca2057` (Aug 17)
  `fix(agent): restore forced agent-name-reuse launches for Agents-tab ,x kill-and-edit`
  — extracted `sase.agent.force_reuse_launch` and threaded `allow_force_reuse` through
  the `RUN_LAUNCH` durable payload.
- `13e9ccbc9` (Aug 17)
  `fix(agent): consume force-reuse plans on the durable launch path` — typed failures
  plus seam coverage.
- `b582f1180` (Aug 18) `feat(cli): add sase agent restart` — the CLI twin of `,x`; moved
  the relaunch-prompt helpers into `sase/agent/relaunch_prompt.py`.
- `cfdaa6577` (Aug 18) `feat(agent): make restart reuse names it used to refuse` —
  `sase agent restart` injects `%id:!NAME` when the stored prompt has none, reported as
  `name_reuse_source="injected"`.
- **`ea31a2b5b` (Aug 18) `fix(agent): rebuild real identity on kill-and-edit` — the
  regression.** Its family-root and clan-preservation work is correct and stays. But it
  also (a) switched the plain path from `force_name_reuse_in_prompt` (a no-op when the
  prompt has no `%id`) to `ensure_forced_name_reuse` (always injects `%id:!<name>`), and
  (b) added `_verify_kill_and_edit_prompt`, whose `_verify_plain_form` raises "rewrite
  has no %id identity" whenever the rewrite carries no name. Its own commit message
  records the change as "Prompts with no %id now get `%id:!<name>`". Together those
  produce both symptoms above.
- `daa095ec3` (Aug 18) `refactor(agent): split restart.py into focused modules` — pure
  move; only relevant because `plan_agent_restart` now lives in
  `src/sase/agent/_restart_planning.py`.

Nothing in the Aug 17 force-reuse-launch plumbing needs to change: after this fix fewer
relaunches request forced reuse at all, and the ones that do are unchanged.

## Change

### 1. `src/sase/agent/retry_prompt.py` — add a tolerant identity probe

Add a public helper:

```python
def prompt_has_id_directive(prompt: str) -> bool:
    """Whether *prompt* explicitly declares a top-level ``%id``/``%i`` with a name."""
```

Implement it with the same scan `force_name_reuse_in_prompt` already performs in this
module: bail early on `"%i" not in prompt`, protect fenced blocks and disabled regions,
then walk `_DIRECTIVE_PATTERN`. A match counts when its `_DIRECTIVE_ALIASES`-resolved
name is `"id"` **and** it carries an argument — `match.group(2) is not None` (the
`%id(...)` paren form) or `match.group(3) is not None` (the `%id:value` colon form).
Return `True` even when the directive already carries `!` (an already-forced `%id:!foo`
still counts as declared). Factor the shared protect/scan preamble so the two functions
cannot drift; keep both public because symvision forbids importing private symbols
across files.

The argument requirement is not incidental: a bare `%id` on its own line names nobody,
and `force_name_reuse_in_prompt` already leaves it alone
(`test_force_name_reuse_leaves_bare_and_missing_name_directives`). Today
`prepare_kill_and_edit_prompt("%id\nDo work", None)` raises "rewrite is missing forced
name reuse"; gating it out fixes that refusal too.

Two properties matter and must be preserved:

- **It agrees with the rewriter.** The gate is true exactly when
  `force_name_reuse_in_prompt` has something to mark, so we never gate on a directive
  the rewrite would ignore.
- **It never raises.** Do _not_ implement this with `extract_prompt_directives`. A scan
  of the 1200 most recent stored prompts in the local artifact store found 197 that
  raise `DirectiveError` today (model aliases such as `@medium_worker` / `@epic_lander`
  that were later removed from config). A parser-based gate would turn `,x` on any of
  those rows into a hard refusal.

### 2. `src/sase/agent/relaunch_prompt.py` — gate the rewrite

In `prepare_kill_and_edit_prompt`, before the existing branch selection:

```python
serial_family_member = bool(family_name and role_suffix and not is_family_root)
if not serial_family_member and not prompt_has_id_directive(raw_prompt):
    return raw_prompt
```

A prompt that never named its agent is returned byte-for-byte: no `%id` injection, no
`_verify_kill_and_edit_prompt` call, no `KillAndEditPromptError`. This fixes both
symptoms at once, because the `agent_name is None` case can no longer reach the
verifier.

Everything downstream of the gate is untouched: the family-root branch, the
serial-family-member branch, `_rewrite_family_member_or_preserve_clan`, the clan
fallback, and all of `_verify_kill_and_edit_prompt` keep their current behavior for
prompts that _do_ declare `%id`. Update the function docstring to state the rule.

### 3. One deliberate exception: serial non-root family members

`serial_family_member` rows keep the current `%id(!<suffix>, family=<family>)` rewrite
even when their stored prompt has no `%id` (e.g. raw prompt `Implement the plan` on
`sase-8u.4.2--code`). Rationale, stated here so it can be vetoed rather than discovered
later:

- The `family=` attachment is **structural**, not a user-requested name. It is
  reconstructed from row metadata (`agent_family`, `role_suffix`), never from the
  prompt, so dropping it would orphan the member from its family/plan chain rather than
  merely renaming it.
- The `!` is load-bearing there: `,x` kills or dismisses the old member but its name
  stays reserved in `~/.sase/agent_name_registry.json`, so a `family=` rewrite without
  `!` would fail the relaunch on a name collision.

Family **roots** get no such exception. A plan-chain root whose prompt is
`#gh:gh_sase-org__sase #plan` (auto-named `06d`, promoted to `06d--plan`) now relaunches
verbatim and forms a fresh family under a newly allocated name.

### 4. `sase agent restart` — no behavior change, one internal path restored

`_rewrite_prompt_identity` in `src/sase/agent/_restart_planning.py` calls
`prepare_kill_and_edit_prompt`, so for an unnamed prompt it now returns the prompt
verbatim. `_plan_name_reuse` then injects `%id:!<meta_name>` via
`ensure_forced_name_reuse` and reports `name_reuse_source="injected"` — exactly the path
`cfdaa6577` built, which `ea31a2b5b` made unreachable by injecting first. The command
still relaunches under the same name, as `docs/cli.md` promises; only the reported reuse
source changes from `prompt` to `injected` (and the preview label from "from prompt" to
"injected"). Make no source change here.

### 5. `docs/ace.md`

The rewrite table and the paragraph beneath it are already correct — leave them. Add one
sentence to that paragraph noting that a serial family member is the one case where a
prompt with no `%id` is still rewritten, because its `family=` attachment comes from the
row rather than the prompt, and that family roots are not treated that way.

## Tests

Baseline before the change: the six files below are green (119 passed).

Flip these to the restored contract:

- `tests/ace/tui/test_retry_edit_agent_name.py`
  - `test_prepare_kill_and_edit_prompt_contract`: change the
    `("#gh:gh_sase-org__sase Describe this repo.", "068", "%id:!068\n…")` case to expect
    the raw prompt unchanged, and restore a `("Do work", None, "Do work")` case.
  - `test_prepare_kill_and_edit_prompt_refuses_missing_identity`: replace with a test
    that `prepare_kill_and_edit_prompt("Do work", None) == "Do work"` and raises
    nothing.
  - `test_prepare_kill_and_edit_prompt_plain_family_root_forces_reuse`: rename to
    `…_plain_family_root_keeps_prompt` and expect `"#gh:gh_sase-org__sase #plan"`
    unchanged.
  - `test_kill_and_edit_agent_injects_forced_id_when_prompt_has_none`: rename to
    `…_keeps_prompt_when_it_has_no_id` and expect the unrewritten prompt to be seeded.
  - `test_prepare_kill_and_edit_prompt_refuses_self_attaching_family` must keep passing
    unchanged (it is a serial-member case, so the gate does not apply).
- `tests/ace/tui/test_agent_bulk_kill_edit.py`: the three marked-set cases whose raw
  prompts have no `%id` now seed verbatim panes — `"Just do it"` (was `%id:!solo\n…`),
  `"part one\n---\npart two"` (was `%id:!multi\n…`), and `"Work live"` (was
  `%id:!live\n…`). The `%i:run` / `%id:done` case and the
  `%id(!code, family=sase-8u.4.2, bead=sase-8u.4.2)` family case stay as they are.
- `tests/ace/tui/test_family_member_relaunch.py`:
  `test_plain_plan_root_relaunch_forces_family_name_reuse` → expects the seeded prompt
  to equal `#gh:gh_sase-org__sase #plan`; rename accordingly. The epic-root
  (`%id(!1, clan=sase-pw, …)`), self-attach, and clan-container tests stay green.
- `tests/test_agent_restart_plan.py`: `test_plan_injects_forced_id_for_plain_prompt` now
  asserts `name_reuse_source == "injected"` and `"injected" in plan.preview.name_reuse`
  (its name finally matches what it exercises).
  `test_plan_name_reuse_source_is_prompt_when_id_already_present` and
  `test_plan_family_member_is_not_double_rewritten` stay green.
- `tests/test_force_reuse_launch_seam.py` and `tests/agent/test_force_reuse_launch.py`:
  expected to stay green untouched (`_clan_kill_and_edit_prompt` declares `%id`,
  `_family_kill_and_edit_prompt` is a serial member). If either turns red, stop and
  re-check the gate rather than editing the assertion.

Add these new cases:

- `prompt_has_id_directive` unit tests. The exact table below was prototyped against the
  real `_DIRECTIVE_PATTERN` and passes 19/19, so it is the contract to encode:
  - `True`: `%id:foo`, `%i:foo`, `%id:!foo`, `%id:@.cld`, `%id:sase-8a.3\n%auto\n…`,
    ``%id:`quoted name` ``, `%id(2, clan=sase-8k, bead=sase-8k.2)`,
    `%id(!code, family=sase-8u.4.2, bead=sase-8u.4.2)`, an epic root with
    `%id(sase-pw.1, …)` plus `%clan(sase-pw, …)`, and
    `%model:@no_such_alias\n%id:foo\n…` (unparseable but declared).
  - `False`: `Do work`, `#gh:… Describe this repo.`, `#gh:… #plan`,
    `Implement the plan`, `part one\n---\npart two`, a bare `%id\nDo work`, a `%id`
    reachable only inside a fenced block or a `%xprompts_enabled:false` region, and
    `%model:@no_such_alias\nDo work`.
- A bare `%id\nDo work` reaches `prepare_kill_and_edit_prompt` unchanged for
  `agent_name` of `"foo"` and `None` (today the first injects `%id:!foo` and the second
  raises).
- Regression test named for the bug: `prepare_kill_and_edit_prompt` returns the exact
  `06y` prompt (`#gh:gh_sase-org__sase Did we ever fix the issue where …`) unchanged for
  `agent_name` of `"06y"`, `"bbugyi200.athena.06y"`, and `None`.
- A prompt whose only `%id` sits inside a fenced block is returned unchanged (the gate
  must not be fooled by prose that quotes a directive).
- A prompt that still fails `extract_prompt_directives` (e.g. `%model:@no_such_alias`)
  and has no `%id` is returned unchanged rather than raising — the tolerance property
  from step 1.

## Verification

1. `just install` first (ephemeral workspace).
2. `.venv/bin/python -m pytest -q tests/ace/tui/test_retry_edit_agent_name.py tests/ace/tui/test_agent_bulk_kill_edit.py tests/ace/tui/test_family_member_relaunch.py tests/test_agent_restart_plan.py tests/test_force_reuse_launch_seam.py tests/agent/test_force_reuse_launch.py`
   while iterating.
3. `just check` before replying.
4. `just check-full` through `/sase_monitor` before landing, since this touches a
   boundary shared by ACE and the CLI.
5. Manual check in ACE: launch an unnamed prompt, press `,x` within a few seconds of
   launch (before the row gains a name), and confirm the prompt bar opens seeded with
   the original text and no toast; then press `,x` on a `%id:`-named agent and confirm
   the seeded prompt still carries `%id:!<name>`.

## Out of scope

- ACE rows carrying `agent_name=None` for the first seconds after launch. The gate makes
  it harmless for unnamed prompts, and a `%id`-bearing prompt already relaunches
  correctly without the row name (`force_name_reuse_in_prompt` marks the prompt's own
  `%id`). Not worth a loader change here.
- `_verify_kill_and_edit_prompt` still refuses a prompt that _does_ declare `%id` but
  cannot be parsed by `extract_prompt_directives` (the ~197 historical prompts with
  removed model aliases). Pre-existing, out of scope, and worth its own task bead.
- Multi-segment prompts where only a later `---` segment declares `%id`. Pre-existing
  ambiguity in `set_prompt_name`; the gate neither improves nor worsens it.
