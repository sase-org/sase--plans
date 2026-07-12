---
create_time: 2026-07-09 13:16:45
status: done
prompt: 202607/prompts/agent_derived_at_suffixes.md
tier: tale
---
# Plan: Use `@` Templates for Derived Agent Names

## Goal

Change derived agent names so fork, wait, and retry launches use the existing agent-name template machinery instead of
custom numeric suffix counters:

- `#fork:foo` / resume-derived launch: `foo.f-0`, `foo.f-1`, ...
- `%wait:foo`-derived launch: `foo.w-0`, `foo.w-1`, ...
- retry launch: `foo.r-0`, `foo.r-1`, ...

The literal template shapes are `foo.f-@`, `foo.w-@`, and `foo.r-@`. This should reuse the same token order, namespace
reservation, planned-name reservation, and template-reference behavior as other `@` agent names.

## Current Behavior and Relevant Code

- Fork/resume-derived names are allocated in `src/sase/agent/names/_resume.py::allocate_resume_name` as `<base>.f<N>`.
- Wait-derived names are allocated in `src/sase/agent/names/_resume.py::allocate_wait_name` as `<base>.w<N>`.
- Retry names are allocated in `src/sase/agent/names/_retry.py::allocate_retry_name` as `<base>.r<N>`.
- Parent-side launch planning uses these functions in:
  - `src/sase/agent/multi_prompt_reference_allocator.py`
  - `src/sase/agent/repeat_launcher.py`
  - `src/sase/agent/launch_cwd_common.py`
  - `src/sase/xprompt/_directive_alt_naming.py`
- Child-side extraction in `src/sase/axe/run_agent_directives.py` accepts a planned fork name only if it matches the
  legacy `base.f<N>` regex.
- Retry entry points call `allocate_retry_name`, then rewrite or inject a concrete `%name:<retry-name>` directive.
- The linked Rust core template parser already accepts arbitrary one-`@` templates, so `foo.f-@`, `foo.w-@`, and
  `foo.r-@` do not require a parser change. User-name validation only rejects `--`, so rendered names like `foo.f-0` are
  valid.

## Design

1. Add small derived-template helpers near the name allocators:
   - `resume_agent_name_template(base) -> f"{base}.f-@"`
   - `wait_agent_name_template(base) -> f"{base}.w-@"`
   - `retry_agent_name_template(base) -> f"{base}.r-@"`

2. Reimplement the three allocation functions on top of `allocate_agent_name_template`:
   - `allocate_resume_name("foo")` returns the lowest available rendering of `foo.f-@`.
   - `allocate_wait_name("foo")` returns the lowest available rendering of `foo.w-@`.
   - `allocate_retry_name("foo")` returns the lowest available rendering of `foo.r-@`.

3. Use the durable reserved-name registry and template namespace index for derived names, matching normal `@`
   allocation. This is an intentional behavior change from the legacy active-only `.f<N>/.w<N>/.r<N>` counters. Legacy
   names remain resolvable historical names, but new allocations start in the new template namespace, so an old `foo.f1`
   does not force the first new fork name to skip `foo.f-0`.

4. Preserve efficient batch allocation:
   - `allocate_resume_names` and `allocate_wait_names` should build one `get_reserved_agent_names()` snapshot and one
     `AgentNameNamespaceReservationIndex`, then allocate all names from it.
   - `PlannedNameAllocator` should allocate derived fork/wait names through its existing `_allocate_template_name(...)`
     path so parent-side planned reservations, rollback, and concurrent launch handling remain consistent.

5. Update child-side planned-name acceptance:
   - Replace `_planned_name_matches_resume_target`'s `base.f<N>` regex with a template match against `f"{base}.f-@"`.
   - Keep a legacy `base.f<N>` acceptance branch if needed for compatibility with already-started parent processes or
     tests that inject legacy planned names.

6. Update fan-out naming:
   - In `_directive_alt_naming.py`, when a model/alt fan-out needs an inferred fork/wait base, use the new allocation
     functions so generated child names become `foo.f-0.<suffix>` and `foo.w-0.<suffix>`.
   - Existing auto-template fan-out (`@.cld`, `@.1`, etc.) stays unchanged.

