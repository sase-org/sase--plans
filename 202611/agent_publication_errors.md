---
tier: epic
title: Stop agent publication false blocks, parked-row floods, and drain starvation
goal: 'The background agent_publication job stops reporting scanner false positives
  every pass. bob-cli''s stuck and starved rows publish. A request withheld by publication
  protection can no longer starve its project or flood the error digest. A future
  scanner-rule correction releases its own stale false blocks with provenance, and
  .land coder rows stop quarantining over a prompt archive that already exists.

  '
decisions:
  revalidation:
    ask: How should blocks recorded under older scanner rules be re-evaluated after
      the rules change?
    choices:
      automatic: The next publish attempt rescans under current rules and supersedes
        clean blocks
      command: Stale blocks stay enforced until a human runs a new report-first revalidate
        command
    default: automatic
    why: A scanner-hit block is a verdict on (bytes, rules); new rules should re-decide
      it
    answer: automatic
  rule_scope:
    ask: Which scanner rules should get the left word-boundary fix?
    choices:
      all_prefix: Every prefix-literal token rule, plus a digit-or-capital body check
        for sk- keys
      openai_only: Only openai-key, as the research proposed; sibling rules still
        match inside words
    default: all_prefix
    why: The Stripe, Slack, GitHub, and Anthropic rules share the same unanchored-prefix
      defect
    answer: all_prefix
  park_notice:
    ask: How should the job report a request that publication protection newly parks?
    choices:
      once: The pass that first parks a request reports one error; later rechecks
        stay silent
      silent: Never a job error; parked rows appear only in doctor and the publisher
        heartbeat
    default: once
    why: A real secret in an agent transcript deserves one digest entry, not zero
      or one per pass
    answer: once
  historic_quarantines:
    ask: Should the 233 bob-cli rows quarantined by the .land prompt-archive bug be
      revived?
    choices:
      leave: They stay quarantined; most of their pages were pushed before the ack
        check refused
      recheck: The verify phase gates a bounded revival of only those rows to prove
        the fix
    default: leave
    why: Reviving mostly re-publishes identical pages; the fix is proven by new .land
      rows
    answer: leave
  planner_hints:
    ask: Should planner prompt-archive rows also append a doorbell hint, as the research
      proposed?
    default: false
    why: The unified drain loop reaches hint-less rows in the same pass; a hint only
      reorders them
    answer: false
phases:
- id: core
  title: Anchor the scanner rules and make scanner-hit blocks rules-versioned (sase-core)
  depends_on: []
  size: medium
  description: 'core: in sase-core, add a left token boundary to the prefix credential
    rules, bump the scanner rules version to 2, make a scanner-hit block enforced
    only while its evidence is current, supersede stale blocks with provenance on
    a clean rescan, add a parked-recheck predicate, and expose all of it through bindings
    with tests.'
- id: drain
  title: One drain loop that never lets a failing row starve its project
  depends_on: []
  size: medium
  description: 'drain: replace the hinted-then-fallback loops in publication_job with
    one loop that excludes keys already attempted this pass, stops only on repository-scope
    failures, reports failure scope on the drain outcome, and compacts claimed hint
    files to at most one per project.'
- id: land_prompt
  title: Recognize a session member's prompt archive for .land coder rows
  depends_on: []
  size: small
  description: 'land_prompt: make the deferred-restore archive check accept an archive
    whose AGENTS label is a member of the request''s own agent session (for example
    the --plan member), indexed once per restore call, so .land coder rows acknowledge
    instead of quarantining.'
- id: protection
  title: Consume the rules-versioned protection policy in sase
  depends_on:
  - core
  size: medium
  description: 'protection: move the sase-core pin, route every ledger lookup through
    the enforced-lookup binding so stale blocks are rescanned, pass the clean rules
    version into resolve so stale blocks are superseded, keep superseded history on
    disk, and teach doctor to separate enforced, stale, and superseded records.'
