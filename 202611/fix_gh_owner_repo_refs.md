---
tier: tale
title: Fix
goal: Routine and human prompts starting with
size: small
decisions:
  stale_rule_text:
    ask:
      Also fix the stale `word`/`agent` rule wording in the sase docs table and
      sase-core catalog?
    choices:
      fix_now:
        Fix both now; the tale also edits a sase docs table and two sase-core strings
      defer: Change only sase-github; file one task bead for the stale wording
    default: fix_now
    why: Stale "no whitespace" wording is how plugin macros kept slash refs on `word`
    answer: fix_now
proposed_by: bbugyi200.athena.1ca
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-13 17:53:06
status: wip
---

# Fix `#gh:<owner>/<repo>` refs rejected by the hardened `word` input type

## Symptom

Every agent launched by the `refresh_docs` routine fails before it starts, for example
`chop.refresh_docs.bob-cli.<run>.1` and `chop.refresh_docs.sase.<run>.1`:

```
Step 'main' failed: WorkflowExecutionError: Workflow '#gh' argument error:
Argument 'gh_ref' expects word (no spaces) matching [A-Za-z0-9._-]+, got 'bobs-org/bob-cli'
```

Each run's `.2` polish agent waits on the failed `.1` agent, so the TUI shows it as
`WAITING ✗1` indefinitely. The same error hits any prompt that starts with
`#gh:<owner>/<repo>`, including the `#gh:bbugyi200/cash` launches.

## Root cause

1. The `projects` `for_each` target source builds `target.workspace` as
   `gh:<owner>/<repo>` (sase `src/sase/axe/_config_targets.py`,
   `_project_workspace_ref`). `refresh_docs` proposes its launches with that workspace,
   and `scaffolded_prompt` (`src/sase/axe/chop_proposal_models.py`) prefixes the prompt
   with `#gh:<owner>/<repo>`.
2. The `#gh` macro ships in the **sase-github** plugin
   (`src/sase_github/xprompts/gh.yml`) and declares `gh_ref` as `type: word`.
3. sase commit `22e1ef1249` (phase sase-1sb.4, shell-argument hardening) narrowed `word`
   from "no whitespace" to `[A-Za-z0-9._-]+` (`_WORD_RE` in `src/sase/macro/models.py`)
   and deliberately rejects `/` and `:`. That commit moved sase's own sibling macro
   `#git` (`git_ref`) from `word` to `line`. The upgrade guide
   (`docs/reference/upgrading/macros.md`) also says "Use `line` for branch references
   containing `/`". The phase audited only macros shipped inside sase, so the
   plugin-shipped `#gh` macro kept `word`.

Evidence:

- Workflow logs show `#gh:<owner>/<repo>` prompts succeeding through Nov 11, and every
  one failing from Nov 13 onward, after the hardening commit was installed.
- Interactive launches still work because the TUI writes the project-name form
  `#gh:gh_<owner>__<repo>`, which is a valid `word`.
- Binding `bobs-org/bob-cli` or `ghe.example.test/acme/widget` against the current
  `gh.yml` raises this exact error. With `gh_ref: line`, both bind, and so do
  `gh_sase-org__sase` and a plain branch or Patch name.

`owner/repo` is a supported `#gh` form:

- sase-github's README says "`#gh:` remains the spelling for `owner/repo` paths".
- `resolve_gh_ref`'s repo-path mode depends on it.
- sase's prompt completion offers `#gh:<owner>/<repo>`
  (`src/sase/macro/vcs_repo_completion.py`).

So the defect is in the macro's input type, not in the routine.

### Rejected alternatives

- **Have `_project_workspace_ref` emit `gh:gh_<owner>__<repo>` instead.** This fixes
  only one caller. Human-typed and completion-inserted `#gh:owner/repo` prompts stay
  broken.
- **Loosen `word` again.** This reverses the reviewed `bash_autoquote` hardening
  decision for every macro.

## Changes

### 1. sase-github: `gh_ref` becomes `line` (the fix)

Open the linked repo with `/sase_repo` (`sase repo open sase-github`) and work in the
path it prints.

- In `src/sase_github/xprompts/gh.yml`, change the `gh_ref` input from `type: word` to
  `type: line`, which mirrors sase's `#git` macro. Leave `workflow_label` as `word`.
- No template edits are needed. `gh_ref` reaches only `python:` steps, always through
  `{{ gh_ref | tojson }}`, and no `bash:` step interpolates it. Confirm this still holds
  when you make the edit.

### 2. sase-github: validate repo-path components where `#gh` turns them into paths

Under `word`, a value containing `/` never reached `resolve_gh_ref`'s repo-path mode.
With `line`, values like `../x` or `owner/re po` reach it again. Mode 1 joins owner and
repo into `~/projects/github/<owner>/<repo>/` (`github_workspace_dir`) without checking
them. Keep the hardening's intent at the point where the value becomes a path:

