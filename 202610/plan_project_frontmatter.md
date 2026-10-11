---
tier: epic
title: Record each local plan's SASE project in its frontmatter
goal:
  "`sase plan propose` stamps the owning SASE project into every machine-local plan
  under `~/.sase/plans/`. `sase plan approve <plan-file>` and `sase bead work
  <plan-file>` use that field to pick the right project, so both work from any
  directory. The field never reaches a project's plan store."
decisions:
  project_value:
    ask: How should the stamped `project` value be spelled?
    choices:
      display: Display name such as `sase`; readable, mapped to the key by the alias map
      key: Project key such as `gh_sase-org__sase`; exact, but opaque to humans
    default: display
    why: Matches the `+sase` tag and `sase project list`; readers accept both spellings
    answer: display
  bead_work_project_flag:
    ask: Add a long-only `--project` override to `sase bead work` for plan files?
    default: false
    why: The frontmatter covers new plans; `-P` is already `--no-push`
    answer: false
phases:
  - id: field
    title: Accept, stamp, and strip the plan `project` field
    size: medium
    depends_on: []
    description:
      "field: sase-core accepts an optional `project` frontmatter field. `sase plan
      propose` stamps the planner's project into the local plan. A shared Python helper
      reads and strips the field, and every writer that copies a plan into a project
      plan store strips it."
  - id: approve
    title: "`sase plan approve` resolves the project from frontmatter"
    size: small
    depends_on:
      - field
    description:
      "approve: gateless `sase plan approve` reads the plan's `project` field right
      after `-P`. Its already-done recovery check and dry-run preview use the resolved
      project instead of the working directory."
  - id: bead-work
    title: "`sase bead work <plan-file>` routes by frontmatter project"
    size: medium
    depends_on:
      - field
    description:
      "bead-work: plan-file mode reads the `project` field. When the working directory
      is not inside that project, it runs in a write-capable operation context at that
      project's primary checkout. That context covers the lock, stores, archive links,
      parent routing, and launch."
proposed_by: bbugyi200.athena.0zl
decided_by: reviewer
decided_via: tui
create_time: 2026-10-11 07:37:53
status: wip
---

- **PROMPT:**
  [prompts/202610/plan_project_frontmatter.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/plan_project_frontmatter.md)

# Record each local plan's SASE project in its frontmatter

## Problem

`sase plan propose` moves a plan into the machine-local archive
`~/.sase/plans/<YYYYMM>/<name>.md`. The plan does not record which SASE project it
belongs to, so later commands guess:

- **`sase bead work <plan-file>`** has no project input. It picks its bead store, plan
  store, workspace, VCS workflow, and macros from the working directory
  (`resolve_plan_file_context` in `src/sase/bead/cli_work_from_plan_store.py`,
  `cli_work_context.py`).
  - From another project's checkout, it silently archives the plan into that project's
    plans sidecar and creates the epic in that project's bead store.
  - From `~`, it falls back to a legacy in-tree store and leaves stray files under
    `~/sdd/`. This has already happened: `~/sdd/plans/202610/xprompts_to_macros.md`.
- **Gateless `sase plan approve <plan-file>`** resolves the project in this order: `-P`,
  then gate history, then the planner agent's artifacts, then a workspace marker, then
  the working directory. The order is in `resolve_project` in
  `src/sase/main/plan_direct_approval_project.py`.
  - Once the gate record or the planner is gone, it refuses from `~` ("cannot tell which
    project").
  - From another project's checkout, it silently picks that project.
  - Its already-done recovery check also resolves plan refs against the working
    directory.

## Design

**The field.** `project: <name>` is an optional, SASE-written frontmatter field, in the
same family as `proposed_by` and `bead`.

- `sase plan propose` writes it.
- It exists only in machine-local plan files under `~/.sase/plans/`.
- It is removed whenever a plan is copied into a project's plan store, whether a sidecar
  or in-tree `sdd/plans`. There the store's location already identifies the project.

**Who reads it.**

- Gateless `sase plan approve` reads it right after `-P`.
- `sase bead work` reads it in plan-file mode.

**Plans without the field.** Older plans and scratch files without the field keep
today's behavior exactly.

**Precedence:**

- For `sase plan approve`: explicit `-P`, then frontmatter `project`, then the existing
  chain.
- For `sase bead work`: frontmatter `project`, then the working directory (see the
  `bead_work_project_flag` decision for an explicit flag).

**Where a routed `sase bead work` writes.** When `sase bead work` routes away from the
working directory, it writes exactly as if it had been run from the project's primary
checkout. That is today's manual workaround, and it is the same user-directed routing
`sase bead work` already uses for a cross-project `parent_bead`. It does not use an
operational lease, because the command must stay synchronous in the terminal.

**No feature flag.** The field is inert metadata until something reads it. Each consumer
phase ships a complete behavior. Routing engages only when the field is present, and it
replaces a silent wrong-project fallback; no old behavior is deprecated.

**Cross-repo landing.** Strict plan validation lives in sase-core: unknown frontmatter
keys are errors (`validate_top_level_keys` in `crates/sase_core/src/plan/validate.rs`).
So the core must accept `project` before sase stamps it.

- Phase `field` changes both repos in one turn. Open the linked core with
  `sase repo open sase-core -r "<why>"`.
- One declaration that commits both repos gets `sase-core-revision.txt` bumped
  automatically by the host.
- Do not touch the `sase-core-rs` window in `pyproject.toml`; the release lane owns it
  (see `docs/rust_backend.md`).

## Phase field

### sase-core (linked repo; read its `AGENTS.md` first)

1. In `crates/sase_core/src/plan/validate.rs`:
   - Add `"project"` to `SYSTEM_FIELDS`.
   - Parse it with `optional_non_empty_string(mapping, "project", &index)` for both
     tiers and every validation mode.
   - Add `pub project: Option<String>` to `ValidatedPlanWire`, with
     `#[serde(default, skip_serializing_if = "Option::is_none")]`. This is additive:
     plans without the field serialize exactly as before, and `PLAN_WIRE_SCHEMA_VERSION`
     stays 3.
   - Add a `project` entry to `plan_frontmatter_schema` beside `proposed_by`, typed
     `non-empty string`, not required, with example `"sase"`. Description: "SASE project
     this machine-local proposal belongs to, written by `sase plan propose`; removed
     when the plan is archived into a project plan store."