- id: parking
  title: Park protection-withheld requests instead of retrying them every pass
  depends_on:
  - drain
  - protection
  size: medium
  description: 'parking: add parked fields to the outbox (schema 8), park withheld
    and prompt-protection rows at settlement, keep parked rows out of selection and
    the self-ring until the Rust recheck predicate says they are due, stop counting
    parks as job errors (except per park_notice), and surface parked counts in the
    heartbeat, chop summary, and doctor.'
- id: verify
  title: Install, verify live on athena, and hand off follow-ups
  depends_on:
  - land_prompt
  - parking
  size: small
  description: 'verify: propose the human-only reinstall through a gate if the installed
    build predates the fix, then confirm on athena that bob-cli''s stuck and starved
    rows drain, claim files compact, stale blocks are superseded, and the digest goes
    quiet; record follow-ups for sase-1ot and sase-1r7.'
proposed_by: bbugyi200.athena.184
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 13:54:34
status: done
bead_id: sase-1r8
---

- **PROMPT:** [prompts/202611/agent_publication_errors.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/agent_publication_errors.md)
- **BEAD:** [sase-1r8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1r8/README.md)

# Plan: Stop agent publication false blocks, parked-row floods, and drain starvation

## Context (verified this turn)

The research report `research:202611/agent_publication_errors_still_occurring.md` and
its Nov 2 predecessor describe three causes. I checked each claim against sase
`d563b9da4d`, sase-core `55532e2d`, and live athena state at about 13:40 EST. All of the
following hold:

- **Unanchored rule.** `crates/sase_core/src/note_attachment/scanner.rs` has
  `openai_re()` = `sk-[A-Za-z0-9_-]{20,}` with no left boundary. The sibling prefix
  rules (`ghp_…`, `sk-ant-`, `AIza`, `AKIA`, `xox?-`, `sk_live_`/`rk_live_`) are
  unanchored in the same way. For example, the Stripe rule matches inside any
  `…task_live_…` identifier.
- **Blocks are permanent.** `agent_publication_protection.rs` keeps every `blocked`
  record forever: `resolve_verified_clean` retains blocks and `retry_allowed` refuses
  them. Every Python call site (`_publication_snapshot_gating._prior_content_blocked`
  and `_prior_bytes_releasable`, `prompt_archive/protection._prior_block`,
  `_git_sync_ops_payload`, `_publication_state_repair_payload`, and
  `publication_recovery`) checks the ledger _before_ rescanning. A blocked digest
  therefore never meets a fixed rule. Every record carries `evidence[].rules_version`,
  and all of them are version 1 today.
- **Live ledgers.** bob-cli has 56 records, all `blocked/scanner_hit/openai-key`, all
  rules v1. tailnet has 1 of the same kind. sase has 4 `openai-key` and 4
  `high-entropy-assignment` records.
- **Live bob-cli outbox.** There are 51 active rows. 5 of them are stuck with
  `attempts=0` and a withheld error: `research.5n.mus`, `bob-cli-fs.8.1.3`,
  `bob-cli-fx.2` ×2, and `bob-cli-g1.2`. The other 46 were never attempted
  (`updated_at == created_at`). 153 `hints.claimed.*.jsonl` files have piled up.
- **Head-of-line block.** In `publication_job._drain_project`, the hinted loop `break`s
  on `outcome.error`, and the fallback loop only runs under `result.error is None`.
  Withheld rows are settled with `increment_attempts=False`
  (`_commit_publication_transaction_steps.acknowledge_completed`). They stay ready, keep
  their hints, re-ring the doorbell, and error on every pass. `preserve_hints` is
  all-or-nothing, so claim files are never unlinked.
- **Secondary class.** 233 of bob-cli's 296 quarantined rows end in "prompt archive is
  unresolved and a missing local source does not prove there was no prompt obligation".
  The `.land` coder run has only `followup_prompt.md`. The session's prompt _is_
  archived (for example `prompts/202611/relation_successor_core_closeout.md`), but under
  the `…fs.2.1.land--plan` AGENTS label, and
  `prompt_archive/deferred._prompt_archive_present` only accepts an exact label match.
  The hood pages are pushed before the acknowledgment check refuses: revision
  `98bfec6aca2a` is in the published `--code` `commits.json`.

