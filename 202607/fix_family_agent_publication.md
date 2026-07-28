---
tier: epic
title: Repair agent-hood publication for family members and un-wedge the agents sidecar
goal: 'Every `sase commit` links to an agent page that actually exists in the agents
  sidecar, family-member commits publish their hood instead of quarantining, a dirty
  sidecar working tree can no longer wedge publication indefinitely, and quarantined
  publication requests are surfaced instead of failing silently.

  '
phases:
- id: identity
  title: Resolve the committing run's own agent name
  depends_on: []
  size: medium
  description: 'identity: make commit-time agent attribution resolve the running member''s
    name rather than the stale lane/family name held in the process environment, so
    footer links and publication requests both name a real run.

    '
- id: publish
  title: Make hood publication tolerate container-named requests
  depends_on: []
  size: small
  description: 'publish: resolve an outbox request that names a pure family/clan container
    to a concrete run in that hood so publication succeeds instead of raising "absent
    from project inventory" and quarantining.

    '
- id: sidecar_tx
  title: Self-healing agents sidecar transactions
  depends_on: []
  size: medium
  description: 'sidecar_tx: guarantee a clean sidecar working tree before every rebase
    pull, clear stale index locks, and revert uncommitted payload writes on every
    early-return and exception path so one failed publish cannot wedge all later ones.

    '
- id: visibility
  title: Surface quarantined and stalled publications
  depends_on:
  - publish
  - sidecar_tx
  size: small
  description: 'visibility: add a doctor check and stronger reporting for the publication
    outbox so quarantined or long-stalled requests are noticed instead of accumulating
    unseen for days.

    '
- id: backfill
  title: Recover the backlog and verify end to end
  depends_on:
  - identity
  - publish
  - sidecar_tx
  size: small
  description: 'backfill: drain the existing outbox, re-publish the hoods that were
    never written, and report which historical commit links remain permanently dead.

    '
create_time: 2026-07-28 07:43:27
status: done
bead_id: sase-ad
---

- **PROMPT:** [202607/prompts/fix_family_agent_publication.md](prompts/fix_family_agent_publication.md)

# Plan: Repair agent-hood publication for family members and un-wedge the agents sidecar

## Symptom

Commit `3bd59cdda` in the `sase` repository carries the footer:

```
SASE_AGENT=[bbugyi200.athena.ms][2]

[2]: https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.ms/README.md
```

That URL 404s. The page was never published to the `sase--agents` sidecar.

Two independent defects each break this commit. Both must be fixed; fixing either alone leaves the link dead.

## Root cause 1: family members publish and link under the family container name

The commit was authored by the run whose artifacts live in
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/28/20260728065539`, whose `agent_meta.json` records
`"name": "ms--code"`, `"agent_family": "ms"`, `"role_suffix": "--code"`. The footer nevertheless names the bare lane
name `ms`.

`ms` is not a run. `~/.sase/agent_name_registry.json` records it as `"container_kind": "family"`, a pure container
reserving the lane for members `ms--plan` and `ms--code`. Consequences:

1. **Dead link.** `resolve_agent_commit_tag` calls the Rust `agent_link_target` (`sase-core`,
   `crates/sase_core/src/agent_identity/identity.rs`). Given a member name it correctly returns
   `families/<global>.md#member-<role>`; given the bare container name it falls into the plain-agent branch and returns
   `agents/<global>/README.md`, which is only ever written for a hood that contains a run of that exact name. The
   intended member behavior is already covered by `tests/agents_sync/test_links.py` — that test passes because it feeds
   `foo.bar--code` directly, so the defect is entirely upstream of the link builder.
2. **Publication is rejected.** `publish_committed_agent_hood` (`src/sase/agents_sync/commit_publication.py`) enqueues
   an outbox item with `local_agent="ms"`, and `publish_agent_hood` (`src/sase/agents_sync/publication.py`) raises
   `committing agent 'ms' is absent from project inventory` because no inventory run has that local name. After
   `DEFAULT_PUBLICATION_MAX_ATTEMPTS` (3) the request is quarantined and the hood is never published at all — not the
   family page, not either member page.