2. Add `project: None` to every `ValidatedPlanWire` struct literal, for example in
   `plan/decisions/resolver.rs` and the validate tests. Update any `sase_core_py`
   fixture or schema snapshot that lists the frontmatter fields.
3. Tests:
   - Accepted for tale and epic in `authoring`, `launch`, and `archived` modes.
   - Empty and non-string values are errors with `field_path == "project"`.
   - The value round-trips through the wire.
   - A plan without the field serializes with no `project` key.
4. Run `sase tool run check` in sase-core.

### sase

5. In `src/sase/sdd/plan_validate.py`:
   - Add `project: str | None = None` to `_ValidatedPlan`.
   - Rehydrate it in `_validated_plan_from_dict` with
     `_optional_str(payload.get("project"))`.
   - Update any Python schema-field expectations in `tests/test_plan_validate.py`.
6. Add a new module `src/sase/sdd/plan_project.py` with these helpers, each tolerant and
   never raising on bad input:
   - `PLAN_PROJECT_FIELD = "project"`.
   - `read_plan_project(content: str) -> str | None` returns the stripped non-empty
     string. It returns `None` when the field is missing, blank, or not a string, or
     when the frontmatter is malformed. Use `parse_frontmatter` from
     `src/sase/sdd/frontmatter.py`.
   - `read_plan_project_file(path) -> str | None` returns `None` on `OSError` or
     `UnicodeError`.
   - `strip_plan_project(content: str) -> str` uses `remove_frontmatter_fields`. It
     returns the input byte-for-byte unchanged when the field is absent.
   - `plan_project_for_artifacts_dir(artifacts_dir: str) -> str | None` returns the
     value to stamp:
     - Derive the project key with `project_name_from_artifacts_dir` from
       `sase.turns.settlement`.
     - Return `None` for `home`, and for any key that is not a registered, non-system
       project. Use the project records behind `src/sase/project_aliases.py`, the same
       data `sase project list --json` shows.
     - Return the record's display name.

