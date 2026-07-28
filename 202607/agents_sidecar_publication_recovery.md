---
tier: epic
title: Repair agents-sidecar publication blocked by legacy agent names
goal: 'Automatic `--agents` sidecar publication works end to end again: legacy agent
  names can never block publication, one failing outbox item can never block the others,
  the drain is fast enough to finish under its lock, the stuck backlog is published,
  and every `SASE_AGENT` commit link resolves to a real page.

  '
phases:
- id: identity-tolerance
  title: Total historical name classification in sase-core
  depends_on: []
  size: medium
  description: '''Phase 1: Total historical name classification in sase-core'' section:
    make hood/family classification of historical agent names total so a legacy name
    is classified rather than raising, while keeping strict validation for newly created
    names.'
- id: inventory-tolerance
  title: Non-fatal inventory classification and diagnostics
  depends_on:
  - identity-tolerance
  size: small
  description: '''Phase 2: Non-fatal inventory classification and diagnostics'' section:
    make hood selection and eligible-hood discovery total, and surface every skipped
    record as a diagnostic instead of aborting the build.'
- id: outbox-isolation
  title: Per-item publication isolation, honest errors, and quarantine
  depends_on:
  - inventory-tolerance
  size: medium
  description: '''Phase 3: Per-item publication isolation, honest errors, and quarantine''
    section: stop head-of-line blocking in the outbox drain, record per-item errors
    instead of broadcasting one error to every item, and quarantine repeatedly failing
    items.'
- id: publication-performance
  title: Single-inventory drain and lock-time reduction
  depends_on:
  - inventory-tolerance
  size: medium
  description: '''Phase 4: Single-inventory drain and lock-time reduction'' section:
    build the project inventory once per drain, deduplicate queued hoods, stop the
    permanently-stale name-registry rebuild, and reduce work done while holding the
    sidecar lock.'
- id: name-generation-guard
  title: Prevent new non-terminal family-role names
  depends_on:
  - identity-tolerance
  size: small
  description: '''Phase 5: Prevent new non-terminal family-role names'' section: stop
    family attachment from generating names whose role suffix is not terminal, so
    no new unclassifiable names enter the artifact store.'
- id: recovery-verification
  title: Backlog drain, link verification, and documentation
  depends_on:
  - outbox-isolation
  - publication-performance
  - name-generation-guard
  size: medium
  description: '''Phase 6: Backlog drain, link verification, and documentation'' section:
    drain the real stuck backlog, prove the first successful v2 publication, verify
    previously dead SASE_AGENT links resolve, and document outbox diagnostics and
    recovery.'
create_time: 2026-07-24 19:41:49
status: done
bead_id: sase-91
---

- **PROMPT:** [202607/prompts/agents_sidecar_publication_recovery.md](prompts/agents_sidecar_publication_recovery.md)
- **BEAD:** [sase-91](https://github.com/sase-org/sase--beads/blob/main/pages/sase-91/README.md)

# Plan: Repair agents-sidecar publication blocked by legacy agent names

## Goal

Automatic publication to the hidden `<project>--agents` sidecar repository, delivered by the `sase-8v` epic, has never
once succeeded in production. Every `sase commit` since that epic landed has queued a publication request that failed,
and the sidecar still contains only pre-epic v1 data. This plan makes publication resilient to historical agent names,
recovers the stuck backlog, and verifies the behavior the original epic promised.

## Observed symptom

- The `sase--agents` sidecar's most recent commit is `174c61f` (`chore(agents): sync from athena`, 2026-07-24 14:21:16
  -0400). It writes only v1 paths such as `agents/athena.jh--code/meta.json`.
- The sidecar's `manifest.json` still declares `schema_version: 1`. Its `users/` directory — the root of the v2
  owner-sharded layout — is **empty**, and no `snapshot.json` exists anywhere. All 338 entries under `agents/` use the
  v1 `athena.<name>` spelling; zero use the v2 `bbugyi200.athena.<name>` spelling.