## Where this plan departs from the research

Each disagreement is a plan decision above. Two more departures need no reviewer input:

- **The parked-recheck policy lives in Rust.** The protection policy and the failure
  accounting (`decide_publication_failure_accounting`) are already in sase-core. Per the
  Rust-core boundary, the "is this parked row due?" predicate belongs there too. The
  outbox store itself stays Python-owned, as it is today.
- **The SSH publickey retry stays out of scope.** It caused 57 entries during GitHub's
  1¾-hour incident, and the job's own cadence already retries it. The verify phase notes
  it on `sase-1r7` and does not add retries inside the sidecar-lock-holding transaction.
- **Agreed as written:** the boundary fix, rules-version-aware recovery of the five
  rows, excluding failing keys and still draining the rest, not charging confirmed
  blocks, and re-checking `sase-1ot`'s 27 historical hits under the corrected rule.

**Hygiene for every phase.** The scanner installed today is v1. Never write a long
hyphenated name that contains `task-` followed by 20+ name characters in prompts, chat,
artifact filenames, or commit messages, because it will withhold your own transcript.
Build such fixtures in code by concatenation (for example `"ta" + "sk-" + tail`) and
refer to them in prose as `task-…`. Also build key-shaped positive controls by
concatenation, so that no literal key-shaped string lands in source. GitHub push
protection reads source too.

## Phase core: Anchor the scanner rules and make scanner-hit blocks rules-versioned (sase-core)

Work in the linked sase-core checkout (`sase repo open sase-core`). Read its `AGENTS.md`
and verify with `sase tool run check`.

1. **Scanner** (`crates/sase_core/src/note_attachment/scanner.rs`):
   - Require a left token boundary before each prefix-literal credential rule: the start
     of the line, or a character outside `[A-Za-z0-9_-]`. Rust `regex` has no
     look-behind, so use a leading `(?:^|[^A-Za-z0-9_-])` group or check the preceding
     byte in code.
   - `scan_windows` splits lines longer than 16 KiB. A window that does not start at the
     true line start must not treat its first byte as a boundary. Pass the preceding
     byte, or ignore matches at offset 0 of non-initial windows.
   - For `openai-key` and `anthropic-key`, also require at least one ASCII digit or
     uppercase letter in the key body. Real keys are random base62 or base64url;
     kebab-case words are not. Matching needs `find_iter` or captures instead of
     `is_match`, so change `CredentialRule` to carry a match predicate instead of a bare
     regex getter.
   - Bump `ATTACHMENT_SCANNER_RULES_VERSION` to `2`, and update its pins in the
     `sase_core` and `sase_core_py` tests (`note_attachment/tests.rs` asserts `1`).
   - Add regression tests:
     - Clean: hyphenated `…task-…` names with long tails, a `…-task-why-…` artifact path
       inside compact JSON, a `test-task-…-*.cjs` path, `…task_live_…` for Stripe, and
       the window-boundary case.
     - Hit (positive controls): line start, after `=`, `:`, a quote, a space, `Bearer `,
       and a URL `/`, for every anchored rule.

   > [!decision] rule_scope = openai_only Anchor and shape-check only `openai_re()`. The
   > other prefix rules stay as they are. The rules version still moves to 2.

