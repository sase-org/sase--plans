---
tier: epic
title: Approve any tale plan file from the CLI
goal: '`sase plan approve <plan>` commits and approves a tale plan the way sase''s
  TUI does — through its live approval gate when one exists, and otherwise by committing
  the plan itself and launching a `#coder` agent into the planner''s agent family,
  or as a standalone agent when no family can be found. `-k/--kind` defaults to `tale`,
  and every success, dry run, refusal, and failure prints a clear, colored summary.

  '
phases:
- id: engine
  title: Direct approval engine
  depends_on: []
  size: medium
  description: 'engine: add the backend for approving a plan with no live gate. This
    covers archive and adoption helpers, direct-approval receipts wired into plan
    history and `sase plan list`, the gate-history classifier, the resolver/decision
    model with `#coder` prompt composition, the executor, and coder follow-up fields
    on gate approval results.

    '
- id: cli
  title: CLI routing, output, and docs
  depends_on:
  - engine
  size: medium
  description: 'cli: default `--kind` to tale with an epic guard, and add `-n/--dry-run`
    and `-P/--project`. Route live gates and gateless plans through one handler, render
    one approval card for every outcome, and update help, docs, and CLI tests.'
proposed_by: bbugyi200.athena.0rr
create_time: 2026-09-24 19:04:53
status: wip
bead_id: sase-18i
---

- **PROMPT:** [prompts/202609/plan_approve_gateless_tales.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/plan_approve_gateless_tales.md)
- **BEAD:** [sase-18i](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18i/README.md)

# Plan: Approve any tale plan file from the CLI

## Why

`sase plan approve` only answers **pending** proposals, which are plans whose approval
gate is still live. The manual command matters most when that has broken down:

- The planner failed before `sase plan propose`. Its scratch `sase_plan_<name>.md` is
  left in its workspace.
- The proposal was archived to `~/.sase/plans/YYYYMM/<name>.md`, but the gate was never
  created, or its gate shell died (orphaned), or the request expired after 24h.

Today all of these end in `✗ <name> is not awaiting approval`, and the user has to
rebuild the coder launch by hand. `-k tale` also shows up in every example, which makes
it look required.

## User-facing design

### Synopsis

```
sase plan approve [PLAN] [-n] [-k KIND] [-m MODEL] [-P PROJECT] [-p PROMPT] [-w SPEC]
```

Options are listed alphabetically by long name: `-n/--dry-run`, `-k/--kind`,
`-m/--model`, `-P/--project`, `-p/--prompt`, `-w/--wait`.

### Two routes, one command

1. **Gate route (unchanged machinery).** If PLAN matches a live pending proposal (the
   existing name-first resolver in `src/sase/main/plan_pending.py`), sase answers its
   gate through `execute_plan_approval_response`. The gate archives the plan, and its
   gate shell launches `<family>--code`. This is exactly what the TUI does.
2. **Direct route (new).** If PLAN is not a live proposal but names a plan **file**,
   sase approves it itself:
   - a path of any kind (scratch or archived)
   - a `plan:` ref, or `<shard>/<name>`
   - a bare name matching exactly one `~/.sase/plans` proposal
   - the notification ID or prefix of an expired or orphaned proposal

   A gate bundle's `plan.md` resource (`PLAN_RESOURCE_PATH`) maps back to its original
   plan via `sase.plan_gate.original_plan_file_for_resource`, so gate history is not
   lost.

The direct route handles files whose gate history is one of:

- **none**: never proposed, or the gate was never created.
- **orphaned**: no live gate shell or planner owns it.
- **expired**: the request went stale.

Anything already handled refuses, using the existing wording: approved, rejected, sent
back for feedback, cancelled, or approved directly (see receipts below).

### Kinds

- `-k` defaults to **`tale`**. The parser keeps `default=None` so the handler can tell
  whether the user passed a kind. The help text says `(default: tale)`, and the handler
  resolves `None` to `tale`.
