---
tier: epic
title: Restore toobig_split chop relaunches after the script-only chop migration
goal: 'The toobig_split chop reliably launches one split_file agent per oversized
  file every hour (unless the split_file hood is active): a deduped head proposal
  no longer cascades and kills the whole wait_on chain, once-per keys are released
  when their launches terminally fail, and agents whose completion markers were removed
  by dismissal are no longer misreported as failed.

  '
phases:
- id: core-release
  title: Once-per key release in the Rust chop engine
  depends_on: []
  description: '''Once-per key release in the Rust chop engine'' section: add a release_chop_once_per
    engine operation to sase-core with a pyo3 binding and a thin axe_chop_facade wrapper
    so the runner can remove recorded once-per keys from the seen document.'
- id: chain-relink
  title: Relink wait_on chains across dedupe-skipped proposals
  depends_on: []
  description: '''Relink wait_on chains across dedupe-skipped proposals'' section:
    change apply_chop_once_per so a proposal whose wait_on dependency was skipped
    as a once-per duplicate is relinked to its nearest accepted ancestor instead of
    being skipped, with launch and preview plumbing for the remapped waits.'
- id: key-release
  title: Release once-per keys for terminally failed launches
  depends_on:
  - core-release
  description: '''Release once-per keys for terminally failed launches'' section:
    record each launch''s dedupe key in the durable launch descriptors and release
    keys through the new engine operation when the launch phase throws or when finalize_launched_chop_runs
    marks an agent''s completion as failed.'
- id: dismissal-evidence
  title: Consult dismissed bundles before fail-closing chop completions
  depends_on: []
  description: '''Consult dismissed bundles before fail-closing chop completions''
    section: make the chop lifecycle''s _agent_completion fall back to the dismissed-bundle
    archive when done.json is missing, so dismissed successful agents finalize as
    action_succeeded.'