2. **Protection policy** (`crates/sase_core/src/agent_publication_protection.rs`):
   - Add `publication_protection_record_enforced(record) -> bool`. It returns true for
     `RemoteSecretRejection` blocks, for any block with `KnownValue` evidence (an
     environment-dependent verdict, not a rules verdict), for blocks with no evidence,
     and for blocks whose maximum `evidence[].rules_version` is at least
     `ATTACHMENT_SCANNER_RULES_VERSION`. Compute staleness from the evidence, never from
     `rules_versions`: a later unverified merge appends to `rules_versions` without
     evidence.
   - Add a lookup binding (for example `publication_protection_lookup_enforced`) that
     returns the record unless it is a stale scanner-hit block, in which case it returns
     `None`. Keep `publication_protection_lookup` raw for diagnostics.
   - `publication_protection_retry_allowed` returns true for stale blocks. Note this
     semantic change in the commit body.
   - `PublicationProtectionResolveVerifiedCleanRequestWire` gains optional,
     serde-defaulted fields `clean_rules_version` and `timestamp`. When both are present
     and the matching record is a stale scanner-hit block whose maximum evidence rules
     version is below `clean_rules_version`:
     - Move that record out of `records`.
     - Append it to a new ledger field
       `superseded: Vec<PublicationProtectionSupersededWire>` (serde default, skip when
       empty). Each entry holds the full prior record, `superseded_at`,
       `superseded_by_rules_version`, and `reason: "rescanned_clean"`.

     Enforced blocks behave exactly as before. Without the new fields, resolve behaves
     exactly as before (fail-closed for old callers). Validate superseded entries in
     `publication_protection_ledger_validate`.

   - Keep `AGENT_PUBLICATION_PROTECTION_WIRE_SCHEMA_VERSION` at 1. The field is
     additive, and an older reader that ignores it just sees no block and rescans under
     its own rules.

   > [!decision] revalidation = command Do not add the enforced lookup or change
   > `retry_allowed`. Stale blocks stay enforced. Instead add
   > `publication_protection_revalidate(ledger, identity, clean_rules_version, timestamp)`,
   > which performs the same supersede-with-provenance transition only when called
   > explicitly. The protection phase wires it to a CLI.

3. **Parked-recheck predicate**: add a function to the `agent_publication_recovery`
   module, for example `publication_parked_recheck_due`. It takes `now`,
   `parked_checked_at`, `parked_rules_version`, `current_rules_version`, and
   `recheck_interval_seconds`. It returns due when the rules versions differ, when the
   interval has elapsed, or when `parked_checked_at` lies in the future (clock skew).
4. **Bindings**: expose the new functions in the `sase_core_py` domains that already
   bind these modules (protection lives in `note_attachment/mod.rs`). Register each one,
   and add round-trip tests. All bindings are additive, so this is `feat:`, not
   breaking.

## Phase drain: One drain loop that never lets a failing row starve its project

Work in `src/sase/agents_sync/`. This phase needs no sase-core change.

1. **Failure scope on the outcome.** `CommitPublicationOutcome`
   (`_commit_publication_types.py`) gains `failure_scope: str | None`.
   `drain_agent_publications` (`_commit_publication_drain.py`) sets it to `repository`
   for failures that are not about specific rows: target or owner unavailable, clone,
   lock busy, pull or rebase, payload clean, commit, push, manifest verify, and a
   repository-scope timeout. It leaves it `None` when only item errors occurred.
   `drain_selected_agent_publications` passes it through unchanged.
2. **One loop.** Replace the two loops in `publication_job._drain_project` with one:
   - Candidates are the ready hinted keys (hint order) followed by the remaining ready
     keys (oldest first, by `publication_sort_key`), excluding every key already
     attempted in this pass.
   - Drain batches of at most `_HOOD_BATCH_LIMIT`. Add each batch to `attempted`.
   - Stop on the deadline, on lock busy, on a repository-scope failure, or when no
     unattempted ready key remains.
   - An item-level error records its diagnostic and the loop continues. It still counts
     as a job error in this phase; the parking phase reclassifies parks.
   - A batch with no progress no longer ends the pass. Its keys are attempted, so the
     loop terminates because `attempted` only grows.
   - Keep the deferred-budget and lock-busy outcomes and the link and Referenced By
     drains as they are.
3. **Claim compaction.** Replace the all-or-nothing `preserve_hints` with a retained
   set. `publication_doorbell.finish_hint_claim(claim, retain=...)` writes the hints
   that still need work into one fresh claim file (atomic write, then rename), then
   unlinks every consumed claim path. Hints that still need work are:
   - hinted keys that are still active and ready but were not attempted because of the
     deadline, lock busy, or a repository failure;
   - link hints whose link drain was deferred or failed.

   Hints for keys that were attempted and remain active are dropped, because the
   oldest-first order reaches them anyway. An interrupted pass still leaves its claims
   on disk, as it does today. The first pass after deploy merges and compacts the 153
   existing claim files.

