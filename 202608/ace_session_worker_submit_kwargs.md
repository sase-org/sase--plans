---
tier: tale
title: Restore session-worker dedup/exclusive-scope kwargs and guard the submit contract
goal:
  Every ACE session-worker producer submits without crashing, mutually exclusive
  update/sync/bead work is serialized again, and a static conformance test makes a
  future `_submit_session_worker`/`_submit_durable_proc` signature change impossible to
  ship with stale call-site kwargs.
size: medium
proposed_by: bbugyi200.athena.036
create_time: 2026-08-15 21:52:18
status: wip
---

# Plan: Restore session-worker dedup/exclusive-scope kwargs and guard the submit contract

## 1. Symptom

Pressing the `,U` leader keymap (`update_sase`) in `sase ace` crashes the TUI:

```
TypeError: ProcActionsMixin._submit_session_worker() got an unexpected keyword
argument 'dedup_key'
```

The traceback frame is `PluginsBrowserPane(id='updates')` →
`_submit_comprehensive_update_task` → `submit(...)`.

## 2. Root cause

Commit `8c4840458` ("feat(ace): observe durable procs read-only") replaced the owned
proc queue with a read-only observer projection. As part of that migration, several
producers were switched from the removed `_submit_tracked_proc(...)` to the new
`_submit_session_worker(...)`, but **the old keyword arguments were left in place at the
call sites** even though the new method never accepted them.

`ProcActionsMixin._submit_session_worker` (`src/sase/ace/tui/actions/proc_actions.py`,
around line 273) accepts exactly:

```
proc_type, body, *, on_complete, display_name, cl_name, project_file
```

The removed `_submit_tracked_proc` had additionally accepted `dedup_key`,
`exclusive_scopes`, `duplicate_message`, `reload_on_complete`, and `notify_on_complete`,
and it enforced the first three: it looked up a running proc by `dedup_key` (falling
back to `cl_name`), then by overlapping `exclusive_scopes`, and on a hit it warned with
`duplicate_message` and returned `None` instead of submitting.

So the migration produced two defects at once:

1. **Hard crash** at every call site that still passes the removed kwargs.
2. **Silently lost mutual exclusion**: the concurrency guard those kwargs requested no
   longer exists for session-local work at all. Two comprehensive updates (or an
   agents-sync racing a comprehensive update) can now run
   `uv tool install --force-reinstall` / repo sync legs concurrently against the same
   tool environment.

## 3. Blast radius — every currently broken call site

