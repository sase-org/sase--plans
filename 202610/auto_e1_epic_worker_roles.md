---
tier: tale
title: Epic workers launch nested epics and tales by default via autonomy.roles
goal:
  Epic phase and land agents auto-approve and launch the epics and tales they author,
  unless sase configuration (autonomy.roles) narrows them, and the %auto roadmap
  reflects this so E2 and E3 build on it.
size: medium
proposed_by: bbugyi200.athena.0za
create_time: 2026-10-09 17:15:03
status: wip
---

# Plan: Epic workers launch nested epics and tales by default (`autonomy.roles`)

## Why

The `%auto` E1 work (epic `sase-1ip`, plan `plan:202610/auto_e1_autonomy_record.md`)
kept a P0 stopgap (D7, shipped in `sase-1id`): every epic phase and land segment
rendered by `src/sase/bead/work_prompt.py::render_multi_prompt` carries a literal
`%auto:tale`. Under the E1 record, that gives every epic worker the `tale` profile. Its
own tale plans auto-approve, but any epic it authors parks until a human approves it.
Two common cases are affected:

- a `large` phase that plans a child epic;
- a land agent that authors a "Finish…" child epic.

So overnight work stops and waits for a human. The user has reversed that stopgap:

- **Default:** epic phase agents and epic land agents can launch epics and tales. They
  run the `standard` profile.
- **Override:** one sase config setting changes this, without code changes.
- **Roadmap:** the research roadmap must say so, so that E2 and E3 build on the new
  default.

The setting is a minimal, forward-compatible slice of E3's planned `autonomy.roles`. In
E1 its values are the four built-in profile names from the core catalog (`manual`,
`standard`, `tale`, `epic`). In E3, roles may also name user-defined profiles.

Read for context:

- `sase bead read sase-1ip -r "<why>"`;
- `sase artifact read plan:202610/auto_e1_autonomy_record.md "<why>"`;
- `sase artifact read research:202610/auto_autonomy_epic_roadmap/auto_autonomy_epic_roadmap.md "<why>"`.

## Before you start: in-flight overlap

The E1 remediation tale `plan:202610/finish_auto_e1_landing.md` (agent `sase-1ip.land`)
may still be running when you start. It closes `sase-1ip` itself. It changes:

- autonomy inheritance and record authority (`src/sase/autonomy/record.py`,
  `axe/run_agent_directive_metadata.py`, direct plan approval);
- single gate evaluation (`notification_gates/`, `src/sase/autonomy/gates.py`);
- `sase autonomy log --since` (`src/sase/autonomy/cli_log.py`);
- the contract-suite isolation fences (`tests/autonomy_contract/harness.py`);
- possibly sase-core `autonomy/mutate.rs`.

Run `git log --oneline -20` and
`sase bead read sase-1ip -r "Check whether the E1 remediation has landed"`. Build on
whatever has landed. Do not wait for it. Keep this tale off those seams. In the shared
test files, edit only the epic-worker pieces named below. Do not reopen `sase-1ip` or
change its plan file.

## 1. Config: `autonomy.roles`

Add a new top-level `autonomy:` section.

**`src/sase/default_config.yml`:**

```yaml
autonomy:
  # Autonomy profile each generated worker role launches with. Values are
  # built-in profile names from `sase autonomy list`: standard (tale plans
  # approve + archive, epic plans approve + launch, questions take the first
  # option), tale (epic plans wait for you), epic (tale plans wait for you),
  # or manual (everything waits). Set a role to tale to make nested epics
  # wait for human approval.
  roles:
    # Epic phase agents launched by `sase bead work` / epic plan approval.
    epic_phase: standard
    # Epic land agents (they may author a tale or a "Finish..." child epic).
    epic_land: standard
```

**`src/sase/config/sase.schema.json`:**

- Add a top-level `autonomy` object with `additionalProperties: false`, holding a
  `roles` object with `additionalProperties: false`.
- `roles` has the properties `epic_phase` and `epic_land`. Each is a string enum of
  `["manual", "standard", "tale", "epic"]` with default `standard` and a description.
- Mention in the descriptions that E3 will widen the values to config profile names.
- The top-level schema is `additionalProperties: false`, so the section must be declared
  there.

**`docs/configuration.md`:**

- Add an `### autonomy` section and a Table of Contents entry.
- Show user-level (`~/.config/sase/sase.yml`, chezmoi-managed for this user) and
  project-level (`sase/sase.yml`) examples. For example, set both roles to `tale` to
  restore "nested epics wait for me".
- Say the setting is read when the epic's worker prompts are rendered, so already
  launched workers keep the profile in their prompt.
- Say `sase autonomy list` shows the effective role assignments.

## 2. A thin role adapter: `src/sase/autonomy/roles.py`