### How the wrong name reaches the commit

`CommitWorkflow` records `publication_agent=resolve_local_agent_name()` (`src/sase/workflows/commit/workflow.py`), and
`resolve_local_agent_name` (`src/sase/workflows/commit/runtime_tags.py`) prefers `$SASE_AGENT_NAME` over
`$SASE_ARTIFACTS_DIR/agent_meta.json`. The same value feeds both the footer tag and the outbox request, which is why the
two symptoms always agree.

Plan-chain family members share one OS process with their predecessor. Comparing `pid` across `agent_meta.json` files
for one day of runs shows the pairs are co-resident, while unrelated agents each have their own pid:

| artifacts dir    | name       | pid     |
| ---------------- | ---------- | ------- |
| `20260728064646` | `mq--plan` | 3964901 |
| `20260728065608` | `mq--code` | 3964901 |
| `20260728065146` | `ms--plan` | 3994876 |
| `20260728065539` | `ms--code` | 3994876 |
| `20260728072005` | `mt`       | 362977  |

`SASE_AGENT_NAME` is written once per process by `_claim_agent_identity`
(`src/sase/axe/run_agent_directive_identity.py`) when the lane first claims the bare name. Family promotion
(`src/sase/agent/_family_promotion.py`) then rewrites `agent_meta.json` (`name` becomes `<lane>--<suffix>`,
`workflow_name` becomes the lane) and the registry, but nothing refreshes the live process environment, and the
in-process successor member never re-claims. So for the whole lifetime of the process the environment keeps saying `ms`
while each run's `agent_meta.json` correctly says `ms--plan` or `ms--code`.

### Evidence that this is deterministic, not intermittent

Across every `ace-run` artifact directory under `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607` that has
both `agent_meta.json` and `commit_result.json`:

- 190 of 190 non-family runs tag their own name — correct.
- 56 of 56 family-member runs tag the family container name — wrong, 100% of the time.

The same signature appears in the durable outbox. All 29 quarantined requests in
`~/.sase/projects/gh_sase-org__sase/agents-publication-outbox.json` fail with
`committing agent '<lane>' is absent from project inventory`, and every one of those names is a family container (`k4`,
`kc`, `ka`, `kq`, `ks`, `km.f0`, `kt`, `l0`, `ku.f1`, `lg`, `lo`, `lr`, `lt`, `lz`, `ly`, `m5`, `m8`, `m9`,
`sase-99.land.f2/f3`, `sase-9r.7`, `sase-9s.land`, `sase-9v.land`, `sase-9x.land`, `sase-9y.land`, `sase-9z.5`). None of
the corresponding `agents/bbugyi200.athena.<lane>` directories exist in the sidecar.

Some family hoods do have pages anyway (`sase-a8.9`, `sase-a9.5`) because `_publish_hoods` publishes a whole hood at
once, so a sibling with a correct name in the same hood incidentally publishes them. Short auto-named lanes like `ms`
have no such sibling, so they are lost entirely.

## Root cause 2: a dirty sidecar working tree wedges all publication permanently

Both publication paths begin with `git pull --rebase --no-autostash` (`pull_agents_rebase` in
`src/sase/agents_sync/git_sync_ops.py`), and neither guarantees the working tree is clean first. Several paths leave
payload writes uncommitted in that long-lived clone:

- `_publish_queued_locked` (`src/sase/agents_sync/commit_publication.py`) calls `integrate_agent_imports_with_receipts`,
  and `_prepare_publications` calls `apply_payload_atomic` for each hood that publishes successfully — then returns
  early at `if not prepared:` without ever reaching `commit_agents_payload_if_dirty`.
- `sync_project_locked` (`src/sase/agents_sync/git_sync_transaction.py`) returns on any exception from the
  integrate/export pass, after that pass may already have written into the tree.
- Any process killed between `apply_payload_atomic` and the commit does the same, and additionally can leave
  `.git/index.lock` behind.

Once the tree is dirty, every later attempt fails at the very first step with
`cannot pull with rebase: You have unstaged changes`, forever. There is no repair path — notably, `sase repo open` does
clear a stale `index.lock`, but agents-sync has no equivalent recovery.