- **Epic guard.** If the plan's authored tier is `epic` and no `-k` was passed, refuse
  instead of silently downgrading an epic into one tale coder. This is the only
  exception to "default tale":

  ```
  ✗ big_epic is an epic plan
    Approve it as an epic:        sase plan approve big_epic -k epic
    Or run it as a single tale:   sase plan approve big_epic -k tale
  ```

- The direct route supports `tale` (commit and launch coder), `commit` (commit only),
  and `approve` (launch coder, no commit).
- For a gateless plan, `-k epic` (or an epic plan with no `-k`) refuses and points at
  the existing idempotent epic launcher:

  ```
  ✗ big_epic is an epic plan without a live approval gate
    Launch it with:               sase bead work <path>
    Or run it as a single tale:   sase plan approve <PLAN> -k tale
  ```

- The direct route always validates the plan as a tale, using
  `require_plan_approval_validation(path, "tale")`. Validation failures render through
  the existing `render_validation_human` path and exit 1. Nothing is mutated.

### Where the direct-route coder lands

The planner identity comes from the first of these that is present:

1. the newest gate-history entry's `action_data` (`agent_name`, falling back to
   `agent_cl_name`)
2. the plan frontmatter `proposed_by`, via `sase.bead.attribution.plan_proposed_by`

With no planner, the coder is **standalone**, with the reason "this plan records no
planner".

With a planner, resolve
`resolve_agent_session_attach_plan(AgentSessionAttachDirective(parent=<planner>, suffix="code"), project_name=<project>)`,
imported from `sase.agent.agent_session_attach`. This is a pure read. Then:

- **Success, parent not running** → **family**. The coder prompt carries
  `%id(code, family=<planner>)`. The predicted member name is `attach.agent_name`, e.g.
  `bob--code`, and the family is `attach.parent_base`.
- **Success, but `attach.parent_is_running`** → refuse with `planner_running`:
  `✗ <name>'s planner <planner> is still running`, plus the line "Its approval gate may
  still appear — check with: sase plan list". This avoids racing a gate that is being
  created.
- **`AgentSessionAttachError`** (missing, dismissed, ambiguous, name taken) →
  **standalone**. The reason is the error message with its
  `Cannot attach session member to '<x>': ` prefix stripped.

Never guess a family from workspace occupancy or other heuristics. Attaching to the
wrong family is worse than launching standalone.

### Project resolution (direct route)

Use the first of these that resolves. An explicit value always wins.

1. `-P/--project`. Canonicalize it through the project-tag catalog; an unknown project
   is refused.
2. `resolve_plan_action_project_name(action_data)` from the newest gate-history entry.
3. The `proposed_by` planner, found with `find_named_agent`; its project comes from
   `parse_agent_artifact_path(artifacts_dir).project_name`.
4. The checkout marker above the plan file: `find_marker_from_cwd(plan.parent)` →
   `CheckoutMarker.project_key`. This covers scratch plans left in a workspace.
5. The cwd project from
   `ensure_project_file_and_get_workspace_num(create_missing=False)`, ignoring home
   mode.
6. Otherwise refuse:

   ```
   ✗ cannot tell which project <name> belongs to
     Pass one with -P/--project (see `sase project list`)
   ```

The prompt spells the project with
`known_project_tag_for(load_project_tag_catalog(), project)`, which gives `+sase` or
`#gh:org/repo`.

`-P` passed on the gate route is an error: "-P/--project only applies to plans without a
live approval gate".

### The `#coder` prompt (direct route)

This is composed by one function and used both for dry-run previews and for recovery
hints:

```
<project tag> %model:<model> [%id(code, family=<planner>) | %id(bead=<bead>)]
#coder(<plan ref or path>)

Additional instructions:
<-p text>
```

- **Model precedence** matches the gate path (`run_agent_exec_plan_accept.py`):
  1. A `%model`/`%m` directive inside `-p` wins, and no prefix is added
     (`custom_coder_prompt_model`).
  2. Otherwise `-m MODEL` via `format_model_directive_value`. `worker` means
     size-derived.
  3. Otherwise `validated_tale_followup_model_directive(plan)`, which gives `@<size>`,
     with legacy sizeless plans treated as `@medium`.
