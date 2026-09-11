---
tier: tale
title: Stop treating prose %if / %proc mentions as directives
goal:
  Prompts that mention %if or %proc as plain prose (bead titles, quoted text) launch
  normally in both typed_launch_units flag states, in the Python extractor and the Rust
  typed-launch planner, while all real directive forms keep their current hard errors.
size: medium
proposed_by: bbugyi200.athena.0j3
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.research.1t.cdx](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1t.cdx/README.md)
  - [bbugyi200.athena.research.1t.cld](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1t.cld/README.md)
  - [bbugyi200.athena.research.1t.final](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1t.final/README.md)
- **COMMITS:**
  - [0e76f6d](https://github.com/sase-org/sase--research/commit/0e76f6db51d21a6873eb16a1793d83c2797bff1a)
    — docs(research): add sase tool dynamic capacity claim research (researcher B)
  - [5209648](https://github.com/sase-org/sase--research/commit/5209648b3f852e90a9697a512b0604011f654a8a)
    — docs(research): design dynamic tool capacity
  - [d6fa54c](https://github.com/sase-org/sase--research/commit/d6fa54cd5811efdab4f344855a3c2dbd4c3e448d)
    — docs(research): consolidate command capacity and load meter design

# Stop treating prose `%if` / `%proc` mentions as directives

## Background: what killed agent `0ik`

The `0ik` pipe chain (user-requested notification task-bead review) died at its third
link. `0ik` and `0ik--review01` piped successfully, but the successor `0ik--review02`
failed during prompt preprocessing before its model ever ran, killing the chain
silently. Evidence: `workflow_state.json` in artifacts dir
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/11/20260911010744` records
`status: failed` with:

```
DirectiveError: %if requires %if:: followed by exactly one closed bash or python fence.
```

raised from `_collect_code_directive` (`src/sase/xprompt/_directive_collect.py`, the
`if` branch near the top of that function) via `preprocess_prompt_early` ->
`extract_prompt_directives`.

The piped prompt contained this bead title as plain prose:

> sase-wk: Abort a gated typed-admission launch bundle so ,X can stop un-admitted
> %if/%proc units

`%if` is preceded by a space, so `_DIRECTIVE_PATTERN`
(`src/sase/xprompt/_directive_types.py`) matched it as a bare directive token (no paren,
no colon arg, no `+`), and `_collect_code_directive` raises unconditionally for any
`%if` that reaches it. Any prompt that quotes prose mentioning `%if` (bead titles,
transcripts, docs excerpts) hard-fails the entire launch. In a pipe chain the failure
happens after the predecessor already terminated, so nobody can recover and the chain
dies without a notification.

## Root cause and fix semantics

Valid `%if::` / `%proc::` fence forms are consumed before directive collection by the
Rust owned-fence scanner (`scan_directive_owned_fences` in
`crates/sase_core/src/fenced_code.rs` of the sase-core repo), which itself diagnoses
missing/unclosed/empty fences and unknown languages (see `take_owned_fence`; the scanner
only recognizes a `%if::`/`%proc::` header alone on its own line). Therefore any _bare_
`%if`/`%proc` token that reaches the later collection passes is either (a) prose, or (b)
a `::` header with trailing text on the same line. Only (b) deserves a hard error.

**New rule, applied identically in both feature-flag states and on both sides of the
Rust core boundary:**

A `%if` or `%proc` token matched with **no parenthesis, no colon argument, no `+`
suffix, and not immediately followed by `::`** is prose. It must be skipped: no error,
no diagnostic, and the token stays in the prompt text delivered to the model. All other
forms keep their current behavior:

- `%if(...)`, `%if:arg`, `%if+` → existing "requires %if:: followed by exactly one
  closed bash or python fence" error.
- `%proc(...)`, `%proc:arg`, `%proc+` → existing paren/colon/body validation.
- Bare `%if`/`%proc` immediately followed by `::` (e.g. `%if:: echo hi` with trailing
  text, which the owned-fence scanner deliberately ignores) → existing hard errors.
- Valid `%if::`/`%proc::` + fence and scanner diagnostics (missing/unclosed/empty fence,
  unknown language) → unchanged.

## Changes

### 1. sase repo — Python extractor (fixes the `0ik` failure path)

`src/sase/xprompt/_directive_collect.py`, `_collect_code_directive`:

- Compute the form flags (`match.group(2)`/`group(3)`/`group(4)`) before the current
  unconditional `if name == "if": raise`.
- When the match is bare (all three groups `None`) and
  `not prompt.startswith("::", match.end())`: return without raising and without
  appending to `collected.regions_to_remove`, for both `if` and `proc`. Keep the
  `typed_launch_units_enabled()` gate check _after_ this prose short-circuit so a prose
  mention no longer trips the flag-off rejection either (see next item for the earlier
  rejection point).
- All non-bare forms and `::`-adjacent bare forms fall through to the existing
  error/parse logic unchanged.

`src/sase/xprompt/code_value.py`, `_mentions_code_directive` / `_is_directive_token`:
the flag-off path (`reject_disabled_code_directives`, called from
`extract_prompt_directives` and `src/sase/core/agent_launch_facade.py`) currently raises
`TYPED_LAUNCH_UNITS_DISABLED_MESSAGE` for _any_ `%if`/`%proc` token outside fences, so
prose would still hard-fail on machines without the flag. Narrow `_is_directive_token`
so a token counts as a use only when followed by `(`, `:`, or `+` (`::` is covered by
`:`; end-of-string and every other boundary character are prose). Existing flag-off
rejections for real forms (`%if::`+fence, `%proc("...")`) must keep raising.

### 2. sase-core repo — Rust typed-launch planner (same false positive)

Open the repo with `sase repo open sase-core` and use only the printed path.

`crates/sase_core/src/agent_launch/mod.rs`:

- `DirectiveOccurrence` (near line 688) does not record which syntactic form matched;
  bare cannot be reliably inferred from `args` (e.g. `%if()` also yields one empty arg).
  Add an explicit form flag (e.g. `is_bare: bool`) set in `directive_occurrences` (near
  line 3470) from the capture groups: true only when neither the paren, colon-arg, nor
  `+` group matched. This struct is internal to the module — no wire/schema change.
- In the second directive loop of `plan_typed_launch_units` (near line 1133): in the
  `"if"` arm, when the occurrence `is_bare` and the prompt at `directive.end` does not
  start with `::`, `continue` without pushing the `invalid-if-form` diagnostic and
  without adding to `regions_to_remove`. Apply the same skip in the `"proc"` arm before
  `parse_proc_directive` (which would otherwise emit the body-required diagnostic for
  prose). `%proc::` fence headers are followed by `::`, so the existing
  owned-span/options handling is unaffected.

### 3. Tests

Python (sase repo) — extend `tests/test_typed_launch_units_code_contract.py` or add a
sibling focused test module:

- Flag ON: `extract_prompt_directives("stop un-admitted %if/%proc units")` (the literal
  string that killed `0ik--review02`) succeeds, extracts no directives, and the cleaned
  prompt still contains `%if/%proc`.
- Flag ON: bare `%if` / bare `%proc` at line start in prose succeed and are preserved.
- Flag OFF: the same prose strings do not raise `TYPED_LAUNCH_UNITS_DISABLED_MESSAGE`.
- Flag OFF: `%if::`+fence and `%proc("cmd")` still raise the disabled message (existing
  tests `test_flag_off_rejects_*` must keep passing).
- Flag ON: `%if:cond`, `%if(true)`, `%if+`, and `%if:: echo hi` (trailing text) still
  raise the fence-form error; `%proc:: echo hi` still raises the body-required error;
  `%if::` with no fence still raises via scan diagnostics (existing
  `test_unknown_language_and_missing_fence_are_hard_errors`).

Rust (sase-core repo) — in the `agent_launch` module tests: a `plan_typed_launch_units`
prompt containing prose `... %if/%proc units ...` produces no `invalid-if-form` or
proc-body diagnostics and leaves the token in the unit prompt text; argument-form
`%if(true)` still produces `invalid-if-form`.

### 4. Landing order and consistency

The Python change alone unblocks the workflow-executor path that killed `0ik`; the Rust
change aligns the typed-launch planner so plan-file launch admission does not reject the
same prose. Land sase-core first, rebuild the binding in the sase checkout
(`just install`, which drives `rust-install` against the linked checkout), then land the
sase change. No wire schema or binding API changes are expected, so the `sase-core-rs`
version floor in `pyproject.toml` / `sase-core-revision.txt` should not need a bump;
verify with `tools/validate_test_environment` via `just check` rather than bumping
preemptively.

## Verification

- sase-core: run that repo's own check entry point (see its `Justfile` / `scripts`)
  including the new Rust tests.
- sase: `just check` (two-speed default gate) with the new Python tests; the implementer
  must read `sase/memory/lint_and_test.md` before finishing, per core memory, and
  `sase/memory/xprompts.md` before touching directive parsing.
- Regression check: feed the exact `0ik--review02` failing prompt fragment
  (`stop un-admitted %if/%proc units`) through `extract_prompt_directives` under the
  enabled `typed_launch_units` flag and confirm no `DirectiveError`.

## Out of scope (operational follow-ups, not repo changes)

- Restarting the dead `0ik` review chain: after this fix lands, the chain can be resumed
  by re-piping the saved prompt `review-02.txt` from the coordinator's temp state
  directory (`/tmp/sase-notification-bead-review-20260911/`).
- The coordinator's temporary `max_agent_pipe_chain: 32` block appended to
  `sase/sase.yml` in the chain's workspace was supposed to be removed by the final chain
  agent, which never ran; that leftover uncommitted state needs manual cleanup when the
  chain is resumed or abandoned.
- Surfacing a notification when a piped successor's workflow fails at prompt preparation
  (the chain died silently); worth a separate task bead if desired.