This is what actually blocked commit `3bd59cdda`. The outbox shows the wedge beginning at 2026-07-28 02:07:06 with the
`toobig-0k.split_file.…` request and then swallowing every subsequent request:

```
02:07:06  toobig-0k.split_file.tests.ac…  att=8  git pull --rebase failed: … You have unstaged changes.
06:01:18  sase-a5.6                       att=7  (same)
06:13:42  sase-a8.8                       att=6  (same)
06:14:58  sase-a9.5                       att=5  (same)
06:28:34  sase-a8.9                       att=4  (same)
07:10:30  mq                              att=3  (same)
07:12:45  ms                              att=2  (same)
07:18:00  toobig-0m.split_file.…v2_io.0   att=1  (same)
```

The sidecar's `origin/main` had not moved since 2026-07-27 17:41 as a result. The `ms--code` agent's own summary in its
`done.json` records the deferral: _"An auxiliary agent-hood publication was deferred by a busy lock and will retry
automatically."_

Note for whoever picks this up: **this state has already been cleared.** Diagnosing it required `sase repo open agents`,
which cleans the workspace and removes stale index locks, so the clone is clean and the `.git/index.lock` is gone. The
outbox entries above are still present and still un-drained, and they are the durable evidence — do not expect to
reproduce the dirty tree by inspection.

## Root cause 3 (contributing): the sidecar clone is shared with `sase repo open`

`resolve_sync_targets` points `ProjectTarget.sidecar_path` at the same `~/.sase/projects/<key>/repos/agents` checkout
that `sase repo open agents` hands to agents. Agents-sync parks uncommitted payload in that tree between write and
commit, while `sase repo open` hard-cleans and resets it. Either side can destroy the other's work. This does not need
to be re-architected here, but the fix in `sidecar_tx` must be correct in the presence of an outside actor resetting the
tree, and the risk should be recorded.

## Root cause 4 (contributing): the failure was invisible

29 requests sat quarantined for three days while `sase commit` kept minting links to pages that would never exist.
`sase commit` prints a single warning and proceeds; the quarantine diagnostics only reach
`~/.sase/agents_sync/status_snapshot.json` and the ACE indicator; there is no `sase doctor` check for the publication
outbox at all.

---

## Phase: Resolve the committing run's own agent name

Make commit-time attribution name the run that is actually committing.

- Change agent-name resolution used for commit attribution so the running run's `agent_meta.json` name wins over a stale
  process-level `SASE_AGENT_NAME`, or refresh `SASE_AGENT_NAME` whenever a family promotion or in-process successor
  changes the current run's name. Pin down which of the two is correct before editing — the deciding question is whether
  any other consumer of `SASE_AGENT_NAME` depends on it naming the lane rather than the run. Audit the other consumers
  (`src/sase/agent/identity.py`, `src/sase/memory/proposals/identity.py`, `src/sase/main/var_handler.py`,
  `src/sase/agent/names/_migration.py`) before choosing.
- Do not reimplement name semantics in Python. `agent_link_target`, `agent_local_hood`, and `parse_agent_family_name`
  already live in the Rust core and are correct; this phase only changes which name is handed to them.
- Regression tests must cover a family member end to end: a run whose `agent_meta.json` says `ms--code` while the
  environment still says `ms` must produce the label `bbugyi200.athena.ms--code` and the link
  `families/bbugyi200.athena.ms.md#member-code`, and must enqueue a publication request naming `ms--code`.
- Add a test that a non-family run is unaffected.

Verification beyond the unit tests: after this phase, a fresh plan-chain `--code` agent's commit footer must name
`<lane>--code`.

## Phase: Make hood publication tolerate container-named requests

Defence in depth, so a container name can never again silently discard a whole hood. This phase is independent of
`identity` and must not depend on it.

- In `publish_committed_agent_hood` / `publish_agent_hood`, when the requested local name has no matching run but names
  a hood that does contain runs, publish that hood rather than raising. `agent_local_hood("ms--code")` and
  `agent_local_hood("ms")` both return `ms`, so the hood being requested is unambiguous.