All of these raise `TypeError` the moment the user triggers them (verified by binding
each call's keywords against the real signature; line numbers are pre-fix anchors):

| #   | Site                                                                                                        | Extra kwargs passed                                   | User-visible action that crashes                                                                                                                                                               |
| --- | ----------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `src/sase/ace/tui/modals/plugins_browser_comprehensive_update.py:335` (`_submit_comprehensive_update_task`) | `dedup_key`, `exclusive_scopes`                       | `,U` / Admin Center comprehensive update (**the reported crash**)                                                                                                                              |
| 2   | `src/sase/ace/tui/modals/plugins_browser_agent_clis_actions.py:291` (`_submit_agent_cli_update_task`)       | `dedup_key`, `exclusive_scopes`, `reload_on_complete` | Admin Center → agent CLIs → update                                                                                                                                                             |
| 3   | `src/sase/ace/tui/actions/agents_sync.py:208` (`action_sync_agents`)                                        | `dedup_key`, `exclusive_scopes`, `reload_on_complete` | Sync agents repositories                                                                                                                                                                       |
| 4   | `src/sase/ace/tui/actions/agents_sync.py:254` (`action_integrate_cached_agents`)                            | `dedup_key`, `exclusive_scopes`, `reload_on_complete` | Import cached incoming agent hoods                                                                                                                                                             |
| 5   | `src/sase/ace/tui/actions/_artifacts_beads_common.py:136` (`_submit_beads_task`)                            | `dedup_key`                                           | Every Beads-pane mutation routed through `_submit_beads_task` — callers in `_artifacts_beads_mutations.py:258`, `_artifacts_beads_launch.py:99,137`, `_artifacts_beads_issue_mutations.py:176` |
| 6   | `src/sase/ace/tui/actions/agents/_notification_modals.py:457` (`_submit_legacy_epic_launch_task`)           | `dedup_key`                                           | Legacy epic launch from a notification modal                                                                                                                                                   |

Note that site 5 means the Beads pane's status toggles / launches are broken too, not
just the update flows.

Reproduction without a TUI:

```python
import inspect
from sase.ace.tui.actions.proc_actions import ProcActionsMixin

inspect.signature(ProcActionsMixin._submit_session_worker).bind_partial(
    object(), "comprehensive-update", lambda: None, dedup_key="comprehensive-update"
)  # TypeError: got an unexpected keyword argument 'dedup_key'
```

## 4. Why no gate caught it

- Every producer reaches the method through
  `getattr(self.app, "_submit_session_worker", None)` (or
  `self._submit_session_worker(...)  # type: ignore[attr-defined]`), so the bound object
  is `Any` and mypy cannot check the call.
- Every test double for the method is written as
  `def _submit_session_worker(self, *args: Any, **kwargs: Any)` (see
  `tests/ace/tui/test_plugins_browser_pane_comprehensive_update_execution.py:63`,
  `tests/ace/tui/test_agents_sync_actions.py:68`,
  `tests/ace/tui/test_notification_plan_gate.py:77`), so behavior tests happily accept
  kwargs the real method rejects.
- `tests/ace/tui/test_proc_producer_inventory.py` already walks the AST of every submit
  call in `src/sase/ace` — it just never inspected the keywords.

That third point is the opening for a durable fix: the scanner infrastructure to prevent
this class of bug forever already exists.

## 5. Goal and non-goals

**Goal.** All six sites submit successfully; the dedup/exclusive-scope semantics those
kwargs asked for are genuinely enforced again for session-local work; a static test
fails the build if any producer call ever passes a keyword the real submit method does
not accept.

**Non-goals** (do not do these here):

- Making session workers visible in the Procs pane / proc indicator. Session workers
  build an `ObservedProc` but never register it with `ProcObserver` or `ProcProjection`,
  and `_apply_proc_observer_snapshot` replaces `self._proc_projection` wholesale, so
  merging session rows into the projection is a separate change. It also means
  `running_background_procs()` (restart gating in
  `plugins_browser_sase_update_procs.py`) cannot see session work. File this as a
  follow-up task bead via `/sase_new_task` (size `medium`) rather than growing this
  plan.
- Migrating any of these producers to durable procs.
- Reintroducing `reload_on_complete` / `notify_on_complete` for session workers (see
  decision D3).

## 6. Design decisions

**D1 — Add the guard to `_submit_session_worker`, do not just delete the kwargs.**
Deleting them would stop the crash and quietly keep the concurrency hole. The old code
deliberately serialized these mutations; concurrent `uv tool install --force-reinstall`
runs against one tool environment are a real corruption risk.

New signature:

```python
def _submit_session_worker[T](
    self,
    proc_type: str,
    body: Callable[[], TrackedProcResult[T]],
    *,
    on_complete: Callable[[TrackedProcCompletion[T]], None] | None = None,
    display_name: str | None = None,
    cl_name: str = "",
    project_file: str = "",
    dedup_key: str | None = None,
    exclusive_scopes: Collection[str] = (),
    duplicate_message: str | None = None,
) -> ObservedProc | None:
```

It returns the session `ObservedProc` on submit and `None` on a collision, mirroring the
old `_submit_tracked_proc` contract.

**D2 — Explicit dedup only; no implicit `cl_name` dedup.** The old code fell back to
deduping by `cl_name` when `dedup_key` was `None`. Do **not** restore that fallback:
several session producers have been running without any dedup since the refactor
(install-many, mode-switch, revert preview, question responses, bead-issue mutations),
and reviving an implicit guard would newly block legitimate concurrent work — e.g. two
different mutations sharing one `cl_name`. Dedup applies if and only if the caller
passes `dedup_key`.

**D3 — Drop `reload_on_complete` from the three call sites that pass it.** Session
workers never auto-reload and never auto-notify; each `on_complete` handler does its own
toasting/reloading. All three sites pass `reload_on_complete=False`, which is already
the effective behavior, so removing the argument changes nothing. Do not add the
parameter just to ignore it.

**D4 — Conflict detection spans session _and_ durable claims.** A session submit must
lose to (a) an in-flight session worker with the same `dedup_key`, (b) an in-flight
session worker whose `exclusive_scopes` intersect, (c) a running/pending durable proc
claiming an intersecting scope (`ProcProjection.scope_conflict`, plus
`self._proc_pending_scopes`). Symmetrically, `_submit_durable_proc`'s existing conflict
check gains the in-flight session claims so the exclusion is not one-directional.

**D5 — Static conformance test over a typed Protocol.** A `Protocol` annotation on each
`getattr` result would also restore mypy checking, but it means touching ~15 sites and
does not cover the `# type: ignore[attr-defined]` direct calls. Extending the existing
AST scanner covers every site uniformly with far less churn.

## 7. Implementation

### Step 1 — `src/sase/ace/tui/actions/proc_actions.py`

1. Extend `_submit_session_worker` to the D1 signature.
2. Build the session `ObservedProc` with `dedup_key=dedup_key` and
   `exclusive_scopes=frozenset(exclusive_scopes)` (both fields already exist on
   `ObservedProc`, `src/sase/ace/tui/proc_observer.py`).
3. Before starting the worker, resolve a conflict per D4. In-flight session claims are
   already available: `self._session_completion_callbacks` maps
   `proc_id -> (on_complete, proc_info)` and entries are popped in both
   `_on_session_worker_completed` and `_on_session_worker_error`, so the stored
   `proc_info` values are exactly the live claims — read them rather than adding a
   parallel dict.
4. On conflict:
   `self.notify(duplicate_message or f"A {existing.proc_type} proc is already running for {humanize_cl_name(cl_name or existing.cl_name)}", severity="warning")`
   and return `None` without starting a worker. `humanize_cl_name` is already imported
   in this module.
5. Factor the conflict lookup into a small private helper (e.g.
   `_session_scope_conflict(dedup_key, requested_scopes)`) so `_submit_durable_proc` can
   reuse it for the D4 symmetry.
6. In `_submit_durable_proc`, fold the session-claim lookup into the existing
   `existing is None` chain that already consults `projection.scope_conflict(...)` and
   `self._proc_pending_scopes`.

### Step 2 — Fix the six call sites

Sites 1–6 from §3 keep `dedup_key` / `exclusive_scopes` (now real parameters) and drop
`reload_on_complete`. Additionally:

- Site 1 (`_submit_comprehensive_update_task`) currently ends with `return True`
  unconditionally. Return `submit(...) is not None` so a collision leaves the Admin
  Center open — its caller at `plugins_browser_comprehensive_update.py:265` only calls
  `_close_admin_center_after_sase_update()` when the submit reports success. Add
  `duplicate_message="A SASE, agent CLI, or agents-repository update is already running."`
  (the wording the pre-refactor site used).
- Sites 3 and 4 add
  `duplicate_message="An agents-repository synchronization is already running."`; site 2
  adds `duplicate_message="An agent CLI update is already running."`.

### Step 3 — Restore the sase-update family's scope claims

`src/sase/ace/tui/modals/plugins_browser_sase_update_procs.py` currently loses its dedup
inputs entirely: `_submit_dev_update_proc` accepts `dedup_key` and `duplicate_message`
and then executes `del dedup_key, duplicate_message` (line 143).

1. Delete that `del` and forward both values to `_submit_session_worker`.
2. Give all three update producers the `sase-update` exclusive scope so they are
   mutually exclusive with the comprehensive update, which claims
   `("sase-update", "agent-cli-update", "agents-sync")`:
   - `_submit_sase_update_proc` → `dedup_key="sase-update"`,
     `exclusive_scopes=("sase-update",)`
   - `_submit_dev_update_proc` → caller-supplied `dedup_key`,
     `exclusive_scopes=("sase-update",)`
   - `_submit_combined_update_proc` → `dedup_key="sase-update"`,
     `exclusive_scopes=("sase-update",)`

   A `dedup_key` and an `exclusive_scopes` entry that share the string `sase-update`
   live in different namespaces and do not conflict with each other — the scope entry is
   what creates the mutual exclusion, so it must be passed explicitly.

### Step 4 — `src/sase/ace/tui/proc_producer_sites.py`

The inventory records a `concurrency_keys` tuple per site, and every `session_worker`
entry currently declares `()` — accurate only because enforcement was lost. Update the
entries that now claim scopes/dedup keys so the catalog matches reality:
`plugin.comprehensive`, `plugin.agent_cli_update`, `plugin.sase_update`,
`plugin.dev_update`, `plugin.combined_update`, `agents.sync`, `agents.cached`,
`bead.mutate`, `notify.legacy_epic`. Use the existing `{placeholder}` convention for
interpolated keys (e.g. `"beads:{operation}:{project}:{bead_id}"`). Do not add or remove
sites — `test_inventory_records_infrastructure_and_classifications` asserts
`len(PRODUCTION_PRODUCERS) == 36`.

## 8. Tests

### 8.1 Static conformance guard (the durable fix)

Extend `tests/ace/tui/test_proc_producer_inventory.py`:

- Add a `keywords: tuple[str, ...] = ()` field to `FoundProducerCall` and populate it in
  `_SubmitCallVisitor.visit_Call`. Keep `site_key` unchanged so the existing inventory
  comparison is unaffected.
- Add `test_producer_calls_match_submit_signatures`: for every discovered call, bind its
  keywords against the real signature —
  `inspect.signature(ProcActionsMixin._submit_session_worker)` or
  `..._submit_durable_proc` — with `bind_partial` (or a plain set-difference against the
  signature's parameter names) and assert no site passes an unknown keyword. The failure
  message must name `path:function:kind` and the offending keywords.
- The visitor's `getattr` binding must stay order-sensitive:
  `plugins_browser_install.py` rebinds the local name `submit` to `_submit_durable_proc`
  at line ~425 and to `_submit_session_worker` at line ~468 in the same module. A
  whole-file "last assignment wins" binding misattributes the first call and produces a
  false positive — the current `visit_Assign`/`visit_AnnAssign` walk already resolves
  this correctly, so do not replace it with a pre-pass.

Sanity check: this test must fail against the pre-fix tree with exactly the six sites in
§3 (and no others) before it passes against the fixed tree.

### 8.2 Behavior tests for the new guard

New tests against `ProcActionsMixin` (a lightweight harness that stubs `run_worker` and
`notify`, in the style of the existing proc-actions tests):

- Same `dedup_key` while one is in flight → second submit returns `None`, starts no
  worker, and emits a `warning` notification carrying `duplicate_message`.
- Overlapping `exclusive_scopes` → rejected; disjoint scopes → both submit.
- No `dedup_key` and no scopes → concurrent submits are allowed (locks in D2).
- A durable proc holding a scope blocks a session submit claiming it, and an in-flight
  session claim blocks a durable submit claiming it (D4, both directions).
- The claim is released after `_on_session_worker_completed` **and** after
  `_on_session_worker_error`, so a later submit with the same key succeeds.

### 8.3 Call-site tests

- `tests/ace/tui/test_plugins_browser_pane_comprehensive_update_execution.py:113`
  asserts `_submit_comprehensive_update_task(preview) is True`; extend it with a
  collision case that asserts `False` and that the Admin Center is not closed.
- Tighten the test doubles for the six repaired paths (at minimum
  `test_plugins_browser_pane_comprehensive_update_execution.py:63`,
  `test_agents_sync_actions.py:68`, and the agent-CLI fake in
  `test_plugins_browser_pane_agent_clis.py:294`) so they validate their kwargs against
  `inspect.signature(ProcActionsMixin._submit_session_worker)` instead of swallowing
  `**kwargs`. A tiny shared helper in a test support module keeps this to one line per
  fake. Leave the remaining `**kwargs` fakes alone — §8.1 already covers them.

## 9. Verification

```bash
just install
just check
```

Run the focused suites while iterating:

```bash
just test-scoped   # or: pytest tests/ace/tui/test_proc_producer_inventory.py \
                   #            tests/ace/tui/test_agents_sync_actions.py \
                   #            tests/ace/tui/test_plugins_browser_pane_comprehensive_update_execution.py \
                   #            tests/ace/tui/test_plugins_browser_pane_agent_clis.py \
                   #            tests/ace/tui/test_artifacts_beads_mutations.py
```

Because this touches shared proc infrastructure that many suites reach, run
`just check-full` before landing, and run it through `/sase_monitor` (never inline) with
a `--next` action.

Manual smoke, if a TUI is available: open `sase ace`, press `,U`, confirm the modal, and
confirm the update starts without a crash; press `,U` again while it runs and confirm a
single warning toast instead of a second concurrent update.

## 10. Risks

- **Over-blocking.** A too-broad scope claim would make legitimate actions silently
  refuse. Mitigation: scopes are copied verbatim from the pre-refactor call sites, and
  D2 forbids the implicit `cl_name` fallback.
- **Leaked claims.** If a worker never reaches either completion path, its claim would
  pin the scope for the session's lifetime. The claim shares the exact lifecycle of
  `_session_completion_callbacks`, which the existing success and error handlers both
  pop, so this adds no new leak surface — but the error-path test in §8.2 must cover it.
- **Restart gating still ignores session work** (§5 non-goal). Worth stating explicitly
  in the follow-up bead so it is not mistaken for something this plan fixed.
