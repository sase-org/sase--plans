---
tier: tale
title: "Rewrite the #coder xprompt to name the plan instead of inlining it"
goal:
  'The #coder xprompt renders "The 202608/foo.md plan file has been reviewed and
  approved. Implement it now.", stripping any plan path or canonical plan reference down
  to its YYYYmm/<name>.md portion, so the coder locates and reads the plan itself
  instead of receiving it pre-inlined.'
size: small
proposed_by: bbugyi200.athena.0ci
---

# Plan: Rewrite the `#coder` xprompt to name the plan instead of inlining it

## Goal

Replace the body of the built-in `#coder` xprompt so it names the approved plan by its
`YYYYmm/<plan_name>.md` reference path instead of inlining the file with `@`:

```
The 202608/foo.md plan file has been reviewed and approved. Implement it now.
```

The `plan_file` input keeps its current name and `path` type — callers still pass
whatever plan path they have (`~/.sase/plans/202608/foo.md`, an absolute archive path, a
canonical `plan:202608/foo.md` reference) — and the xprompt strips it down to the
`YYYYmm/<name>.md` portion at render time.

The point is to stop handing the agent pre-inlined bytes and instead make it locate and
read the plan itself, which in the good case routes it through
`sase artifact read plan:202608/foo.md "<reason>"` and leaves an audited read edge in
the artifact graph.

## Current behavior

`src/sase/xprompts/coder.md` is a single prompt-part `.md` xprompt:

```markdown
---
name: coder
description: Ask an agent to implement an approved plan file.
input:
  - name: plan_file
    type: path
    description: Path to the approved plan the agent should implement.
---

@{{ plan_file }}

The above plan has been reviewed and approved. Implement it now.
```

Verified today:

```
$ sase xprompt expand '#coder:~/.sase/plans/202608/accept_command_line_tools.md'
@~/.sase/plans/202608/accept_command_line_tools.md

The above plan has been reviewed and approved. Implement it now.
```

`@{{ file }}` inlines the file contents at launch (per the xprompts memory note), so the
coder currently never has to find or read the plan itself.

## Design

### 1. Normalization helper (`src/sase/sdd/plan_refs.py`)

`src/sase/sdd/plan_refs.py` already owns plan-reference parsing and canonicalization and
delegates the grammar to `sase_core_rs`. Add one public function there rather than
inventing string surgery elsewhere:

```python
def plan_reference_display_path(
    value: str,
    *,
    roots: tuple[Path, ...] | None = None,
) -> str:
    """Return the ``YYYYmm/<name>.md`` portion of a plan path or reference."""
```

Algorithm:

1. Strip surrounding whitespace. Empty input returns _value_ unchanged.
2. `parse_plan_reference(text)`. When the result is **not** `legacy`, the input was
   already a canonical `plan:` reference — return `parsed.path`.
3. Otherwise resolve plan roots
   (`resolve_plan_roots(*workspace_context_for_plan_resolution(Path.cwd()))` when
   _roots_ is `None`) and call
   `canonicalize_plan_reference_from_roots(text, roots=roots)`. On a hit, return the
   path portion of the canonical reference.
4. On any miss or any raised exception, return _value_ unchanged.

Step 4 is the important contract: **only strip when the value is provably a plan** — a
canonical reference, or a filesystem path that lives below an active plans root.
Anything else is passed through verbatim so the prompt still names something the agent
can actually open. That deliberately avoids a naive "keep the last two path components"
rule, which would turn `/tmp/notes.md` into the meaningless `tmp/notes.md`.

The `roots` keyword mirrors the module's existing `resolve_plan_reference_from_roots` /
`canonicalize_plan_reference_from_roots` shape and lets tests inject roots instead of
depending on the machine's SDD store. Add the new name to the module's `__all__`.

Behavior this produces (all four canonicalization results below were confirmed against
the live `sase_core_rs` binding in this workspace):

