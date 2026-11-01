---
tier: tale
title: Make agent prompt selection independent of host filesystem ordering
goal:
  Fix the raw-versus-expanded prompt and hint failures for sase-1qt.1 and verify them on
  apollo and athena.
size: medium
proposed_by: bbugyi200.athena.sase-1qt.1
proposed_with: codex/gpt-6-astra
bead: sase-1qt.1
create_time: 2026-11-01 18:19:07
status: wip
---

- **PARENT:**
  [202611/task_backlog_epic_5.md](https://github.com/sase-org/sase--plans/blob/main/202611/task_backlog_epic_5.md)
- **BEAD:**
  [sase-1qt.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1qt/sase-1qt.1.md)

# Make agent prompt selection independent of host filesystem ordering

## Scope and completion owner

Complete assigned phase `sase-1qt.1` of `sase-1qt`, implementing the prompt-render
portion of `plan:202611/task_backlog_epic_5.md`. The seed tasks are `sase-1ia`,
`sase-1l3`, and `sase-1hy`; recheck `sase-1i9` as related evidence. Read those beads and
the parent plan through their audited commands before implementation.

The parent's final decision is `optional_scope = keep_five_seeds`: repair the selected
defects only. This tale authorizes no memory changes, adjacent features, new task beads,
or changes to the other two parent-epic phases. Record discovered work as
`PROPOSED FOLLOW-UP:` notes on `sase-1qt.1`. Only this assigned phase may be closed by
its worker; the parent land agent owns the seed-task and ancestor closures.

This is one bounded policy change with cross-language integration and cross-host
verification. The phase had no previous implementation or remaining-work notes at
planning time. A child epic is unnecessary unless new evidence materially widens the
work.

## Evidence and root-cause hypothesis

Planning inspected clean sase HEAD `9066546900`, immediately after the parent plan's
reproduction base `31a28d9618`. Earlier work has not removed the defect:
`src/sase/agent/artifact_files_cache.py::_ArtifactFileCache.select_prompt_file` still
accepts every `*_prompt.md` except commit-finalizer re-prompts, then sorts only by
descending `st_mtime_ns`.

The four primary files passed on athena during planning: **30 passed in 4.54s** with the
workspace Python 3.14.7 venv and `pytest -q -n 0 -p no:cacheprovider`. This is baseline
evidence only; the implementation must still run the matched cross-host recipe and the
expanded sibling coverage below.

Consequently `raw_prompt.md` is a candidate for the expanded AGENT PROMPT body. Python's
stable sort preserves the unsorted `os.listdir` order on timestamp ties. The cache
signature is sorted, so it can also retain a previously chosen order-dependent winner. A
read-only, mocked-filesystem probe of the actual method gave these results:

| Directory order | Raw mtime | Expanded mtime | Selected file   |
| --------------- | --------: | -------------: | --------------- |
| raw, expanded   |       100 |            100 | `raw_prompt.md` |
| expanded, raw   |       100 |            100 | `01_prompt.md`  |
| raw, expanded   |       100 |            101 | `01_prompt.md`  |
| expanded, raw   |       102 |            101 | `raw_prompt.md` |

The affected fixtures write raw and expanded files consecutively without setting their
timestamps. This confirms a product-level precedence defect; it does not yet prove which
timestamp granularity or directory order apollo supplies. Confirm that last connection
before editing product code. Do not mask it by sleeping between fixture writes or
forcing only favorable timestamps.

The shared path is:

- `prompt_panel/_agent_clan_member_content.py::_load_member_prompts` loads raw
  separately and calls `get_prompt_content` for AGENT PROMPT.
- `prompt_panel/_agent_display_content.py::get_prompt_content` delegates selection to
  `_ArtifactFileCache` and reads the selected file through the cache.
- Normal panels, session panels, hint rendering, and node-finder previews consume this
  selection. Selecting raw twice explains both missing expanded content and duplicate
  raw-path hints.

Existing selection tests are in `tests/agent/test_artifact_files_cache.py`. They cover
unequal timestamps, workflow-step preference, and finalizer exclusion but not raw-file
exclusion or timestamp ties. Canonical and legacy raw names are `raw_prompt.md` and
`raw_xprompt.md`, defined in `src/sase/legacy_xprompt_names.py`.

## Implementation

1. Establish before results on both hosts using the same source snapshot and core wheel.
   Follow the parent's apollo recipe: export the current workspace HEAD with
   `git archive`, transfer to a uniquely named `/tmp` scratch directory over
   `ssh apollo`, create a Python 3.12 venv with `~/.local/bin/uv`, and install `.[dev]`
   with an override pointing to the copied, exact athena abi3 wheel. Never run tests in
   an apollo SASE workspace. Do not browse or clone another checkout to obtain source;
   use the authorized local checkout snapshot. Remove owned scratch paths after
   retaining the evidence.

   Run the four primary files listed below with `-n 4 -p no:cacheprovider`. Capture
   actual candidate basenames, `st_mtime_ns`, directory order, and selected basename for
   a failing fixture on apollo and the corresponding athena case. Use test-local
   instrumentation or a small standalone selector probe; avoid logging real prompt
   bodies or scanning live agent artifacts. A serial probe distinguishes filesystem
   input from xdist scheduling. Force equal timestamps with `os.utime(ns=...)` and both
   enumeration orders on athena to reproduce the same wrong selection. If this does not
   account for every primary failure, trace the remaining failure before claiming the
   cluster resolved.

2. Put expanded-prompt eligibility and ranking in `sase_core`, since every frontend
   reading agent prompt artifacts should agree on it. Open `sase-core` using
   `/sase_repo`, read its `AGENTS.md`, and implement a small pure API over candidate
   records rather than moving filesystem I/O or the Python cache into Rust.

   A suitable domain is a focused `agent_prompt_selection` module exported from
   `crates/sase_core/src/lib.rs`; expose its serde request/result through a PyO3 binding
   registered in the existing `artifact_refs` binding domain. The nearby
   `prompt_artifact` module concerns referenced-document manifests, not selecting an
   agent's expanded prompt, so do not overload its manifest schema. Give the request the
   candidate paths/basenames and integer nanosecond mtimes plus `is_workflow_child` and
   optional `step_name`; return the selected path or None. Do not convert timestamps to
   floating point.

   The policy must:
   - Accept ordinary expanded `*_prompt.md` files, including existing submitted and
     follow-up prompt forms when they otherwise qualify.
   - Exclude the exact canonical and legacy raw-prompt names and historical
     `commit_finalizer_pass_<digits>_prompt.md` files. Raw has its own display surface;
     raw-only directories produce no expanded prompt selection.
   - Preserve matching `-<step_name>_prompt.md` preference for workflow children, with
     the existing fallback to other eligible prompts when no step matches.
   - Prefer the newest eligible integer mtime. For ties, choose the lexicographically
     greatest basename/path so the result is deterministic across directory orders and
     conventional numbered prompts prefer the later name. Document this tie rule; never
     rely on iteration order.

   Keep Python directory observation, stat error handling, signature computation,
   locking, and file-content caching in `_ArtifactFileCache`. Replace its selection
   policy with a thin call via `require_rust_binding`; do not add a Python fallback.
   Cache hits must continue avoiding policy recomputation. Account for candidate
   additions, removals, mtime changes, and an empty eligible set. Keep observations on
   the current execution path; add no extra render-time I/O.

   Do not broadly migrate other prompt readers or change feedback-replan/workflow
   selection semantics. If inspection finds the same defect there, record it as proposed
   follow-up unless it is directly required for these phase assertions.

3. Add focused regressions at the policy, binding, cache, and rendered-output seams.
   Rust tests and a binding round trip must prove raw exclusion with older, equal, and
   newer raw mtimes; both listing orders; deterministic ties between two real expanded
   files; raw-only and empty candidates; finalizer exclusion; and workflow step
   preference/fallback. Include legacy raw naming without removing existing
   legacy-reader coverage. Exercise cache invalidation with file additions/removals and
   changed mtimes in Python.

   Add a small end-to-end regression, preferably by parameterizing the existing clan
   precedence test or using its fixture setup, that pins equal mtimes and reversed
   enumeration while retaining the exact raw-versus-expanded assertions. Keep all
   existing full-section, path-map, fold-level, and hint-number assertions. Do not add
   skips, xfails, sleeps, suppressions, flake-baseline entries, or larger timeout
   budgets. Demonstrate that reverting the selection fix makes the forced host-input
   regression fail.

4. Build the edited core into the local workspace venv through the repository's
   supported recipes. Use the resulting identical abi3 wheel for the apollo after run; a
   published wheel without the new binding cannot verify the patch. Export the actual
   modified primary tree, not only pre-edit HEAD, for the remote after run, and record a
   source digest and wheel checksum. Do not globally install SASE. Read `/sase_monitor`
   before commands that need a long-command handoff and preserve the evidence and
   remaining instructions in that handoff.

## Verification and evidence

The required four-file suite under `tests/ace/tui/widgets/` is:

- `test_agent_clan_aggregation.py`
- `test_agent_display_agent_session_render.py`
- `test_identity_header_raw_prompt.py`
- `test_agent_display_raw_prompt.py`

Also run the sibling coverage named in the task records on both hosts:

- `test_agent_clan_aggregation_async.py`
- `test_agent_display_agent_session_hints.py`
- `test_agent_display_raw_prompt_hints.py`, including
  `TestAgentRawPromptHintMode::test_hint_mode_renders_timestamp_file_hints_before_body_hints`
  from `sase-1i9`
- `test_agent_header_panel_basic.py` and `test_agent_header_panel_preview.py`
- `test_agent_display_hint_cache.py` and `test_agent_prompt_semantic.py`

Use `rg` to locate any additional named node from the audited bead records if it moved;
preserve its assertion coverage. Run `tests/agent/test_artifact_files_cache.py` and the
new Rust and binding tests. Normal and forced-tie runs must agree on the expanded prompt
and hint mapping, with the four primary files passing on both hosts. Record host, source
SHA/digest, Python version, core source/wheel identity, commands, before/after counts,
and the measured host input that caused the failure.

Read `lint_and_test.md` before finishing. Format, then run `sase tool run check` in each
changed repository. Do not run `just check-full`. Core tests use `just` recipes, never
bare cargo. If rendered output or snapshot coverage changes, read `tui_screenshot.md`
and run targeted `just fix-tui-screenshots -- <selectors>` for the affected
agent/session prompt panels, including the monitor-metadata session snapshot named in
`sase-1l3`. Inspect the report and every changed golden; a partial capture is not
evidence that skipped snapshots pass.

When a separate check failure reproduces identically on the clean base, append a
`PROPOSED FOLLOW-UP:` note with the reproduction and any existing task owner and
continue completion as the phase prompt directs. The selected prompt-render defects
themselves remain the scope to fix.

## Completion

Record `TASK RESOLVED: <id> — <cause and verification>` for `sase-1ia`, `sase-1l3`, and
`sase-1hy` on `sase-1qt.1`, plus the actual apollo result for `sase-1i9`. Run
`sase bead epic-symbols sase-1qt.1`; resolve entries or re-key them to a still-open
owner before closing. Planning found no entries, but recheck at close. Close only
`sase-1qt.1` with `sase bead close sase-1qt.1 --note "<verified results>"` once the
whole scope is verified. Never set its status by hand and never close `sase-1qt` or
another ancestor.

Declare both changed repositories through `/sase_final`. Host completion commits core
first and updates `sase-core-revision.txt` to that pushed commit before the primary
commit; do not fabricate a SHA or alter the release-owned PyPI version pin. If the
binding was already landed separately, advance the CI source revision to the real
containing commit through the supported ratchet. Intermediate monitor or plan handoffs
keep the assigned bead open.