> [!decision] project_value = key `plan_project_for_artifacts_dir` returns the project
> key itself. The docs and test fixtures use key spellings such as `gh_sase-org__sase`
> instead of `sase`.

7. Stamp in `src/sase/main/plan_propose_handler.py`:
   - Add `stamps["project"]` from `plan_project_for_artifacts_dir(artifacts_dir)` next
     to `proposed_by`, for both tiers.
   - It overwrites any authored value. When nothing is derivable, leave the content
     untouched.
   - It goes through the existing `set_frontmatter_fields` call, before prettier.
   - Authoring validation still runs on the unstamped `original`.
8. Strip the field wherever a plan's content is written into a project plan store. The
   local `~/.sase/plans` file always keeps it.
   - `archive_plan_file` (`src/sase/sdd/plan_archive.py`): strip right after reading and
     formatting the source, before `validate_plan_for_commit`. This covers live, TUI,
     and gateless approvals, the `sase bead work` epic archive, and the plan-archive
     doctor.
   - `handle_sase_plan` (`src/sase/workflows/commit/plan_hooks.py`): strip in the
     `should_copy` branch, which copies `SASE_PLAN` from `~/.sase/plans` into the store
     when the coder commits.
   - `write_sdd_files` (`src/sase/sdd/_write.py`): this is the legacy runner path.
   - Grep for any other code that writes plan content under a store's
     `kind_root("plans")` or `plans_root`, and strip there too. Plans already in a store
     never carry the field.
9. Tests:
   - Propose stamps `project` from a project artifacts dir and omits it for `home`.
   - `archive_plan_file` output has no `project`, and the source file keeps it.
   - The `handle_sase_plan` copy has no `project`.
   - The `write_sdd_files` output has no `project`.
   - The helper edge cases.
   - A stamped plan passes `validate_plan` in `authoring`, `launch`, and `archived`
     modes.
10. Docs: in `docs/sdd.md`, add `project` to the SASE-managed fields paragraph (near
    "SASE-managed `create_time`, `status`, ..."). Explain that it is local-only and
    stripped on archive.

## Phase approve

1. In `resolve_project` (`src/sase/main/plan_direct_approval_project.py`), add a step
   after `-P` and before gate history:
   - Read `read_plan_project_file(source_path)`.
   - Canonicalize it like `-P` does (`_canonicalize_project` maps display names and
     aliases to the key).
   - Confirm it names a registered, enabled project, for example with
     `origin_for_project_ref` in `src/sase/bead/cross_project.py`.
   - If the value is present but unknown, return an `unknown_project` refusal. Name the
     frontmatter value, and hint `-P/--project` and `sase project list`. Never fall
     through to the working directory in that case.
2. Make the already-done recovery check independent of the working directory.
   - `_committed_plan_is_done` in `src/sase/main/plan_direct_approval_recovery.py`, and
     its re-check under the lock in `plan_direct_approval_run.py`, should resolve
     committed plan refs against the resolved project's plan roots. Find them through
     `resolve_primary_workspace_for_project(project)` in `src/sase/bead/workspace.py`.
   - Use the working directory only when the primary checkout cannot be resolved.
3. `src/sase/main/plan_approve_render.py` prints a hard-coded `→ <ref> in sase`. Print
   the resolved project's name instead.
4. Leave the live-gate path unchanged. It already takes the project from the gate record
   and refuses `-P`.
5. Tests, each from a temporary working directory outside any project:
   - A `~/.sase/plans` proposal with `project:` and no gate history or planner resolves
     to that project.
   - Frontmatter beats the working directory's project.
   - `-P` beats frontmatter.
   - A display name and a key both resolve.
   - An unknown frontmatter project refuses.
   - The recovery done-check finds the committed plan in the frontmatter project's
     store.
6. Docs: in `docs/sdd.md` (the direct-approval paragraph) and the `sase plan approve`
   row of `docs/cli.md`, say that a proposal's frontmatter `project` supplies its
   project when `--project` is omitted.

## Phase bead-work

1. In `work_from_plan_file` (`src/sase/bead/cli_work_from_plan.py`), read
   `read_plan_project_file(source_path)` after validation.
   - Resolve the value with `origin_for_project_ref`.
   - An unknown or ambiguous value fails before any mutation. Use the existing
     resume-hint error style and give a remedy.