- In `src/sase_github/workspace/refs.py`, make `_repo_path_coords` return `None` unless
  `owner` and `repo` each fullmatch `[A-Za-z0-9._-]+` and are not `.` or `..`. This is
  GitHub's own owner/repo name charset. Define a module-level regex locally; do not
  import new sase helpers, because the plugin's dependency floor is `sase>=0.18.0`.
- `resolve_gh_ref` already raises `ValueError` when `_repo_path_coords` returns `None`.
  Extend that message to name the allowed characters, but keep the existing
  `expected 'user/project' or 'host/user/project'` text so
  `tests/test_workspace_plugin.py::...::test_invalid_repo_path` still matches.
- `peek_gh_ref` already returns `None` on invalid coordinates. Keep that behavior.
- Host handling is unchanged: the 3-part form still requires a configured GitHub host.

### 3. sase-github: regression tests

sase-github CI checks out sase `master`, so these tests run against the strict
validator.

- Add a test that loads the plugin's `src/sase_github/xprompts/gh.yml` with
  `sase.macro.workflow_loader_definition.load_workflow_from_file` and binds it with
  `sase.macro.input_binding.bind_input_args(workflow.inputs, [ref], {})`. The test must
  bind successfully, with the bound `gh_ref` equal to the input, for:
  - `bobs-org/bob-cli`
  - `ghe.example.test/acme/widget`
  - `gh_sase-org__sase`
  - a plain Patch/branch name such as `my-feature`

  Name it after the behavior (owner/repo refs bind), not the type. It fails on today's
  `gh.yml`.

- Add `resolve_gh_ref` / `peek_gh_ref` tests showing that `../widget`, `acme/..`,
  `acme/re po`, and `acme/$(id)` fail closed. `resolve_gh_ref` should raise `ValueError`
  and `peek_gh_ref` should return `None`, with no clone attempted (assert that
  `clone_gh_repo` is not called). The existing owner/repo and host-qualified tests must
  stay green.
- Run `sase tool run check` from inside the sase-github checkout.

### 4. Stale rule wording

> [!decision] stale_rule_text = fix_now
>
> Correct both published descriptions of the `word`/`agent` rule:
>
> - **sase** `docs/reference/workflows.md`, Typed Inputs table: the `word` row says
>   "Single word, no whitespace" and the `agent` row says "Agent name (no whitespace)".
>   Reword them to match `docs/reference/macros/arguments.md`: `word` is non-empty with
>   ASCII letters, digits, `.`, `_`, `-` only, and `line` is for refs containing `/`;
>   `agent` uses the same characters plus an optional leading `@`. This updates an
>   existing Reference page, which the docs routing table allows. Run `just fmt` (the
>   table is width-aligned), then `sase tool run check` in sase.
> - **sase-core** `crates/sase_core/src/macro_input_types/catalog.rs`: the builtin
>   `word` row (`"A single word with no whitespace."`) and the `agent` row
>   (`"A non-empty agent name with no whitespace."`) feed `sase macro types`. Reword
>   them to the same rule. Open the repo with `sase repo open sase-core`. This changes
>   only strings, not the binding API, so sase's `sase-core-revision.txt` pin does not
>   need to move. Grep sase and sase-core for the old strings first; today nothing
>   asserts them. Run `sase tool run check` in the sase-core checkout.

> [!decision] stale_rule_text = defer
>
> Skip step 4's edits. File one task bead through `/sase_new_task` naming both stale
> descriptions and the files above.

## Out of scope (file through `/sase_new_task`, do not fix here)

- Notification-gate form schemas (sase `src/sase/notification_gates/model_inputs.py`)
  still compile `word`/`agent` fields to `^\S+`, while `InputArg.validate_and_convert`
  now enforces `[A-Za-z0-9._-]+`. That module's own comment requires the two layers to
  agree. This is the same incomplete-audit gap, but it is a separate validator change
  with cross-client schema consequences. File it as a bug after checking for a
  duplicate.

## Rollout and recovery

- When the sase-github change lands, its conventional `fix:` commit feeds the plugin's
  release-please patch release. The local editable sase-github install picks the change
  up once its primary checkout is updated, with no sase release needed.
- After that, retry the failed `chop.refresh_docs.*.1` agents (or let the routine's next
  scheduled run replace them). The waiting `.2` polish agents resume once their `.1`
  succeeds.

## Acceptance

- `#gh:bobs-org/bob-cli`, `#gh:ghe.example.test/acme/widget`, `#gh:gh_sase-org__sase`,
  and `#gh:<patch-name>` all bind against the plugin's `gh.yml` under sase master's
  strict `word` rule.
- Repo-path refs with `.`/`..` components or characters outside GitHub's name charset
  are refused before any path join or clone.
- `sase tool run check` passes in every repo this tale touched.