- The durable outbox at `~/.sase/projects/gh_sase-org__sase/agents-publication-outbox.json` holds 30 queued items, the
  oldest created 2026-07-24 15:44:57 with 29 attempts. Every one of the 30 carries the identical `last_error`:

  ```text
  could not publish agent hood 'toobig-r': invalid family name '4x--epic.f-0':
  expected a solo name or one terminal '--<role>' suffix
  ```

- Every primary commit since the epic landed writes a correct-looking linked footer, for example
  `SASE_AGENT=[bbugyi200.athena.jq.f0.f1][2]` pointing at
  `https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.jq.f0.f1/README.md`. Because no v2 payload
  has ever been published, **every one of those links is a 404**.

The user-visible failure is therefore broader than "no recent sidecar commits": two headline deliverables of the
original epic — automatic publication on `sase commit`, and browsable agent and family pages linked from commit footers
— have never worked.

## Root cause

`ProjectHoodInventory.hood_runs()` in `src/sase/agents_sync/inventory_models.py` selects a hood's runs by calling
`agent_name_in_hood(run.local_name, hood)` for **every run in the whole project inventory**:

```python
def hood_runs(self, hood: str) -> tuple[InventoryRun, ...]:
    return tuple(
        run for run in self.runs if agent_name_in_hood(run.local_name, hood)
    )
```

`agent_name_in_hood` is a thin binding over the Rust `agent_name_in_hood` in the linked `sase-core` repository
(`crates/sase_core/src/agent_identity/identity.rs`). It begins with `parse_agent_family_name(name)?`, which delegates to
`parse_normalized_family_name`. That function counts `--` delimiters and **returns `Err(InvalidFamilyName)`** when there
is more than one, or when a single delimiter's role part contains a `.`:

```rust
1 => {
    let (base, role) = normalized.rsplit_once("--").expect("one match");
    if base.is_empty() || role.is_empty() || role.contains('.') {
        return Err(AgentIdentityError::InvalidFamilyName { ... });
    }
    ...
}
_ => Err(AgentIdentityError::InvalidFamilyName { ... }),
```

So a name whose `--<role>` suffix is not terminal raises instead of being classified. Because `hood_runs()` evaluates
the predicate against every run in the project, **one such record anywhere in the project makes every hood publication
raise** — including hoods that have nothing to do with the offending name.

A scan of all 4,566 `agent_meta.json` records across every project found exactly five malformed records, all in the
`sase` project and all predating the epic:

| Field          | Value               | Example artifact                             |
| -------------- | ------------------- | -------------------------------------------- |
| `name`         | `4x--epic.f-0`      | `artifacts/ace-run/202607/10/20260710170207` |
| `name`         | `fi--code.f0--plan` | `artifacts/ace-run/202607/19/20260719214644` |
| `agent_family` | `fi--code.f0`       | `artifacts/ace-run/202607/19/20260719214644` |
| `name`         | `fi--code.f0--code` | `artifacts/ace-run/202607/19/20260719215401` |

These names were produced by SASE itself: family attachment appends `--<role>` to a parent key that may already contain
`--` or acquire a later dotted segment. The `sase-8v` epic introduced a stricter identity model that retroactively
rejects names its own runtime had already written to disk. No test in that epic exercised a realistic artifact store
containing pre-existing names, so the defect was invisible to CI and appeared only against real history.

### Why nothing recovered

The epic designed the durable outbox so that a failed publish is retried by the next `sase commit`, by
`sase agent sync`, and by the Updates-tab `a` action. All of those paths are broken by the same defect:

1. **`eligible_hoods()` has the identical flaw.** It calls `agent_local_hood(run.local_name)` for every run, and
   `agent_local_hood` also starts with `parse_agent_family_name(name)?`. So `reconcile_agent_hoods` — the full-sync
   entry point behind `sase agent sync` and Updates-tab `a` — raises before it can drain anything. There is no working
   recovery path.
