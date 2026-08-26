---
tier: tale
title: Stop injecting the VCS workflow tag into
goal:
  A "#fork" prompt renders the parent transcript verbatim, with no "#gh:<project>" or
  "#git:<project>" tag spliced into the injected history or in front of the "# New
  Query" heading, while genuine "---"-separated multi-prompt segments keep inheriting
  the tag exactly as they do today.
size: small
proposed_by: bbugyi200.athena.0e4
create_time: 2026-08-26 08:18:52
status: wip
---

# Stop injecting the VCS workflow tag into `#fork`-injected history

## Problem

Every agent launched with `#fork` receives a prompt in which the launch's VCS workflow
tag has been spliced into the _content_ of the injected parent transcript. The most
visible artifact is the `# New Query` heading, which reaches the model as:

```text
---

#gh:gh_sase-org__sase # New Query

<the actual new query>
```

The tag is also spliced into the middle of the transcript, once per `---` line, and the
noise compounds: a fork of a fork of a monitor-handoff chain observed **8** copies in a
single prompt.

### Confirmed evidence (already gathered, do not re-derive)

Final prompts handed to the model (`workflow-*-main_prompt.md` in the agent's artifacts
directory) contain the injected tag:

- `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/25/20260825144003/workflow-tmp_260825_144217-main_prompt.md:133`
  → `#gh:gh_sase-org__sase # New Query`
- `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/18/20260818102201/workflow-tmp_260818_102301-main_prompt.md`
  → the same line at 122, 428, 691, 1001, 1264, 1527, 1634, 1690 (8 copies)

The corresponding saved chat transcript for that same run
(`~/.sase/chats/202608/gh_sase_org__sase-ace_run-05z_f0_f0__plan-260818_102201.md:135`)
has a **clean** `# New Query`. The chat records `state.current_prompt`, which is
captured _before_ the injection happens, so the divergence between the two files
localizes the defect precisely to the runtime step described below.

## Root cause

`---` on its own line is SASE's multi-prompt segment separator. The `#fork` prompt part
emits `---` as ordinary _content_, and so does the transcript flattener, but the VCS-tag
inheritance pass splits on `---` without first neutralizing the inert region those
separators live inside.

The chain, in order:

1. `_wrap_fork_history` (`src/sase/history/chat_fork/build.py:138`) wraps the parent
   transcript as:

   ```text
   %xprompts_enabled:false
   <heading>

   <body>

   ---

   %xprompts_enabled:true
   # New Query
   ```

   and `load_chat_for_resume` (`src/sase/history/chat_resume.py:268`) joins the
   flattened turns inside `<body>` with `"\n\n---\n\n"`. Every one of those `---` lines
   is prose inside a `%xprompts_enabled:false` … `%xprompts_enabled:true` disabled
   region, not a segment boundary.

2. The agent runner expands the launch-deferred `#fork` reference into the prompt at
   `expand_deferred_launch_xprompts` (`src/sase/axe/run_agent_runner_setup.py:353`), so
   `state.prompt` now carries that whole block, with the launch's `#gh:<project>` tag
   still leading it.

3. The runner passes the launch's tag to the workflow as the inherited VCS tag
   (`src/sase/axe/run_agent_exec.py:113`), and the prompt step applies it at
   `src/sase/xprompt/workflow_executor_steps_prompt.py:213`:

   ```python
   step_prompt = inherit_vcs_workflow_tag(step.agent, self.inherited_vcs_tag)
   ```

4. `inherit_vcs_workflow_tag` (`src/sase/xprompt/_parsing_vcs_tags.py:172`) protects
   fenced blocks, then splits on `_SEGMENT_SEPARATOR_RE`
   (`src/sase/xprompt/_parsing_vcs_tags.py:21`) and hands each piece to
   `_inherit_vcs_workflow_tag_segment` (`:148`), which prefixes any piece lacking a VCS
   ref with the inherited tag. It never protects disabled regions, so every `---` inside
   the fork block is treated as a real segment boundary and each resulting "segment"
   gets a tag. For the tail piece, `_DIRECTIVE_PREFIX_RE` consumes the closing
   `%xprompts_enabled:true` line, so the tag lands immediately in front of `# New Query`
   — exactly the reported symptom.

Reproduced directly against the installed tree:

```python
from sase.history.chat_fork.build import _wrap_fork_history
from sase.xprompt._parsing import inherit_vcs_workflow_tag

body = ("**User:**\n\nturn one\n\n**Assistant:**\n\nreply one\n\n---\n\n"
        "**User:**\n\nturn two\n\n**Assistant:**\n\nreply two")
prompt = "#git:home \n" + _wrap_fork_history("# Previous Conversation", body) + "\n\nCan you now do X?"
inherit_vcs_workflow_tag(prompt, "#git:home ")
```