- **Plan argument:**
  - For `tale`: the canonical `plan:YYYYMM/<name>.md` returned by the archive.
  - For `approve`: the absolute plan path, or the canonical ref when the plan is already
    committed.
  - Quote the argument if it contains characters the xprompt argument grammar needs
    quoted.
- **`-w SPEC`** is applied with `set_prompt_wait(prompt, PromptWaitDirective(...))` from
  the parsed spec, as the gate path does.
- **`bead=`** is added only for **standalone** coders, and only when the plan
  frontmatter has `bead: <id>` and that bead exists and is not closed (best-effort
  read). This way the coder keeps the association a family coder would have inherited
  from the planner's metadata.
- The launch passes `extra_env={"SASE_PLAN": <local plan path>}`, so commit footers and
  the done-marking plan hook work as they do on the gate path.

### Local proposal adoption and receipts

- **Adoption.** A plan outside `~/.sase/plans` and outside the SDD store (a scratch
  file) is **copied, never moved**, into `~/.sase/plans/YYYYMM/<name>.md`.
  - The `sase_plan_` prefix is stripped, and the dedup counter matches
    `move_plan_to_sase`.
  - Before copying, reuse any existing local copy with byte-identical content, so
    re-running on the same scratch maps to the same local plan.
  - The user's file is never touched.
- **Receipt.** Every direct approval writes a JSON receipt at
  `sase_subdir("plan_approvals")/<shard>/<name>.json`, keyed by the local plan's shard
  and stem.
  - Written atomically, with `schema_version: 1`.
  - Fields:
    - `plan_path`: the absolute local plan path
    - `action`
    - `approved_at` (ISO)
    - `source: "cli"`
    - `route` (`family`, `standalone`, or `none`)
    - `project`
    - `plan_archive_ref`
    - `saved_plan_path`
    - `coder_agent`, `coder_pid`, `coder_error`
    - `family`
    - `retired_gate_id`
    - `original_path`
  - The receipt is the direct route's durable "approved" fact:
    - **Idempotency.** A receipt means
      `was already approved as a tale via sase plan approve (2h ago) · coder kx7`
      (refusal code `already_approved`).
    - **History.** `sase plan approve` and `sase plan reject` misses report it.
    - **Inventory.** `sase plan list` shows it under **Approved**, not as an inferred
      rejection.
- **Already committed.**
  - A PLAN inside the SDD plans store refuses for `tale`/`commit` with
    `already_committed`:

    ```
    ✗ foo is already committed as plan:202609/foo.md
      A committed tale is already approved. To run another coder on it:
        sase run '<exact composed coder prompt>'
    ```

  - `-k approve` stays allowed, since it explicitly means "run a coder on this plan".
  - The archive also refuses to overwrite an existing SDD destination (see engine),
    which covers races and same-name collisions.

### Direct-route execution order

Every step before the launch is idempotent or guarded:

1. Validate, resolve, and decide. This is all read-only, and `-n/--dry-run` stops here.
2. Adopt the plan into `~/.sase/plans` (scratch only), then re-check that no receipt
   exists.
3. For `tale`/`commit`, publish the plan to the SDD store:
   - Call `preflight_plan_archive_credential(("commit",))`.
   - Then call `archive_approved_plan(..., project_name=<project>, if_exists="refuse")`.
4. Write the receipt with coder fields still empty.
5. Retire a stale or orphaned gate, all best-effort:
   - `cancel_gate(bundle, reason="approved_directly", source="plan_approve")`
   - then `mark_already_handled(id, source="plan_approve", action=<kind>)`
   - then `dismiss_notification_best_effort(id)`
   - Marking must come **after** cancelling, so history reads "approved as a tale".
   - If the cancel reports `already_answered`, the gate was answered concurrently. Do
     **not** launch a second coder; record a warning instead.
6. For `tale`/`approve`, compose the final prompt with the real plan ref and call
   `launch_agents_from_cwd(prompt, extra_env={"SASE_PLAN": ...})`. Any exception becomes
   `coder_error`; the plan stays committed.
