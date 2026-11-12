---
tier: epic
title: Auto-restart heals update-skew failures in self-hosted dev installs
goal: "Agents torn apart by a `sase update` of the editable dev install are relaunched
  automatically again: the classifier no longer mistakes the editable checkout for
  agent-workspace code, incidental log strings no longer veto recovery, the at-most-once
  ledger key is derived correctly for day-sharded artifact layouts, and launcher rows
  that died before naming get a working relaunch path.

  "
decisions:
  launcher_relaunch_mode:
    ask:
      Should the healer relaunch launcher rows that died before naming by resubmitting
      their raw prompt?
    choices:
      resubmit:
        Resubmit raw_prompt.md as a fresh launch; today's failure class heals end to end
      escalate_only:
        Decline these rows loudly with "retry by hand"; smaller change, bootstrap deaths
        stay manual
    default: resubmit
    why:
      All six 2026-11-12 failures died pre-naming; classifier fixes alone cannot restart
      them
    answer: resubmit
  marker_scope:
    ask:
      Should never-restart markers be scanned over exception text only when exception
      evidence exists?
    choices:
      exception_first:
        Scan markers over error+traceback when an exception exists; log tail only for
        log-only rows
      filter_json_lines:
        Keep scanning the full log tail but strip JSON log lines first; still
        prose-vulnerable
    default: exception_first
    why:
      The 052959 veto came from an unrelated procfs JSON line; exceptions are better
      signal
    answer: exception_first
  stale_ledger_keys:
    ask: Should existing mis-keyed ledger records (202610__*, 202611__*) be cleaned up?
    choices:
      leave:
        Leave them; terminal declined records are harmless once new claims key correctly
      cleanup:
        Add a one-shot migration deleting records whose project component is a YYYYMM
        date
    default: leave
    why: They are inert history; a migration adds risk for no behavioral gain
    answer: leave
phases:
  - id: core-origin-precedence
    title:
      "sase-core: managed-root precedence and exception-scoped never-restart markers"
    depends_on: []
    size: medium
    description:
      "core-origin-precedence: in sase-core, a traceback file under a managed root
      counts as managed origin even when the agent workspace contains that root
      (self-hosted editable installs), and never-restart markers are scanned over
      exception text instead of the raw log tail whenever exception evidence exists."
  - id: ledger-project-key
    title: Healer ledger keys derive the project from the canonical artifact path parser
    depends_on: []
    size: small
    description:
      "ledger-project-key: `_target_for_artifacts_dir` derives the ledger project via
      `parse_agent_artifact_path` instead of `parents[1].name`, so day-sharded ace-run
      rows key as `<project>__<lineage>` and the at-most-once guarantee holds across
      both healer enumeration sources."
  - id: launcher-relaunch
    title: Healer relaunches launcher rows that died before naming
    depends_on:
      - ledger-project-key
    size: medium
    description:
      "launcher-relaunch: the healer detects failed rows whose runner died before
      assigning a name (ace-run bootstrap deaths), and relaunches them by resubmitting
      the row's raw_prompt.md through the existing launch machinery instead of deferring
      forever on `plan_agent_restart` name lookup."
  - id: pin-docs-verify
    title: Ratchet the core pin, align sase tests and docs, replay the incident
    depends_on:
      - core-origin-precedence
      - ledger-project-key
      - launcher-relaunch
    size: medium
    description:
      "pin-docs-verify: move sase's sase-core-revision.txt past the
      core-origin-precedence commit, update sase-side tests that assert the old
      classifier verdicts, refresh docs/reference/agent-auto-restart.md, and verify with
      a read-only `sase agent auto-restart scan` replay of the 2026-11-12 sase@22e1ef1
      episode."
proposed_by: bbugyi200.athena.1bq
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 06:15:55
status: wip
---

- **PROMPT:**
  [prompts/202611/auto_restart_update_skew_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/auto_restart_update_skew_fixes.md)