2. **Head-of-line blocking.** `_prepare_publications` in `src/sase/agents_sync/commit_publication.py` iterates queued
   requests in creation order and `return`s the first exception. It returns before `commit_agents_payload_if_dirty` is
   ever reached, so nothing is committed even for hoods that would have succeeded. The oldest item, hood `toobig-r`, has
   blocked the other 29 for every attempt.
3. **Error broadcast destroys the diagnostics.** `_record_failure` takes the single returned error string and writes it
   to _every_ queued item with `increment_attempts=True`. That is why all 30 items show a `toobig-r` error even though
   only the head item produced it, and why attempt counts are inflated for items that were never individually attempted.

### Contributing amplifiers

These did not cause the outage, but they made it slower, noisier, and harder to read. An earlier investigation observed
`agents sync lock is busy` on every item and a live `sase commit` holding `.git/sase-agents-sync.lock` for minutes at
high CPU; that was this amplifier layer masking the real error underneath.

- **Repeated whole-project inventory scans.** `publish_agent_hood` accepts an optional `inventory=` argument, but
  `_prepare_publications` never passes it. Each of the 30 queued items therefore rebuilds the complete project inventory
  from scratch, over ~4,566 artifact directories, while holding the exclusive sidecar lock. Queued items also repeat
  hoods (`sase-8v` appears several times), so the same hood is rebuilt and republished repeatedly within one drain.
- **A permanently stale name registry.** `_source_signature()` in `src/sase/agent/names/_registry_store.py` summarizes
  registry sources as `{count, max_mtime_ns}` over directories returned by `source_signature_paths()`, which include
  `sase_projects_dir()` and every project's `artifacts/` and workflow subdirectories. Any running agent writing into its
  artifact directory bumps `max_mtime_ns`, so the cached registry is effectively always stale and `load_name_registry()`
  performs a full rebuild on nearly every call. `_prepare_publications` calls `integrate_agent_imports_with_receipts`
  **inside** the lock, which drives those rebuilds under the lock.
- **A 2-second lock timeout.** `DEFAULT_SYNC_LOCK_TIMEOUT_SECONDS = 2.0` in `src/sase/agents_sync/git_sync.py`.
  Concurrent `sase commit` runs give up almost immediately, record `agents sync lock is busy`, and re-queue — enlarging
  the next drain and worsening contention.

## Gaps against the original epic request

The original request behind `sase-8v` asked for several things that are implemented in code but not observably
delivered. This plan closes them:

- "We should start pushing all necessary new agent data automatically when an agent runs the `sase commit` command." —
  implemented, but has never succeeded once.
- "We should start also pushing a beautiful agent family overview markdown file whenever a new agent family is pushed.
  This will be linked to from sase commits." — the footer link is generated correctly, but no family or agent page has
  ever been published, so every link is dead.
- Publishing the committing agent's complete top-level hood recursively — never exercised against real history.

Confirmed **not** at issue, and out of scope: locally owned names are correctly bare on disk (the outbox records
`local_agent: "toobig-r.split_file..."` against `global_agent: "bbugyi200.athena.toobig-r..."`), `SASE_MACHINE` is
absent from new commits, and the `id.username` / `id.machine_name` identity resolves. The `gh_bobs-org__bob-cli` project
has an empty outbox and is unaffected, because it contains no malformed names.

## Design contracts

### Strict on write, total on read

Name validation must be strict where a name is _created_ and total where history is _read_. A record already on disk is
a fact; classifying it must never raise. Publication reads history, so every classification helper it uses must return
an answer for any input.

### Classify, do not skip

Skipping unclassifiable records would silently under-publish the hood, quietly violating the epic's acceptance criterion
that a commit publishes its complete top-level hood. The preferred outcome is a _correct classification_ for these
legacy names; skipping is reserved for records that are genuinely unsafe (empty segments, path-unsafe characters, failed
containment checks) and every skip must appear in `ProjectHoodInventory.diagnostics` rather than being silent.