This is Python glue over core and config. Do not hardcode profile semantics: the core
catalog stays the source of truth (`rust_core_backend_boundary`).

- **Constants:** `EPIC_PHASE_ROLE = "epic_phase"`, `EPIC_LAND_ROLE = "epic_land"`, and
  `DEFAULT_ROLE_PROFILE = "standard"`.
- **`role_profile(role) -> RoleAssignment`.**
  - Reads `autonomy.roles.<role>` from `sase.config.load_merged_config()`, the same way
    `sase.bead.config` getters read their sections.
  - Strips whitespace and validates the name against `profiles_catalog()` names.
  - Returns `{role, profile, source}`, where `source` is `config`, `default`, or
    `invalid`.
  - A missing or malformed section, or a non-string value, falls back to the default,
    following the repo's config-getter convention.
  - An unknown name also falls back to the default, with a `logging` warning that names
    the bad value and the valid names. Its `source` is `invalid`.
- **`role_auto_directive(role) -> str`.** Returns the `%auto` line for the role's
  profile.
  - Derive the selection from the catalog entry: the entry whose `kind` is `default` is
    selected by bare `%auto`. Every other built-in is selected by `%auto:<name>`.
  - So: `standard` → `%auto`, `tale` → `%auto:tale`, `epic` → `%auto:epic`, `manual` →
    `%auto:manual`. Emit `manual` explicitly so a dry run shows it.
  - Verify the derived selection through `record.resolve_selection(...)`. The resolved
    record's `profile` must equal the configured name. If it does not, raise
    `RuntimeError`, which means core and config disagree.
  - This must not need a sase-core change. If you find it truly needs one, keep it
    minimal and make Python tolerate an older core in the `sase-core-rs` minor window.
- **`role_assignments() -> list[RoleAssignment]`** for both roles, in a fixed order.
- If the package re-exports helpers through `src/sase/autonomy/__init__.py`, follow that
  pattern.

## 3. Render worker prompts from the roles

In `src/sase/bead/work_prompt.py::render_multi_prompt`:

- Replace the two literal `lines.append("%auto:tale")` /
  `land_lines.append("%auto:tale")` calls with `role_auto_directive(EPIC_PHASE_ROLE)`
  and `role_auto_directive(EPIC_LAND_ROLE)`.
- Resolve each role once per render, not once per segment.
- Keep the line position unchanged: after `%model:`, before `%queue`/`%w`.
- Update the docstring.

No other launch path changes:

- Workers resolve their record from their own prompt (`source: prompt`).
- Their host-composed successors inherit it structurally.
- `creator_role: epic_worker` decision logging is unchanged.

Check every `render_multi_prompt` caller with `rg -n "render_multi_prompt" src`. This
covers plan-approval epic launch, `sase bead work` (including `--dry-run`), and
resumed/retried epics. All of them must get the role directive without a second config
read path.

## 4. Worker macro wording

`src/sase/default_config.yml` has two macros that state "Such a child epic plan waits
for human approval before its clan launches.":

- `bd/land_epic`, in the child-epic path paragraph;
- `bd/work_phase_bead`, in its last sentence.

That claim is no longer true by default. Replace it with wording that depends on
autonomy. For example: "Whether such a child epic plan launches automatically or waits
for human approval depends on your autonomy profile; the SASE autonomy block in your
prompt says which."

Keep the rest of each macro byte-identical. If any test pins the macro text, update it.

## 5. `sase autonomy list` shows the roles

In `src/sase/autonomy/cli_profiles.py::handle_autonomy_list`:

- **Rich output:** after the profile lines and before the coverage line, print a "Roles"
  block, one line per role. For example:
  `epic_phase → standard (default) · set autonomy.roles.epic_phase`. Name the source. An
  `invalid` source says which value was ignored.
- **`--json`:** add a `roles` key, a list of `{role, profile, source}`. Keep `profiles`
  and `coverage` exactly as they are. This is an additive object key.
- Add no new options. The CLI completion snapshot should not change; if it does,
  regenerate it with `just sync-completion-spec`.

## 6. Docs

**`docs/beads.md`:**

- Rewrite the three passages near the current lines 2797, 2819, and 2862.
- Every phase and land segment carries the `%auto` line for its `autonomy.roles`
  profile. The default is `standard`: submitted tale plans are approved and archived,
  and submitted epic plans are approved and launched as child epics beneath the phase or
  land bead.
- Setting the role to `tale` makes nested epic plans park for human review.
- Link `configuration.md#autonomy`.

**`docs/macros.md`, Auto Directive section:**

- Add one short paragraph: generated epic workers take their profile from
  `autonomy.roles` (default `standard`), and `sase autonomy list` shows it.
- Adjust any sentence that implies epic workers are `tale`.

