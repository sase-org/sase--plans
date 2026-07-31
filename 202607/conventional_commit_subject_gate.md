---
tier: epic
title: Reject non-conventional commit subjects in sase commit
goal: '`sase commit` refuses to create a commit, proposal, or pull request whose subject
  line is not a Conventional Commit, printing an actionable error that tells the agent
  exactly how to rewrite it, while staying lenient enough that legitimate subjects,
  merge/revert/fixup subjects, and projects that opt out are never blocked.

  '
phases:
- id: core_grammar
  title: Conventional-subject grammar in the Rust core
  depends_on: []
  size: medium
  description: 'core_grammar: add a pure `commit_subject` domain module to sase-core
    that parses a commit subject into type/scope/breaking/description, classifies
    exemptions, and reports a stable violation code; expose it through the `sase_core_py`
    binding.

    '
- id: python_surface
  title: Python facade, policy config, and shared header helper
  depends_on:
  - core_grammar
  size: small
  description: 'python_surface: add the typed Python facade over the new binding,
    the `commit.message` configuration block with its schema and defaults, and retarget
    the existing init-memory conventional-header check at the shared helper.

    '
- id: commit_enforcement
  title: Reject invalid subjects in CommitWorkflow
  depends_on:
  - python_surface
  size: medium
  description: 'commit_enforcement: gate `CommitWorkflow.run()` on the subject policy
    before any side effect runs, render the actionable rejection message, record the
    telemetry reason, and update the existing tests that commit with non-conventional
    messages.

    '
- id: docs_and_guidance
  title: Document the gate and guard against type-list drift
  depends_on:
  - commit_enforcement
  size: small
  description: 'docs_and_guidance: document the new configuration and the rejection
    behavior, mention the gate in the git commit skill, and add a regression test
    that keeps the repo''s PR-title CI type list compatible with the validator defaults.

    '
create_time: 2026-07-31 07:14:04
status: wip
bead_id: sase-bj
---

- **BEAD:** [sase-bj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bj/README.md)

# Plan: Reject non-conventional commit subjects in `sase commit`

## Problem

Commit `e0d2476f1` landed on `master` with the subject:

```
Update built-in model aliases for Claude and Codex catalog
```

It has no Conventional Commits header, so release-please cannot classify it: no changelog entry, no version bump
contribution. Nothing in the toolchain stopped it.

Today the format is only _advised_, never _enforced_ at commit time:

- `src/sase/xprompts/skills/sase_git_commit.md` step 2 tells agents to pick a conventional tag.
- `.github/workflows/pr-title.yml` validates the **PR title** with
  `^(feat|fix|perf|docs|ci|test|chore|refactor|build|deps)(\([A-Za-z0-9._/-]+\))?!?: [^[:space:]].*`.
- `src/sase/main/init_memory/commit_message.py` has a private `_CONVENTIONAL_HEADER_RE` used only to decide whether a
  memory-fold subject needs a default prefix.

`create_commit` never squash-merges through a PR title, so the CI check does not cover it at all. Direct-to-`master`
commits — which is how most SASE agent work lands in this repo — are completely ungated. That is the gap that let
`e0d2476f1` through.

## Approach

Add one enforcement point where every agent-authored commit message already passes through: `CommitWorkflow.run()`. The
subject line is validated against a Conventional Commits grammar **before any side effect runs**; a violation fails the
workflow with an actionable message, and the `-M` message file is preserved (existing behavior in
`handle_commit_command`) so the agent can fix the subject and re-run the identical command.

The grammar itself goes in the Rust core. `crates/sase_core/src/commit_footer.rs` already owns the _footer_ half of the
commit-message grammar; putting the _header_ half in Python would split one grammar across two languages and two repos.
It is pure, I/O-free domain behavior — the textbook case for the boundary rule in `CLAUDE.md`.

### Design decisions

Each of these is a judgment call the reviewer may want to overturn; the rationale is stated so that is cheap to do.

1. **Enforced by default, opt-out per project.** New config `commit.message.require_conventional_subject`, default
   `true`. `sase commit` runs in every repo SASE manages, and the `sase_git_commit` skill already instructs agents to
   use conventional tags in all of them, so defaulting on matches the instructions agents already receive. A project
   that genuinely does not use Conventional Commits sets it to `false`.