### Recommended canonical rule

Treat `--<role>` as a family-role marker **only in the final dot-separated segment**. A `--` inside any earlier segment
is an opaque part of that segment's text, not a family marker. Under this rule:

| Name                | Terminal role | Family base    | Top-level hood |
| ------------------- | ------------- | -------------- | -------------- |
| `foo.bar.baz--code` | `code`        | `foo.bar.baz`  | `foo`          |
| `4x--epic.f-0`      | none (solo)   | `4x--epic.f-0` | `4x`           |
| `fi--code.f0--code` | `code`        | `fi--code.f0`  | `fi`           |

Top-level hood extraction takes the first dot-separated segment and strips any `--<...>` remainder, so `4x--epic.f-0`
and `fi--code.f0--code` land in hoods `4x` and `fi` respectively. This preserves current behavior for every well-formed
name while giving legacy names a coherent home. Phase 1 owns this decision: validate the rule against the existing Rust
test matrix and adjust it if a concrete case contradicts it, but do not leave classification partial.

## Safety and scope boundaries

- Do not edit SASE memory files, generated agent-instruction files, or provider shims.
- Do not mass-rename or rewrite historical artifact directories, chat files, or primary commits. The five malformed
  records stay exactly as they are on disk; the code learns to read them.
- Do not delete or rewrite existing v1 sidecar payload. V1 and v2 coexist, as the original epic settled.
- Shared identity, hood, and family classification rules belong in `sase-core`. Python must call the binding rather than
  reimplementing parsing. Open `sase-core` through the repository-access workflow and land it according to its own
  instructions; rebuild the binding before dependent phases run.
- Do not manually edit release-plz-managed versions in the core repository.
- Treat the sidecar and outbox as durable state: a partial failure must never lose a queued request or create a
  duplicate primary commit.

## Phase 1: Total historical name classification in sase-core

**ID:** `identity-tolerance` **Depends on:** none **Size:** medium

In the linked `sase-core` repository, make classification of historical names total.

- Settle the canonical rule described above in `crates/sase_core/src/agent_identity/identity.rs`. Terminal-segment role
  parsing replaces the current whole-name `--` count in `parse_normalized_family_name` and
  `parse_normalized_family_name_unchecked`.
- Make `agent_name_in_hood` total with respect to its _candidate_ name: an unclassifiable candidate returns `Ok(false)`,
  never an error. Keep returning an error for an invalid `hood` argument, which is a caller bug rather than historical
  data.
- Make `agent_local_hood` total: return the best-effort top-level hood for any non-empty, path-safe name.
- Keep a strict entry point for validating newly created names, and keep the existing `InvalidFamilyName` variant for
  that path. Do not weaken validation used at name creation.
- Apply the same treatment to `agent_name_ancestors` and any other helper publication reaches through `hood_runs`,
  `eligible_hoods`, or container/relationship construction, so no read path can raise on a historical name.
- Expose the behavior through the existing pyo3 bindings without changing call signatures where possible, so Python
  callers need no churn.
- Test the exact real-world names `4x--epic.f-0`, `fi--code.f0`, `fi--code.f0--plan`, and `fi--code.f0--code` alongside
  the existing well-formed matrix; assert hood membership, hood extraction, ancestors, and idempotence. Include a
  regression asserting `agent_name_in_hood` never raises for any name in a representative corpus, and that `foo` still
  does not match `foobar`.

Run core formatting, clippy, and workspace tests. Land and rebuild the binding before dependent phases.

## Phase 2: Non-fatal inventory classification and diagnostics

**ID:** `inventory-tolerance` **Depends on:** `identity-tolerance` **Size:** small

Make inventory selection in `src/sase/agents_sync/` unable to abort on one bad record.

- With Phase 1 in place, `hood_runs()` and `eligible_hoods()` in `src/sase/agents_sync/inventory_models.py` become
  total. Add a defensive boundary anyway so a future classification error degrades to "record excluded, diagnostic
  recorded" rather than a failed publish.