4. **Tests** (`tests/agents_sync/`):
   - A hinted batch whose row hits a persistent item error, next to unhinted ready rows:
     every unhinted row drains in the same pass.
   - A repository-scope failure stops the pass and retains hints.
   - Claim files compact to at most one per project.
   - The deadline is honored.
   - The `agent_publication` job description in `src/sase/default_config.yml` stays
     accurate. Update it if the wording no longer fits.

> [!decision] planner_hints Also make `prompt_archive/publish.py` append a doorbell hint
> (`append_publication_hint` with the same fields the commit path uses) when it enqueues
> a planner prompt-archive row, and cover this in tests.

## Phase land_prompt: Recognize a session member's prompt archive for .land coder rows

In `src/sase/agents_sync/prompt_archive/deferred.py`:

- **Session-aware labels.** `_prompt_archive_present` accepts an archive whose AGENTS
  entry label, normalized to its agent session, equals the request's agent. Normalize
  with the same mapping `prompt_runs_by_request` uses (`sase_agent_ref_for_turn`), and
  handle both global (`owner.host.name`) and local spellings. Compare exact names after
  normalization: `X.land--plan` satisfies `X.land`, but `X.land.2--plan` and
  `X.landing--plan` do not.
- **Index once.** The current check parses every archive in every month for every
  request. Build a label index once per `restore_deferred_prompt_archives` call and pass
  it in.
- **No new archive files.** Do not create archives for `followup_prompt.md`. The plan
  snapshot that the planner archived is the session's prompt record. Archiving coder
  follow-ups would change sidecar conventions for about 1,400 historical runs.
- **Tests.**
  - A `.land` coder row with only `followup_prompt.md` and a `--plan`-labelled planner
    archive acknowledges.
  - A prefix-sharing session does not match.
  - A row with a real local raw prompt still restores.
  - A row with no archive for any member still fails closed.

## Phase protection: Consume the rules-versioned protection policy in sase

1. **Pin.** Move `sase-core-revision.txt` past the core phase's commit with
   `just ratchet-core-revision` (see `docs/rust_backend.md`).
2. **Adapter.** Add the new bindings to `src/sase/core/agent_publication_protection.py`.
   Update `tools/validate_sase_core_rs` if it lists bindings.
3. **Enforced lookups.**
   `_publication_protection_ledger.lookup_publication_content_record` uses the enforced
   lookup, so every caller listed in the Context section rescans a stale block. Under v2
   the rescan decides: still a hit means re-blocked with v2 evidence; clean means
   superseded.
4. **Supersede on clean.** `_record_scan_disposition` passes the clean disposition's
   `rules_version` and the timestamp to resolve. `_resolve_verified_clean` keeps the
   ledger file whenever `records` or `superseded` is non-empty.
5. **Doctor.** `src/sase/doctor/checks_agent_publication_protection.py` reports three
   groups: enforced blocks (WARN, as today), stale blocks awaiting revalidation, and the
   superseded count. It never prints content.
6. **Tests.**
   - A v1 `openai-key` block on a run's `meta.json` whose bytes contain a `…task-…`
     name. Drain it, and assert: rescanned clean, superseded with provenance, run
     published, request acknowledged.
   - The same for a blocked prompt archive.
   - A `RemoteSecretRejection` block stays enforced.
   - A `KnownValue` block stays enforced.
   - A stale block whose bytes still hit under v2 stays blocked and gains v2 evidence.
   - Update the existing fixtures that pin `rules_version: 1` only where the assertion
     is about the current version.

> [!decision] revalidation = command Leave lookups as they are. Read the CLI-rules
> reference memory through `/sase_memory_read`, then add a report-first `revalidate`
> subcommand next to the existing agent publication commands. It lists the stale
> scanner-hit blocks per project, and with `--write` it rescans each block's bytes from
> the published or private source and calls `publication_protection_revalidate`.
> Document it in `docs/reference/cli/agent.md`.