7. Rewrite the receipt with the coder's name, PID, or error.
8. When the planner's artifacts dir is known (from gate `action_data` or the attach
   plan's `parent_artifacts_dir`), record `plan_approved`, `plan_action`, and
   `plan_committed` on the planner's metadata, as a TUI approval does. This is
   best-effort.

A running SASE agent (`SASE_AGENT` set) must not use the direct route to bypass
LaunchApproval. Refuse before step 2 (dry runs are allowed). The message tells the agent
that plan approval launches agents and must be run by the user.

### Output

Output follows the shared color contract: `should_colorize`, `NO_COLOR`, and
`FORCE_COLOR`. Success and dry runs print to stdout; refusals and errors print to
stderr.

- Labels are dim, lowercase, and fixed-width.
- Tier colors come from `sase.plan_style.kind_style`.
- The coder name is bold, `family <name>` is cyan, and `standalone` is yellow; the
  standalone reason goes on a dim continuation line.
- Best-effort warnings print as yellow `!` lines.
- When stderr is a TTY, a transient Rich status spinner shows the slow steps
  ("Committing plan to sase…", "Launching coder…").

**Direct route, family:**

```
✓ Tale approved · updates_tab_cached_open
  Cache the Updates tab's first open

  plan    plan:202609/updates_tab_cached_open.md · committed to sase
  coder   bob--code · joined family bob · %model:@medium
  gate    a1b2c3d4 · expired approval gate closed

  follow  sase agent show bob--code
```

**Direct route, standalone.** Show the agent name when `AgentLaunchResult.agent_name` is
known; otherwise show `PID <pid>` and follow with `sase agent list`.

```
  coder   kx7 · standalone · %model:@medium
          no agent family: this plan records no planner
```

**Dry run (either route), exit 0:**

```
◇ Dry run · updates_tab_cached_open would be approved as a tale
  Cache the Updates tab's first open

  plan    ~/.sase/plans/202609/updates_tab_cached_open.md
          → plan:202609/updates_tab_cached_open.md in sase
  coder   standalone · %model:@medium
          no agent family: this plan records no planner
  gate    none · never proposed
  prompt  +sase %model:@medium #coder(plan:202609/updates_tab_cached_open.md)

  Nothing was changed. Re-run without -n/--dry-run to approve.
```

On the gate route, the dry run names the gate and says "the gate shell launches the
coder into family <family>".

**Gate route success.** This uses new result fields from the engine phase; fall back to
"the gate shell launches it next" when they are empty. Epic approvals show
`launch  monitor 42 · sase monitor show 42 --follow`, or `proc …`, instead of `coder`.

```
✓ Tale approved · updates_tab_cached_open
  ...
  plan    plan:202609/updates_tab_cached_open.md · committed
  coder   bob--code · launched by gate shell bob--gate
  gate    a1b2c3d4 → <response path>

  follow  sase agent show bob--code
```

**Partial failure (committed, coder launch failed), exit 1:**

```
✓ Plan committed · updates_tab_cached_open · plan:202609/updates_tab_cached_open.md
✗ Coder launch failed: <error>
  Launch it yourself:
    sase run '<exact prompt>'
```

**Errors.** Every `PlanApprovalActionError` renders as `✗ <message>` plus code-specific
hints, replacing today's bare `Error:` lines:

- `git_credential_denied` → the docs hint
- `plan_archive_failed` → "nothing was approved; fix and re-run"
- `conflict_already_handled` / `not_found` → `sase plan list`

Omitting PLAN with nothing pending also mentions the new capability: "Nothing is
awaiting approval. To approve a plan without a live gate, pass its name or file:
`sase plan approve <PLAN>`".

**Exit codes:**

| Code | Meaning                                                      |
| ---- | ------------------------------------------------------------ |
| 0    | success, or dry run                                          |
| 1    | validation failure, or partial failure (coder launch failed) |
| 2    | selection miss or ambiguity, refusal, or action error        |

