---
tier: tale
title: Stop approved epic plans from losing their PROMPT link
goal:
  Approved epic plans always enter the SDD archive carrying their reciprocal PROMPT bullet, so `sase validate` stops
  reporting spurious missing-link/reverse-link errors that turn `just check` red in every workspace.
create_time: 2026-07-31 08:59:54
status: done
---

- **PROMPT:** [prompts/202607/epic_prompt_link_race.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/epic_prompt_link_race.md)
- **AGENTS:**
  - [bbugyi200.athena.q9](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.q9.md)
- **COMMITS:**
  - [ab650a2](https://github.com/sase-org/sase--plans/commit/ab650a28d6e0f945969677c97c02b85f2a09214a) — fix: restore missing PROMPT link on conventional\_commit\_subject\_gate plan

# Plan: Stop approved epic plans from losing their `PROMPT` link

## 1. What the user asked for

SASE agents keep reporting "flakey" `sase validate` failures during `just check`. Bead `sase-bk` captured the latest
instance and was closed with "I think this is just a temporary `sase validate` failure that we should figure to mitigate
in the future." Bead `sase-bi` captured an earlier instance of the same family. Diagnose the root cause and fix it.

## 2. Reproduction

`sase plan links validate` in a clean workspace (no local changes) currently prints:

```
error: 202607/conventional_commit_subject_gate.md: missing 'prompt' link to 202607/prompts/conventional_commit_subject_gate.md (missing-link)
error: 202607/prompts/conventional_commit_subject_gate.md: 202607/conventional_commit_subject_gate.md is missing a valid 'prompt' link (reverse-link)
SDD validation failed: 2 errors, 537 warnings
```

`sase validate` runs `plan links validate` as one of its four checks (`src/sase/main/validate_handler.py:33`), and
`just check` runs `just validate` (`Justfile:405`), so this ends `just check` red in **every** workspace that has pulled
the plans sidecar, regardless of what the agent changed.

## 3. Root cause

### 3.1 The failure is not random per run — it is random per _plan_

The check is deterministic in a given tree. What varies is which plans got archived in a broken state. Because the plans
sidecar is shared, one bad pair pushed to `origin/main` turns `just check` red for every agent that pulls it, until
somebody hand-repairs it. Agents then report it as unrelated pre-existing drift, which reads as flakiness. That is the
exact noise `sase-bi` warned about ("trains agents to ignore red checks").

### 3.2 Only approved **epic** plans can enter the archive unlinked

The plan/prompt pair is written by two different processes, in two different sidecar checkouts, with no ordering between
them:

| Writer                                                                 | Process           | Checkout      | Writes                                  |
| ---------------------------------------------------------------------- | ----------------- | ------------- | --------------------------------------- |
| `run_agent_exec_plan_accept.py:334-341`                                | the axe **agent** | workspace `N` | prompt snapshot only (`write_sdd_spec`) |
| `plan_approval_actions.py:462` / `_notification_plan_background.py:54` | the **host**      | workspace `1` | the archived plan                       |

`plan_approval_actions.py:167` writes `plan_response.json` **before** `run_plan_side_effects` archives the plan (`:169`
→ `:354`). The agent is polling for that response, so the moment it appears both processes run concurrently. The comment
at `run_agent_exec_plan_accept.py:387` ("Epic prompt snapshots are committed independently so they cannot race the host
command's plan archive/link commits") decoupled the _git_ operations but left the _content_ dependency racing.

The two sides then apply **asymmetric** link policy:

- The prompt side is optimistic. `write_sdd_spec` (`src/sase/sdd/_write.py:45`) always writes
  `- **PLAN:** [../<name>.md]`, pointing at a plan that does not exist yet.
- The plan side is pessimistic. `archive_plan_file` → `_resolved_prompt_path` (`src/sase/sdd/plan_archive.py:121-131`)
  computes the canonical counterpart with `archived_prompt_path_for_plan(destination)` — a **pure path derivation**,
  `plan.parent / "prompts" / plan.name`, no I/O — and then throws it away unless `archived_prompt.is_file()` is true in
  _its own_ checkout. `project_plan_header_sections` (`src/sase/sdd/plan_header_writes.py:49`) silently emits nothing
  when handed `None`.

So when the host archives before the agent's prompt push has been fetched into workspace `1`, the plan is committed with
no `- **PROMPT:**` bullet, permanently. Nothing afterwards repairs it.

### 3.3 Why tales are immune

For `tier: tale` the agent calls `write_sdd_files` (`run_agent_exec_plan_accept.py:344`), which writes **both** files
and commits them together (`Add SDD files for <name>`). Even if the host archived a bullet-less plan first, the agent's
own later commit rewrites the plan with the correct bullet. For `tier: epic` the agent writes only the prompt
(`sdd_commit_paths = [sdd_prompt_path_obj]`), so no second write ever heals the plan. That is why only epics break.

### 3.4 Confirmed against the live evidence

`sase-bk`'s plan came through exactly this path — three commits in the plans sidecar, 2 seconds apart:

```
5d583f2f 07:14:03  Add SDD prompt for conventional_commit_subject_gate     <- agent, workspace N
ec7674af 07:14:05  Archive approved plan conventional_commit_subject_gate  <- host, workspace 1, lost the race
10dc6713 07:14:37  Link approved epic plan to its bead: ...                <- host, header rewrite
```

`git show ec7674af` shows the plan entering the archive with no `- **PROMPT:**` bullet. Every other `202607/*.md` plan
carries one as its first body element. `sase plan links repair` (dry run) already identifies the exact missing bullet,
proving the pair is unambiguously inferable and that only the writer, not the data, is at fault.

### 3.5 Second symptom in the same family: orphaned prompts

`sase-bi` was the mirror image — `202607/prompts/commit_vars_finalizer.md` existed with no plan at all
(`link-missing-target`), and had to be deleted by hand (`e52c3b23`). Both host archive call sites swallow every
exception with no signal: `plan_approval_actions.py:472` is a bare `except Exception: return None` with **no logging at
all**, and `_notification_plan_background.py:70` logs at `debug`. A materialization or lock failure there (for example
`SddIntegrationStatus.LOCK_UNAVAILABLE` raised through `_refresh_materialized_store`, which is likely precisely because
the agent is committing concurrently) leaves the already-pushed prompt orphaned and nobody is told.

### 3.6 Boundary check (Rust core)

The link _grammar_ is Rust-owned (`sase-core/crates/sase_core/src/plan/artifact_link.rs`: `parse_sdd_artifact_link`,
`upsert_sdd_artifact_link`, ...) and is **not** changing. What is wrong here is Python-side write orchestration — which
process projects the reciprocal bullet and under what precondition. No `sase-core` change is required.

## 4. The fix

Make the plan side as deterministic as the prompt side already is, but only where the prompt snapshot is contractually
guaranteed. For an approved epic, `write_sdd_spec` always runs, so the counterpart path is known by construction and the
`is_file()` probe adds nothing except a race.

### Step 1 — `archive_plan_file` gains an explicit "prompt snapshot is expected" mode

In `src/sase/sdd/plan_archive.py`:

- Add keyword-only `expect_prompt_snapshot: bool = False` to `archive_plan_file`, and thread it into
  `_resolved_prompt_path`.
- In `_resolved_prompt_path`, keep the current precedence — an explicit `prompt_path` argument still wins — then:
  - if `expect_prompt_snapshot` is true, return `archived_prompt_path_for_plan(destination)` **unconditionally**;
  - otherwise keep today's `archived_prompt.is_file()` probe.
- Document in the docstring that `expect_prompt_snapshot` is for callers that own a guaranteed prompt snapshot, and that
  a genuinely missing snapshot then surfaces as one honest `link-missing-target` error rather than a silently
  half-linked pair.

The default is unchanged, so every existing caller and the
`test_archive_leaves_unpaired_plan_unlinked_without_prompt_snapshot` contract keep their current behavior.

### Step 2 — the two approval archive sites opt in for epics

- `src/sase/plan_approval_actions.py:462` (`_archive_plan_for_approval`)
- `src/sase/ace/tui/actions/agents/_notification_plan_background.py:54` (`archive_plan_for_approval`)

Both already compute `tier`. Pass `expect_prompt_snapshot=(tier == "epic")` to `archive_plan_file`.

Do **not** change `src/sase/bead/cli_work_from_plan.py:231`. That is `sase bead work <plan-file>`, which can be handed a
hand-authored epic plan that has no prompt snapshot; leaving it on `auto` keeps it from minting a dangling link.

Do **not** change `src/sase/bead/epic_from_plan.py:159`. `project_plan_header_sections` already preserves an existing
`PROMPT` section when passed `None`, so its `is_file()` gate never erases the bullet Step 2 installs.

### Step 3 — stop swallowing archive failures silently

- `src/sase/plan_approval_actions.py:472`: replace the bare `except Exception: return None` with a
  `_logger.warning(..., exc_info=True)` that names the plan, then return `None`. Keep it non-fatal — approval must not
  fail on archive trouble — but make an orphaned prompt diagnosable instead of invisible.
- `src/sase/ace/tui/actions/agents/_notification_plan_background.py:70`: raise the existing `log.debug` to `log.warning`
  with the same information.

### Step 4 — make the validation failure actionable

In `src/sase/main/plan_links_handler.py::_print_validation`, when validation fails and at least one issue has code
`missing-link`, `reverse-link`, or `link-format`, print a remediation line after the issue list, e.g.:

```
hint: `sase plan links repair --write` can restore inferable bidirectional links.
```

`sase validate` relays the child process's stdout verbatim (`validate_handler.py:97-116`), so this single change also
fixes the `sase validate` and `just check` experience. Keep `validate` read-only — it must not auto-write to a shared
sidecar from every agent's `just check`.

### Step 5 — repair the currently broken pair

The bad data is live on `origin/main` of the plans sidecar and will keep failing `just check` until it is fixed. Use the
`/sase_repo` skill to open the `plans` sidecar, then from that checkout run `sase plan links repair --write` (it reports
exactly one action, adding the `PROMPT` bullet to `202607/conventional_commit_subject_gate.md`) and commit the result
via the normal SDD commit path. Re-run `sase plan links validate` and confirm 0 errors.

Do not hand-edit the file; use the repair command so the bullet's canonical layout matches what the validator expects.

## 5. Tests

Add to `tests/sdd/test_plan_archive.py`:

- `test_archive_installs_prompt_section_when_snapshot_is_expected_but_absent` — archive with
  `expect_prompt_snapshot=True` and **no** prompt file on disk; assert the archived plan carries
  `- **PROMPT:** [<yyyymm>/prompts/<name>.md](prompts/<name>.md)`. This is the direct regression test for the race: it
  reproduces the loser-of-the-race filesystem state and asserts the link is written anyway.
- `test_archive_default_still_skips_prompt_section_without_snapshot` — assert the `expect_prompt_snapshot=False` default
  is byte-for-byte unchanged (guards `sase bead work` against dangling links). The existing
  `test_archive_leaves_unpaired_plan_unlinked_without_prompt_snapshot` covers most of this; extend rather than duplicate
  if it is a clean fit.
- `test_archive_with_expected_snapshot_produces_valid_bidirectional_pair` — write the prompt snapshot **after**
  archiving with `expect_prompt_snapshot=True`, then assert `validate_sdd_tree` reports zero errors. This is the
  end-state the bug currently makes unreachable.

Add to the approval-path tests (extend `tests/test_bead/test_epic_from_plan.py` or add a focused module next to the
existing plan-approval tests — the implementing agent should place these next to whatever already covers
`_archive_plan_for_approval`):

- an epic approval passes `expect_prompt_snapshot=True` to `archive_plan_file`;
- a tale approval passes `expect_prompt_snapshot=False`.

Add to the plan-links handler tests:

- a failing validation whose issues include `missing-link` prints the `sase plan links repair --write` hint;
- a passing validation prints no hint.

## 6. Verification

1. `just install` (ephemeral workspaces may have stale deps).
2. `just lint` and `just fmt`.
3. `just test`.
4. `just check` — the SASE validation stage must now be green, which requires Step 5 to have landed.
5. Manually confirm `sase plan links validate` reports 0 errors.

## 7. Out of scope (deliberate)

- **Auto-repairing during `sase validate` / `just check`.** Every agent's `just check` would then write and potentially
  commit to a shared sidecar; a read-only check plus an actionable hint is the safer contract.
- **Serializing the host archive against the agent's prompt push.** Making the host block on the agent's push would put
  network latency on the approval UI path. Step 1 removes the dependency instead of ordering it, which is strictly
  simpler.
- **Changing the Rust artifact-link grammar.** See §3.6.
- **Backfilling historical archives.** Only one pair is currently broken (Step 5). If more appear,
  `sase plan links repair --write` handles them.

## 8. Follow-ups to consider (file as beads, do not implement here)

- The orphan-prompt path (§3.5) is mitigated by logging but not prevented. A durable retry — re-attempting the archive
  when the epic launch notices the plan is missing — would close it. File a task bead.
- `ensure_workspace_sdd_clone` gates its fetch behind a 120s freshness TTL (`_integration_marker.py:12`,
  `_store_link.py:300-307`). That is a reasonable optimization but it widens every cross-checkout visibility window in
  the SDD store. Worth a bead to audit which callers need `fresh=True`.
