---
tier: tale
title: Give the finalizer declaration-recovery turn evidence of its own run
goal:
  A declaration-recovery turn can tell which dirty paths its own run produced, so it
  stops refusing the commit finalizer on the false belief that the files predate it.
size: medium
proposed_by: bbugyi200.athena.0bq
create_time: 2026-08-23 09:54:51
status: wip
---

# Plan: Give the finalizer declaration-recovery turn evidence of its own run

## Incident this plan fixes

Agent `sase-s9.2` (run `20260823120149`, artifacts under
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/23/20260823120149/`)
implemented its phase in full, then died with:

```
BuiltinCommitFinalizerError: commit finalizer refused dirty repository main:
This turn is a declaration-recovery turn with no actual user request or work
performed by this agent; the dirty files predate this conversation and I lack
context to write an accurate commit message or authorize a commit.
```

The refusal was factually wrong. `tool_calls.jsonl` for that run records 16
`Edit`/`Write` calls between 13:09 and 13:25 UTC, covering exactly the seven paths the
finalizer listed as its obligation (`Justfile`,
`src/sase/ace/query_profile/{__init__,pane_registry,profiles}.py`,
`src/sase/ace/tui/_proc_query.py`, `tests/ace/tui/test_proc_query.py`,
`tests/test_query_profile.py`). The agent refused to commit its own work, the run
failed, and the workspace was later reset — the phase's implementation is gone.

## Root cause

`tool_calls.jsonl` for that run contains **two** Claude session IDs:

- `d7ac5ea3-…` — the main turn. Every `Edit`/`Write` call belongs to it.
- `ac22a157-…` — the declaration-recovery turn. Four tool calls: load `/sase_final`,
  `sase final context -f json`, one orientation command
  (`git status; git diff --stat; git log --oneline -5`), then the refusing manifest.

They are different sessions because every SASE provider invocation is stateless.
`ClaudeCodeProvider._invoke_loop` mints a fresh `--session-id` UUID per call
(`src/sase/llm_provider/claude.py:318`); `grok.py:314` does the same, and no provider
resumes a prior session. So when `ensure_final_declaration_or_recover` calls
`provider.invoke(prompt, …)` (`src/sase/finalizers/declaration_recovery.py:61`), the
recovery turn starts with an **empty context window**.

Its entire input was `_declaration_recovery_prompt`
(`src/sase/finalizers/declaration_recovery.py:107`), which is five sentences long and:

- asserts "after your normal response" while supplying zero evidence of that response;
- says "Do not perform unrelated work, do not mutate repositories, and do not answer the
  user yet" — which, read from an empty context, sounds like _you did nothing, touch
  nothing_;
- carries no original prompt, no file list, no diff, and no provenance.

The agent did try to orient itself, but `git` cannot attribute a dirty tree to an
author: `git log` describes committed history, and the two brand-new files were
untracked so `git diff --stat` showed nothing for them. Given no evidence either way, it
picked the reading that felt safest and refused. `dispatch_commit_decisions` converts
any refusal into a terminal `BuiltinCommitFinalizerError`
(`src/sase/finalizers/commit_dispatch.py:87-96`).

**The host already had the proof and never published it:**

| Evidence                                                   | Where it lives at recovery time                                                                       |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| The main turn's full response, naming each file it created | `invoke_result.content` — already a parameter of `ensure_final_declaration_or_recover`                |
| The expanded prompt the run was given                      | `original_prompt` — already a parameter of `run_finalizers` (`controller.py:57`), never threaded down |
| Which paths were already dirty before the run started      | `finalizer_baseline.json` in the artifacts dir                                                        |
| Which paths this run wrote                                 | `tool_calls.jsonl` in the artifacts dir                                                               |

The baseline evidence is decisive on its own. `collect_dirty_state` already subtracts
every path that is provably unchanged since the run-start snapshot
(`src/sase/llm_provider/commit_finalizer_state.py:135` →
`_exclude_pre_existing_baseline`). For run `20260823120149` the baseline recorded only
the `beads` linked repo — the main repo was clean at run start. So _by construction_ not
one of those seven paths could have predated the conversation, and the host could have
said so in one sentence.

## Goal

The declaration-recovery turn receives a host-authored, bounded evidence brief
describing what the run was asked to do, what it reported doing, and what the host knows
about the provenance of each dirty path — so a context-blind refusal stops being the
path of least resistance.

## Non-goals

- **Do not** add provider session resumption. It would be the "real" fix, but every
  runtime differs (Muse documents that it has no headless resume,
  `src/sase/llm_provider/muse.py:438`) and the `Uniform Agent Runtimes` convention in
  `CLAUDE.md` forbids runtime-specific capability branching.
- **Do not** change the finalizer wire contract. `FinalizerObligationWire`
  (`src/sase/core/finalizer_wire.py:90`) feeds `validate_finalizer_context`, which is a
  Rust binding (`src/sase/core/finalizer_facade.py:91`); adding a provenance field there
  means changing `../sase-core` and every context digest. The evidence brief is
  host→agent prompt text and a plain artifact file, so nothing crosses the Rust
  boundary.
- **Do not** change refusal policy. `finalizers.instances.<id>.refusal` currently
  accepts only `"fail"` (`src/sase/finalizers/config.py:260-264`); adding a softer value
  is a config-schema change with its own wire implications. Note it as follow-up
  instead.
- **Do not** fix why `sase-s9.2` skipped `/sase_final` in the first place (it looped
  waiting on a background `just test-scoped` monitor and returned without declaring).
  That is a distinct bug; this plan only makes the recovery path work correctly. File it
  separately.

## Design

### 1. New module: `src/sase/finalizers/declaration_recovery_evidence.py`

One public entry point:

```python
def build_recovery_evidence(
    *,
    context: FinalContextPublication,
    original_prompt: str | None,
    response_text: str,
    artifacts_dir: str | None,
) -> str:
    """Render the host's bounded, best-effort brief for a recovery turn."""
```

It returns Markdown with these sections. **Every section is independently byte-bounded**
and, when trimmed, ends with an explicit `… (truncated; showing N of M characters)`
marker so the agent never mistakes a cut for the whole story. Suggested caps: 2000 chars
for the prompt, 4000 for the response, 100 listed paths per repository.

1. **`## What this run was asked to do`** — the head of `original_prompt`.
2. **`## What this run reported doing before it stopped`** — the **tail** of
   `response_text` (the end of a turn is closest to the final tree state).
3. **`## Uncommitted paths the host is asking you to decide about`** — for each
   `kind: repository` obligation in `context.context.obligations`, the display name and
   its paths, each tagged with a provenance label from step 2 below.
4. **`## How the host knows whose changes these are`** — the provenance statement.
5. **`## Files this run wrote directly`** — best-effort, from `tool_calls.jsonl`.

Any section whose source is missing or unreadable is **omitted**, never faked. The whole
builder is best-effort: it must never raise into the recovery path. Wrap the body so
that an unexpected failure logs a warning and yields an empty brief, matching how
`capture_dirty_baseline` degrades
(`src/sase/llm_provider/commit_finalizer_baseline.py:32-57`).

### 2. Path provenance from the run-start baseline

Map obligation → repository path with
`load_accepted_host_repositories(Path(artifacts_dir))`
(`src/sase/finalizers/declaration_store.py:90`), which reads `final_context_host.json` —
already written by `publish_final_context` before recovery runs. Load the snapshot with
`load_dirty_baseline(Path(artifacts_dir))`
(`src/sase/llm_provider/commit_finalizer_baseline.py:114`) and key it with
`normalize_path` from `commit_finalizer_git`.

Emit exactly one of three labels per path, and **only** what the data supports:

| Condition                                                         | Label                                                                                           |
| ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Baseline loaded, repo has no baseline record                      | `new since run start` — the repo was clean when this run began                                  |
| Baseline loaded, repo recorded, path absent from its fingerprints | `new since run start`                                                                           |
| Baseline loaded, repo recorded, path present in its fingerprints  | `changed since run start` — was already dirty when this run began, and this run edited it again |
| Baseline missing or unreadable                                    | `provenance unknown`                                                                            |

The section-4 statement is gated on the same fact, because overclaiming here is exactly
the failure mode in reverse:

- **Baseline available** — say that the host snapshotted dirty paths before the first
  turn and that `collect_dirty_state` already subtracts every path provably unchanged
  since; therefore each listed path is this run's own work, not another agent's
  in-flight edit.
- **Baseline unavailable** — say plainly that no run-start baseline was captured, so the
  host cannot rule out pre-existing dirt, and the agent should inspect the diff before
  deciding.

### 3. Best-effort `tool_calls.jsonl` scan

`tool_calls.jsonl` rows carry `tool_name` and `tool_input_summary.file_path` for
`Edit`/`Write` (verified against run `20260823120149`). Extract the distinct `file_path`
values for `event == "ToolUse"` and `tool_name` in `{Edit, Write, NotebookEdit}`, render
them workspace-relative, and cap the list.

Do this with a small local JSONL scan inside the new module. **Do not** import
`sase.ace.tui.tools.reader`: that is TUI presentation code and the finalizer path must
not depend on it. Skip malformed lines silently; drop the section entirely if the file
is absent.

### 4. Rewrite `_declaration_recovery_prompt`

Keep the guardrails that still make sense (single recovery turn, don't do unrelated
work, don't mutate repositories, use `/sase_final`, the context digest) and add:

- the evidence brief, when non-empty;
- an explicit statement that the work described above **is this turn's own run**, so the
  agent should write the commit message from that evidence;
- an explicit list of what is **not** a valid refusal reason: "I have no context", "this
  is only a recovery turn", "these files predate me", "I did not do this work". Name
  what _is_ valid: protected paths, changes that genuinely belong to another repository
  or agent, or a tree the agent can see is unsafe to commit.

Keep the existing sentinel string `single declaration-recovery turn` intact — the
current test asserts on it (`tests/test_finalizer_declaration_channel.py:324`).

### 5. Thread `original_prompt` down

`run_finalizers` already receives `original_prompt`
(`src/sase/finalizers/controller.py:57`) and drops it on the floor. Add a keyword-only
`original_prompt: str | None = None` parameter to:

- `ensure_final_declaration_or_recover` (`declaration_recovery.py:27`)
- `ensure_current_declaration` (`controller_context.py:99`)

and pass it at **all three** `_ensure_current_declaration` call sites in
`controller.py`: line 96 (pre-loop), line 151 (per-cycle, before the built-in commit
instance), and line 363 (inside `_run_budgeted_commit`, on the
`stale_commit_declaration` retry). The third site is the easy one to miss —
`_run_budgeted_commit` does not currently receive `original_prompt`, so it needs the
parameter added and forwarded from its caller too.

Defaulting to `None` throughout keeps the existing tests and any external callers
working.

### 6. Persist the brief as an artifact

Write the rendered brief to `final_declaration_recovery_evidence.md` in the artifacts
dir via the existing `write_text_atomic` already imported in `declaration_recovery.py`,
next to the prompt and response files. Postmortems then show exactly what the recovery
turn was told. Add the filename as a module constant alongside
`FINAL_DECLARATION_RECOVERY_PROMPT_FILENAME`.

Keep using `FINAL_DECLARATION_RECOVERY_PROMPT_FILENAME` as the one-shot-budget sentinel
(`controller_context.py:declaration_recovery_spent`); do not move that marker to the new
file.

### 7. Skill guidance: `src/sase/xprompts/skills/sase_final.md`

Under `## Rules`, add that a `refuse` decision needs a substantive reason about the
_changes_, and that missing conversational context is not one — in a recovery turn,
build the commit message from the host's evidence brief rather than assuming no work
happened.

This file is a skill source template, not a `sase/memory/*.md` note, so the memory-file
permission gate does not apply. It **is** a generated-skill source: per
`sase/memory/generated_skills.md`, land the template change on the canonical branch
first and only then run `sase skill init --force`. Do not deploy from a dirty tree as
part of this work.

## Implementation steps

1. Add `src/sase/finalizers/declaration_recovery_evidence.py` with
   `build_recovery_evidence` plus private helpers for truncation, baseline provenance,
   and the `tool_calls.jsonl` scan.
2. Thread `original_prompt` through `controller.py` → `controller_context.py` →
   `declaration_recovery.py` as described in design step 5.
3. Rewrite `_declaration_recovery_prompt` to take the brief and emit the new refusal
   guidance; write the brief artifact; export the new filename constant and re-export it
   from `src/sase/finalizers/declaration.py` if the existing constants are re-exported
   there (they are, at lines 56-58 and 455-456).
4. Update `src/sase/xprompts/skills/sase_final.md`.
5. Add tests (below).
6. Run `just install` first (workspaces are ephemeral), then `just check`. Resolve any
   symvision findings for the new public symbols — read `sase/memory/symvision.md` with
   `/sase_memory_read` if they appear.

## Testing

New `tests/test_finalizer_declaration_recovery_evidence.py`, unit-testing the builder
against a temp artifacts dir:

- Baseline present and repo absent from it → every path labelled `new since run start`,
  and section 4 carries the strong "this run's own work" statement.
- Baseline present with the repo recorded and a path fingerprinted → that path labelled
  `changed since run start`, others `new since run start`.
- Baseline file missing → paths labelled `provenance unknown` and section 4 carries the
  hedged wording, not the strong claim. **This is the regression guard against
  overclaiming.**
- Oversized prompt and response are truncated, and the truncation marker is present.
- The response section shows the **tail**, not the head, of a long response.
- A malformed or absent `tool_calls.jsonl` drops that section without raising.
- A `tool_calls.jsonl` with `Edit`/`Write` rows produces the distinct file list.
- An unreadable artifacts dir yields an empty brief rather than an exception.

Extend `tests/test_finalizer_declaration_channel.py`:

- Reshape `test_missing_required_declaration_gets_one_fresh_recovery_turn` so the fake
  provider asserts the prompt now contains the obligation paths and the "not a valid
  refusal reason" guidance, alongside the existing `single declaration-recovery turn`
  and `/sase_final` assertions.
- New test: with `original_prompt` supplied, the recovery prompt contains an excerpt of
  it; with `original_prompt=None`, the prompt still renders and the section is simply
  absent.
- New test: `final_declaration_recovery_evidence.md` is written to the artifacts dir,
  and `declaration_recovery_spent` still keys off the prompt file.

**Incident-shaped regression test** — the one that would have caught this. Reconstruct
run `20260823120149`'s shape: a baseline with no `main` record, an obligation listing
several main-repo paths, and a fake provider that submits a `refuse` manifest. Assert
the recovery prompt handed to that provider states the paths are this run's work. This
pins the fix to the actual failure rather than to the helper's internals.

## Verification

`just check` (after `just install`). If the scoped selection escalates or reports
anything unusual, follow up with `just check-full` through `/sase_monitor` using
`--start-status TESTING --stop-status TESTED`.

## Risks

- **Prompt bloat.** The brief lands in every recovery turn. Hard per-section caps keep
  it to a few KB; prefer dropping a section over emitting an unbounded one.
- **Overclaiming provenance.** Asserting "this is your work" when the baseline is
  missing would re-create this bug with the polarity flipped, and a wrong commit is
  worse than a wrong refusal. The gating in design step 2 and its dedicated test are the
  mitigation.
- **Best-effort I/O in the failure path.** Every new read touches the artifacts dir
  while the run is already failing. The builder must swallow its own exceptions; a
  missing brief degrades to today's behavior.

## Follow-up worth filing separately

- `sase-s9.2` never called `/sase_final`: it entered a poll/wait loop on a background
  `just test-scoped` monitor and returned. Six consecutive "I'll wait for the background
  task" replies appear in its `live_reply.md`. That is the trigger that made recovery
  necessary at all.
- The conflict-repair turn (`src/sase/finalizers/commit_repair.py:315`) invokes the
  provider the same context-free way. It is more self-orienting because conflict markers
  are visible in the files, but it has the same structural blind spot.
- `refusal` policy accepts only `"fail"`, so a bad refusal is always terminal and always
  strands a held workspace.