| `plan_file` input                             | rendered value                     |
| --------------------------------------------- | ---------------------------------- |
| `plan:202608/foo.md`                          | `202608/foo.md`                    |
| `/home/bryan/.sase/plans/202608/foo.md`       | `202608/foo.md`                    |
| `~/.sase/plans/202608/foo.md`                 | `202608/foo.md`                    |
| `sase/repos/plans/202608/foo.md` (store root) | `202608/foo.md`                    |
| `202608/foo.md`                               | `202608/foo.md` (passed through)   |
| `/tmp/scratch.md`                             | `/tmp/scratch.md` (passed through) |

Note that the Rust canonicalizer does not require the file to exist, expands `~`, and
resolves relative paths against the process CWD — all verified.

### 2. Jinja filter (`src/sase/xprompt/jinja_filters.py`, new)

A `.md` prompt part can only reach Python through a Jinja filter, so expose the helper
as one:

```python
def register_prompt_filters(env: Environment) -> None:
    env.filters["plan_ref_path"] = _plan_ref_path
```

`_plan_ref_path` returns non-`str` values untouched, imports `sase.sdd.plan_refs`
**lazily inside the function** (the `_jinja` module is deliberately low-level and avoids
importing heavier packages at module scope), and never raises — the helper already
swallows failures and returns the input.

Call `register_prompt_filters` from both environments so the filter behaves the same
wherever a prompt body is rendered:

- `src/sase/xprompt/_jinja.py::_get_jinja_env()` — xprompt bodies and top-level prompts.
- `src/sase/xprompt/workflow_executor_utils.py::create_jinja_env()` — workflow step
  bodies.

`src/sase/xprompt/jinja_inspect.py::jinja_filter_names()` already returns
`sorted(get_jinja_env().filters)`, so `plan_ref_path` shows up in the ACE Jinja
completion popup (`src/sase/ace/tui/widgets/jinja_completion.py:71`) with no extra
wiring.

Name rationale: `plan_ref_path` says what it returns — the path portion of a canonical
plan reference. `plan_path` would read like a filesystem path (it is not), and
`plan_ref` would imply the `plan:` prefix is kept (it is not).

### 3. New `src/sase/xprompts/coder.md`

```markdown
---
name: coder
description: Ask an agent to implement an approved plan file.
input:
  - name: plan_file
    type: path
    description: Path to the approved plan the agent should implement.
---

The {{ plan_file | plan_ref_path }} plan file has been reviewed and approved. Implement
it now.
```

Frontmatter is unchanged: same input name, same `path` type, same description. Only the
body changes.

`just fmt` runs prettier over `**/*.md` with `proseWrap: always` and `printWidth: 88`
(`package.json`), so the sentence will wrap as shown. That is cosmetic; the rendered
prompt is one paragraph either way.

## Explicitly out of scope — a decision for the approver

**The automated coder hand-off will not change.**
`src/sase/axe/run_agent_exec_plan_accept.py:535-539` builds the real post-approval coder
prompt from a hardcoded copy of the same text:

```python
prompt=(
    f"{model_prefix}{vcs_prefix}"
    f"@{coder_plan_ref}\n\n"
    "The above plan has been reviewed and approved. "
    f"Implement it now.{coder_extra}\n{embedded_refs}"
),
```

So after this plan lands, only a hand-typed or hand-composed `#coder:<plan>` gets the
new wording. Every coder launched by plan approval keeps receiving the inlined
`@plan:...` and will not be nudged toward `sase artifact read`.

I am leaving that site alone because changing it is a materially riskier change on the
primary coding path, not because it is uninteresting:

- Real production-driven assertions pin the current shape and would all need updating:
  `tests/test_axe_run_agent_exec_plan_followup_approval_plan_refs.py:60,103,104,151,152`,
  `tests/test_axe_run_agent_exec_plan_followup_coder_prompt.py:90,108`,
  `tests/test_axe_run_agent_exec_plan_followup_questions.py:553`. (The two string
  assertions in `tests/test_epic_approval.py:128,140` build their expected prompt inline
  and never call production code, so they are tautological either way.)