- Keep a genuine error for a name whose hood has no publishable runs at all; that case must still be reported, not
  silently succeed.
- Preserve outbox idempotency: the `logical_key` is `(global_agent, primary_revision)`, so re-resolving the name must
  not orphan or duplicate an in-flight request.
- Tests: a request naming a pure family container publishes the hood (family page plus every member page); a request
  naming a hood with no runs still errors.

## Phase: Self-healing agents sidecar transactions

Make a failed or interrupted publication unable to poison every subsequent one.

- Before `pull_agents_rebase` in both `sync_project_locked` and `_publish_queued_locked`, bring the sidecar working tree
  to a clean state under the existing sync lock: clear a stale `.git/index.lock`, and discard uncommitted changes to the
  payload paths. Discarding is safe because the payload is fully regenerated from local artifacts on every run; state
  the reasoning in a comment so a future reader does not "fix" it back into a stash.
- Guarantee cleanup on the failure paths that currently leak: the `if not prepared:` early return in
  `_publish_queued_locked`, and the exception return from the integrate/export pass in `sync_project_locked`. Prefer a
  single `try/finally` or context manager over patching each return site.
- Model the shared-clone hazard from root cause 3: an outside `sase repo open` may reset the tree at any moment, so the
  recovery must be idempotent and must not assume its own prior writes survived.
- Tests: a sidecar clone left dirty by a previous failed publish still publishes on the next attempt; a stale
  `index.lock` does not block a publish; a publish that fails after `apply_payload_atomic` leaves the tree clean.

## Phase: Surface quarantined and stalled publications

- Add a `sase doctor` check that reads each project's `agents-publication-outbox.json` and reports quarantined requests
  and requests whose `attempts` or age exceed a threshold, with the remediation command
  (`sase agent sync --retry-quarantined`) in the message.
- Strengthen what `sase commit` reports when publication is deferred: the current single warning reads as routine. It
  should distinguish "queued, will retry" from "this project already has quarantined requests", because the second means
  the link just written to the commit is likely to stay dead.
- Tests: doctor reports a clean outbox as healthy, and reports quarantined and stalled requests as problems.

## Phase: Recover the backlog and verify end to end

Operational recovery, run after the three code phases land. This phase pushes to the real `sase--agents` remote — do not
run it until the code phases are merged, and report exactly what was pushed.

- Confirm the agents sidecar working tree is clean and the outbox still holds the 37 recorded requests before starting.
- Drain the outbox with `sase agent sync --retry-quarantined` for the `sase` project, then confirm that
  `agents/bbugyi200.athena.ms--code/README.md` and `families/bbugyi200.athena.ms.md` exist in the sidecar, along with
  the other previously quarantined hoods.
- Re-run for every other project that has quarantined requests; the status snapshot at
  `~/.sase/agents_sync/status_snapshot.json` lists them.
- Report the residue honestly: commit `3bd59cdda` and the other historical commits already carry immutable footers
  pointing at `agents/<user>.<machine>.<lane>/README.md`. Publishing the hood correctly creates `families/…md` and the
  member pages but does **not** make those old URLs resolve. Do not rewrite published history to fix them. State plainly
  in the phase report how many commits keep dead links, and leave the decision about any cosmetic redirect (for example,
  whether the sidecar should emit a stub page for a family container) to the user rather than inventing one.

## Out of scope

- Rewriting or force-pushing any already-published commit in `sase` or `sase--agents` to repair historical footers.
- Re-architecting the shared sidecar clone so that `sase repo open` and agents-sync no longer collide. Record it; do not
  attempt it here.
- Migrating `sase/agents_sync` into the Rust core.

## Risks

- The `identity` phase changes a value several subsystems read. The audit of `SASE_AGENT_NAME` consumers is the gating
  step, not an optional courtesy.
- Discarding uncommitted sidecar changes in `sidecar_tx` is only safe because the payload is regenerated from local
  artifacts. If any part of the payload ever becomes non-regenerable, that assumption breaks.
- The `backfill` phase pushes a large batch of previously unpublished hoods in one sync. Expect a sizeable sidecar
  commit and confirm the push succeeded rather than assuming it.