## Phase parking: Park protection-withheld requests instead of retrying them every pass

1. **Outbox schema 8.** Add optional fields to `AgentPublicationOutboxItem`:
   `parked_reason`, `parked_since`, `parked_checked_at`, and `parked_rules_version`.
   Read schemas 1–8 and write 8. The doctor outbox notice uses the constant.
2. **Park at settlement.** In `acknowledge_completed`, the withheld-run and
   withheld-hood path and the uncharged prompt-protection path park the row:
   - set the reason, `parked_checked_at = now`, and the current scanner rules version
     (from `attachment_scanner_rules_version`);
   - preserve `parked_since` across re-parks;
   - keep attempts uncharged.

   Any charged update, an acknowledgment, an enqueue that merges into the same logical
   key, and the explicit revive and retry-key paths clear the parked fields.

3. **Selection.** `publication_job._publication_is_ready` treats a parked row as not
   ready unless the Rust predicate says it is due. Use a one-hour recheck interval
   constant. A row that is parked and not due does not count toward the self-ring, so a
   project with only parked rows falls back to the job's five-minute `max_quiet`
   cadence. Explicit `sase agent sync` paths still include parked rows.
4. **Accounting.** The drain outcome reports `parked_keys` and `newly_parked_keys` (a
   row that was not parked before this settlement). Parked keys never count as job
   errors. Under `park_notice = once`, a pass with newly parked keys adds one error
   whose diagnostic names each `agent@revision` and its reason. A re-park after a
   recheck is silent.

   > [!decision] park_notice = silent Newly parked keys add no error. They appear only
   > in the heartbeat, the chop summary, and doctor.

5. **Visibility.**
   - The heartbeat `project_stats` gains `parked_count` and `oldest_parked_age_seconds`.
   - The chop summary (`sase_chop_agent_publication.py`) gains `parked`.
   - The doctor outbox check lists parked rows as WARN, with reason and age.
6. **Tests.**
   - Withheld rows park and are not reselected until due.
   - A rules-version change makes them due immediately.
   - An enqueue clears the park.
   - Parks are not errors, except one error per new park under `once`.
   - The self-ring is suppressed for parked-only projects.
   - Schema 7 documents still read.

## Phase verify: Install, verify live on athena, and hand off follow-ups

1. **Install if needed.** If the installed `sase_core_rs` reports scanner rules version
   1, or the installed sase cannot read outbox schema 8, propose the human-only
   reinstall through `/sase_gate` (decision `global-install-is-human-only`). Continue
   after it runs.
2. **Read-only live checks** after at least two publication passes:
   - bob-cli's five formerly stuck keys are acknowledged.
   - The 46 starved rows have drained; only fresh rows remain active.
   - Each project has at most one `hints.claimed.*` file.
   - The republished v1 `openai-key` blocks appear under `superseded`.
   - The heartbeat shows `parked_count`.
   - No new `publication / agent_publication` entries appear in
     `~/.sase/axe/error_digests/` after the install, except genuine ones.
   - A new `.land` coder row acknowledges instead of quarantining.
3. **Follow-ups.** Record `PROPOSED FOLLOW-UP:` notes on this phase's bead:
   - `sase-1ot` should re-scan its 27 historical hits under rules v2 before any rotation
     or history purge.
   - `sase-1r7` could add the agents_sync publication pull/push as a fifth
     credential-retry site.
   - Any residual failure class seen in the live checks.

> [!decision] historic_quarantines = recheck After the live checks pass, propose through
> `/sase_gate` a bounded revival of only the bob-cli rows whose `last_error` is the
> unresolved-prompt class: one
> `sase agent sync --retry-quarantined --retry-key … --retry-limit 1 -p bob-cli` per
> row, in small batches. Confirm that they acknowledge.

## Out of scope

- `sase-1qh.7`, which predates this bug.
- The 4 sase `high-entropy-assignment` blocks. They are rescanned under v2 and stay
  blocked if they still hit.
- Moving the outbox store to Rust.