## Engine

Everything here is Python. Plan-approval orchestration, plan inventory, and the
pending-plan resolver already live in this repo, so no `sase-core` change is needed.
Keep every new module comfortably under the `toobig` limits.

1. **`src/sase/_plan_archive_approval.py`**
   - Add keyword-only `project_name: str | None = None`. When set, it overrides
     `resolve_plan_action_project_name(action_data)`.
   - Add `if_exists: Literal["replace", "refuse"] = "replace"`.
   - With `"refuse"`:
     - Compute `plan_archive_destination(src_plan, sdd_store)` inside the lease
       **before** `reset_and_replay`. Raise a new `PlanAlreadyArchivedError` if the
       destination exists or the source is already under the plans root.
       - It subclasses `PlanApprovalActionError` with code `already_committed`.
       - It carries `path` and the canonical `plan_archive_ref`.
     - Inside the replay, pass `preserve_existing=True` and raise the same error when
       `written` is false. The refusal must propagate without replay retries or commits.
   - Existing callers are unchanged.
2. **`src/sase/llm_provider/_plan_utils.py`**
   - Add `adopt_plan_into_sase(plan_file) -> Path`: copy with content-dedup reuse, as
     specified above.
   - Share the name and dedup logic with `move_plan_to_sase` instead of duplicating it.
3. **Receipts: new module `src/sase/plan_approval_receipts.py`**
   - `DirectApprovalReceipt` (frozen dataclass)
   - `receipt_path_for(local_plan)`
   - `read_direct_approval_receipt(local_plan) -> DirectApprovalReceipt | None`, which
     tolerates corrupt files by returning `None`
   - `write_direct_approval_receipt(receipt)` (atomic)
   - `iter_direct_approval_receipts()`
4. **Gate-history classifier**
   - In `src/sase/main/plan_pending_diagnosis.py`, extract
     `classify_plan_gate_history(located: Path) -> PlanGateHistory`.
   - Its `kind` is `none`, `orphaned`, `expired`, `handled`, or `direct`. It also
     carries `action`, `age`, `notification_id`, `bundle_path`, and `action_data`.
   - It consults the receipt first (`direct`), then `_gate_history_for_plan`.
   - Drive `diagnose_located_plan_miss` from it so there is one classifier. Keep
     existing wording and error codes, and add the `direct` wording.
   - Also expose `locate_plan_candidates(selector) -> tuple[Path, ...]`, a
     generalization of `_locate_named_plan` that reports ambiguity.
   - Map notification IDs and prefixes of unavailable PlanApproval notifications to
     their plan file (`durable_plan_file_for_context`).
5. **Inventory**
   - In `src/sase/main/plan_inventory_collectors.py` / `plan_inventory.py`, turn
     receipts into `ApprovedPlan` rows:
     - `agent` = coder name or `-`
     - `action` = receipt action
     - `meta_path` = receipt path
   - Their plan keys join `represented_paths`. Dedup against planner-meta rows by plan
     key.
6. **Gate route coder fields**
   - Add optional `coder_agent: str | None = None`, `coder_error: str | None = None`,
     and `gate_shell_member: str | None = None` to `PlanApprovalActionResult`.
   - Populate them in `execute_neutral_plan_approval_response` after the synchronous
     `settle_gate_shell`, by reading the gate-shell member's metadata:
     - `gate_followup_agent` → `coder_agent`
     - `gate_followup_error` → `coder_error`
     - the record's `member_agent_name` → `gate_shell_member`
   - Best-effort; never fail an approval over this.
7. **`src/sase/plan_approval_actions.py`**
   - Add public `record_plan_approval_metadata(context, action, *, plan_committed)`, a
     thin wrapper over `_write_plan_action_metadata`.
