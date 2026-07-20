---
tier: tale
title: Preserve dismissed clan completion for waits and forks
goal: 'Agents waiting on or forking from a completed clan continue automatically after
  successful clan members are dismissed, while failed, queued, missing, or mismatched
  archived members remain blocking.

  '
create_time: 2026-07-20 10:53:56
status: wip
prompt: 202607/prompts/dismissed_clan_waits.md
---

# Plan: Preserve dismissed clan completion for waits and forks

## Context and diagnosis

The `sase-7z.f3` launch exposed a lifecycle disagreement rather than a delayed refresh. ACE showed the relevant epic
members as complete, but the wait dependency index reported the clan unresolved. The persisted artifacts explain the
split:

- Successful phase members `sase-7z.1` through `sase-7z.6` were dismissed before the later `#fork:sase-7z` launch.
  Dismissal correctly archived a top-level bundle with status `DONE` and then removed `done.json` and workflow markers
  from each artifact directory.
- The wait index still scans each surviving `agent_meta.json`, but it treats a missing `done.json` as unfinished and
  never consults the dismissed-agent archive. The newest `sase-7z` clan generation therefore contained six false
  unfinished candidates plus the current successful members, so `WaitDependencyIndex.is_resolved("sase-7z")` remained
  false indefinitely.
- This is not only a wait-marker defect. A manual retry reached `agent_chat_from_name` and failed with
  `Clan 'sase-7z' is not complete: 3/9 members done` for the same reason. Even if the wait index alone were fixed, clan
  fork resolution would still reject the parent, and transcript lookup currently reads only `done.json` rather than the
  archived bundle's `response_path`.
- Existing cleanup-time memoization protects waiters that already exist when a successful artifact is dismissed. It
  cannot protect a future waiter, such as this `#fork` launch, created after cleanup. The durable archive must therefore
  participate in normal completion lookup.

The dismissed-bundle archive and its indexed suffix queries already provide the authoritative preserved lifecycle
record, including status, identity, artifact suffix, bundle path, and transcript metadata. The fix should consume that
surface instead of retaining duplicate markers or changing dismissal semantics. No new Rust-core behavior is required;
this is a Python adapter and aggregation gap around existing backend archive data.

## Archived completion contract

Introduce one non-TUI archive lookup seam that batches requested artifact suffixes and returns only exact, top-level
dismissed records. Match records against the artifact's timestamp, project/ChangeSpec identity, and agent name so
same-second artifacts in different projects, workflow children, stale bundles, and unrelated names cannot supply
completion accidentally. Read the bundle payload only when fields unavailable from the summary, such as `response_path`,
are requested.

Translate archived display states into the same canonical lifecycle semantics used by live `done.json` records.
Successful terminal states must behave like their original successful outcomes; `PLAN REJECTED` retains its accepted
identity-terminal meaning; stopped repeat slots retain stopped semantics; and failed/killed records remain failures that
cannot satisfy a successful wait. Missing, corrupt, revived, ambiguous, or identity-mismatched archive data must fail
closed and leave the dependency unresolved. Keep this translation in one shared helper so wait aggregation, name/group
lookup, and fork transcript resolution cannot drift again.

Batch archive reads per artifact scan or clan/family lookup rather than issuing one archive query per member. Preserve
current live-marker precedence: `done.json` remains authoritative when present, and the archive is only a fallback after
dismissal removed the marker.

## Wait dependency integration

Teach `src/sase/core/wait_dependency_resolution/` to accept the archived terminal projection while building candidates.
A successfully dismissed artifact should populate `is_resolved`, `is_done`, identity-success, and failure flags exactly
as its original terminal marker did. An artifact with neither a live terminal marker nor a matching archive record stays
unfinished, including queued members that carry `waiting.json`.

Apply the same archive-aware construction to every entry path, not only the periodic `wait_checks` chop:

- single-project initial checks in `build_wait_dependency_index`, used by a runner before it starts polling;
- the all-project `wait_checks` scan that writes `ready.json`;
- all-project fork/tribe index construction and cleanup-time waiter checks.

This makes immediate checks, periodic healing, clan waits, tribe-backed clan waits, and identity waits agree on the same
terminal history. Preserve newest generation selection, queued-member barriers, successful-only resolution, and the
existing exact-identity memoization behavior.

## Clan and fork integration

Make the agent family/clan lookup layer use the same archived outcome fallback when `done.json` is absent.
`find_agent_clan`, clan completeness diagnostics, resume selection, and related helpers should therefore classify
dismissed members consistently with the wait index rather than reporting them as live or unfinished.

Update `src/sase/scripts/agent_chat_from_name.py` so a completed clan or clan-backed tribe fork resolves each member's
transcript from the live `done.json` first and then from the exact matching dismissed bundle. Continue to include every
successful member of the selected generation in launch order, validate that each transcript is readable, reject
incomplete/failed clans, and retain duplicate-transcript detection. Do not silently omit a dismissed member: a
whole-clan fork must either provide the complete successful context or fail with an actionable member-specific
diagnostic.

Keep ordinary dismissal behavior unchanged. Successful agents may still have their loader markers removed, dismissed
rows remain hidden, archives remain revivable, and no synthetic `done.json` files should be recreated merely to release
a wait.

## Regression coverage

Add focused fixtures that persist realistic top-level dismissed bundles and then remove live terminal markers, matching
the production cleanup order. Cover at least these cases:

1. A clan containing both current successful members and earlier successfully dismissed members resolves through the
   shared index, causes `wait_checks` to write `ready.json`, and no longer leaves a future waiter parked.
2. A queued/live member with no terminal archive still blocks, preserving the recent queued-clan regression fix.
3. A dismissed failed member, corrupt or absent archive, revived record, or project/name mismatch does not resolve the
   clan or identity dependency.
4. Same suffixes across projects cannot cross-contaminate completion.
5. `#fork:<clan>` accepts the mixed live/dismissed completed clan, emits all member transcript paths in launch order,
   and reads dismissed transcripts from their bundles.
6. Clan fork diagnostics remain correct for genuinely incomplete or failed members, and unreadable archived transcripts
   fail explicitly.
7. Family, tribe-backed clan, and direct named/identity lookups retain their existing success/failure and
   newest-generation behavior where the shared helper affects them.

Run the focused wait, clan, tribe, cleanup, archive, and fork-source test files first. Then run `just install` to
refresh this ephemeral workspace and execute the repository-required `just check`. Finally, rerun the focused production
shape as a read-only smoke check: a clan generation modeled after `sase-7z` with six archived `DONE` members plus
current completions must resolve, while changing any archived member to failure must make it unresolved again.

## Risks and guardrails

- Treating every dismissed record as success would release failed work, so the status-to-outcome mapping and failure
  regressions are mandatory.
- Timestamp alone is insufficient because archive suffixes can collide across projects; exact project and name
  validation is part of the contract.
- Repeated full-archive scans would make every wait poll more expensive. Use indexed, suffix-batched queries and reuse
  the projection throughout one dependency-index build.
- Fork resolution needs payload fields that summary rows do not carry. Read only already-matched bundle paths, and keep
  corrupt/unreadable payloads a closed failure rather than guessing.
- Do not broaden this fix into archive schema redesign, dismissal retention, or clan-generation policy changes. The
  acceptance criterion is lifecycle equivalence before and after successful dismissal.