2. **No per-invocation bypass flag and no bypass environment variable.** A `--no-verify`-style escape hatch would be
   found and used by the exact agents this gate exists to correct. The remedy for a rejection is a one-line rewrite, not
   a bypass. The rejection message explicitly tells the agent not to flip the config off to get past the gate.
   (Deliberate: if the reviewer wants a human escape hatch, the cheapest addition is a `--no-verify-message` flag on
   `sase commit`, and this is the place to say so.)

3. **Lenient everywhere it does not cost correctness.** Specifically:
   - The scope is optional and may contain anything except parentheses and newlines.
   - `!` before the colon is optional.
   - One _or more_ spaces after the colon are accepted, not exactly one.
   - No subject-length limit is enforced, and no capitalization rule is applied to the description. The repo's own
     history contains both `fix(artifact): protect artifacts...` and `chore: Add SDD prompt...`; both stay valid.
   - Leading and trailing whitespace on the message is stripped before the subject is read.
   - Nothing below the subject line is inspected: no body rules, no blank-line rule, no `BREAKING CHANGE:` footer
     requirement.

4. **The type must be lowercase, and that gets its own error.** Conventional Commits calls types case-insensitive, but
   release-please's changelog and version-bump classification is not reliably case-insensitive across versions — so
   accepting `Fix:` risks silently producing exactly the missed-changelog-entry failure this epic exists to prevent.
   `Fix: ...` is therefore rejected, but with a distinct, self-correcting message
   (`type must be lowercase: use "fix:" not "Fix:"`) rather than the generic one.

5. **Type allowlist, configurable per project.** An unknown type such as `feet:` is structurally valid but is silently
   ignored by release tooling, so an allowlist is worth keeping. The default set is the union of the tags documented in
   the `sase_git_commit` skill and the types `.github/workflows/pr-title.yml` accepts:

   ```
   build, chore, ci, deps, docs, feat, fix, perf, refactor, revert, style, test
   ```

   `deps` is non-standard and is included solely because this repo's PR-title CI already accepts it. Projects may
   replace the set with `commit.message.allowed_types`; a configured list replaces the default rather than extending it.

6. **Exempt prefixes are skipped entirely** — subjects beginning with `Merge `, `Revert "`, `fixup!`, `squash!`, or
   `amend!`. These are mechanical git-generated or rebase-directive subjects; rejecting them would block legitimate
   recovery flows for no benefit.

7. **Applies to all three commit methods.** `create_commit`, `create_proposal`, and `create_pull_request` all produce a
   real commit whose subject becomes a changelog input or a squash-merge PR title.

8. **Blank messages are rejected too.** `handle_commit_command` always sets `payload["message"]`, so the existing
   `"message" not in payload` check in `CommitWorkflow.run()` never catches an empty string; today an empty message
   reaches the provider and fails with a raw git error. The new gate rejects it with a clear reason instead. This
   applies to `create_pull_request` as well: an empty PR title is never correct.

### What already goes through this path (verified)

- The only in-tree caller that shells out to `sase commit` is `src/sase/axe/run_agent_exec_plan_sdd.py`, and it already
  builds `chore: Add SDD prompt and plan for <name>` — conventional, unaffected.
- The only in-process constructor of `CommitWorkflow` is `src/sase/main/commit_handler.py`.
- Every other SASE-authored auto-commit (`bare_git_init.py`'s `Initial commit`, `_commit_bare_git.py`'s
  `Initialize SDD`, `init_memory/project_deploy.py`, `init_workspace_handler.py`, `bead/_sync_git.py`,
  `llm_provider/commit_finalizer_git.py`) calls `git` directly and bypasses `CommitWorkflow` entirely. Those are **out
  of scope** — see Non-goals.

## Non-goals

- Changing the SASE-authored auto-commit subjects that bypass `CommitWorkflow` (`Initial commit`, `Initialize SDD`, and
  the workspace/bead-sync messages). Several are not conventional. Bringing them in line is a separate, larger change;
  file a task bead for it rather than widening this epic.
- Making `.github/workflows/pr-title.yml` call `sase` to single-source the type list. That job is deliberately a
  checkout-free bash step; installing `sase` into it would cost far more than the drift it prevents. `docs_and_guidance`
  adds a cheap in-repo drift test instead.
- Enforcing subject length, description capitalization, body structure, or `BREAKING CHANGE:` footers.
- Adding a new CLI subcommand or new `sase commit` flags. This epic adds no CLI surface, so `sase/memory/cli_rules.md`
  does not apply.