8. **Resolver/decision: new `src/sase/main/plan_direct_approval.py`**
   - `DirectApprovalRequest`:
     - `selector`
     - `kind` (one of `tale`, `commit`, `approve`, `epic`)
     - `kind_explicit`
     - `coder_model`, `coder_prompt`, `wait`, `project`, `cwd`
   - `CoderPlacement`:
     - `mode` (`family` or `standalone`)
     - `parent`, `member_name`, `family`, `reason`, `planner_artifacts_dir`
   - `RetiredGate`:
     - `notification_id`
     - `state` (`orphaned` or `expired`)
     - `bundle_path`, `action_data`
   - `DirectApprovalPlan`:
     - `request`
     - resolved `kind`
     - `source_path`
     - `location` (`scratch`, `proposal`, or `committed`)
     - `name`, `title`, `size`
     - `project`, `project_tag`
     - `planner`, `gate`, `placement`
     - `model_directive`, `bead`
     - `predicted_plan_ref`, `coder_prompt_preview`
   - `DirectApprovalRefusal`:
     - `code`, `header`, `detail_lines`, `hints`
     - Hints are copy-pasteable commands.
   - `resolve_direct_approval(request) -> DirectApprovalPlan | DirectApprovalRefusal | None`
     - Returns `None` when PLAN names no plan file, so the caller renders the existing
       miss.
     - Raises `PlanApprovalValidationError` on invalid plans.
     - Implements every rule in "User-facing design".
   - `compose_coder_prompt(...)` is a pure function.
9. **Executor: new `src/sase/main/plan_direct_approval_run.py`**
   - `execute_direct_approval(plan) -> DirectApprovalOutcome`
   - `DirectApprovalOutcome` carries:
     - `plan`
     - `local_plan_path`, `plan_ref`, `saved_plan_path`
     - `coder_prompt`
     - `coder` (`AgentLaunchResult | None`), `coder_error`
     - `warnings`
   - Follows the eight-step order above.
   - Raises `DirectApprovalRefused(refusal)` for races detected after resolution
     (receipt appeared, archive refused), so the CLI renders them like any other
     refusal.

**Engine tests** go in new `tests/test_plan_direct_approval.py`,
`tests/test_plan_direct_approval_run.py`, and `tests/test_plan_approval_receipts.py`.
Reuse `tests/plan_validation_helpers.py`, `patched_operational_lease`, and
`patched_sdd_policy`. Mock the launcher, attach resolver, and archive at module seams.

- **Resolution matrix:**
  - never-proposed scratch → standalone
  - `proposed_by` planner attachable → family `…--code`
  - planner dismissed → standalone, with the stripped reason
  - planner running → `planner_running`
  - orphaned gate → direct, with a retired gate
  - expired gate → direct
  - gate handled → refusal, with existing wording
  - receipt present → `already_approved`
  - SDD-committed PLAN → `already_committed` for tale/commit, allowed for approve
  - epic-authored plan with no kind → epic refusal
  - `-k epic` gateless → `sase bead work` hint
  - bundle resource → mapped to the original
  - ambiguous bare name → refusal listing `<shard>/<name>` candidates
- **Project precedence:** `-P` > gate `action_data` > `proposed_by` > marker > cwd >
  refusal.
- **Prompt composition:**
  - `-p` `%model` beats `-m`, which beats size
  - `worker`
  - wait directive
  - extra instructions
  - `bead=` only when standalone and the bead is open
  - family `%id(code, family=…)`
  - project tag
  - quoting
- **Execution:**
  - Step order is asserted via a call log.
  - Scratch adoption copies, content-dedups, and leaves the source intact.
  - Cancel happens before `mark_already_handled`, and the final history reads "approved
    as a tale".
  - `already_answered` means no launch plus a warning.
  - A launch exception gives committed plus `coder_error`, and the receipt records it.
  - `commit` never launches; `approve` never archives.
  - Planner metadata is written only when its artifacts dir is known.
- **Archive and adoption:**
  - `if_exists="refuse"` raises without committing.
  - `project_name` overrides `action_data`.
  - `adopt_plan_into_sase` dedup.
- **Receipts:**
  - round trip, and corrupt-file tolerance
  - inventory Approved rows, with no inferred rejection
  - diagnosis `direct` wording