# Auto-Restart Heals Update-Skew Failures In Self-Hosted Dev Installs

## Background: the 2026-11-12 incident

On 2026-11-12 between 05:29:56 and 05:30:01 EST, ACE launched six agents (five in
`gh_sase-org__sase`, one in `gh_bobs-org__bob-cli`). At 05:31:52 a dev-mode
`sase update` fast-forwarded the editable install checkout
`/home/bryan/projects/github/sase-org/sase` from `671ff5bde` to `ea4121387` (journal:
`~/.sase/logs/dev_update.jsonl`, episode `sase@22e1ef1`). All six runners died
mid-bootstrap with a torn import:

```
ImportError: cannot import name 'is_valid_git_revision_name' from
'sase.vcs_provider.plugins._git_revision_ops'
(/home/bryan/projects/github/sase-org/sase/src/sase/vcs_provider/plugins/_git_revision_ops.py)
```

raised in
`bootstrap_agent_run → extract_directives_and_write_meta → detect_vcs → entry-point load`
— i.e. lifecycle phase `booting`, before any model turn. This is exactly the Tier 1
"torn Python imports in SASE-managed code" family the auto-restart feature exists for,
and every witness was available: W1 (meta `code_identity` commit `671ff5b` vs current
`ea41213`), W2 (journal entry inside the boot→death window), and the runner even printed
the precise skew explanation ("the sase source checkout … moved … while this runner was
live"). The dying runners did their part: they wrote failed `done.json` markers with
`failure_facts` and `recovery.state = "pending"`, and dropped doorbells.

The healer claimed all six lineages at 05:33 and **declined every one**. Nothing was
relaunched. Three distinct bugs caused this:

### Bug 1 — workspace-origin veto defeats self-hosted installs (5 of 6 rows)

`origin_is_managed` in `sase-core/crates/sase_core/src/agent_auto_restart/catalog.rs`
vetoes managed origin whenever **any** candidate traceback file sits under
`context.workspace_dir`, and the regex-fallback branch in `classify.rs` declines
`workspace_origin` ("ImportError originated in the agent workspace, not managed sase
code") under the same rule. On this machine sase is installed editable from
`/home/bryan/projects/github/sase-org/sase` — which is **also** workspace #0 of the
`gh_sase-org__sase` project. Every runner traceback frame therefore sits under both a
managed root and the workspace dir, the veto wins, `match_family` returns `None`, and
genuine update skew is declined as workspace code. Ledger evidence:
`~/.sase/agent_auto_restart/ledger/gh_sase-org__sase__20261112052956.json`
(`decline_reason: "workspace_origin"`) and four siblings.

### Bug 2 — never-restart markers grep the raw log tail (1 of 6 rows)

Row `20261112052959` had the identical torn import, but `never_restart_reason`
(catalog.rs) scans PROVIDER/KILL/RESOURCE/DIRECTIVE/FINALIZER markers over
`combined_text` = error + traceback + the 200-line runner log tail — **before** family
matching, so no witness can rescue a hit. Its log tail contained an unrelated structured
cleanup line (`"… environ unreadable: Permission denied (os error 13)"` inside a
`launch_scratch_cleanup` JSON record), `permission denied` matched `RESOURCE_MARKERS`,
and the row declined `resource_exhausted`. Log:
`~/.sase/workflows/202611/gh_sase-org__sase_ace-run-261112_052959.txt`.

### Bug 3 — ledger project keys break on the day-sharded layout (all rows)

`_target_for_artifacts_dir` (`src/sase/agent/auto_restart/healer_targets.py:389`)
derives the ledger project as `artifacts_dir.parents[1].name`. For the day-sharded
layout `~/.sase/projects/<project>/artifacts/ace-run/<YYYYMM>/<DD>/<stamp>` that yields
the **month** (`202611`), not the project. Doorbell-sourced healer passes claimed
`202611__<lineage>` records at 05:33:09 (and escalated loudly), then a pass 43 seconds
later re-enumerated the same failures through the scan facade (correct project), found
no record under `gh_sase-org__sase__<lineage>`, and **claimed each lineage a second
time**. The at-most-once ledger guarantee is keyed `<project>__<lineage_root>`, so a
wrong project component defeats it — with a `relaunch` verdict this double-claims could
have double-launched. Evidence: paired `202611__*.json` / `gh_sase-org__sase__*.json`
files under `~/.sase/agent_auto_restart/ledger/`, claimed 43s apart by different pids.

### Bug 4 (latent) — no relaunch path for rows that died before naming

Even with Bugs 1–3 fixed, today's rows could not have been relaunched: the runners died
inside `extract_directives_and_write_meta`, before the `%id(...)` directive assigned a
name, so `agent_meta.json` has `name: null` and the healer's target name falls back to
the artifacts timestamp. `relaunch_claimed`
(`src/sase/agent/auto_restart/healer_relaunch.py`) plans via `plan_agent_restart(name)`
→ `find_named_agent(timestamp)` → `not_found` →
`defer_heal("restart planning refused: …")`, looping until `max_defer_seconds` and then
declining. Bootstrap is precisely the window where update skew strikes, so this class is
common, not exotic.

(Non-bugs, for completeness: the missing `done.json` files on these rows today are user
dismissals — the failure output instructs dismissing to release workspace #0 — and the
W4 probe module scoping in `probe.py` already uses managed roots without a workspace
veto, so it needs no change.)

## Constraints for every phase

- The classifier fixes live in the linked `sase-core` repo (Rust). Open it with
  `sase repo open sase-core -r "<why>"` and work only in the printed path; read its
  `AGENTS.md` first (guarded `sase tool run check`, Conventional Commits, never edit
  versions/changelogs, no `macro_rules!`).
- Python-side phases change `src/sase/agent/auto_restart/` in the sase repo and must
  follow the repo's lint/test obligations before finishing.
- Never weaken the at-most-once guarantee: one automatic restart per lineage, claims
  before any mutation, `write_recovery` never recreates a wiped artifacts dir.
- Do not add a feature flag: these are bug fixes to recovery behavior already guarded by
  the existing `agent_auto_restart.enabled` kill switch, and the broken branches must
  not stay reachable.

## Phase core-origin-precedence: sase-core classifier fixes

All in `crates/sase_core/src/agent_auto_restart/` (`catalog.rs`, `classify.rs`,
`tests.rs`) of the linked `sase-core` repo.

**Managed-root precedence.** Replace the workspace veto with workspace-only semantics: a
candidate file blocks managed origin (and triggers the `workspace_origin` fallback
decline) only when it is under `context.workspace_dir` **and not under any managed
root**. Concretely:

1. Add a helper beside `origin_is_workspace` (or change its semantics and docs —
   implementer's choice, but both call sites must use the new rule) that answers "does
   any candidate file sit under the workspace dir while resolving under no managed
   root?". Keep the existing normalization and whole-path-component comparison
   (`path_is_under`), per the repo's path-canonicalization convention.
2. Use it in `origin_is_managed` in place of the current
   `origin_is_workspace(...) → return false` veto.
3. Use it in the `classify.rs` regex-fallback branch that currently declines
   `workspace_origin` when `looks_like_import_or_attribute_error ∧ origin_is_workspace`.
4. Update the module docs that state "never under the agent's workspace directory"
   (catalog.rs header, classify.rs comments) to state the new precedence rule.

**Exception-scoped never-restart markers** (decision `marker_scope`, default
`exception_first`): in `never_restart_reason`, when the row has exception evidence —
structured facts with `import_error`, `attribute_error`, or a non-empty
`exception_chain`, or a non-empty `error_text`/`traceback_text` — scan the
PROVIDER/KILL/RESOURCE/DIRECTIVE/FINALIZER marker lists over `exception_text` (error +
traceback) only. Scan `combined_text` (including the log tail) only for log-only rows
with no exception evidence. The earlier `outcome`/`kill_source` checks for user kills
are unchanged and still run first.

**Tests** (in `tests.rs`, following the module's existing test style):

- Self-hosted overlap: `workspace_dir` equal to (and a parent of) a managed root, frames
  under `<root>/src/sase/...`, structured `import_error` facts, lifecycle `booting`, W2
  journal witness, probe ok → verdict `relaunch` (`tier1_torn_python`,
  `cannot_import_name`).
- Regression: an ImportError whose frames sit under the workspace dir but under no
  managed root still declines `workspace_origin`.
- Mixed frames: managed-root frames plus one workspace-only frame still blocks
  (workspace-only evidence present).
- The 052959 reproduction: ImportError exception text plus
  `"Permission denied (os error 13)"` only in `log_tail` → not `resource_exhausted`;
  family matches tier 1.
- Log-only row (`error_text`/`traceback_text` empty, facts absent) with
  `"out of memory"` in the tail → still `resource_exhausted`; a `429` in `error_text` →
  still `provider_error`.

Run the guarded gate (`sase tool run check`) before finishing. Commit as
`fix(agent_auto_restart): …` — no wire or binding signature changes, so this is not a
breaking release.

## Phase ledger-project-key: canonical project derivation for healer targets

In `src/sase/agent/auto_restart/healer_targets.py`:

- `_target_for_artifacts_dir` resolves the project through
  `sase.core.agent_artifact_paths.parse_agent_artifact_path(artifacts_dir)`
  (`path_info.project_name`); keep the current `parents[1].name` / `parent.name` chain
  only as the fallback when parsing returns `None`. Name resolution (meta `name` →
  `workflow_name` → dir name) is unchanged.
- Audit the rest of the `auto_restart` package for other path-positional project
  derivations (`rg 'parents\[' src/sase/agent/auto_restart/`) and route any through the
  same parser.

**Tests** (extend `tests/test_agent_auto_restart_healer*.py` /
`_agent_auto_restart_healer_helpers.py`): a doorbell target with a day-sharded layout
`<projects>/gh_x__y/artifacts/ace-run/202611/12/20261112052956` produces ledger key
`gh_x__y__20261112052956` — and, end to end, a doorbell-sourced pass and a scan-sourced
pass claim the **same** ledger record for one failure (the duplicate claim from the
incident becomes impossible). Keep a legacy-layout case green.

Existing mis-keyed `202610__*`/`202611__*` records are left in place (decision
`stale_ledger_keys`, default `leave`): they are terminal `declined` records.

## Phase launcher-relaunch: relaunch rows that died before naming

> [!decision] launcher_relaunch_mode = resubmit

In `src/sase/agent/auto_restart/healer_relaunch.py` (plus whatever small seams it needs
in `healer_flow.py` / `_healer_common.py`):

- **Detection.** A claimed target is a _launcher row_ when neither `agent_meta.json` nor
  `done.json` carries a `name`/`workflow_name` (the healer's `agent_name` is the
  artifacts-timestamp fallback). Detect this in `relaunch_claimed` before the
  `plan_restart` call.
- **Relaunch by resubmission.** For launcher rows, build the replacement from the row's
  `raw_prompt.md` instead of `plan_agent_restart`: resubmit the prompt through the
  existing non-interactive launch machinery (the same seam `sase run` / the TUI relaunch
  flow use — see `src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py` and
  `sase.agent.retry_prompt` for prior art; reuse, don't reinvent). The prompt's own
  `%id(...)`/clan directives re-create the intended identity, and since the dead
  launcher never registered a name there is nothing to wipe — forced name reuse must NOT
  be invoked.
- **Ledger and evidence parity with the named path.** The launcher branch still goes
  through `begin_launch → launched/settled_failed`, writes evidence via
  `_write_evidence`, records provenance on the replacement (reuse `_attach_provenance`
  or its equivalent for the resubmission path), sets `launched_artifacts_dir` from the
  launch result so pinned dependency waits can follow the replacement mapping, and
  publishes the episode relaunch notification. One restart per lineage still holds — the
  ledger record is the same one.
- **Refusals.** Decline (through the existing `decline_heal`, honoring the silenced
  rules) when `raw_prompt.md` is missing or empty, or when the prompt is multi-segment
  (match `plan_agent_restart`'s `_refuse_multi_segment` semantics). Deferral on
  quiescence/probe is unchanged.
- **Workspace holds — investigate before coding.** The failed launcher holds a workspace
  (the incident rows held workspace #0). Verify the resubmitted launch can claim a
  workspace while the failed row's hold exists
  (`src/sase/workspace_provider/occupancy_conflicts.py`), and if it cannot, release the
  failed row's hold as part of a successful relaunch **without** wiping or dismissing
  the failed row's artifacts (the evidence must survive). If release turns out to
  require dismissal-equivalent destruction, stop and record the finding on the phase
  bead instead of destroying evidence.

**Tests** (healer suites in `tests/`): an unnamed launcher row with a relaunch verdict
resubmits its raw prompt and the ledger reaches `launched` with the replacement dir
recorded; a missing `raw_prompt.md` declines; a multi-segment prompt declines; a named
row still takes the `plan_agent_restart` path; at-most-once is preserved when the healer
runs twice.

## Phase pin-docs-verify: pin ratchet, test alignment, docs, incident replay

- **Core pin.** Move `sase-core-revision.txt` past the `core-origin-precedence` commit
  with `just ratchet-core-revision` (see `docs/rust_backend.md`), or confirm the
  scheduled ratchet already has.
- **Test alignment.** Update sase-side tests that assert the old classifier behavior
  through the binding — start with `tests/test_core_agent_auto_restart.py` and the scan
  suites (`tests/test_agent_auto_restart_scan*.py`,
  `tests/test_agent_auto_restart_incident_replay.py`) — so expectations match
  managed-root precedence and exception-scoped markers.
- **Docs.** Refresh `docs/reference/agent-auto-restart.md`: the classifier table and
  "Who the healer considers"/"Never restarted" wording (markers now scope to exception
  text when exception evidence exists; a managed root inside the agent's workspace —
  self-hosted editable checkouts — counts as managed), and document the launcher-row
  relaunch mode next to the phases/modes table. Follow the repo's docs-writing guidance
  (read the `docs_writing` reference memory first).
- **Incident replay.** Read-only verification: `sase agent auto-restart scan -s 30d -j`
  must classify the `sase@22e1ef1` (2026-11-12) rows with a tier-1 `cannot_import_name`
  signature and no `workspace_origin`/`resource_exhausted` decline. Rows whose artifacts
  were dismissed replay from bundles without lifecycle breadcrumbs and may land on
  `ask`/`phase_unknown` — that is acceptable; the assertion is the origin and marker
  classification, not the final mode. Record the replay summary on the phase bead.

## Acceptance criteria

1. A torn-import failure whose frames resolve under an editable managed root that is
   also the agent's workspace directory classifies as `tier1_torn_python` and, with
   witnesses and a passing probe, relaunches — verified by sase-core unit tests and the
   sase-side incident replay.
2. An incidental resource/provider marker string in the runner log tail no longer vetoes
   a row that has concrete exception evidence; log-only rows keep today's marker
   behavior.
3. Healer ledger keys for day-sharded artifacts use the real project name; a doorbell
   pass and a scan pass can no longer double-claim one lineage.
4. A launcher row that died before naming is relaunched once by raw-prompt resubmission,
   with ledger, evidence, provenance, and notifications matching the named path; refusal
   cases decline cleanly instead of defer-looping.
5. `docs/reference/agent-auto-restart.md` describes the new behavior, and sase's core
   pin covers the sase-core commit.