yields `#git:home **User:**` in the middle of the transcript and `#git:home # New Query`
at the end.

### Why the injected tag is inert today (and why that is luck, not design)

`_expand_embedded_workflows_in_prompt`
(`src/sase/xprompt/workflow_executor_steps_embedded_expand.py:89`) _does_ protect
disabled regions before it splits on `---`, and the resulting placeholder abuts the
injected tag with a NUL byte. `XPROMPT_REFERENCE_LEADING_CONTEXT`
(`src/sase/xprompt/_parsing_references.py:14`) only matches a reference at a string
start, after whitespace, or after a bracket, so the tag is not recognized as a reference
and survives into the final prompt as literal text instead of expanding a second VCS
workflow. That is the only reason this shows up as prompt noise rather than as
`Multiple VCS-tagged workflows in one prompt`
(`src/sase/xprompt/workflow_executor_steps_embedded_expand.py:203`). The fix must remove
the injection, not rely on that adjacency.

### Second-order hazard that must be fixed with it

`_wrap_fork_history` builds its disabled region by hand and does **not** escape
marker-shaped text in the body, unlike `wrap_disabled_region`
(`src/sase/xprompt/_disabled_regions.py:39`). A parent transcript that contains a
line-initial `%xprompts_enabled:true` — which real chats do;
`grep -rl '^%xprompts_enabled:' ~/.sase/chats/202608/` matches many files, because
assistant responses are never marker-stripped the way `sanitize_resume_prompt` strips
stored prompts — closes the region early. Verified: with such a body,
`protect_disabled_regions` captures only a prefix and two `---` separators remain
visible. Disabled-region protection alone would therefore still leak in exactly the
recursive-fork case that produces the worst noise, so the region must be made
trustworthy too.

## Fix

### 1. Add a shared, disabled-region-aware segment splitter

Add one helper (suggested home: a small new `src/sase/xprompt/_prompt_segments.py`, or
alongside `_SEGMENT_SEPARATOR_RE` in `src/sase/xprompt/_parsing_vcs_tags.py` if that
reads better with the existing imports) that returns a prompt's top-level segments and
the separators between them:

- protect fenced blocks, then protect disabled regions (this order matters: a marker
  quoted inside a code fence must stay inert, which is the order used by
  `src/sase/agent/multi_prompt_reference_rewriting.py:42-44`);
- split / find on `^---\s*$`;
- restore each piece fully (unprotect disabled regions first, then fenced blocks, so a
  fence nested inside a region is restored correctly) before returning it, so callers
  keep operating on real text and existing span arithmetic stays valid.

Callers must be able to both map-and-rebuild (VCS tag inheritance) and read spans
(artifact ref contexts), so return the restored pieces plus the matched separators.

### 2. Route the VCS-tag passes through it

In `src/sase/xprompt/_parsing_vcs_tags.py`, replace the ad-hoc `protect_fenced_blocks` /
`_SEGMENT_SEPARATOR_RE.split` / rebuild bodies in `inherit_vcs_workflow_tag` (`:172`)
and `normalize_default_vcs_workflow` (`:236`) with the shared helper. Keep
`_split_frontmatter_block` (`:201`) running first and unchanged — a leading YAML
frontmatter block is still split off before segmentation.

Also harden tag insertion in `_inherit_vcs_workflow_tag_segment` (`:148`) and
`normalize_default_vcs_workflow_segment` (`:108`): when the body a tag is being inserted
in front of begins with a `%xprompts_enabled:false` marker, insert `f"{tag}\n"` rather
than `f"{tag} "`, so the marker keeps its own line and the region stays parseable. This
is the same invariant `ensure_disabled_region_at_line_start`
(`src/sase/xprompt/_disabled_regions.py:21`) protects on the expansion side. Today this
path is unreachable (a fork-led prompt with no leading tag also has no inherited tag to
apply), so it is a guard rather than a live bug — but it becomes reachable the moment
either caller is used on an already-expanded prompt, and getting it wrong silently
re-enables xprompt expansion across a whole parent transcript.

### 3. Route the artifact-ref segment pass through it too

`_prompt_segments` (`src/sase/artifact_ref_prompt_context.py:305`) is the same splitter,
copied, and it runs on an already-fork-expanded prompt: once through
`prompt_segment_vcs_refs` (`:270`) from `preprocess_prompt_early`
(`src/sase/llm_provider/preprocessing.py:121`), and again from `preprocess_prompt_late`
(`:230`). It must move to the shared helper **in the same change**, not as a follow-up:

- it has the same defect today (a fork block's `---` lines fragment the prompt into
  bogus segments for artifact-ref resolution), and
- once step 2 stops injecting tags, an unfixed `_prompt_segments` would newly see a tail
  segment with **no** VCS tag where it previously saw the injected one, changing which
  project `@`-refs in the new query resolve against. Fixing both together keeps the
  whole prompt one segment carrying the launch's real leading tag.