- There is a real timing hazard. `@<path>` inlines the plan when the successor's prompt
  is expanded; "go read it yourself" defers the read until after the VCS pre-step runs
  `git checkout . && git clean -fd`, which is exactly the wipe that
  `commit_sdd_files_for_exec_plan` in `src/sase/axe/run_agent_exec_plan_sdd.py` exists
  to work around. Durable-archive plans under `~/.sase/plans` are outside the repo and
  safe, but in-tree hand-offs such as the `@sdd/plans/202605/scratch_plan.md` case
  pinned by `tests/..._approval_plan_refs.py:60` are not obviously safe, and proving
  that out is its own investigation.
- Losing the inlined plan means a coder that ignores the instruction implements nothing.
  Across every approved plan that is a much larger blast radius than a hand-typed
  `#coder`.

**If you want the automated hand-off changed too, say so in plan feedback** and I will
extend this plan to cover `run_agent_exec_plan_accept.py`, the seven assertions above,
and the `git clean` timing question. A third option worth naming: make that site emit
`#coder:{coder_plan_ref}` and let normal xprompt expansion produce the body, which
removes the duplication permanently but also makes a user's local `~/xprompts/coder.md`
override the real coder hand-off.

Also considered and rejected for this change:

- **Keeping the `plan:` prefix** (`The plan:202608/foo.md plan file has been reviewed…`)
  would make `sase artifact read plan:202608/foo.md` a literal copy-paste for the agent
  and is arguably a better nudge — but you specified the bare `202608/foo.md` form, so
  that is what this implements. Easy to revisit.
- **A pure-Jinja body** such as `{{ plan_file.split("/")[-2:] | join("/") }}` needs no
  Python at all, but it cannot strip a `plan:` prefix, cannot be unit tested, and
  mangles non-plan paths. Rejected.
- **A new `InputType`** (e.g. `type: plan`) would normalize at bind time and keep the
  body as `{{ plan_file }}`, but `InputType` members are enumerated across notification
  gates, typed input forms, and gate input panels (~10 files). Far too much surface for
  one xprompt.
- **Converting `coder.md` to a `.yml` workflow** with a `python:` step would give direct
  Python access, but it changes `#coder` from an inline prompt part into a workflow and
  breaks composition. Rejected.

## Docs

`docs/xprompt.md` currently states something that this change makes false, so it must be
updated in the same commit:

- Line ~2309-2312: "Once the plan is approved, sase launches a follow-up **coder** agent
  using the same handoff body as the `#coder` built-in xprompt … `#coder` takes the
  approved plan file as its `plan_file` input, **injects it with `@`**, and instructs
  the agent to implement the plan."

  Rewrite so it (a) no longer claims the two bodies are identical, (b) describes
  `#coder` as naming the plan by its `YYYYmm/<name>.md` reference and asking the agent
  to read it (pointing at `sase artifact read`), and (c) states plainly that the
  automated hand-off still inlines the plan with `@`. Keep the surrounding `%model:` /
  size-routing prose unchanged.

- The **Template Context** table under `## Jinja2 Integration` (line ~792) documents the
  variables available to prompt bodies but has no filter list. Add a short subsection
  after that table documenting `plan_ref_path`: what it accepts, that it returns the
  `YYYYmm/<name>.md` portion, and that it passes non-plan values through unchanged.
  `| tojson` is mentioned at line ~2849 as the only other prompt-visible filter; keep
  the new entry equally brief.

## Tests

New file `tests/test_xprompt_plan_ref_filter.py`:

- `plan_reference_display_path` with explicit `roots` covering the six rows in the
  behavior table above, plus: empty string, whitespace-only, a value with surrounding
  whitespace, and a nested `plan:202608/sub/foo.md` (the parser returns
  `202608/sub/foo.md`, and the filter must not truncate it further).
