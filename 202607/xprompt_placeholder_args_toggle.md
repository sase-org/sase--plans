---
tier: tale
title: Wire the xprompt_placeholder_args toggle and land epic sase-9q
goal: 'Make `ace.prompt_inputs.xprompt_placeholder_args: false` actually disable the
  gx/gX raw-placeholder-to-input-argument conversion, correct the docs that currently
  describe the key as unread, then close out epic sase-9q (bead close, symvision cleanup,
  plan-file status).

  '
bead: sase-9q
parent: sase/repos/plans/202607/raw_placeholder_inputs.md
create_time: 2026-07-26 11:29:42
status: done
---

- **PROMPT:** [202607/prompts/xprompt_placeholder_args_toggle.md](prompts/xprompt_placeholder_args_toggle.md)

# Plan: Wire the `xprompt_placeholder_args` toggle and land epic sase-9q

## Background

Epic sase-9q made raw `<placeholder>` tags behave like prompt input arguments. Its plan
(`sase/repos/plans/202607/raw_placeholder_inputs.md`, section "submit.2") added an `ace.prompt_inputs` config section
with two keys, both defaulting to `true`:

- `collect_raw_placeholders` — gates submit-time collection. **Wired**: read by `_collect_raw_placeholders_enabled()` in
  `src/sase/agent/prompt_placeholder_inputs.py`.
- `xprompt_placeholder_args` — was meant to gate the phase-`xpromptargs` conversion (raw placeholders become `text`
  input arguments when saving a draft as a new xprompt via `gx`/`gX`). **Dead**: it exists in
  `src/sase/default_config.yml` (line ~134) and `src/sase/config/sase.schema.json` (line ~695), but no code reads it.
  `docs/configuration.md` (line ~813) and `docs/ace.md` (lines ~2758-2761) currently document it honestly as
  accepted-but-not-read.

This plan wires the toggle, fixes the docs, and finishes landing the epic. Everything else in sase-9q has been verified
complete; do not touch the rest of the feature.

## Where the conversion happens

Both save paths funnel through one pure function, so one gate covers both:

- `gx` (save draft as a global/config xprompt): `on_prompt_input_bar_save_as_xprompt_requested` in
  `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt.py` (line ~70) calls
  `convert_placeholders_to_inputs(body, existing=...)`, merges `conversion.inputs` into the frontmatter, and hands
  `conversion.body` to the save picker/preview.
- `gX` (convert active pane to a frontmatter-local xprompt): `convert_active_pane_to_local_xprompt` in
  `src/sase/ace/tui/widgets/_prompt_input_bar_local_xprompt_actions.py` (line ~65) calls
  `infer_local_xprompt_inputs(body)`, which itself calls `convert_placeholders_to_inputs` in
  `src/sase/ace/tui/widgets/_local_xprompt_conversion.py`.

## Change 1 — Gate `convert_placeholders_to_inputs` on the toggle

In `src/sase/ace/tui/widgets/_local_xprompt_conversion.py`:

1. Add a module-private reader mirroring `_collect_raw_placeholders_enabled()` / `_config_section()` from
   `src/sase/agent/prompt_placeholder_inputs.py` (lines 70-92):

   ```python
   def _xprompt_placeholder_args_enabled() -> bool:
       """Return whether gx/gX placeholder-to-input conversion is enabled.

       Reads ``ace.prompt_inputs.xprompt_placeholder_args`` through the cached
       merged config. A missing, unreadable, or unparsable value falls back to
       enabled.
       """
   ```

   Use `load_merged_config` from `sase.config.core`, a tolerant `try/except` returning `True` on any failure (log at
   debug level like the launch-path reader does), and `bool(...)` coercion of the configured value. Duplicate the tiny
   `_config_section` helper locally rather than importing the private one from `sase.agent.prompt_placeholder_inputs`
   (cross-module private imports trip Symvision private-misuse checks).