- Record every excluded record in `ProjectHoodInventory.diagnostics` with its artifact path and reason.
  `build_project_hood_inventory` already accumulates diagnostics for per-record parse failures; selection-time
  exclusions must join the same channel.
- Ensure diagnostics reach the publication outcome so they are visible in `V2PublicationCounts.diagnostics` and in
  CLI/status output, rather than being discarded.
- Test with a fixture artifact store containing the four real malformed names: assert that building the inventory
  succeeds, that hoods `4x` and `fi` are discoverable, that unrelated hoods publish normally, and that no exclusion is
  silent.

Run `just install` before `just check`.

## Phase 3: Per-item publication isolation, honest errors, and quarantine

**ID:** `outbox-isolation` **Depends on:** `inventory-tolerance` **Size:** medium

Make one bad queued item unable to block the rest, in `src/sase/agents_sync/commit_publication.py` and
`src/sase/agents_sync/publication_outbox.py`.

- Change `_prepare_publications` from return-on-first-error to skip-and-continue. Record the failure against that
  request only, continue preparing the remaining requests, and proceed to `commit_agents_payload_if_dirty` so
  successfully prepared hoods are still committed and pushed.
- Acknowledge only the requests that actually succeeded. A request that failed preparation must remain queued with its
  own error and its own incremented attempt count.
- Fix the error broadcast in `_record_failure`. A whole-drain failure (lock busy, clone failure, pull/push failure)
  legitimately applies to every item; a single request's preparation failure must not. Distinguish the two so
  `last_error` is always attributable.
- Add bounded retry semantics. After a configurable attempt threshold, move an item to a quarantined state that is
  reported in status and CLI diagnostics and no longer blocks or slows ordinary drains. Quarantine must be visible and
  manually clearable, never a silent drop.
- Preserve the epic's existing guarantees: a failed sidecar publish never fails the primary commit, never re-dispatches
  it, and never creates a duplicate sidecar record.
- Test a mixed queue where one item fails preparation and others succeed: assert the good hoods are committed and
  pushed, the bad item stays queued with its own error, other items' attempt counts are untouched, and a repeated drain
  is idempotent. Test the quarantine threshold and that a drain with every item quarantined still completes cleanly.

Run `just install` before `just check`.

## Phase 4: Single-inventory drain and lock-time reduction

**ID:** `publication-performance` **Depends on:** `inventory-tolerance` **Size:** medium

Make a drain of a large backlog finish comfortably inside its lock.

- Build the project inventory once per drain and pass it through the existing `inventory=` parameter of
  `publish_agent_hood`, instead of letting each queued item rebuild it.
- Deduplicate queued requests by hood before publishing so a hood appearing several times in the backlog is built and
  rendered once, while every underlying request is still acknowledged.
- Fix the permanently stale registry signature in `src/sase/agent/names/_registry_store.py`. The `{count, max_mtime_ns}`
  summary over live artifact directories is invalidated by any running agent, forcing a full rebuild on nearly every
  `load_name_registry()` call. Base staleness on something that does not change merely because an unrelated agent is
  writing output, and cache the loaded registry across a single import/publication loop.
- Move `integrate_agent_imports_with_receipts` out of the exclusive lock in `_prepare_publications`, or reduce what it
  does while holding it, so import work is not serialized behind publication.
- Raise `DEFAULT_SYNC_LOCK_TIMEOUT_SECONDS` in `src/sase/agents_sync/git_sync.py` so a concurrent `sase commit` waits
  briefly rather than immediately re-queueing. Keep it bounded and keep the typed retry/skip contract: lock contention
  must never become an unbounded UI wait.
- Add a performance regression test with a realistic multi-item backlog across several hoods over a large synthetic
  artifact store, asserting the inventory is built once and that lock hold time stays within a defined bound.

Run `just install` before `just check`.

## Phase 5: Prevent new non-terminal family-role names