**`docs/sdd.md`:** Grep for wording about epic workers or nested epics parking, and
correct it if present.

Leave `docs/blog/` alone. It is historical.

## 7. Tests

**Rendering.**

- Default config: every phase and land segment carries a bare `%auto` line, and no
  `%auto:tale`.
- With `autonomy.roles` patched (monkeypatch `load_merged_config`, following existing
  config-getter tests), test `epic_phase: tale`, `epic_land: manual`, and an invalid
  value (warning plus default).
- Update the fixture strings that assert `%auto:tale` in:
  - `tests/test_bead/work_test_helpers.py`
  - `tests/test_bead/test_work_epic_plan.py` (flip its assertion)
  - `tests/test_bead/test_work_rendering_models.py`
  - `tests/test_bead/test_work_rendering_changespec.py`
  - `tests/test_bead/test_cli_work_epic_dry_run.py` (the dry run shows the role line)

**Role adapter unit tests** (e.g. `tests/test_autonomy_roles.py`):

- each built-in name maps to the right directive;
- the token round-trips through `resolve_selection` to the same profile;
- `source` values `config`, `default`, and `invalid`;
- whitespace is stripped;
- non-dict sections.

**Contract suite (`tests/autonomy_contract/`).**

- In `rows.py` `STATE_ROWS`, replace row `tale_epic_worker`
  (`APPROVE_ARCHIVE, ASK, FIRST`, context `epic_worker`) with these rows. Add a comment
  marking them as the deliberate post-E1 amendment. Rows need a way to carry a config
  override; add one optional field or encode it in the context. Every module that
  parametrizes over `STATE_ROWS` must handle the new contexts (today that includes
  `test_autonomy_cli_parity.py::_state_meta`; `rg STATE_ROWS tests/autonomy_contract`).

  | Row                       | Config                             | Tale plan         | Epic plan        | Question |
  | ------------------------- | ---------------------------------- | ----------------- | ---------------- | -------- |
  | `epic_phase_worker`       | default                            | `APPROVE_ARCHIVE` | `APPROVE_LAUNCH` | `FIRST`  |
  | `epic_land_worker`        | default                            | `APPROVE_ARCHIVE` | `APPROVE_LAUNCH` | `FIRST`  |
  | `epic_worker_role_tale`   | `autonomy.roles.epic_phase: tale`  | `APPROVE_ARCHIVE` | `ASK`            | `FIRST`  |
  | `epic_worker_role_manual` | `autonomy.roles.epic_land: manual` | `ASK`             | `ASK`            | `ASK`    |

- Rewrite `harness.adapt_epic_worker_prompt` so it no longer inspects source text for
  `"%auto:tale"`. It should render a minimal real epic multi-prompt through
  `render_multi_prompt` (or the smallest production entry that reaches it) and return
  the phase or land segment's actual `%auto` line. Then the rows exercise the real
  renderer plus config.
- Update `tests/autonomy_contract/test_autonomy_cli_parity.py` (it special-cases
  `tale_epic_worker`) so explain-equals-runtime parity covers the new rows.
- Keep every other row and expectation unchanged.

**CLI:** `sase autonomy list` Rich and `--json` output includes roles, and `profiles`
and `coverage` are unchanged.

**Schema:** the existing bundled-schema validity test passes. `default_config.yml` still
validates against the schema, wherever that is tested.

## 8. Update the roadmap in the research sidecar

Open the sidecar with
`sase repo open sase--research -r "Amend the %auto roadmap for epic-worker roles"`. Edit
only `202610/auto_autonomy_epic_roadmap/auto_autonomy_epic_roadmap.md`. Leave the
`__final`/`__<model>` source reports and the PNG/JPG images alone. Keep the report's
voice and Markdown style.