- Fixing the interaction where `vcs_provider.use_project_pr_prefix: true` prepends `[project] ` to a PR title _after_
  validation, producing a final title that the PR-title CI would reject. That option is `false` in this repo and the
  prefix behavior predates this epic; `docs_and_guidance` documents the interaction rather than changing it.

---

## Phase `core_grammar`: Conventional-subject grammar in the Rust core

Work happens in the **sase-core** repository. Open it with `sase repo open sase-core -r "<reason>"` and use the printed
path as the only path for reads and writes. Do not edit `[workspace.package].version` or crate versions — release-plz
owns those (see sase-core's `AGENTS.md`).

### New module `crates/sase_core/src/commit_subject.rs`

Pure domain code: no filesystem, no git, no host access. Model it on the existing `commit_footer.rs` — same wire-struct
conventions, same `schema_version` field, same `serde` derives.

Public surface:

```rust
pub const COMMIT_SUBJECT_WIRE_SCHEMA_VERSION: u32 = 1;

/// Default Conventional Commit types accepted when a project configures none.
pub fn default_commit_subject_types() -> Vec<String>;

pub struct CommitSubjectWire {
    pub schema_version: u32,
    /// The first non-empty-trimmed line of the message, as validated.
    pub subject: String,
    pub valid: bool,
    /// True when an exempt prefix skipped validation (`valid` is then also true).
    pub exempt: bool,
    pub commit_type: Option<String>,
    pub scope: Option<String>,
    pub breaking: bool,
    pub description: Option<String>,
    /// Stable machine code, `None` when `valid`.
    pub violation: Option<String>,
    /// The offending type as written, when `violation` concerns the type.
    pub found_type: Option<String>,
}

pub fn parse_commit_subject(message: &str, allowed_types: &[String]) -> CommitSubjectWire;
```

`default_commit_subject_types()` returns, in this order:
`build, chore, ci, deps, docs, feat, fix, perf, refactor, revert, style, test`.

### Parsing rules

1. Trim the whole message; take everything up to the first `\n` as the subject; trim the subject.
2. If the subject is empty → `violation = "empty_subject"`, `valid = false`.
3. If the subject starts with any of `Merge `, `Revert "`, `fixup!`, `squash!`, `amend!` → `exempt = true`,
   `valid = true`, `violation = None`. Match these prefixes case-sensitively; they are produced verbatim by git.
4. Otherwise split on the first `:`. If there is no `:`, or the token before it is empty →
   `violation = "missing_type_separator"`.
5. The token before the `:` must match `^([A-Za-z]+)(\(([^()\n]*)\))?(!)?$`. If it does not (stray characters, empty or
   nested parentheses, text after `!`) → `violation = "missing_type_separator"`. An empty scope — `fix(): x` — is a
   violation; require at least one non-whitespace character inside the parentheses.
6. If the captured type contains any uppercase character → `violation = "uppercase_type"`,
   `found_type = Some(<as written>)`. Report this in preference to `unknown_type`, and report it even when the
   lowercased form is not in `allowed_types` — the lowercase fix is the first thing the author needs to know.
7. If the type is not in `allowed_types` → `violation = "unknown_type"`, `found_type = Some(<type>)`.
8. Everything after the `:` must contain at least one non-whitespace character after stripping the one-or-more
   separating spaces. If not → `violation = "empty_description"`. `description` is the remainder with leading and
   trailing whitespace trimmed.
9. On success populate `commit_type`, `scope` (trimmed, `None` when absent), `breaking` (the `!`), and `description`;
   set `valid = true`, `violation = None`.

Implement without a regex crate dependency unless one is already in the workspace — plain `str` scanning is clearer here
and avoids adding a dependency to a leaf domain module. Check `Cargo.toml` before deciding.

### Rust unit tests (in-module `#[cfg(test)]`, matching the file's existing convention)

Cover at minimum: plain `fix: x`; scoped `feat(bead): x`; breaking `feat!: x` and `feat(cli)!: x`; multi-space
`fix:   x`; multi-line message where only the first line is read; leading/trailing whitespace; each exempt prefix; the
real failing subject `Update built-in model aliases for Claude and Codex catalog` → `missing_type_separator`; `Fix: x` →
`uppercase_type`; `feet: x` → `unknown_type`; `Feet: x` → `uppercase_type` (rule 6 precedence); `fix:` and `fix:   ` →
`empty_description`; `fix(): x` → `missing_type_separator`; empty and whitespace-only messages → `empty_subject`; a
custom `allowed_types` list that excludes a default type and confirms it is rejected.

### Binding in `crates/sase_core_py/src/lib.rs`

Follow the `py_parse_commit_footer` pattern exactly: a `#[pyo3(name = "parse_commit_subject")]` function taking the
message and the allowed-type list and returning the wire struct as a Python dict, plus
`#[pyo3(name = "commit_subject_wire_schema_version")]` and `#[pyo3(name = "default_commit_subject_types")]`. Register
all three with `wrap_pyfunction!` in the module init, add the `pub use` re-export in `crates/sase_core/src/lib.rs`
alongside the `commit_footer` re-exports, and add a binding round-trip test mirroring the existing `commit_footer`
binding test.

### Verification

From the sase-core checkout: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and
`cargo test --workspace` — the three commands sase-core CI runs.

### Landing

Commit and push to sase-core `master` before the dependent phases start. Downstream `sase` CI builds the core wheel from
sase-core `master` HEAD (`.github/workflows/ci.yml`, `build-core` job), and local `just install` builds `sase_core_rs`
from the linked checkout, so **no `sase-core-rs` version bump in `pyproject.toml` is needed for the later phases to
work**. Leave the `sase-core-rs>=0.17.0,<0.18.0` window alone; release-plz will publish the new version on its own
cadence.

---

## Phase `python_surface`: Python facade, policy config, and shared header helper

Work happens in the **sase** repository. Run `just install` first — this workspace may be stale, and it also rebuilds
`sase_core_rs` from the linked sase-core checkout so the `core_grammar` binding is available.

### `src/sase/core/commit_subject_facade.py`

Typed facade over the binding, modeled on `src/sase/core/commit_footer_facade.py` (which uses
`sase.core.rust.require_rust_binding`). Expose:

```python
COMMIT_SUBJECT_WIRE_SCHEMA_VERSION = 1

@dataclass(frozen=True)
class CommitSubject:
    subject: str
    valid: bool
    exempt: bool
    commit_type: str | None
    scope: str | None
    breaking: bool
    description: str | None
    violation: str | None      # "empty_subject" | "missing_type_separator"
                               # | "uppercase_type" | "unknown_type" | "empty_description"
    found_type: str | None

def default_commit_subject_types() -> tuple[str, ...]: ...
def parse_commit_subject(message: str, allowed_types: Sequence[str] | None = None) -> CommitSubject: ...
```

`allowed_types=None` means "use the core defaults". Keep all user-facing wording out of this module — it is a contract
facade, not a presenter.

### `src/sase/workflows/commit/message_validation.py` (new)

Two responsibilities, no side effects:

- `load_commit_message_policy()` → a frozen dataclass
  `CommitMessagePolicy(require_conventional_subject: bool, allowed_types: tuple[str, ...])`. Read it from
  `load_merged_config()` exactly the way `src/sase/llm_provider/commit_finalizer_config.py` reads `commit.finalizer`:
  tolerate a missing or malformed `commit` / `commit.message` mapping by falling back to defaults, never raise. A
  non-list or empty `allowed_types` falls back to the core defaults; entries are lowercased and de-duplicated while
  preserving order.
- `check_commit_message(message, policy)` → `str | None`. Returns `None` when the message is acceptable (including when
  the policy is disabled, and including exempt subjects); otherwise the fully rendered, multi-line rejection text. This
  is where the user-facing wording lives.

Rendering, per violation code — the generic form:

```
Commit message rejected: the subject line is not a Conventional Commit.

  subject: Update built-in model aliases for Claude and Codex catalog

Expected: <type>[(<scope>)][!]: <description>
Allowed types: build, chore, ci, deps, docs, feat, fix, perf, refactor, revert, style, test
Examples:
  fix(commit): reject non-conventional subjects
  feat!: drop the legacy config format

Rewrite the subject line and re-run the same command. Do not disable
commit.message.require_conventional_subject to get past this check.
```

Specialize the first line per code so the fix is unambiguous:

- `empty_subject` → `Commit message rejected: the message is empty.` (omit the `subject:` line)
- `uppercase_type` → `Commit message rejected: the commit type must be lowercase — use "fix:" not "Fix:".` substituting
  the actual `found_type` and its lowercased form.
- `unknown_type` → `Commit message rejected: "feet" is not an allowed commit type.` substituting `found_type`.
- `empty_description` → `Commit message rejected: the subject has no description after the type.`
- `missing_type_separator` → the generic first line above.

The `Allowed types:` line always renders the policy's effective list, sorted, so a project with a custom list gets
accurate guidance.

### Configuration

`src/sase/default_config.yml`, extending the existing `commit:` block:

```yaml
commit:
  finalizer:
    enabled: true
    max_passes: 2
  message:
    # Reject `sase commit` messages whose subject line is not a Conventional
    # Commit (`type(scope)!: description`). Merge, revert, and fixup subjects
    # are always exempt.
    require_conventional_subject: true
    # Commit types this project accepts. A configured list replaces the
    # built-in set rather than extending it.
    allowed_types: [build, chore, ci, deps, docs, feat, fix, perf, refactor, revert, style, test]
```

`src/sase/config/sase.schema.json`: add `message` under `commit.properties` — the `commit` object has
`additionalProperties: false`, so an unlisted key breaks editor LSP validation. Give `require_conventional_subject` type
`boolean` with `default: true`, and `allowed_types` type `array` of `string` with the default list and `minItems: 1`.
This repo's `gotchas` mentor profile flags a missed schema update as its single most common defect — do not skip it.

Do **not** add anything to `sase/sase.yml`; the defaults are what this repo wants.

### Retarget the existing duplicate

`src/sase/main/init_memory/commit_message.py` carries its own `_ALLOWED_TAGS` and `_CONVENTIONAL_HEADER_RE`. Replace
`_is_conventional_header` with a call to the facade (`parse_commit_subject(subject).valid`) and delete the local tag
tuple and regex. Behavior must not change: `_compose_fold_subject` still prefixes `MEMORY_FOLD_DEFAULT_PREFIX` only when
the subject is not already conventional. Note the one intentional widening — the local list omitted `deps`, the shared
default includes it — and confirm `tests/main/test_init_memory_commit_message.py` still passes.

Symvision may flag the new public helpers until `commit_enforcement` consumes them; if it does, read
`sase/memory/symvision.md` through `/sase_memory_read` before adding any pragma.

### Tests

- `tests/core/` (match the directory convention used by the other facade tests): facade round-trip — valid, each
  violation code, exempt prefixes, custom `allowed_types`.
- Policy loading: default when `commit` is absent, when `commit.message` is absent, when the block is a non-mapping,
  when `allowed_types` is a non-list or empty; and honoring an explicit `false` and an explicit custom list.
- `check_commit_message` returns `None` when disabled, `None` for exempt subjects, and text containing the offending
  subject plus the effective allowed-type list for each violation code.

### Verification

`just install` then `just check`.

---

## Phase `commit_enforcement`: Reject invalid subjects in CommitWorkflow

Run `just install` before anything else.

### The gate

In `src/sase/workflows/commit/workflow.py`, inside `CommitWorkflow.run()`, add the check **immediately after the
existing payload-shape validation** (the `create_pull_request`-name check, currently around line 111) and **before
`cwd = os.getcwd()`**. Placement is the substance of this phase: everything after that point mutates state that a later
failure cannot undo —

- `apply_bead_commit_tag` rewrites the payload,
- `handle_beads` closes beads,
- `handle_sase_plan` stages plan files,
- `run_before_commit_hook` runs `just fix`.

A subject rejected after any of those would leave beads closed for a commit that never happened.

```python
policy = load_commit_message_policy()
rejection = check_commit_message(str(self._payload.get("message") or ""), policy)
if rejection is not None:
    print_status(rejection, "error")
    _log_commit_failed(self._method, "invalid_message")
    return RunResult.FAILED
```

Apply it to all three methods. `--resume` must **not** re-validate: the resume path replays post-dispatch bookkeeping
for a commit that already exists, and re-running the gate there could strand a checkpoint. `CommitWorkflow.resume()`
delegates to `resume_commit_workflow` and does not call `run()`, so no change is needed — confirm this rather than
assume it.

`handle_commit_command` already preserves the `-M` file and prints the "re-run with the same `-M` flag" hint on failure,
so no change is needed there. Verify it, and verify the rejection text and the preservation hint read sensibly together
in the actual terminal output.

`invalid_message` is a new value for the existing free-form `reason` field on the `commit_failed` run-log event; there
is no enumeration to extend (`before_hook_failed` and friends are plain strings).

### Existing tests that will break

Several tests drive `CommitWorkflow` with non-conventional messages and will now fail. Known cases — sweep for more
rather than trusting this list:

- `tests/test_file_hook_engine.py` — `{"message": "done"}`.
- `tests/workflows/test_commit_workflow.py`, `tests/test_commit_runtime_tags.py`, `tests/test_pr_tags.py` — check each
  payload.

Fix them by making the fixture messages conventional (`chore: done`), not by disabling the policy — a conventional
fixture is just as valid a test input and keeps the suite honest. `tests/test_commit_workflow_hooks.py` already uses
`fix: bug` and should need no change.

### New tests

- A non-conventional message fails with `RunResult.FAILED`, and — the important assertion — `handle_beads`,
  `handle_sase_plan`, `run_before_commit_hook`, and the provider dispatch are **never called**. Patch them and assert
  zero calls; ordering is the whole point of this phase.
- The printed error contains the offending subject and the expected format.
- `commit_failed` is logged with `reason="invalid_message"`.
- A conventional message still reaches dispatch unchanged, for each of `create_commit`, `create_proposal`, and
  `create_pull_request`.
- `require_conventional_subject: false` lets a non-conventional message through to dispatch.
- An exempt subject (`Merge branch 'x'`) reaches dispatch.
- An empty message is rejected for all three methods.

### Verification

`just install` then `just check`. Then exercise it for real in a scratch git repo — the gate's whole value is the
terminal output an agent sees, so read it once with your own eyes:

```bash
sase commit -m "Update built-in model aliases for Claude and Codex catalog"   # expect rejection, exit 1
sase commit -m "chore: update built-in model aliases"                          # expect normal behavior
```

---

## Phase `docs_and_guidance`: Document the gate and guard against type-list drift

Run `just install` before anything else.

### Documentation

- `docs/configuration.md`, the `### commit` section (around line 1077): add a `commit.message` subsection with the YAML
  block and a settings table row for each of `commit.message.require_conventional_subject` and
  `commit.message.allowed_types`, matching the existing `commit.finalizer` table's format. State that merge, revert, and
  fixup subjects are exempt, and that a configured `allowed_types` replaces the default set. Update the `Source:` line
  to name the new modules. Add the anchor to the table of contents at the top of the file if that section's entries are
  listed there.
- `docs/commit_workflows.md`: document the gate in the `## How It Works` → `### 3. CommitWorkflow orchestrates` flow,
  placing it explicitly _before_ bead handling and the before-hook, and add a line to the ordered step list. Note the
  `use_project_pr_prefix` interaction from the Non-goals section: the subject is validated as the agent authored it, and
  a `[project] ` PR-title prefix is applied afterward.
- `src/sase/xprompts/skills/sase_git_commit.md`: in step 2, note that `sase commit` now **rejects** a non-conventional
  subject, so the tag choice is mandatory rather than advisory, and that the fix is to rewrite the message file and
  re-run the same command. Keep it to a sentence or two — the step already explains the tags well. Because this is a
  generated-skill source, follow `sase/memory/generated_skills.md` (read it through `/sase_memory_read`) for the
  regeneration and deployment steps.

Do **not** edit `sase/memory/*.md`, `AGENTS.md`, or the generated `CLAUDE.md` / `GEMINI.md` / `OPENCODE.md` / `QWEN.md`
shims. That requires explicit user permission in the conversation, and this plan does not grant it. If the gate seems
worth a memory entry, file a task bead proposing it.

### Drift guard

`.github/workflows/pr-title.yml` hardcodes `feat|fix|perf|docs|ci|test|chore|refactor|build|deps`. If the validator's
default set and that list diverge, a subject can pass `sase commit` and then fail the PR-title job (or vice versa). Add
a test that parses the `allowed_types=` line out of that workflow file and asserts every type in it is present in
`default_commit_subject_types()`. Subset, not equality — the defaults intentionally also allow `style` and `revert`,
which the CI list omits. Make the failure message say which type drifted and which of the two files to update.

### Verification

`just install` then `just check`. Confirm `docs/` still builds if the repo has a docs build target, and confirm the
regenerated skill file matches its source.