7. Update retry launch surfaces:
   - TUI retry edit/relaunch and mobile retry should get `foo.r-0` from `allocate_retry_name` and continue using the
     existing prompt rewrite path.
   - Tests that patch `allocate_retry_name` can keep patching the public API; only expected concrete examples should
     change where the real allocator is exercised.

8. Update docs, comments, and test fixtures that describe the derived-name format. Avoid changing historical fixtures
   whose purpose is to prove old names can still be loaded, grouped, or displayed.

## Test Plan

Focused tests to update or add:

- `tests/test_agent_names_resume.py`
  - first fork allocation returns `foo.f-0`
  - existing `foo.f-0.<child>` reserves token `0`
  - `allocate_resume_names("foo", 3)` returns `foo.f-0`, `foo.f-1`, `foo.f-2`
  - legacy `foo.f1` / `foo.r1` remains historical data but does not reserve the new `foo.f-@` namespace

- `tests/test_agent_names_wait_retry.py`
  - wait allocation returns `foo.w-0`
  - existing `foo.w-0.<child>` reserves token `0`
  - retry allocation returns `foo.r-0`
  - existing `foo.r-0.<child>` reserves token `0`

- `tests/test_agent_names_extract.py` and `tests/test_axe_run_agent_phases_tags.py`
  - child-side extraction names fork-derived agents as `foo.f-0`
  - child-side extraction names wait-derived agents as `foo.w-0`
  - planned fork names like `foo.f-0` are accepted for `#fork:foo`

- `tests/test_launch_planned_agent_name.py`
  - single-prompt planned wait names use `foo.w-0`

- `tests/test_multi_prompt_launcher_planned_name_allocator.py`, `tests/test_multi_prompt_launcher_launch.py`,
  `tests/test_multi_prompt_launcher_template_refs.py`, and `tests/test_multi_prompt_launcher_launch_env.py`
  - parent-side fork/wait planned names use `.f-<token>` / `.w-<token>`
  - sibling `%wait:foo` segments get `foo.w-0`, `foo.w-1`
  - template references still rewrite to planned concrete names
  - uncommitted planned template reservations still release on spawn failure

- `tests/test_repeat_launcher.py`
  - repeat over `#fork:foo` uses `foo.f-0`, `foo.f-1`, ...
  - repeat over `%wait:foo` uses `foo.w-0`, `foo.w-1`, ...
  - injected `%wait:<previous-name>` lines use the new concrete names

- `tests/test_directives_split_models_naming.py` and `tests/test_directives_split_models_alts.py`
  - model/alt fan-out derived bases become `foo.f-0.<suffix>` and `foo.w-0.<suffix>`

- `tests/ace/tui/test_retry_edit_agent_name.py`, `tests/test_mobile_agent_kill_retry.py`, and
  `tests/test_mobile_agent_bridge_smoke.py`
  - retry examples use `.r-0` where they exercise the real allocator or the new public convention

Verification:

1. Run `just install` in the SASE workspace before checks, per repo guidance.
2. Run targeted pytest files listed above while iterating.
3. Run `just check` before finalizing implementation changes.

## Risks and Mitigations

- **Legacy fixture churn:** Many UI/grouping tests intentionally use historical `.f1/.w1/.r1` examples. Only change
  tests whose expected output comes from allocation behavior; leave historical loader/rendering fixtures intact.
- **Parent/child disagreement:** The parent allocator and child extractor must both accept the new fork-derived shape.
  Update `_planned_name_matches_resume_target` and add a direct test.
- **Batch reservation semantics:** Reusing `_allocate_template_name` in `PlannedNameAllocator` is safer than adding
  parallel derived-name reservation state because it already handles template namespaces, planned registry entries,
  rollback, and concurrent launches.
- **Behavioral migration:** Old `.f<N>/.w<N>/.r<N>` names remain valid for lookup, grouping, and wait/fork references,
  but they no longer reserve the new `@` token namespace. This should be called out in release notes or a changelog if
  the project has one for user-visible naming changes.