1. **Amendment callout** directly under the infographic. Add an "Amendment (2026-10-09):
   epic workers launch by default" callout covering:
   - the user reversed D7's worker stopgap;
   - epic phase and land agents run the `autonomy.roles` profile, default `standard`, so
     they auto-approve and launch the tales and epics they author;
   - `autonomy.roles.epic_phase` / `epic_land` narrows them;
   - this landed as a follow-up tale to E1 (name this tale's plan reference);
   - the infographic predates it.
2. **Bottom line.**
   - P0 row: mark "Epic workers park nested epics instead of launching them" as
     superseded by the amendment. The tier-mismatch → `ask` half stays.
   - E1 row: "No behavior change" now has one exception, the epic-worker default.
   - E3 row: generated workers' config roles already exist. E3 lets them name config
     profiles.
   - Hoops list: rewrite the "two-step worker rule". P0 emitted `%auto:tale`. The E1
     amendment moved worker autonomy into `autonomy.roles` (default `standard`). E3
     widens role values to config profiles.
3. **The yardstick table.**
   - Change "epic phase or land worker" to: approve + archive | approve + launch clan |
     first option.
   - Add rows for a role set to `tale` (approve + archive | ask | first) and to `manual`
     (ask | ask | ask).
4. **P0 table.** Annotate the worker tale row: shipped in `sase-1id`; its `%auto:tale`
   emission is superseded by the amendment; its tier-mismatch → `ask` change stands.
5. **E1 section.**
   - Add a short "Amendment: epic-worker roles" subsection describing exactly what this
     tale shipped: the config, the rendering, the macro wording, `sase autonomy list`
     roles, and the contract rows.
   - Replace the watch metric "parked nested epics and how long they wait" with "nested
     epic auto-launches by epic workers (`creator_role: epic_worker`) and their nesting
     depth".
6. **E2 section.**
   - Worker-launched nested epics are announced like any epic auto-launch. The
     announcement names the role and how to change it (`autonomy.roles`).
   - Add an exit criterion: an epic auto-launched by an epic worker rings with
     **Manual** and **Pause all**.
   - Note that the brake is now the stop for overnight nesting.
7. **E3 section.**
   - Rewrite the Roles result and the E3.3 `roles` phase. `autonomy.roles` already
     exists with built-in names. E3 lets role values name user/project profiles, and
     worker prompts then emit `%auto:<profile>`.
   - Drop the built-in `epic_worker` profile. Defaults stay `standard`.
   - State three rules:
     1. A role is a config grant. A parent or planner ceiling never clamps an epic
        worker below its role, so a human-approved epic from a Manual planner still runs
        `standard` workers.
     2. A project layer may only narrow a role relative to the user layer.
     3. Agent-authored `%auto` in worker follow-ups may only narrow.
   - Plan Decisions: replace `epic_worker` (**`epic: ask`** | `approve(max_depth=1)`)
     with an opt-in depth-bound decision, for example `epic_depth`: **offer
     `approve(max_depth=N)` as opt-in vocabulary; default roles stay unbounded** | defer
     to E5.
   - Add a constraint to `standard_epics`: if `standard` stops auto-launching epics, the
     default roles must move to a profile that still launches. The user requires that
     epic workers launch by default.
   - Exit criteria: add "a project layer that widens a role is clamped" and "default
     roles reproduce E1's worker rows".
8. **Risks.**
   - Mark the "E1's own epic workers still run with a literal `%auto`" row as
     historical.
   - Add a risk row: unbounded nested epics overnight are now the default. Mitigations:
     `autonomy.roles` per user/project; `sase autonomy log` filtered to epic workers;
     E2's announcements and brake; E3's opt-in depth bound.
9. **Alternatives rejected.** Add "Keep epic workers on `tale` (the D7 stopgap)": it
   leaves overnight work parked until a human wakes up, and the user rejected it on
   2026-10-09.
10. **What would change this recommendation.** Add: if worker-authored "Finish…" epics
    start chaining more than about two deep, set `autonomy.roles.epic_land: tale` or
    adopt E3's opt-in depth bound.
11. **Evidence: "Who auto-approves epics now".** Leave the historical numbers. Add one
    sentence: after the amendment, epic-worker volume is expected to rise, and E1's log
    separates it by `creator_role`.

Then run `sase repo log` / `git -C <printed path> status` to confirm the sidecar change
is staged for the final declaration. Do not hand-commit; the host commits.

## 9. Verify and land

- Read `lint_and_test.md` through `/sase_memory_read` before finishing, because tracked
  sase files change.
- Run `just fix`, then `sase tool run check` in sase. If sase-core changed, also run it
  there. Never run `just check-full`.
- Spot-check with the checkout CLI (`.venv/bin/python -m sase` if the global install is
  older; never reinstall globally):
  - `sase bead work <some epic plan bead or fixture> --dry-run` shows `%auto` on every
    phase and land segment;
  - `sase autonomy list` and `sase autonomy list --json` show both roles;
  - `sase autonomy explain -p '%auto'` shows epics approve + launch.
- Use a conventional commit subject that names the behavior change in plain words, for
  example
  `feat(autonomy): epic phase and land workers launch nested epics and tales by default (autonomy.roles)`.
  `CHANGELOG.md` is generated; never edit it.
- **No memory edits.** No SASE memory note states the old worker behavior.
- Declare both changed repos (sase and the research sidecar) in the final declaration.

## Out of scope

- Workers that were already rendered or launched keep the `%auto:tale` in their prompt
  and their `tale` record. E1 has no human widening command (that is E4's
  `sase autonomy set`). Do not rewrite live agents' records or prompts.
- Task-bead workers, workflow workers, and top-level `%auto` agents.
- Profile vocabulary, depth bounds, and ceilings. These are E3/E5 work, recorded only in
  the roadmap.