**ID:** `name-generation-guard` **Depends on:** `identity-tolerance` **Size:** small

Stop the runtime from writing more names it cannot classify strictly.

- `src/sase/agent/_family_attach_resolution.py` composes a member name as `<parent_key><AGENT_FAMILY_SEPARATOR><role>`
  with no guard against a `parent_key` that already contains `--`. `validate_user_agent_name` in
  `src/sase/agent/launch_validation.py` rejects a user-typed `--`, but generated names bypass it. Add validation at the
  generation site.
- When attachment would produce a non-terminal role suffix, either derive a valid name or fail with an actionable error
  naming the parent and the requested role. Do not silently rename an agent the user explicitly asked for without saying
  so.
- Apply the same guard to clan membership and any other site that appends a dotted segment to a name that already
  carries a terminal role suffix, since that is how `4x--epic` became `4x--epic.f-0`.
- Test attaching a family member to a parent that is already a family member, attaching to a parent carrying a clan
  generation suffix, and the ordinary well-formed cases, asserting no regression to existing family and clan naming.

Run `just install` before `just check`.

## Phase 6: Backlog drain, link verification, and documentation

**ID:** `recovery-verification` **Depends on:** `outbox-isolation`, `publication-performance`, `name-generation-guard`
**Size:** medium

Prove the behavior against the real stuck state rather than only against fixtures.

- Drain the real backlog in `~/.sase/projects/gh_sase-org__sase/agents-publication-outbox.json` through the ordinary
  supported path. Confirm the v2 layout finally materializes: a populated `users/<username>/machines/<machine>/` tree
  with `manifest.json` and per-hood `snapshot.json`, plus `agents/<global-name>/` and `families/<global-family>.md`
  entries using the `bbugyi200.athena.*` spelling.
- Confirm the existing v1 payload is left intact and that the v1 `manifest.json` is not corrupted by the v2 write.
- Verify link recovery: take `SASE_AGENT` footers from primary commits made while publication was broken — for example
  `bbugyi200.athena.jq.f0.f1` and `bbugyi200.athena.sase-8v.land` — and confirm the referenced paths now exist in the
  sidecar. Report any that remain unresolved and why.
- Confirm the legacy hoods `4x` and `fi` publish with their members classified into the right hoods, and that
  publication diagnostics list any record still excluded.
- Verify a fresh `sase commit` publishes its hood promptly and leaves an empty, non-quarantined outbox, and that
  `sase agent sync` and the Updates-tab `a` action both complete against real data.
- Update `docs/agents_sidecar.md` with outbox diagnostics, quarantine semantics, and the manual recovery procedure, and
  document the legacy-name tolerance contract so a future strictness change does not reintroduce this outage.
- Record any deferred cleanup as a bead rather than weakening acceptance.

Run `just install` before `just check`, and run `just test-visual` if any rendered output changes.

## Acceptance criteria

- No agent name already present in the artifact store can cause a publication or full sync to fail. Classification of
  historical names is total; strict validation still applies to newly created names.
- A single failing outbox item never prevents other queued hoods from being committed and pushed. Each item carries its
  own `last_error` and its own attempt count, and repeatedly failing items are visibly quarantined rather than retried
  forever or silently dropped.
- One drain builds the project inventory once, publishes each distinct hood once, and completes within a bounded lock
  hold; a concurrent `sase commit` no longer fails immediately with `agents sync lock is busy`.
- The `sase--agents` sidecar contains a populated v2 owner-sharded layout for `bbugyi200.athena`, alongside the
  untouched v1 payload.
- `SASE_AGENT` links in primary commits resolve to real agent or family pages, including commits made while publication
  was broken.
- Family attachment can no longer generate a name whose `--<role>` suffix is not terminal.
- `sase commit`, `sase agent sync`, and the Updates-tab `a` action each leave the outbox empty of non-quarantined items.
- Every changed repository passes its required checks, no SASE memory or generated instruction file is edited, and no
  historical artifact directory or primary commit is rewritten.
