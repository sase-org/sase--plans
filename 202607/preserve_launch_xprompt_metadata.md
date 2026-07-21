---
tier: tale
title: Preserve launch xprompt metadata across deferred expansion
goal: 'Agent metadata continues to show every xprompt authored at launch even when
  a deferred workflow expansion runs after ordinary prompt parts have been expanded.

  '
create_time: 2026-07-21 11:52:43
status: wip
prompt: '[202607/prompts/preserve_launch_xprompt_metadata.md](prompts/preserve_launch_xprompt_metadata.md)'
---

# Plan: Preserve launch xprompt metadata across deferred expansion

## Diagnosis

The affected `gy.f1.f6.f0.w0.f2` artifacts establish that this is a persistence regression rather than an ACE rendering
bug. Both `submitted_xprompt.md` and `raw_xprompt.md` contain `#gh`, `#fork`, `#beau`, and `#plan`, while the shared
`xprompts.json` contains only `#gh` and `#fork`. The metadata panel correctly renders that shared launch/root artifact,
so changing its filtering or layout would only mask the underlying data loss.

The runner initially calls `preprocess_prompt_xprompts`, which records the full launch-boundary prompt before ordinary
prompt parts are expanded. That stage is specifically intended to preserve references such as `#plan`. The prompt-part
processor then expands `#beau` and `#plan` but deliberately leaves `#fork` deferred until dependency admission. After
the wait, `expand_deferred_launch_xprompts` invokes the generic `expand_embedded_workflows_in_query` helper with the
same artifacts directory and restricts execution to `#fork`. Before applying that restriction, the helper
unconditionally calls `write_used_xprompts` in its default overwrite mode. Its input is now the partially expanded
prompt, so this second capture replaces the correct launch record with only the surviving `#gh` and `#fork` references.
The main prompt step subsequently uses the existing step-only preservation behavior, faithfully preserving the
already-corrupted shared file.

This regression was exposed by the deferred-fork launch ordering: the deferred pass introduced a second root-level
metadata write without declaring that the launch boundary already owns the shared artifact.

## Metadata ownership and preservation contract

Extend the generic query embedded-workflow expansion API with an explicit, keyword-only way for callers to preserve an
existing shared xprompt-usage artifact. Route that mode through the existing `write_used_xprompts` preservation
semantics so it has two important properties:

- If `xprompts.json` already exists, later expansion passes do not replace the launch/root record with a transformed
  subset of the prompt.
- If no launch record exists, the later pass may still seed the shared artifact, retaining the best-effort fallback used
  by foreground and named-workflow paths.

Keep the new mode disabled by default. Existing callers such as foreground `sase run`, CRS, mentor, fix-hook, and direct
xprompt expansion should retain their current first-capture behavior unless they explicitly identify themselves as a
later pass. Have `expand_deferred_launch_xprompts` opt into preservation, because `preprocess_prompt_xprompts` is the
authoritative launch-boundary writer for detached agents. Keep workflow execution, `only_workflow_names` filtering,
embedded workflow metadata, and prompt contents otherwise unchanged.

Do not add ACE-side reconstruction from `raw_xprompt.md` or merge partially expanded records in the renderer. The shared
file should remain a complete, stable launch snapshot, while `xprompts_<step>.json` continues to describe individual
workflow steps.

## Regression coverage

Add focused tests around the ownership boundary rather than only the visual symptom:

1. Extend the query-expansion metadata tests to prove the default mode still performs its current write, preservation
   mode leaves an existing shared record intact, and preservation mode seeds the record when the launch writer did not
   create one.
2. Add a runner/deferred-expansion regression matching the reported lifecycle: begin with launch metadata that includes
   ordinary prompt parts plus deferred workflows, run the deferred `#fork` expansion on the already-partially-expanded
   prompt, and assert that `xprompts.json` still contains the original authored set in its original order.
3. Retain or strengthen the existing assertion that deferred expansion remains restricted to the launch-deferred
   workflow set, so the metadata fix cannot accidentally execute `#gh` or rollover/completion workflows early.
4. Run the focused runner setup, used-xprompt, and embedded-workflow tests, then run the repository-required
   installation and `just check` gate. Inspect the final diff to confirm no renderer or unrelated artifact behavior
   changed.

## Expected outcome

For the reported prompt, the agent metadata panel will list all four authored references—`#gh`, `#fork`, `#beau`, and
`#plan`—because later deferred expansion can no longer clobber the launch snapshot. Agents without deferred workflows
and callers that rely on query expansion to create their first metadata artifact will continue to behave as before.