2. Early-return an identity conversion from `convert_placeholders_to_inputs` when the toggle is off:

   ```python
   if not _xprompt_placeholder_args_enabled():
       return _PlaceholderArgConversion(body=body, inputs=[], renames={})
   ```

   This single gate gives both consumers the correct disabled behavior:
   - `gx`: `conversion.inputs` is empty (frontmatter untouched) and `conversion.body` is the original body, so the draft
     saves verbatim with `<tags>` intact.
   - `gX`: `infer_local_xprompt_inputs` still returns its Jinja-derived inputs (one required `TEXT` input per undeclared
     Jinja variable — the pre-epic behavior) with the body unchanged, so placeholders survive as literal text in the new
     local helper.

3. Update the docstrings of `convert_placeholders_to_inputs` and `infer_local_xprompt_inputs` to mention the toggle.

Do NOT gate at the call sites instead; keeping the gate inside the shared function is what keeps `gx` and `gX` behavior
identical, mirrors how `build_prompt_input_plan` internalizes `collect_raw_placeholders`, and leaves the callers
untouched.

## Change 2 — Tests

Extend `tests/ace/tui/widgets/test_local_xprompt_conversion.py`. Reuse the established config-disable pattern from
`tests/ace/tui/test_prompt_input_collection_launch.py` (lines 41-47): monkeypatch `load_merged_config` **on the module
under test** to return `{"ace": {"prompt_inputs": {"xprompt_placeholder_args": False}}}`.

Cover at least:

1. Toggle off → `convert_placeholders_to_inputs("Deploy <service> now")` returns the body unchanged, no inputs, and
   empty renames.
2. Toggle off → `infer_local_xprompt_inputs` on a body containing both `{{ target }}` and `<service>` returns only the
   `target` Jinja input, and the returned body still contains the literal `<service>` tag.
3. Toggle off, invalid Jinja → `infer_local_xprompt_inputs` still returns `None` (the failure contract is not affected
   by the gate ordering).
4. Config read raises (monkeypatch `load_merged_config` to raise) → conversion proceeds as enabled (fallback).

Also add one action-level `gx` test beside the existing conversion coverage in
`tests/ace/tui/actions/test_prompt_save_xprompt.py`: with the toggle off, a draft containing a raw placeholder reaches
the save modal with its body unchanged and no minted `input:` entries in the frontmatter.

## Change 3 — Documentation

1. `docs/configuration.md` (table row at line ~813): replace the "This key is accepted by the schema, but the current
   `gx` and `gX` conversion paths do not read it" description with the real behavior: when `false`, saving a draft as a
   new xprompt (`gx`) or frontmatter-local helper (`gX`) keeps raw `<placeholder>` tags as literal text and mints no
   `text` inputs; Jinja-variable input inference for `gX` is unaffected.
2. `docs/ace.md` (lines ~2758-2761): rewrite the sentence claiming the conversion paths do not read the key so it
   describes the working toggle instead.
3. `docs/ace.md` (`gx`/`gX` conversion paragraph at lines ~2834-2837) and `docs/xprompt.md` ("When an ACE draft is saved
   as an xprompt ..." section at line ~646): add a brief note that `ace.prompt_inputs.xprompt_placeholder_args: false`
   disables the conversion.

## Verification

1. `just install` first (ephemeral workspaces may have stale dependencies).
2. Focused tests:
   `.venv/bin/pytest tests/ace/tui/widgets/test_local_xprompt_conversion.py tests/ace/tui/actions/test_prompt_save_xprompt.py`.
3. `just check` (required before finishing since this plan changes files outside the exempt categories).

## Final phase — Land epic sase-9q

After the code, tests, docs, and `just check` are green, finish landing the epic:

1. Close the epic bead: `sase bead close sase-9q`.
2. Run `just symvision` (epic-symbol whitelist entries for sase-9q expire at close). Remove any stale sase-9q entries
   and unused code it reports. As of this plan's writing the Justfile carries no sase-9q whitelist entries, so expect
   this to be a confirmation pass.
3. Edit `sase/repos/plans/202607/raw_placeholder_inputs.md` (the epic's plan file, path also shown by
   `sase bead show sase-9q`) and set `status: done` in its YAML frontmatter.
4. If symvision cleanup changed any files, rerun `just check`.