- The filter itself renders through the real xprompt environment: assert
  `"plan_ref_path" in jinja_filter_names()` so the ACE completion wiring stays covered,
  and render a small template through `sase.xprompt._jinja` to confirm it is registered
  on the environment that actually renders prompt bodies.
- A non-`str` value (e.g. `None`) passes through the filter untouched.
- A failure path: monkeypatch `resolve_plan_roots` to raise and assert the filter
  returns the input rather than propagating — prompt expansion must never break on a bad
  workspace context.

New file `tests/test_xprompt_coder_builtin.py`:

- Load the bundled `coder` xprompt through the normal loader and expand it with a
  `plan:`-style input and with an archive-path input; assert the rendered text is
  exactly the target sentence with the stripped reference and that it contains no `@`
  injection.

Existing tests: none need editing. The only other places holding the old wording are
`tests/test_epic_approval.py:128,140` and
`tests/test_plan_approval_launch_reliability_integration.py:383`, which describe the
hardcoded runtime prompt (unchanged by this plan), and
`tests/sdd_store/test_workspace_clone_reconciliation.py:214`, which is a fixture string.

Symvision: `plan_reference_display_path` and `register_prompt_filters` are both public
and both consumed from non-test files (`jinja_filters.py`, `_jinja.py`,
`workflow_executor_utils.py`), so no pragma or whitelist entry is needed.

## Verification

`just install` first (ephemeral workspace), then:

```bash
sase xprompt expand '#coder:~/.sase/plans/202608/accept_command_line_tools.md'
sase xprompt expand '#coder:plan:202608/accept_command_line_tools.md'
sase xprompt expand '#coder:/tmp/scratch.md'
sase xprompt show coder
```

The first two must both print exactly:

```
The 202608/accept_command_line_tools.md plan file has been reviewed and approved.
Implement it now.
```

and the third must pass `/tmp/scratch.md` through unchanged.

Then `just fmt` (prettier will rewrap `coder.md` and the docs edits) and `just check`.
This change touches `src/sase/xprompt/_jinja.py`, which is imported broadly, so if the
scoped test lane escalates or reports an unusual selection, run `just check-full`
through `/sase_monitor` instead of inline.

## Risks

- **Coders that ignore the instruction.** Removing `@` means a coder that does not act
  on the sentence has no plan at all. Contained here because only hand-typed `#coder` is
  affected; see the approver decision section for why the automated path is untouched.
- **Pass-through masking a mistake.** If a caller passes a plan path that is not under
  an active plans root (a plan copied elsewhere, an unusual SDD store layout), the
  filter silently renders the full path instead of `YYYYmm/name.md`. That is the
  intended fallback — it still names a readable file — but it means "the strip did not
  happen" is not surfaced as an error.
- **CWD sensitivity.** With no injected `roots`, root resolution keys off the process
  CWD. Prompt expansion runs in the agent's workspace, so this is correct in practice,
  but expanding `#coder` with a relative path from an unrelated directory will fall back
  to pass-through.
- **New global filter surface.** `plan_ref_path` becomes visible to every prompt and to
  ACE's Jinja completion. It is additive and cannot collide with an existing filter
  name.

## Deliverables

| File                                          | Change                                                   |
| --------------------------------------------- | -------------------------------------------------------- |
| `src/sase/sdd/plan_refs.py`                   | add `plan_reference_display_path` + `__all__` entry      |
| `src/sase/xprompt/jinja_filters.py`           | **new** — `register_prompt_filters` and the filter impl  |
| `src/sase/xprompt/_jinja.py`                  | register filters in `_get_jinja_env()`                   |
| `src/sase/xprompt/workflow_executor_utils.py` | register filters in `create_jinja_env()`                 |
| `src/sase/xprompts/coder.md`                  | new body using `{{ plan_file \| plan_ref_path }}`        |
| `docs/xprompt.md`                             | correct the coder-handoff paragraph; document the filter |
| `tests/test_xprompt_plan_ref_filter.py`       | **new**                                                  |
| `tests/test_xprompt_coder_builtin.py`         | **new**                                                  |
