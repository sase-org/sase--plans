---
tier: epic
title: Claimed bead status for pre-launch agent reservations
goal: 'A bead whose agent has started but has not yet reached model execution shows
  a durable `claimed` status across every bead surface, the claim is released back
  to `open` when that agent dies before launching, and `sase bead work` keeps deciding
  purely from live agent liveness.

  '
phases:
- id: core-status
  title: Rust core claimed status value
  depends_on: []
  size: medium
  description: '''Phase core-status: Rust core claimed status value'' section: add
    StatusWire::Claimed to sase-core with parsing, rendering, filters, blocker semantics,
    and stats.'
- id: core-claims
  title: Rust core claim and release mutations
  depends_on:
  - core-status
  size: medium
  description: '''Phase core-claims: Rust core claim and release mutations'' section:
    add compare-and-swap claim_for_agent_wait and release_agent_claim mutations plus
    their pyo3 bindings.'
- id: py-status
  title: Python status model and presentation
  depends_on:
  - core-status
  size: medium
  description: '''Phase py-status: Python status model and presentation'' section:
    add Status.CLAIMED, the SQLite migration, the shared bead status presentation
    module, and every read-side CLI surface.'
- id: py-claim-runtime
  title: Runner claim lifecycle
  depends_on:
  - core-claims
  - py-status
  size: medium
  description: '''Phase py-claim-runtime: Runner claim lifecycle'' section: claim
    the canonical bead store when a bead-carrying agent starts waiting, promote on
    launch, and release on shutdown.'
- id: chop-reconciler
  title: Stale claim reconciler and doctor check
  depends_on:
  - py-claim-runtime
  size: medium
  description: '''Phase chop-reconciler: Stale claim reconciler and doctor check''
    section: add the bead_claim_checks chop that releases claims held by dead pre-launch
    agents and a doctor advisory for unowned claims.'
- id: tui-visuals
  title: ACE and clan summary visuals
  depends_on:
  - py-status
  size: small
  description: '''Phase tui-visuals: ACE and clan summary visuals'' section: render
    the claimed status in the ACE plans pane, the epic clan summary, the plan filter
    bar, and the manual status cycle.'