Preserve the existing span semantics exactly: spans are computed from the restored piece
lengths plus separator lengths, so they must continue to index the original prompt.

### 4. Make the fork block's disabled region trustworthy

Change `_wrap_fork_history` (`src/sase/history/chat_fork/build.py:138`) to build its
region through `wrap_disabled_region` (`src/sase/xprompt/_disabled_regions.py:39`) — or,
if the exact surrounding whitespace is easier to keep by hand, to run the same
`_escape_disabled_region_markers` neutralization over the heading and body before
wrapping. The emitted shape must not otherwise change: the region still opens on its own
line, still ends with the trailing `---`, and the block still ends with
`%xprompts_enabled:true\n# New Query`, because `build_fork_injected_history`'s callers
and `tests/test_fork_history.py:191-212` pin that shape.

## Out of scope (deliberate, with reasons)

- `parse_multi_prompt` / `is_multi_prompt` (`src/sase/agent/multi_prompt.py:102`) and
  its callers. `#fork` is launch-deferred (`LAUNCH_DEFERRED_XPROMPT_NAMES`,
  `src/sase/xprompt/processor.py:80`), so every `parse_multi_prompt` caller sees the
  prompt before the fork block exists. The one in-runner caller,
  `src/sase/axe/run_agent_directives.py:93`, immediately re-joins the segments with
  `"\n---\n"`, so a spurious split there is a no-op.
- `xprompt_segment_count` (`src/sase/xprompt/segment_separators.py:18`), which runs on
  authored xprompt source content rather than on an expanded agent prompt.

Note in the implementation PR/commit that these two remain fenced-block-only, so the
next reader does not mistake the inconsistency for an oversight.

## Notes for the implementer

- This stays in Python. The surrounding segmentation and VCS-tag logic already lives in
  `src/sase/xprompt/`, with only leaf scanners (e.g. `fenced_block_details`) behind the
  Rust binding; this is a bug fix inside existing Python behavior, not new shared domain
  behavior that needs to move to `sase-core`.
- No feature flag: this is a defect fix with no old branch that must stay reachable.
- `just install` may be required before anything else runs in a cold workspace.

## Tests

Add real regression coverage, not just refactor-proof assertions:

1. `tests/test_xprompt_vcs_tag_replacement.py` (it already has the `_patch_ref_patterns`
   / `get_known_project_workspaces` harness used by
   `test_inherit_vcs_tag_applies_per_multi_prompt_segment:149`):
   - a fork-shaped prompt — leading tag, `%xprompts_enabled:false` … multi-turn body
     with an internal `---` … `%xprompts_enabled:true`, `# New Query`, query — is
     returned **unchanged** by `inherit_vcs_workflow_tag`; assert specifically that
     `# New Query` is not preceded by a tag and that no tag appears inside the region.
   - the existing genuine multi-prompt case still gets a tag per segment (keep
     `test_inherit_vcs_tag_applies_per_multi_prompt_segment` passing as-is).
   - a prompt mixing both: real `---` segments _outside_ a disabled region still
     inherit, while the region's internal `---` lines do not.
   - `normalize_default_vcs_workflow` gets the mirrored coverage.
   - the step-2 guard: inserting a tag in front of a body that opens with
     `%xprompts_enabled:false` leaves that marker at a line start.
2. `tests/test_fork_history.py`: a transcript body containing a line-initial
   `%xprompts_enabled:true` (and one containing `%xprompts_enabled:false`) is escaped so
   `protect_disabled_regions` captures the whole block as exactly one region and no
   `---` separator survives protection.
3. `tests/test_fork_workflow.py`: extend the end-to-end path at
   `tests/test_fork_workflow.py:355-381`, which already drives
   `_expand_embedded_workflows_in_prompt` with a fake `gh` workspace workflow but
   constructs its `WorkflowExecutor` **without** an inherited tag. Add a sibling that
   passes `inherited_vcs_tag="#gh:sase "` through the same `inherit_vcs_workflow_tag` →
   expand → `preprocess_prompt_late` sequence the runner uses, and assert the final
   prompt contains no `#gh:` text at all and ends with a clean `# New Query` followed by
   the query.
4. `tests/` coverage for `prompt_segment_vcs_refs`: a fork-shaped prompt yields a single
   segment whose VCS ref is the launch's leading tag, rather than one entry per `---` in
   the transcript.

## Verification

- `just check` must pass.
- Because the change touches shared prompt-preprocessing used by every agent launch,
  hand `just check-full` to `/sase_monitor` (`TESTING` / `TESTED`) before landing.
- Sanity check the real artifacts after a subsequent fork launch: the new run's
  `workflow-*-main_prompt.md` must contain zero `#gh:<project>` occurrences that the
  user did not type, and its `# New Query` line must be bare.
