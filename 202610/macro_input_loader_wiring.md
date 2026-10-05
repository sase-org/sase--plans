---
tier: tale
title: Wire Python macro loaders through the input-type catalog
goal: "Unknown macro input types follow strict_macro_input_types, bad enum declarations
  skip only their macro, and handoff JSON keeps choices, named_type, and value_role.
  Then close epic sase-1g4.1.1 and phase sase-1g4.1.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1g4.1.1.land
bead: sase-1g4.1.1
status: done
---

- **PARENT:**
  [202610/macro_input_type_vocab.md](https://github.com/sase-org/sase--plans/blob/main/202610/macro_input_type_vocab.md)
- **BEAD:**
  [sase-1g4.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1g4/sase-1g4.1.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1g4.1.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1g4.1.1.land.md)
- **COMMITS:**
  - [95291ab](https://github.com/sase-org/sase/commit/95291ab31a447dcb720dcbdf1b87ae21590f2968)
    — feat(macros): wire loaders through input type catalog

# Plan: Wire Python macro loaders through the input-type catalog

Epic `sase-1g4.1.1` is otherwise landed. Phases `sase-1g4.1.1.1` (Rust catalog and
bindings) and `sase-1g4.1.1.2` (Rust parsers and frontmatter diagnostics) are in pinned
sase-core `0279de6b00a6c053fb82e0375f3c17469f581ab8`, which is HEAD of the linked
checkout and an ancestor of catalog commit `2838c7eb`. Phase `sase-1g4.1.1.4` landed
schemas, `config.macro_input_types`, the `#pr` status enum, docs, the parity corpus, and
`InputArg.validate_and_convert`'s enum arm (`check_input_value`). Phase `sase-1g4.1.1.3`
closed without the Python loader integration. Its notes #2 and #3, and `sase-1g4.1.1.4`
note #1, are this tale. Do not redo the landed surface work.

## Already true

- Bindings on `sase_core_rs`: `macro_input_type_catalog`, `resolve_input_type`,
  `validate_enum_choices`, `pyyaml_plain_scalar_is_non_string`, `check_input_value`.
  `resolve_input_type` takes `{name, raw, plugins?}` and returns
  `{base, named_type, value_role, choices, deprecated}` or raises `ValueError` whose
  text is the resolver message. `validate_enum_choices` takes `{items: [...]}` and
  returns `{choices: [{value, label?, description?}], issues: [{severity, message}]}`
  with severity `error` or `warning`. Pass raw YAML scalars. Do not call `str()` first.
  Unquoted `yes` is already a bool; the message is
  `choice arrived as a boolean and must be quoted` (likewise `an int`, `a float`, or
  `null`).
- `check_config_macro_definitions` already ignores kind `input_type_warning`.
  `check_config_macro_input_types` already reports kinds `input_type` and
  `input_type_warning`.
- Workflow longform already reads `choices`, `description`, and `repeatable`.
  `bind_input_args` already calls `validate_and_convert` once per repeatable element.
  Leave both.
- `sase-1g4.1.1.3` note #1: 13 project/home macro definitions and both installed plugin
  macro resources contain zero `choices` declarations. Enum choice-value errors and
  non-member defaults are unconditional load errors. Only unknown type names obey the
  flag.
- Corpus scan found no bundled `type: enum` before `#pr` was dogfooded. Do not rescan
  unless a load of current bundled macros fails.

## Do not

- Edit sase-core, ratchet `sase-core-revision.txt`, or add a binding.
  `check_closed_set_default` is Rust-only. Emit its message from Python:
  ``default `{value}` is not one of fast | thorough`` (`|` between members; more than
  eight members becomes `{n} choices`).
- Rename JSON key `local_xprompts` in `serialize_local_macros`. Open bead `sase-1eq.10`
  owns that rename. Commit `dc8aee0fbc` already dropped the dual env spellings and moved
  the pin; add fields beside the current keys.
- Create task beads. Follow-up triage is done (see Closeout).
- Run `just check-full`. Read `lint_and_test.md` with `sase memory read` before
  finishing. Run `just fix`, then `sase tool run check` (or `just check` if that is what
  the memory note names). A `just check` pass is the gate.

## 1. Flag

Read `sase_flags.md` with `sase memory read`, then create the flag only with:

```bash
sase flag new strict_macro_input_types -k sunset \
  --when-enabled "Unknown macro input type names are a per-macro load error with suggestions." \
  --when-disabled "Unknown macro input type names silently resolve to line, as they did before this flag." \
  --remove-when "No maintained macro source still relies on an unknown type name resolving to line."
```

Paste the printed registry entry into `src/sase/feature_flags/registry.py`. Do not
hand-edit a bead and do not call `sase bead create`. Sunset defaults on. Resolve it at
the load site with `current_flags().enabled(FeatureFlag.strict_macro_input_types)`. No
import-time read. If `tools/check_feature_flags` or the schema drift check fails,
regenerate with the existing `tools/sync_feature_flags_schema` flow. Test both states
with `override_flags` from `sase.feature_flags.snapshot`.

## 2. Resolver adapter

Replace `parse_input_type` in `src/sase/macro/loader_parsing.py`. Return the base
`InputType` plus `named_type`, `value_role`, and warnings. A helper that returns only
`InputType` drops those fields, so update every caller:

- shortform and longform in `loader_parsing.py`
- `parse_workflow_inputs` in `workflow_loader_parse.py`
- `src/sase/ace/tui/modals/input_item_modal.py`
- `src/sase/ace/tui/modals/macro_item_modal.py`
- `src/sase/ace/tui/widgets/_frontmatter_panel_cell_editing.py`

Map `base` through `InputType`. `string` arrives as base `line` with `deprecated` true:
load it as `line` and record an `input_type_warning`. That warning is not flag-gated. An
unknown name with the flag on raises `MacroValidationError` with the resolver message.
With the flag off, return base `line`, null `named_type` and `value_role`, and no error.
The three TUI callers already reject an unknown type against `input_type_schema` before
they call the adapter. Keep that. They must copy `named_type` and `value_role` onto the
saved `InputArg`. They do not consult the flag.

`InputChoice` gains `description: str | None = None`. `InputArg` gains
`named_type: str | None = None` and `value_role: str | None = None`. `type` stays the
base `InputType`. Leave `__post_init__` declaration checks as they are.
`validate_and_convert` for `ENUM` already calls `check_input_value`.

## 3. Choices and defaults

`parse_input_choices` calls `validate_enum_choices` with the raw YAML list. Copy
`value`, `label`, and `description` from the returned choices. Each `error` issue raises
`MacroValidationError` with that message. Each `warning` is recorded with
`record_load_issue(..., kind="input_type_warning")` and does not skip the macro.
Warnings need the source path; thread it from the loaders. A warning must not raise, or
`load_workflow_from_mapping` will record it as a `workflow` skip.

After a successful choice parse, a concrete default that is not `UNSET` and not explicit
`None` must be a string member. A non-member string uses the default message above. A
non-string concrete default says it arrived as a boolean, int, float, or null and must
be a quoted choice. That check is a load error. Do not check defaults again at bind
time.

## 4. Per-macro isolation

One bad declaration never escapes the loader:

- `load_macro_from_file` catches `MacroValidationError` around
  `parse_inputs_from_front_matter` (and local-macro parsing if it can raise), records
  kind `input_type`, and returns `None`.
- `parse_macro_entries` catches the same error per entry, records kind `input_type`, and
  continues with the other entries.
- `load_plugin_markdown_macros` catches it, records kind `input_type`, and does not
  yield that macro. Skills already go through `load_macro_from_file`.
- `load_workflow_from_mapping` already catches `MacroValidationError` as kind
  `workflow`. Keep that kind. A warning must not take that path.

## 5. Handoff and frontmatter write-back

In `src/sase/agent/multi_prompt_macros.py`, round-trip `choices` (`value`, `label`,
`description`), the input `description`, `repeatable`, `named_type`, and `value_role`.
Old files omit those keys and deserialize as empty choices, `repeatable` false, and null
names. Keep the `local_xprompts` nested key.

`prompt_frontmatter._input_to_yaml` writes `type: <named_type>` when `named_type` is
set, and writes choice `description`. It does not write resolved choices back onto a
named type. The only named type in this epic is `agent`, whose base is already `agent`.

## 6. Tests

Add or extend tests so all of these hold:

- A file containing `choices: [yes, no]` fails in the Python loader. The message says
  the value arrived as a boolean and must be quoted.
- A non-member enum default is rejected at load, and sibling macros in the same file or
  config mapping still load.
- A longform workflow enum loads, including `description` on a choice.
- `type: enmu` with the flag on (the sunset default) records an `input_type` issue
  naming `enum` and skips only that macro. With the flag off, the input loads as `line`
  and the macro stays.
- `type: string` loads as `line` and records an `input_type_warning`.
- An enum local macro survives `serialize_local_macros` and `deserialize_local_macros`
  with choices, description, repeatable, and named_type. A JSON object that omits the
  new keys still loads.
- Existing gate enum tests still expect the did-you-mean message. Do not weaken
  `tests/test_pr_status_enum.py`, `tests/test_macro_input_schemas.py`,
  `tests/test_macro_input_type_parity.py`, or
  `tests/doctor/test_checks_config_macro_input_types.py`.

`#pr(x, status=ready)` still binds, `status=Ready` still suggests `ready`, and
`#pr:ready` still binds `name`.

## 7. Closeout

This tale has no land agent. Finish the landing in this same turn, after `just fix` and
a passing `sase tool run check`. Do not wait for this turn's commit SHA, push, or CI.

1. Run `sase bead epic-symbols sase-1g4.1.1`. The list was empty at landing review. If
   it is still empty, continue. If any `--epic-symbol` entry remains, resolve it (wire
   it up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy) or, only when a still-open later bead needs the exemption,
   re-key the Justfile line to that open bead (`sase-1g4` or phase `sase-1g4.2`). Do not
   leave the judgment. `sase bead close` refuses while any entry remains.
2. Close the epic:

   ```bash
   sase bead close sase-1g4.1.1 --note "<verification>"
   ```

   The note must say what was verified: the Rust catalog and both parsers agree; unknown
   types follow `strict_macro_input_types` (flag bead from step 1); a bad enum skips
   only its macro; `#pr` status is the `wip | draft | ready` enum; the schema-sync check
   passes; handoff round-trips the new fields. Also record the follow-up triage below.
   Never use `--force` to make the close succeed. If the close is rejected for leftover
   symbols, fix them and close again. The four phase beads are already closed `done`; do
   not force them.

3. Run `just symvision`.
4. Set `status: done` in the frontmatter of
   `sase/repos/plans/202610/macro_input_type_vocab.md` (the PLAN path from
   `sase bead read sase-1g4.1.1`).
5. Parent is phase bead `sase-1g4.1`, not a plan bead. Re-read it and confirm this child
   plan completed that phase: one catalog, both Rust and Python parsers on it, strict
   enum declarations, per-macro isolation, the sunset flag, generated schemas, the
   doctor check, and `#pr` status as an enum. Run `sase bead epic-symbols sase-1g4.1`
   first (empty at landing review). Re-key any entry that still names `sase-1g4.1` to
   open bead `sase-1g4` or `sase-1g4.2`. Then close only that phase:

   ```bash
   sase bead close sase-1g4.1 --note "<what you verified against the phase description>"
   ```

   Do not close `sase-1g4`. Do not mark `macro_named_input_types.md` done. Do not use
   `--force`. Stop after `sase-1g4.1`. Its land agent owns the containing epic.
   `sase-1g4.2` stays open.

If `just check` fails a node that passes in isolation and this diff does not touch that
area, triage it with `/sase_new_task` before closing and name the outcome in the close
note. Do not treat it as unfinished loader work. The three flakes below are already
filed.

### Follow-up triage already recorded

Record this same disposition in the epic close note:

- `sase-1g4.1.1.2` prompt-catalog flake
  `test_prompt_source_token_changes_for_project_file`: declined. It reproduced on clean
  `25cc3c475d` and was fixed by `404b0e2ac2` (`sase-1eq.5.1.6`), which is an ancestor of
  HEAD. No task names that node.
- `sase-1g4.1.1.3` pin ratchet to `2838c7eb`: declined. Pin `0279de6b` (`dc8aee0fbc`,
  bead `sase-1eq.10`) contains that commit and the `macro_input_types` bindings.
- `sase-1g4.1.1.3` notes #3 and `sase-1g4.1.1.4` note #1 (loader integration): this
  tale, not a new task.
- `test_cancel_saves_once_submit_path_quiet`: filed `sase-1gg` (flake, large, ready).
  Same missing-widget class as `sase-1fy` and `sase-1g5`, different node. `sase-j7` note
  #85 already records it; not a confirmed process-global leak.
- `test_sudo_runner_invocation_keeps_canary_out_of_process_argv_and_env`: filed
  `sase-1gh` (flake, large, ready). Sibling of `sase-19d`, different node. `sase-j7`
  note #86 already records it.
- `test_agent_run_stops_on_a_new_item`: filed `sase-1gi` (flake, large, ready). Closed
  `sase-114` fixed a different nested-diagnostics leak. `sase-j7` note #86 already
  records it.