- **Gate route:** `PlanApprovalActionResult` coder fields are populated from gate-shell
  metadata and stay empty on lookup failure.

## CLI

1. **`src/sase/main/parser_plan.py`**
   - Add `-n/--dry-run` and `-P/--project NAME`, keeping alphabetical order by long
     name.
   - Keep `--kind` `default=None` with help "(default: tale)", and document the epic
     guard.
   - Rewrite the `approve` description and epilog to describe both routes. Examples:
     - bare
     - name
     - `~/.sase/plans/...` path
     - `./sase_plan_my_feature.md --dry-run`
     - `./sase_plan_my_feature.md -P sase`
     - `--prompt`
     - `--wait`
     - `big_epic --kind epic`
     - `--kind commit`
     - Drop `-k tale` from the examples, since it is now the default.
   - Update the group epilog example to `sase plan approve abcdef12`.
2. **`src/sase/main/plan_approve_handler.py`**
   - Route: pending match → gate route; ambiguity → existing render; omitted PLAN with
     nothing pending → improved empty state; otherwise → `resolve_direct_approval`, then
     refusal, dry run, or execute. The existing miss rendering is kept when the resolver
     returns `None`.
   - Apply the kind default and the epic guard on both routes.
   - Error when `-P` is used on the gate route.
   - Apply the agent-context guard, but only to direct execution; dry runs are allowed.
   - Gate-route dry run: validate with `resolve_plan_approval_choice`, which is
     read-only.
   - Exit codes as specified.
   - Keep the auto-approval and tmux/notification helpers in this module untouched.
3. **New renderer `src/sase/main/plan_approve_render.py`**
   - `render_gate_approval`, `render_gate_approval_dry_run`
   - `render_direct_approval`, `render_direct_approval_dry_run`
   - `render_direct_approval_refusal`
   - `render_approval_error`
   - Reuse the console helpers from `plan_pending_render.py`.
   - `sase plan reject` output is unchanged.
4. **Docs**
   - `docs/cli.md`: plan paragraph and command table row.
   - `docs/configuration.md`: `sase plan approve` options row.
   - `docs/sdd.md`: "defaults to the tier authored" → tale default plus epic guard.
   - `docs/xprompt.md`, "Plan Approval and Coder Follow-up":
     - the default change
     - gateless approval launching `#coder`, family vs standalone
     - receipts
     - `--dry-run`
   - Do not edit `CHANGELOG.md`, which release-please generates.

**CLI tests:**

- Update `tests/main/test_parser_plan.py`: kind default semantics, new options with
  short aliases, sorted help, examples.
- Update `tests/test_plan_approve_cli.py`: tierless plan now approves as tale, epic
  guard, `-P` on the gate route, coder line from result fields.
- Add `tests/test_plan_approve_render.py`, run with `NO_COLOR`. Cover every card and
  refusal above: key lines, `follow` hints, standalone reason, recovery command, exit
  codes.
- Add end-to-end handler tests for the direct route with the engine mocked:
  - dry run never mutates (assert no archive, launch, receipt, or gate calls)
  - the agent-context refusal
  - partial failure exits 1

## Out of scope and follow-ups

- Shell TAB completion of file paths for PLAN. The completion kind stays `pending_plan`.
- Guessing the planner of a provenance-free scratch plan from workspace occupancy. Such
  plans launch standalone.
- Gateless epic approval. `sase bead work <plan>` already owns that, idempotently.
- `sase plan reject` for gateless plans.
- Telegram or other frontends.

## Verification

Run `sase tool run check` (the wrapped `just check`) after each phase. Do not run
`just check-full` unless explicitly asked.

Manual smoke test after the `cli` phase:

1. Write a valid tale to a scratch `sase_plan_smoke.md` in a scratch directory inside a
   project checkout.
2. `sase plan approve ./sase_plan_smoke.md -n` shows the standalone card and prompt, and
   nothing changes.
3. Without `-n` it commits, launches a standalone coder, and writes a receipt.
4. Re-running refuses with `already_approved`.
5. `sase plan list` shows the plan under Approved.