2. Decide whether to route:
   - **Working directory inside the same project** (primary checkout, a numbered
     workspace, or a host-leased checkout): keep today's behavior exactly. This
     preserves the host's leased epic-launch monitor and in-workspace runs.
   - **Otherwise** (no project, `home`, or a different project): route. When the working
     directory is in a different project, print a one-line notice.
3. Add
   `resolve_operation_context_for_project(project_ref, *, cwd=None, for_write, materialize)`
   to `src/sase/bead/operation_context.py`:
   - Find the primary checkout with `resolve_primary_workspace_for_project`.
   - Resolve the store with `_resolve_beads_location(cwd=primary, ...)`.
   - Build the ownership context with `_user_directed_context_for_project`.
   - Apply the same writable-store and read-only checks as `_location_for_store_route`.
   - Populate `project_key`, `project_label`, and `primary_workspace`. Keep
     `invocation_cwd` as the real working directory.
   - Pass the result through the existing `bead_context` slot. It then drives the
     launch-lock anchor, `_resolve_plan_file_context`, `launch_epic_bead_work` (VCS
     workflow, macros, project name), and the creator-entry project.
4. Parent beads while routing:
   - Resolve the plan's `parent_bead` or `--parent` with
     `resolve_operation_context_for_targets([...], cwd=<project primary>)`, so shorthand
     ids resolve in the plan's project.
   - If the parent belongs to a different project than the frontmatter, refuse. Name
     both projects and suggest `--parent top-level`.
   - Without routing, parent behavior is unchanged.
5. In `src/sase/bead/cli_work_from_plan_launch.py`, pass `primary_root=workspace_dir` to
   `archive_plan_file`. The archived plan's PROMPT, PARENT, and BEAD links then come
   from the routed project's checkout, not from the working directory's
   `resolve_checkout_anchor()`.
6. Output:
   - Human output gains a `Project` line before the `Store` line when the project came
     from frontmatter.
   - Add an additive `project` key to the `--json` object.
   - `--dry-run` routes read-only: `for_write=False`, `materialize=False`.
   - Each plan-file target in a multi-target call routes on its own.
7. Tests:
   - From a temporary non-project directory, a plan with `project:` uses that project's
     stores and never creates a legacy `sdd/` store under the working directory.
   - A working directory in another project routes to the frontmatter project, with the
     notice.
   - A working directory in the same project makes no routing call.
   - A plan without the field behaves as before.
   - A parent-project mismatch refuses.
   - Dry-run does not materialize.
   - `archive_plan_file` receives the routed `primary_root`.
8. Docs: in the plan-file-mode paragraph of `docs/beads.md` (beside the cross-project
   `parent_bead` sentence), describe frontmatter routing.

> [!decision] bead_work_project_flag Read `cli_rules.md` with `/sase_memory_read` first.
> Add a long-only `--project <ref>` option (`-P` is `--no-push`) in
> `src/sase/main/parser_bead_lifecycle_work.py`. It applies to plan-file targets only
> and is refused for bead-id targets. Thread it through `cli_work_entry.py` into
> `work_from_plan_file(project=...)`. Precedence is flag, then frontmatter, then working
> directory. A flag that disagrees with the frontmatter wins, with a notice. Document it
> in the `docs/beads.md` options table and add parser tests.

## Non-goals

- Backfilling `project` into existing `~/.sase/plans` files.
- Stamping scratch plans that gateless `sase plan approve <scratch>` adopts.
  `adopt_plan_into_sase` reuses byte-identical local copies, and stamping would break
  that dedupe.
- Changing the live-gate approval path, the host's leased epic launch,
  `sase plan reject`, or plan search and inventory display.

## Verification

- sase-core: `sase tool run check`.
- sase: follow the `lint_and_test.md` memory.
- Manual check after landing, from `~`:
  - `sase plan approve --dry-run ~/.sase/plans/<YYYYMM>/<tale>.md` names the plan's
    project.
  - `sase bead work --dry-run ~/.sase/plans/<YYYYMM>/<epic>.md` shows that project's
    store.
  - Neither command creates anything under `~/sdd/`.
  - A committed sidecar copy has no `project` line.