- id: recovery-verify
  title: Stale-state recovery and end-to-end verification
  depends_on:
  - core-release
  - chain-relink
  - key-release
  - dismissal-evidence
  description: '''Stale-state recovery and end-to-end verification'' section: clear
    the poisoned toobig_split seen state on the host, exercise a real chop run end-to-end,
    and confirm agents launch for currently oversized files while inhibit_if and dedupe
    still behave correctly.'
create_time: 2026-07-19 13:19:55
status: done
bead_id: sase-7i
---

# Plan: Restore toobig_split chop relaunches after the script-only chop migration

## Problem

Since the sase-6v migration ("Script-only chops with structured launch proposals"), the hourly `toobig_split[sase]` chop
finds oversized files but launches no agents. Every recent run ends with status `skipped` and reason
`all 18 proposal(s) skipped by once-per dedupe`, even though the `split_file` hood is empty and most of those 18 files
have never had a split agent complete successfully.

### Diagnosed root causes (from `~/.sase/axe/lumberjacks/run_every/chops/toobig_split[sase]/`)

The chop script (`bugyi_chop_toobig_split` from the bugyi-chops package) emits one proposal per oversized file, chained
via `wait_on` so split agents run one at a time, and attaches a per-proposal `dedupe_key` of the form
`toobig_split:<workspace>:<path>:<content-digest>`. Three runner defects interact with that shape:

1. **Dedupe-skip cascades through `wait_on` chains.** In `src/sase/axe/chop_policy.py::apply_chop_once_per`, a proposal
   whose `wait_on` dependency was not accepted is itself skipped ("proposal dependency ... was skipped by once-per").
   Because the chop chains _every_ proposal to the previous one purely as a concurrency throttle, a single duplicate at
   the head of the chain cascades and skips all remaining proposals — including files whose keys were never recorded at
   all. This is exactly what the latest run shows: proposal 0 is a genuine duplicate and proposals 1–17 are all
   "dependency was skipped" casualties (5 of them have no key in `seen.json` whatsoever).

2. **Once-per keys are consumed at accept time and never released.** Keys are recorded and persisted in
   `apply_chop_once_per` (called from `src/sase/axe/chop_runner_script_result.py::process_script_chop_result`) _before_
   `launch_chop_proposals` runs, and nothing ever removes them. When a launched agent dies, is killed, or is dismissed
   without changing its file, the file's content digest — and therefore its key — stays the same, so the chop will never
   retry that file. The run history shows 49 keys recorded on 2026-07-18T23:05 and 3 more on 2026-07-19T09:08; both
   batches finalized `action_failed`, yet all keys remained consumed. The intended behavior (and the pre-migration
   behavior) is: retry every still-oversized file each hour unless the `split_file` hood is active
   (`inhibit_if: agent_hood` — which works correctly today).

3. **Dismissal destroys completion evidence, so successes finalize as `action_failed`.** Agent dismissal/cleanup
   (`src/sase/ace/tui/actions/agents/_killing_utils.py::delete_agent_artifacts`) deletes `done.json` so the agent row
   won't reload. The chop lifecycle (`src/sase/axe/chop_lifecycle.py::_agent_completion`) fails closed when the PID is
   dead and `done.json` is missing ("exited without completion artifact"). The overnight 2026-07-18 batch demonstrably
   completed real splits (agent output logs end with "Done marker written to ... done.json" and pushed commits), but
   ~323 dismissal bundles were written before the 09:08 finalization pass, so the whole run was recorded
   `action_failed`. This poisons run history and would also mis-drive `checkpoint: on_action_success` policies for other
   chops.

The bugyi-chops script itself is behaving as designed (chained proposals, content-digest dedupe keys) and needs no
changes. All fixes land in the runner (this repo) and the shared Rust engine (sase-core).

### Intended semantics after this plan

- One duplicate proposal only removes _itself_ from a chain; dependents relink to the nearest surviving ancestor so
  ordering/throttling is preserved.
- A recorded once-per key means "an agent for this exact work was launched and not known to have failed". Terminal
  launch failure releases the key so the next scheduled run retries. An agent that completes successfully but leaves the
  file unchanged (a deliberate no-op) keeps its key, preventing hourly churn on files an agent already examined.
- A dismissed agent whose bundle records a successful terminal state counts as a success during chop-run finalization.

## Design

### Once-per key release in the Rust chop engine

The once-per seen document is owned by the shared engine (`sase_core`; Python only persists it — see
`src/sase/core/axe_chop_facade.py`, whose `check_and_record_chop_once_per` binding performs the check-and-record
transform). Removal must live next to that logic per the Rust core backend boundary.

- In the sase-core repo (open it with `sase repo open sase-core -r "..."`), add a `release_chop_once_per` operation to
  the chop engine: input `{schema_version, document, keys}`, output `{document, released}` where `released` is the count
  of entries removed by exact key match. Validate the document shape the same way `check_and_record_chop_once_per` does;
  unknown keys are a no-op, not an error (release must be idempotent).
- Export it through the pyo3 binding (`sase_core_rs`) alongside the existing chop engine functions, with Rust unit tests
  covering removal, idempotent re-release, and schema validation.
- Add the thin wrapper in `src/sase/core/axe_chop_facade.py` mirroring the existing facade functions, plus a Python-side
  helper in `src/sase/axe/chop_policy.py` (e.g. `release_chop_once_per_keys(lumberjack_name, chop_name, keys)`) that
  takes the policy lock, reads the seen document, calls the facade, and atomically persists the updated document —
  reusing the private `_chop_policy_lock` / `_read_seen_document` / `_seen_path` helpers.
- Update the Rust wire/API tests in sase-core and the facade contract tests here as needed.

### Relink wait_on chains across dedupe-skipped proposals

In `src/sase/axe/chop_policy.py::apply_chop_once_per`:

- When a proposal's `wait_on` dependency (index or id) resolves to a proposal that was skipped as a once-per duplicate,
  do not skip the dependent. Instead walk the dependency chain upward to the nearest _accepted_ transitive ancestor and
  relink the dependent's effective wait target to it (or to nothing if no accepted ancestor remains). Proposals are
  processed in list order and `wait_on` may only reference earlier proposals, so a single forward pass suffices.
- Record a decision entry for relinked proposals (e.g. outcome `accept`, reason "wait dependency `<id>` was deduped;
  relinked to `<ancestor-or-none>`") so run history stays explainable.
- Extend `_ChopOncePerOutcome` with the effective wait mapping (accepted index → remapped `wait_on` value or `None`). In
  `src/sase/axe/chop_runner_script_result.py::process_script_chop_result`, build the accepted proposal list with the
  remapped `wait_on` (e.g. `dataclasses.replace`) before calling `launch_chop_proposals`, and make `proposal_previews`
  display the effective wait so previews and launch records agree.
- A dependent of a proposal that is itself relinked must follow the remapped chain (transitive case), and
  self-consistency must hold when several consecutive chain members are duplicates.

Tests in `tests/test_axe_chop_policy.py` (chain head duplicate, mid-chain duplicate, several consecutive duplicates, id-
and index-based `wait_on`, no accepted ancestor) and runner-level coverage that the launched prompts carry the relinked
`%wait:` lines.

### Release once-per keys for terminally failed launches

- `src/sase/axe/chop_proposals.py::launch_chop_proposals`: include each proposal's `dedupe_key` in the durable launch
  descriptor dict (older run entries without the field are simply not releasable — acceptable backward compatibility).
- Launch-phase exception path in `process_script_chop_result` (the `action_failed` branch that wraps
  `launch_chop_proposals`): release the keys of all accepted proposals that were persisted this run but have no
  successful launch, using the new `release_chop_once_per_keys` helper, so a crashed launch loop cannot permanently
  consume keys.
- `src/sase/axe/chop_lifecycle.py::finalize_launched_chop_runs`: when a linked agent's completion is
  terminal-and-failed, release that launch's `dedupe_key` (matching completion records to launch descriptors by
  artifacts timestamp, falling back to PID). Successful completions keep their keys. Release failures must not abort
  finalization (best-effort with logged output in the run history via `append_chop_run_output`).
- Note the interplay with the dismissal-evidence phase: a completion proven successful via a dismissed bundle must _not_
  release its key.

Tests in `tests/test_axe_chop_agents.py` / lifecycle tests: failed agent releases exactly its own key, successful agent
retains it, mixed batches release only the failed subset, missing `dedupe_key` field is tolerated, and a follow-up
`apply_chop_once_per` accepts a re-proposed key after release.

### Consult dismissed bundles before fail-closing chop completions

In `src/sase/axe/chop_lifecycle.py::_agent_completion`, when the PID is dead and `done.json` is missing, consult the
dismissed-bundle archive before declaring failure:

- Look up the agent's bundle by its artifacts timestamp (`raw_suffix`) through the existing dismissed-bundle helpers
  (`src/sase/ace/dismissed_agents_bundles.py`; note bundle filenames may carry step suffixes like `__c0` — match on the
  root agent bundle). Map a successful terminal bundle status (DONE) to `succeeded=True` with a detail that names the
  evidence source; map failed/killed statuses to failure; keep the current fail-closed behavior when no bundle exists.
- Keep this dependency direction acceptable: `sase.axe` may import the bundle _reading_ helpers, or extract a small
  shared reader if importing from `sase.ace` creates a cycle — follow the existing pattern used by other axe modules
  that read agent state.

Tests: dead PID + no done.json + DONE bundle → `action_succeeded`; FAILED/KILLED bundle → `action_failed`; no bundle →
unchanged fail-closed message.

### Stale-state recovery and end-to-end verification

Runs on the host after the code phases land (this phase mutates durable axe state, not the repo):

- Remove the poisoned seen state so already-recorded keys from the failed 2026-07-18/19 batches stop suppressing
  launches: delete `~/.sase/axe/lumberjacks/run_every/chops/toobig_split[sase]/seen.json` (the bounded event log is safe
  to reset; it only re-allows launches). Leave run history intact.
- Confirm the installed sase (uv tool env) actually contains the fixed runner before exercising (`sase update` /
  reinstall as appropriate on the host).
- Exercise `sase axe chop run` for `toobig_split[sase]` (manual runner path) and verify:
  - proposals are accepted for all currently oversized files, with correct relinked `%wait:` chains in the previews;
  - agents launch into the `split_file` hood, and an immediate re-run reports
    `inhibited by active agent ... in hood split_file`;
  - after killing a launched agent without letting it modify its file, the finalization pass releases its key and the
    next run re-proposes that file;
  - `sase axe chop doctor` stays OK.
- Record observed outcomes (run ids, statuses) in the phase's completion notes.

## Testing

Each code phase carries its own unit tests as described above; sase-core changes include Rust unit tests and binding
coverage. Run `just check` in this repo for every phase that changes files here, and the sase-core repo's own gates for
the engine phase. The final phase is a live end-to-end exercise of the real chop on the host.

## Risks

- **Relink changes behavior for other chops using `wait_on`.** Strictly less-skipping than today and matches the
  sequencing intent documented in the chops redesign; no configured chop relies on cascade-skip semantics.
- **Key release could retry hopeless files hourly.** Bounded by `run_every: 60m`, the `inhibit_if` hood guard, and the
  fact that a _successful_ no-op agent keeps its key; only failed launches retry.
- **Bundle-based success evidence is heuristic.** Only used when `done.json` is already gone; strictly reduces false
  `action_failed` outcomes and never overrides a real `done.json`.
- **Launch descriptor schema gains a field.** Old run entries lack `dedupe_key`; all readers must treat it as optional
  (they do — the plan keeps release best-effort).