- id: docs-and-skill
  title: Documentation and agent-facing guidance
  depends_on:
  - py-claim-runtime
  - chop-reconciler
  - tui-visuals
  size: small
  description: '''Phase docs-and-skill: Documentation and agent-facing guidance''
    section: document the claimed status in the bead docs, the sase_beads skill, and
    the bead xprompt wording.'
create_time: 2026-07-24 16:21:11
status: done
bead_id: sase-8y
---

- **PROMPT:** [prompts/202607/claimed_bead_status.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/claimed_bead_status.md)
- **BEAD:** [sase-8y](https://github.com/sase-org/sase--beads/blob/main/pages/sase-8y/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-8y.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-8y.land/README.md)
  - [bbugyi200.athena.sase-8y.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-8y.land.md#member-code)
- **COMMITS:**
  - [d0495f1](https://github.com/sase-org/sase/commit/d0495f1cba07b4706cc7696a1561d9fa0a0c3343) — fix: finish claimed status landing cleanup (sase-8y)

# Plan: Claimed bead status for pre-launch agent reservations

## Goal

Today a bead only changes status at the last possible moment: `claim_bead_for_agent_launch` sets `in_progress`
immediately before model execution. Everything that happens earlier — dependency waits (`%wait`), runner-slot waits,
duration/time waits — is invisible in the bead store. Launch `sase bead work` on a ten-phase epic and nine beads sit at
`open` while nine agent processes are already alive and irreversibly assigned to them.

This epic adds a fourth bead status, `claimed`, that fills exactly that window:

```
open ──claim──▶ claimed ──promote──▶ in_progress ──close──▶ closed
  ▲                │
  └────release─────┘        (claim owner died before launching)
```

`claimed` means: _a live SASE agent owns this bead but has not started working yet._ It is created by the agent runner
when a `%id(..., bead=<id>)` agent enters its wait phase, promoted to `in_progress` by the existing just-in-time claim
when the runner reaches execution, and released back to `open` when the owning agent dies before it ever launched.

## Design decisions

These decisions are settled; phase agents implement them rather than relitigating them.

### 1. A real persisted status, not a derived overlay

`claimed` is a fourth `Status`/`StatusWire` variant, persisted in the bead event store exactly like the other three. A
derived "is some agent waiting on this bead" overlay would self-heal for free, but it would not survive a store push,
would not be visible to any non-Python reader, and would not satisfy the requirement that `sase bead list` and friends
show the status. Persisting it means claims can go stale, which is why the reliability model below has three independent
layers.

### 2. Claims are written to the canonical project bead store

The claim happens _before_ the agent owns a workspace. A waiting agent launched with a deferred workspace runs with
`workspace_num = 0` inside workspace #1, so "this runner's workspace store" is not a stable target. Every claim and
release therefore resolves its store through `sase.bead.store_locator.canonical_beads_dir_for_project(project_name)` —
the same store `sase bead list`, the mobile helper, and the wait chop already read. The promotion to `in_progress` keeps
its current behavior (the agent's own workspace store) and converges through the normal bead event merge, since the
claim event is strictly older than the promotion event.

### 3. Claim and release are compare-and-swap operations

Neither operation ever overwrites someone else's state:

- **Claim** succeeds only from `open`. From `claimed` held by the same agent it is a no-op success (this is what makes
  the post-wait runner re-exec idempotent). From `closed`, `in_progress`, or `claimed` by another agent it declines
  without mutating anything and without raising.
- **Release** succeeds only when the bead is still `claimed` _and_ its assignee is the releasing agent. Anything else
  declines.

This is what makes every layer of the reliability model safe to run concurrently and repeatedly.

### 4. Claiming must never fail an agent launch

The claim is advisory visibility, not an admission gate. `sase.bead.claims` swallows every failure (unreadable store,
git contention, missing project) into a printed warning and returns "no claim held". A bead that is already
`in_progress` or closed does not block startup either — the existing launch-time claim remains the authoritative check
and keeps its current error behavior.

### 5. `sase bead work` stays liveness-driven and status-blind

`sase bead work` already decides everything from agent artifacts and PID liveness (`preview_bead_work_force_reuse`, the
orphan-meta scan, force-reuse wipes) and reads no bead status other than "not closed". Nothing in this epic adds a
status read to that path. Concretely:

- `build_epic_work_plan` continues to schedule every non-closed phase, so a `claimed` phase is still scheduled.
- Relaunching an epic over its own claimed phases works: the relaunched agent reuses the same `%id(!name)`, so the claim
  CAS sees its own name and no-ops.
- No phase in this epic may add a `Status::Claimed` check to `work.rs`, `cli_work*.py`, or the force-reuse preview.

### 6. Automation only reverses claims it can prove it created

The reconciler releases a claim only when it can point at the artifact of the owning agent, see that the agent is dead,
and see that the agent never promoted the claim (`bead_claim_promoted` in `agent_meta.json`). A `claimed` bead whose
owner cannot be resolved — a hand-set status, a wiped artifact — is never mutated; `sase doctor` reports it instead.
This also removes the only dangerous failure mode: a bead that a long-running agent already carried to `closed` in its
own workspace store can never be reopened by the reconciler, because that agent's claim was promoted.

### 7. Claim and release commits are local and unpushed

Claims are committed with the existing pathspec-scoped, lock-protected `_commit_bead_state` helper so the canonical
checkout never accumulates uncommitted bead churn, but they are not pushed: a claim is host-local runtime state and
per-launch pushes would be pure network churn. Because `_commit_bead_state` stages every changed bead-state file, a
burst of concurrent claims naturally coalesces into far fewer commits than claims.

### Rejected alternatives

- **Release from each killer** (`user_kill`, name wipes, hook process kills, partial-launch rollback): too many call
  sites to keep correct, and it still would not cover SIGKILL, crashes, or reboots. The runner shutdown funnel plus the
  reconciler cover every case in two places.
- **Repurposing `preclaim_epic_work_plan` to write `claimed`**: that mutation has no production caller today (tests and
  benchmarks only), so changing it would add churn without changing behavior. Leave it alone.
- **A new bead event operation kind**: claims reuse the existing `IssueUpdated` event with `status` and `assignee`
  fields, exactly like `claim_for_agent_launch`, so the event log format, merge driver, and parity tests are unchanged.

## Visual language

One status, one look, everywhere. The chosen glyph is `◎` (U+25CE BULLSEYE): it reads as _reserved / spoken for_ rather
than _partially done_, and it stays inside the circle family the other three statuses already use, giving a coherent
progression:

| Status        | Glyph                 | CLI color            | TUI / Rich color | Label         |
| ------------- | --------------------- | -------------------- | ---------------- | ------------- |
| `open`        | `○`                   | cyan (`\x1b[36m`)    | `#87D7FF`        | `OPEN`        |
| `claimed`     | `◎`                   | magenta (`\x1b[35m`) | `#AF87FF`        | `CLAIMED`     |
| `in_progress` | `◐`                   | yellow (`\x1b[33m`)  | `#FFD700`        | `IN_PROGRESS` |
| `closed`      | `✓` (CLI) / `●` (TUI) | green (`\x1b[32m`)   | `#5FD787`        | `CLOSED`      |

Two constraints that drove this choice, both of which phase agents must respect:

- **Font coverage.** The ACE PNG snapshot suite pins Fira Code. Fira Code has `○ ◐ ● ◉ ◎ ◇ ◆ □ ■ ◊`, and does **not**
  have `◔` (U+25D4), `◌`, `⊙`, `⧗`, or `✓`. `◎` is safe in both the terminal and the PNG renderer; the visually obvious
  quarter-circle `◔` is not, and must not be used.
- **Cross-surface resonance.** `#AF87FF` is the amethyst ACE already uses for `WAITING` agents. A claimed bead _is_ a
  waiting agent, so the two surfaces now agree by color.

`sase bead show` gains one extra line for claimed beads so the claim reads as an event rather than a mystery:

```
◎ sase-1a.3 · Wire the reconciler chop   [CLAIMED]
Type: phase · Owner: bryan
Assignee: sase-1a.3
Claimed by: sase-1a.3 (agent has not started working yet)
```

## Reliability model

Three independent, idempotent layers. Each one is a compare-and-swap; running all three concurrently is safe.

1. **Runner shutdown release (fast path, covers ~every ordinary kill).** `finalize_runner_shutdown` releases a held,
   unpromoted claim. A SIGTERM'd waiting runner notices `was_killed()` within its 2s poll and exits through this funnel,
   so the bead reopens seconds after `sase agent kill` or an ACE kill — well before `user_kill`'s SIGKILL escalation.
   Skipped when `has_pending_handoff(artifacts_dir)` is true, mirroring the workspace-release rule, so retry handoffs
   keep their claim.
2. **`bead_claim_checks` chop (backstop, covers SIGKILL/crash/reboot).** Releases claims whose owning agent is dead and
   unpromoted, within one `waits` lumberjack interval.
3. **`sase doctor` advisory (covers everything else).** Reports claims with no resolvable owner and never mutates them.

**Ordering invariant (mandatory).** The reconciler's authoritative pass must read bead stores _before_ scanning agent
artifacts. The runner writes `agent_meta.json` (with `bead_id`) strictly before it writes the claim; reading beads first
therefore makes it impossible to observe a fresh claim without also observing its artifact, which is the only way a live
agent's claim could be falsely released. A cheap artifact pre-pass that only decides _whether to do any work_ is allowed
and does not weaken this, because it never releases anything by itself.

## Non-goals

- No change to when `in_progress` is set, and no release of `in_progress` on kill. An agent that actually ran leaves its
  bead `in_progress`; that behavior is unchanged.
- No bead-status column, chip, or filter in the ACE Agents tab.
- No new configuration keys.
- No change to `sase bead ready` semantics beyond what falls out for free: `ready` lists `open` beads only, so claimed
  beads correctly drop out of `sase bead ready` and out of the `bd/next` selection pool.

## Phase core-status: Rust core claimed status value

Open the Rust core repo with the `/sase_repo` skill first and use the path it prints for every read and write. Land this
as a non-breaking `feat:` commit — the variant is additive and old stores still parse — so release-plz keeps the crate
inside the published `sase-core-rs>=0.12.0,<0.13.0` window that `pyproject.toml` pins.

Add `StatusWire::Claimed` (serde `claimed`, ordered between `Open` and `InProgress`) in
`crates/sase_core/src/bead/wire.rs` and thread it through every exhaustive `match` and every status list in the bead
module:

- `read.rs`: `parse_status`, `status_as_str`, and `has_active_blocker` — a claimed blocker is an _active_ blocker, so
  the match becomes `Open | Claimed | InProgress`. The ready filter keeps `status == Open`.
- `search.rs`: `status_value`.
- `cli.rs`: `parse_status`, `status_icon` (`◎`), `status_value`, `status_upper` (`CLAIMED`), `color_status_icon` with a
  new `ANSI_MAGENTA` constant, `has_active_blocker`, the `parse_list_filters` default status set (now
  `Open + Claimed + InProgress`), and `handle_stats` (a `Claimed:` line between `Open:` and `In Progress:`).
- `cli.rs` `handle_show`: when the issue is claimed, emit the
  `Claimed by: <assignee> (agent has not started working yet)` line described in the visual language section.
- `mutation.rs`: the status parser near the update path accepts `claimed`.
- `work.rs`: confirm (and cover with a test) that a claimed phase is still scheduled — the plan builder filters on
  `!= Closed`, so no code change is expected here.

Tests: unit coverage for round-tripping `claimed` through serde, for `claimed` blocking a dependent bead in
`has_active_blocker`, for the new default list filter, for the stats line, and for a claimed phase still appearing in an
epic work plan. Run the repo's normal Rust checks (`cargo fmt`, `cargo clippy`, `cargo test`) before finishing.

## Phase core-claims: Rust core claim and release mutations

Also in the Rust core repo, opened through `/sase_repo`.

Add two compare-and-swap mutations to `crates/sase_core/src/bead/mutation.rs`, modeled on `claim_for_agent_launch` (same
blank-agent-name validation, same `with_bead_mutation_lock`, same durable-store error wrapping, same `IssueUpdated`
event shape — no new event operation kind):

```rust
pub fn claim_for_agent_wait(beads_dir, issue_id, agent_name, now) -> Result<BeadMutationOutcomeWire, BeadError>
pub fn release_agent_claim(beads_dir, issue_id, agent_name, now) -> Result<BeadMutationOutcomeWire, BeadError>
```

`claim_for_agent_wait`:

- `Open` → set `Claimed`, set `assignee`, bump `updated_at`, append the `IssueUpdated` event, save, return
  `changed: true`.
- `Claimed` with the same assignee → return `changed: false` with the current issue attached and **no** write, so the
  post-wait runner re-exec neither churns the event log nor produces a second commit.
- `Claimed` by another assignee, `InProgress`, or `Closed` → return `changed: false` with the current issue attached and
  a `message` naming the current status and holder. Declining is not an error.
- Unknown id → `not_found` error, as elsewhere in the module.

`release_agent_claim`:

- `Claimed` with a matching assignee → set `Open`, clear `assignee` to the empty string, bump `updated_at`, append the
  `IssueUpdated` event, save, return `changed: true`.
- Any other status, or a non-matching assignee → `changed: false`, no write.
- Unknown id → `not_found` error.

Export both from the bead module and add pyo3 wrappers `bead_claim_for_agent_wait` and `bead_release_agent_claim` in
`crates/sase_core_py/src/lib.rs`, following the `bead_claim_for_agent_launch` wrapper's signature and return shape
(`(issue | None, outcome)`).

Tests: each CAS branch above, including that a decline leaves `issues.jsonl` and the event streams byte-identical, that
claim→release→claim round-trips, and that a released bead has an empty assignee.

## Phase py-status: Python status model and presentation

In the sase repo. Run `just install` before `just check`, since workspace virtualenvs go stale.

**Model and storage.** Add `CLAIMED = "claimed"` to `Status` in `src/sase/bead/model.py`, ordered between `OPEN` and
`IN_PROGRESS`. In `src/sase/bead/db.py` (the compatibility SQLite layer), widen the `status` CHECK constraint in
`_SCHEMA` and add a `_migrate_relax_status_check` migration wired into `init_db` alongside the existing migrations,
following `_migrate_relax_size_check` as the template so pre-existing databases accept the new value.

**Shared presentation module.** Every surface that maps a status to a glyph, color, or label today does it with an
inline dict literal keyed by `Status`, which means a new member is a latent `KeyError` in at least six modules. Add
`src/sase/bead_status_presentation.py`, modeled on the existing `src/sase/phase_size_presentation.py`, exposing the
single source of truth for the table in the visual language section: ordered status values, glyph, CLI/Rich style,
uppercase label, and a helper returning the display order. Route `sase.bead.cli_common.status_icon` through it and
delete the inline dict there.

**Read surfaces.**

- `src/sase/bead/cli_query.py`: the implicit default status set in `handle_bead_list` becomes
  `OPEN + CLAIMED + IN_PROGRESS`; `handle_bead_stats` prints a `Claimed:` line between `Open:` and `In Progress:`; and
  `handle_bead_show` prints the same `Claimed by: <assignee> (agent has not started working yet)` line the Rust
  `handle_show` emits. These are independent implementations of the same commands — the Rust fast path defers `show` to
  argparse in some contexts — so both must stay in parity. `ready`, `blocked`, and the stats counts themselves come from
  the Rust read facade and need no Python change.
- `src/sase/main/parser_bead.py`: add `claimed` to every `--status` choice list (list, update, search).
- `src/sase/integrations/_mobile_helper_beads.py`: `_bead_status_filter`'s active default gains `CLAIMED`.
- `src/sase/doctor/checks_beads.py`: include claimed counts wherever status stats are summarized.
- `src/sase/bead/cli_admin.py`: refresh the status line in the command help text.

**Tests and fixtures.** Extend the bead CLI tests (`tests/test_bead/test_cli_list.py`, `test_model.py`,
`test_db_migrations.py`, `tests/test_bead/golden/*` as needed) to cover the new default filter, the migration from a
legacy CHECK constraint, and glyph/label rendering through the presentation module. Finish with `just check`.

## Phase py-claim-runtime: Runner claim lifecycle

In the sase repo, and the heart of the feature.

**Facade and project methods.** Add `claim_for_agent_wait` and `release_agent_claim` wrappers to
`src/sase/core/bead_mutation_facade.py` (mirroring `claim_for_agent_launch`), and thin `BeadProject` methods in
`src/sase/bead/project.py` that call them and refresh the projection. Both return enough for the caller to distinguish
_claimed by us_ from _declined_.

**Commit helpers.** Add `commit_bead_claim(beads_dir, bead_id, agent_name)` and
`commit_bead_claim_release(beads_dir, bead_id, agent_name)` to `src/sase/bead/sync.py` next to
`commit_bead_work_launch`, both delegating to `_commit_bead_state` with `auto_commit_type="beads"` and messages
`chore(beads): claim <id> for <agent>` / `chore(beads): release claim on <id> from <agent>`. No push.

**Claim lifecycle module.** Add `src/sase/bead/claims.py` with two functions usable both from the runner and from a chop
(so neither takes an artifacts directory as a hard requirement):

- `claim_bead_for_waiting_agent(*, project_name, bead_id, agent_name) -> bool` — resolve
  `canonical_beads_dir_for_project`, open the project, CAS-claim, commit when the store changed, print a one-line notice
  (claimed, or declined with the current holder), and return whether _we_ now hold the claim. Returning `True` for an
  already-ours claim is what makes the post-wait re-exec correct.
- `release_bead_claim_for_agent(*, project_name, bead_id, agent_name) -> bool` — the mirror image.

Both catch every exception, degrade to a warning, and never raise. This is decision 4 and it is not optional.

**Runner integration** in `src/sase/axe/run_agent_runner.py`:

- After `extract_directives_and_write_meta` and the retry-chain metadata merge, and _before_ `wait_for_dependencies`,
  claim when `info.bead_id`, `agent_name`, and `project_name` are all present **and** `has_wait` is true. Gating on
  `has_wait` keeps a straight-through launch from paying for a store write and commit it would undo milliseconds later.
  Track the held claim in a local variable.
- After `claim_bead_for_agent_launch` succeeds (the existing just-in-time promotion), set
  `agent_meta["bead_claim_promoted"] = True`, persist it with `write_agent_meta`, and clear the local held-claim state
  so shutdown does not release a promoted claim. Do the metadata write in `src/sase/axe/run_agent_runner_bead.py` or at
  its call site, but make it durable before execution starts — the reconciler depends on it.

**Shutdown release** in `src/sase/axe/run_agent_runner_lifecycle.py`: add the held-claim fields (bead id, agent name,
project name) to `RunnerShutdownState` and, in `finalize_runner_shutdown`, release the claim unless
`has_pending_handoff(artifacts_dir)` is true. Place it next to the existing workspace release so both cleanup paths read
as one story, and let a failure print a warning rather than derail shutdown.

**Tests.** Cover: a waiting agent claims and a launching agent promotes (extend
`tests/test_bead/test_epic_jit_claim_integration.py`, whose current assertion is that claims happen only at execution —
it becomes claim-then-promote); a killed pre-launch runner reopens its bead; a promoted claim is _not_ released at
shutdown; a pending handoff suppresses the release; a decline (bead already `in_progress`) leaves the store untouched
and the launch unaffected; and re-claiming an already-ours claim produces no second commit. Finish with `just check`.

## Phase chop-reconciler: Stale claim reconciler and doctor check

In the sase repo.

**New chop.** Add `src/sase/scripts/sase_chop_bead_claim_checks.py` using the `@builtin_chop("bead_claim_checks")` /
`run_builtin_chop` pattern (`sase_chop_wait_checks.py` is the closest model), register its console script in
`pyproject.toml` alongside the other `sase_chop_*` entries, and register it in `src/sase/default_config.yml` under
`lumberjacks.waits.chops` with a description like "Release bead claims held by dead pre-launch agents". The `waits`
lumberjack (10s) is the right home: a SIGKILLed agent's bead reopens within one interval, and this chop reconciles the
same waiting-agent population `wait_checks` already serves.

Algorithm, in this order:

1. **Cheap pre-pass.** Scan agent artifacts through the shared `scan_agent_artifacts` facade for records that carry a
   `bead_id`, lack `bead_claim_promoted`, and are not alive. If there are none, emit a zero summary and return without
   opening a single bead store. This keeps the steady-state cost near zero.
2. **Authoritative pass, beads first.** For each project with candidates, read its canonical store and collect issues
   with status `claimed`. This read must happen _before_ the next step — see the ordering invariant.
3. **Re-scan artifacts** and build `agent_name -> (artifact_dir, pid, stopped_at, bead_id, bead_claim_promoted)`.
4. **Release** a claimed bead only when its assignee resolves to a record whose `bead_id` matches, whose
   `bead_claim_promoted` is absent, and whose process is dead per `is_process_alive`. Anything else is left alone — in
   particular an unresolvable assignee is never touched.
5. Release through `release_bead_claim_for_agent` so the CAS and the commit are shared with the runner path, and emit a
   summary (projects scanned, claims examined, claims released).

**Doctor advisory.** Extend `src/sase/doctor/checks_beads.py` to report `claimed` beads whose owner cannot be resolved
to any artifact, as a warning naming the beads and suggesting `sase bead open <id>`. It must never mutate.

**Tests.** Cover: dead unpromoted owner → released; live owner → untouched; promoted owner (dead or alive) → untouched;
unresolvable assignee → untouched and reported by doctor; empty steady state → no bead store reads. Finish with
`just check`.

## Phase tui-visuals: ACE and clan summary visuals

In the sase repo. Every map below must consume `src/sase/bead_status_presentation.py` rather than growing another inline
dict.

- `src/sase/ace/tui/widgets/artifacts/plans_detail.py`: `_status_chip`, `_status_glyph`, `_status_style` gain the
  claimed entry, and `_readiness_label` returns a new `"claimed"` label (color `#AF87FF`, glyph `◎`) before it consults
  the snapshot — otherwise a claimed bead misreports as `waiting`, which is exactly the confusion this epic removes.
- `src/sase/ace/tui/widgets/artifacts/plans_rendering.py`: `_status_glyph` and `_status_style`.
- `src/sase/ace/tui/widgets/artifacts/plan_filter_bar.py`: add `claimed` to the status completion hint.
- `src/sase/ace/tui/actions/artifacts_plans.py`: the manual cycle stays `open → in_progress → closed → open` because
  `claimed` is machine-managed; cycling _from_ `claimed` goes to `in_progress` (a human taking the bead over). Add that
  entry explicitly so the lookup cannot raise.
- `src/sase/scripts/sase_clan_summary_epic.py`: `_STATUS_STYLES` gains the claimed entry. This is the surface where the
  win is most visible — an epic clan summary now shows which phases already have agents attached.

Add a claimed bead to the relevant ACE test fixtures and, if any PNG snapshot covers a claimed row, regenerate it with
`--sase-update-visual-snapshots` and confirm the glyph renders (Fira Code has `◎`; it does not have `◔`). Finish with
`just check` plus `just test-visual`.

## Phase docs-and-skill: Documentation and agent-facing guidance

In the sase repo. Read the `generated_skills` long-term memory with the `/sase_memory_read` skill before touching
anything under `src/sase/xprompts/skills/`. Do not edit any file under `sase/memory/`, `AGENTS.md`, or the generated
provider instruction shims — this phase has no permission to do so.

- `docs/beads.md`: add `claimed` to the status table (glyph `◎`), to both `--status` option tables, and to the blocked
  definition ("at least one dependency with status `open`, `claimed`, or `in_progress`"). Extend the lifecycle section
  that currently documents the just-in-time `in_progress` claim with the earlier claim, the release-on-death behavior,
  the reconciler, and the explicit statement that `sase bead work` ignores bead status and decides from agent liveness.
- `src/sase/xprompts/skills/sase_beads.md`: document `claimed` in the status list and state plainly that agents should
  not set it by hand — the runtime owns it.
- `src/sase/default_config.yml`: refresh the `bd/work_phase_bead` xprompt wording so it survives both windows (the bead
  is claimed for the agent and is `in_progress` by the time the agent is reading the prompt).
- Check `docs/configuration.md` and `docs/mobile_gateway.md` for status enumerations that need the new value.

Finish with `just check`.
