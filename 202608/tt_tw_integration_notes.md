---
tier: tale
title: Reconcile epics sase-tt and sase-tw with integration notes
goal: "Epics `sase-tt` and `sase-tw` stop colliding unseen on the Artifacts Agent pane,
  the agent-catalog query wire, sase-core releases, and the PNG goldens, because every
  phase bead that shares a surface with the other epic carries a note saying what it
  shares, who lands first, and which invariant it must hold.

  "
size: small
proposed_by: bbugyi200.athena.0ds
create_time: 2026-08-25 16:04:31
status: wip
---

# Plan: Reconcile epics `sase-tt` and `sase-tw` with integration notes on nine beads

## 1. Problem

Two epics were proposed 35 minutes apart on 2026-08-25 and neither plan mentions the
other:

- **`sase-tt`** — "Make Artifacts sub-tab queries fast, starting with the Agent pane"
  (`plan:202608/artifacts_query_performance.md`), created 14:59:11 EDT.
- **`sase-tw`** — "Artifact links that survive, derive themselves, and pay for the turn"
  (`plan:202608/artifact_link_durability_and_derivation.md`), created 15:34:34 EDT.

A third epic, **`sase-tj.10`** ("Agent pane landing gaps"), is in progress, and its
phase `sase-tj.10.3` is rebaselining every `artifacts_*` PNG golden right now.

`sase-tw`'s plan has a section titled "Sequencing note: the Agent pane is still landing"
that gates its `agent-pane-filters` phase on `sase-tj.10`. That gate is correct and
insufficient: it predates nothing — `sase-tt` already existed when `sase-tw` was written
— and `sase-tt` rewrites the Agent pane far more invasively than `sase-tj.10` does.
`sase-tt`'s plan has no sequencing section at all.

No phase-worker agent has launched for either epic yet
(`sase agent search 'name:sase-tt.'` returns zero rows), so notes added now will be read
before any phase starts. That window is the reason this is worth doing as its own unit
of work.

### 1.1 The collisions, each verified against the tree

**Four shared files.** `sase-tw.13` (`agent-pane-filters`) lists
`src/sase/agents/catalog/_query.py`, `src/sase/ace/tui/widgets/artifacts/query_rows.py`,
`src/sase/ace/tui/widgets/artifacts/agents_data.py`, and
`src/sase/agents/cli_search.py`. `sase-tt.3` (`agent-paint`) and `sase-tt.5`
(`entry-projection`) rewrite all four.

**The wire-shape invariant versus three new fields.** `sase-tt.5`'s plan section
requires the emitted agent-catalog wire dict stay "byte-identical" and proposes a golden
test as the guard. That invariant exists so `sase-tt.5` composes with `sase-tt.4` in
either order — not as a permanent schema freeze. `sase-tw.13` adds `relation`,
`artifact`, and `linked` to that same wire. Whichever lands second must regenerate the
golden deliberately.

No new wire _types_ are introduced, so `sase-tt.4`'s direct `PyDict`-to-`QueryRow` path
composes: `linked` is a bool like the existing `hidden` / `dismissed` / `revivable` /
`attention`, and `relation` / `artifact` are multi-valued like the existing `label` /
`text` / `project` (`src/sase/agents/catalog/_query.py:49-62`). Only the golden
collides.

**`sase-tw.13` adds per-row cost to the exact function `sase-tt.5` is shrinking.**
`agent_catalog_query_entry` costs a measured 408ms over 11,783 rows. `sase-tt.5` shrinks
it; `sase-tw.13` then joins a link-facet lookup and two list fields onto every row.
Nothing in `sase-tw.13`'s acceptance mentions performance, and `sase-tt.8` will by then
have certified a ≤400ms Agent first-paint target into `tests/perf/README.md`.

**`artifact_links` is already on the Agent snapshot-load path, and `sase-tt` does not
know it.** `AgentsSnapshot` already carries an `artifact_links: ArtifactLinksSnapshot`
field and `load_agents_snapshot` already calls `load_artifact_links_snapshot(project)`
(`src/sase/ace/tui/widgets/artifacts/agents_data.py:24-56`). `sase-tt`'s 1,529ms
breakdown never itemizes it, because it is cheap today at 112 machine-global aggregate
rows. `sase-tw`'s stated goal is ~1,600+ rows.

**`sase-tw.13`'s acceptance invariant is undefined under a two-stage pane.**
`_filtered_agents_snapshot`
(`src/sase/ace/tui/widgets/artifacts/agents_query.py:439-460`) short-circuits only on a
blank query remainder, so any `linked:` / `relation:` / `artifact:` term with
`_query_index is None` returns `pending` — which is exactly `sase-tt.3`'s intended grow
trigger, and fine. The seam is the other order: `matched_rows` is computed by
intersecting `result.matched_row_ids` with `snapshot.rows`, so a full-corpus index over
a still-bounded snapshot yields a silently short `match_count`. `sase-tw.13`'s
acceptance — "`linked:false`'s count plus `linked:true`'s equals the unfiltered total" —
is a whole-corpus assertion. `sase-tw.13` waits on `index-durability` precisely because
a filter over a lossy index "returns confidently wrong answers"; `sase-tt.3` introduces
a second, different incompleteness, and it needs the same answer.

**Three sase-core releases across two epics, and one wrong instruction.** `sase-tt.4`
(corpus construction), `sase-tw.3` (`BeadLinkWire` gains `direction`), and `sase-tw.5`
(relation registry fields) each release sase-core. `sase-tt.4`'s plan section says to
"release sase-core and raise the `sase-core-rs` floor in this repo's `pyproject.toml`".
`tools/validate_sase_core_rs_version:250-260` says the opposite in as many words: a
checkout ahead of the published window "is the normal dev state now that the
release-branch reconciler owns the window", editing `pyproject.toml` there is "the
conscription `@plan:202608/core_window_ratchet.md` removed", and
`tools/ratchet_core_window` moves the window at release time on the release branch. The
declared window today is `sase-core-rs>=0.31.12,<0.32.0` (`pyproject.toml:46`).
`sase-tw.3` states this correctly and is the model. This is a defect in `sase-tt.4`'s
instruction independent of `sase-tw`.

**Three efforts rebaseline the same PNG goldens.** `sase-tj.10.3` (in progress)
rebaselines every `artifacts_*` golden and adds six new Agent-pane snapshots;
`sase-tt.3` may add an "index still building" affordance and `sase-tt.8` explicitly
flags `just test-visual`; `sase-tw.12` updates relation snapshot tests and `sase-tw.13`
adds filter-bar completions. `sase-tw.13` has a `sase-tj.10` gate. `sase-tt.3` — the
most invasive of the three — has none.

## 2. Approach

Append nine `INTEGRATION:` notes across the two epic beads and seven phase beads. Notes
are the right vehicle: the archived plan files are immutable durable artifacts, every
phase worker reads its own bead before starting, and no phase worker has launched yet.

Change no code and no plan file. The only writes are bead notes.

Each note is reproduced verbatim in §4 below. Write each one to a scratch file and pass
it with the `@<path>` form, because the text is multi-line and contains backticks:

```bash
sase bead note <id> @/tmp/note_<id>.md
```

Verify each with `sase bead show <id>` afterwards. Notes are append-only and attributed;
`sase bead note --edit <n>` fixes a mistake, and ordinals shift after any edit.

### 2.1 Before writing, re-verify

These beads are live. Between this plan's approval and its execution, phases may have
started or landed. Before appending each note:

- Run `sase bead show <id>` and confirm the bead has no note already covering the same
  ground. If one does, skip that note rather than duplicating it.
- If a phase named in a note has already **closed**, keep the note but replace its "do
  not start until X lands" wording with a past-tense pointer at what changed; a stale
  gate on a landed phase is worse than no gate.
- Spot-check the three code citations that carry the most weight, since they are what
  make the notes actionable: `agents_data.py`'s `artifact_links` field and
  `load_artifact_links_snapshot` call, `agents_query.py`'s `matched_row_ids`
  intersection, and `validate_sase_core_rs_version`'s remediation text. If any has
  moved, update the line reference in the note text rather than dropping the claim.

### 2.2 Non-goals

- Do not edit either archived plan file. They are durable artifacts of what was
  proposed.
- Do not reorder, re-parent, or add `depends_on` edges between the two epics' beads. The
  dependency is a start-ordering constraint on three phases, not a structural one, and
  cross-epic bead dependencies would deadlock `sase bead work`'s per-epic launch.
- Do not open new task beads for the `sase-tt.4` `pyproject.toml` defect. It is an
  instruction defect inside an in-flight epic phase, so it belongs on that phase's bead,
  which is what note 5 does.
- Do not touch `sase-tj.10` or its phases. Its own work is correct and in progress; the
  notes here only tell `sase-tt` and `sase-tw` phases to wait for it.

## 3. Verification

This plan changes no code, so `just check` has nothing to gate. Verify instead that:

- `sase bead show` on each of the nine beads renders the new note, attributed and
  timestamped, with its Markdown intact and no shell-mangled backticks.
- The nine ids are exactly: `sase-tt`, `sase-tt.3`, `sase-tt.4`, `sase-tt.5`,
  `sase-tt.8`, `sase-tw`, `sase-tw.3`, `sase-tw.12`, `sase-tw.13`.
- No other bead was modified: `sase bead history` on each shows exactly one appended
  `note_appended` event from this run.

## 4. The notes, verbatim

### 4.1 `sase-tt` (epic)

```text
INTEGRATION: epic sase-tw ("Artifact links that survive, derive themselves, and pay for
the turn", plan:202608/artifact_link_durability_and_derivation.md) was proposed 35
minutes after this epic, and its phase sase-tw.13 (agent-pane-filters) edits four of the
same files: src/sase/agents/catalog/_query.py,
src/sase/ace/tui/widgets/artifacts/query_rows.py,
src/sase/ace/tui/widgets/artifacts/agents_data.py, and src/sase/agents/cli_search.py.
Neither plan mentions the other. sase-tw.13 already carries a "do not start until
sase-tj.10 has landed" gate; it now also waits for this whole epic to land, and has been
told so on its own bead.

This epic lands first and owns the Agent pane's first-paint budget, so §2.1's ≤400ms
Agent target is a shared budget rather than a private one. sase-tw.13 adds a per-row
artifact-link facet join to agent_catalog_query_entry — the exact function sase-tt.5 is
shrinking — and has been told to re-run tests/perf/bench_artifacts_first_paint.py and
hold that number. See the note on sase-tt.8 for what that requires of the perf recipe.

One fact this epic's measured breakdown does not itemize: AgentsSnapshot already carries
an artifact_links: ArtifactLinksSnapshot field, and load_agents_snapshot already calls
load_artifact_links_snapshot(project) (agents_data.py:24-56). It is cheap today at 112
machine-global aggregate rows. Epic sase-tw's stated goal is ~1,600+ rows, so treat it
as a growing cost on the load path rather than a constant.
```

### 4.2 `sase-tt.3`

```text
INTEGRATION: three constraints this phase's plan section does not state.

1. Do not start before sase-tj.10.3 lands on master. That phase ("Put the Agent pane in
   the fast-startup inventory and rebaseline the goldens") is IN_PROGRESS, rebaselines
   every artifacts_* PNG golden, and adds six new Agent-pane snapshots. An "index still
   building" affordance from this phase would need golden updates on top of that work.
   sase-tw.13 already carries this gate; this phase needs it too and does not have it.
   Check sase bead show sase-tj.10.3 first and wait rather than rebasing around it.

2. Give artifact_links a defined answer on the bounded pass. AgentsSnapshot already
   carries artifact_links: ArtifactLinksSnapshot, and load_agents_snapshot already calls
   load_artifact_links_snapshot(project) (agents_data.py:24-56). Decide explicitly
   whether the bounded head-slice load populates it or leaves it empty until the
   extension pass, and record which in the code. The aggregate is project-scoped, not
   row-scoped, so slicing rows does not shrink it, and epic sase-tw grows it from ~112
   to ~1,600+ rows.

3. State the semantics a filter term sees mid-extension, because phase sase-tw.13 of
   epic sase-tw will build three new filter fields (relation:, linked:, artifact:) on
   top of them. _filtered_agents_snapshot (agents_query.py:439-460) short-circuits only
   on a blank remainder, so any filter term with _query_index None returns pending —
   this phase's intended grow trigger, and correct. The seam to close is the other
   order: it computes matched_rows by intersecting result.matched_row_ids with
   snapshot.rows, so a full-corpus index over a still-bounded snapshot yields a silently
   short match_count. Either make it impossible to present a full index against a
   partial snapshot, or make the count explicitly a lower bound — _sync_agent_query_bar
   already accepts lower_bound and coverage_label for exactly that case.
```

### 4.3 `sase-tt.4`

```text
CORRECTION and INTEGRATION: this phase's plan section says to "release sase-core and
raise the sase-core-rs floor in this repo's pyproject.toml". Do not hand-edit the floor.
tools/validate_sase_core_rs_version:250-260 says the opposite in as many words: a
checkout ahead of the published window "is the normal dev state now that the
release-branch reconciler owns the window", editing pyproject.toml there is "the
conscription @plan:202608/core_window_ratchet.md removed", and tools/ratchet_core_window
moves the window at release time on the release branch. The declared window today is
sase-core-rs>=0.31.12,<0.32.0 (pyproject.toml:46). Release sase-core and let the
reconciler ratchet; use the just _core-overrides-arg local-override flow for dev
installs, as the plan section already says correctly. Phase sase-tw.3 of epic sase-tw
states this right and is the model.

Coordination: three phases across two concurrent epics release sase-core — this one
(direct dict-to-QueryRow corpus construction, crates/sase_core_py/src/lib.rs), sase-tw.3
(BeadLinkWire gains a direction field), and sase-tw.5 (relation registry gains direction
sentences, worked examples, and recommended endpoint kinds). They touch different
modules and do not conflict in Rust, but their releases serialize. Check whether either
sase-tw phase has an unreleased core change pending before you release, and do not
hand-edit the version line that all three would otherwise touch.
```

### 4.4 `sase-tt.5`

```text
INTEGRATION: the "byte-identical wire shape" invariant in this phase's plan section is
scoped to composing with sase-tt.4 in either order. It is not a permanent schema freeze.
Phase sase-tw.13 of epic sase-tw adds three fields — relation, artifact, and linked — to
the same agent-catalog wire dict. Write the golden test so that is a deliberate
one-line regeneration with a visible diff, and say so in the test's docstring, so the
next agent reads it as a change-detector rather than a wall.

Two things that make sase-tw.13 cheap if you leave room for them:

- agent_catalog_query_entry's signature is (row, *, project_ref_display=None).
  sase-tw.13 threads a link-facet map through as a second optional keyword-only
  argument. Keep the function keyword-extensible, and keep sase.agents.catalog
  Textual-free.
- The three new fields introduce no new wire types: linked is a bool like the existing
  hidden / dismissed / revivable / attention, and relation / artifact are multi-valued
  like the existing label / text / project. sase-tt.4's direct PyDict-to-QueryRow path
  therefore composes with them; only the golden collides.

sase-tw.13 adds per-row work to this exact function, which costs a measured 408ms over
11,783 rows today. It has been told on its own bead to re-run
tests/perf/bench_artifacts_first_paint.py and hold this epic's ≤400ms Agent target.
```

### 4.5 `sase-tt.8`

```text
INTEGRATION: record §2.1's targets in tests/perf/README.md in a form a later epic can
re-measure against — the exact bench command, the corpus shape it assumes, and the
number each pane actually hit — not prose alone. Phase sase-tw.13 of epic sase-tw joins
artifact-link facets onto agent_catalog_query_entry after this epic lands, and has been
told on its bead to re-run tests/perf/bench_artifacts_first_paint.py and hold the Agent
number this phase certifies. That instruction is only actionable if the recipe you write
is.

Worth measuring while the harness is up: AgentsSnapshot carries an
artifact_links: ArtifactLinksSnapshot loaded by load_agents_snapshot
(agents_data.py:24-56) that this epic's breakdown never itemized. It holds 112
machine-global aggregate rows today; epic sase-tw targets ~1,600+. Record its current
cost so that growth has a baseline to be judged against.

Visual goldens are contended in this window: sase-tj.10.3 rebaselines every artifacts_*
golden and adds six new Agent-pane snapshots, and sase-tw.12 updates the relation
snapshot tests. If this epic's panes changed their loading or empty-state rendering,
rebaseline against master after sase-tj.10 lands and look at each regenerated golden
rather than force-accepting it.
```

### 4.6 `sase-tw` (epic)

```text
INTEGRATION: the plan's "Sequencing note: the Agent pane is still landing" is
incomplete. It names sase-tj.10 only. A second in-flight epic, sase-tt ("Make Artifacts
sub-tab queries fast, starting with the Agent pane",
plan:202608/artifacts_query_performance.md, created 2026-08-25 14:59, 35 minutes before
this epic), rewrites the Agent pane harder than sase-tj.10 does:

- sase-tt.3 (agent-paint) gives load_agents_snapshot a limit parameter and
  AgentsSnapshot a complete flag, paints from a bounded head slice, and moves the
  full-corpus query-index build into a background extension worker.
- sase-tt.5 (entry-projection) rewrites agent_catalog_query_entry and
  compile_artifact_query_index's marshalling, and lands a golden test on the
  agent-catalog wire dict.
- sase-tt.4 (core-corpus) changes how sase-core consumes that wire, and releases
  sase-core.

Read that sequencing note as covering sase-tt as well. sase-tw.13 (agent-pane-filters)
must not start until sase-tj.10 and epic sase-tt have both landed on master; its own
bead carries what changes as a result. sase-tw.3 and sase-tw.5 are not blocked by
sase-tt, but they share sase-core releases with sase-tt.4 — see the note on sase-tw.3.

Nothing else in this epic conflicts with sase-tt. The durability, derivation, citation,
read-payoff, and migration phases touch disjoint code.
```

### 4.7 `sase-tw.3`

```text
INTEGRATION: your plan section correctly says to let the release-branch reconciler
ratchet the pyproject window rather than hand-editing it. Hold that line — it matches
tools/validate_sase_core_rs_version:250-260 and tools/ratchet_core_window.

Two other phases across two concurrent epics also release sase-core: sase-tw.5 (relation
registry gains direction sentences, worked examples, and recommended endpoint kinds) and
sase-tt.4 of epic sase-tt (direct dict-to-QueryRow corpus construction in
crates/sase_core_py/src/lib.rs). Different modules, no Rust conflict, but the releases
serialize. Check whether either has a core change pending before you release. Do not
hand-edit the sase-core-rs version line in this repo's pyproject.toml —  sase-tt.4's
plan section wrongly instructs its agent to do exactly that, and has been corrected by a
note on that bead.
```

### 4.8 `sase-tw.12`

```text
INTEGRATION: this phase updates relation snapshot tests and the relation rail's
rendering under src/sase/ace/tui/widgets/artifacts/. Two other efforts touch the same
goldens in the same window: sase-tj.10.3 rebaselines every artifacts_* PNG golden and
adds six new Agent-pane snapshots, and sase-tt.3 of epic sase-tt may add an "index still
building" affordance to the Agent pane. Rebaseline against master after those land, and
hold this phase's own rule — updated deliberately, not force-accepted. That rule is the
only thing keeping three concurrent rebaselines from silently dropping each other's
changes.

One seam between the two epics: sase-tt.3 lists "relation-panel targets" among the
behaviors that must not regress when the Agent pane paints from a bounded head slice. If
this phase changes what a relation edge carries, the bounded-pass answer for relation
targets belongs to both epics. Verify it after both land rather than assuming either
side covered it.
```

### 4.9 `sase-tw.13`

```text
INTEGRATION: this phase's "Do not start until sase-tj.10 has landed" gate is necessary
but not sufficient. A second in-flight epic, sase-tt ("Make Artifacts sub-tab queries
fast, starting with the Agent pane", plan:202608/artifacts_query_performance.md),
rewrites all four files this phase lists. Wait for epic sase-tt to land on master as
well, and re-read src/sase/ace/tui/widgets/artifacts/agents_data.py and
src/sase/agents/catalog/_query.py before starting — both will have changed shape.

Three consequences.

1. The wire golden. sase-tt.5 lands a test asserting the agent-catalog wire dict is
   byte-identical for a fixed corpus. Adding relation, artifact, and linked changes it.
   That regeneration is expected and deliberate: look at the diff, do not force-accept
   it. No new wire types are introduced — linked is a bool like the existing hidden /
   dismissed, and relation / artifact are multi-valued like the existing label / text /
   project — so sase-tt.4's direct PyDict-to-QueryRow path composes with them.

2. Hold the perf budget. sase-tt.5 shrinks agent_catalog_query_entry, which costs a
   measured 408ms over 11,783 rows today, and sase-tt.8 certifies an Agent-pane
   first-paint target of ≤400ms into tests/perf/README.md. The facet join this phase
   adds is per-row work on that exact function. Build the facet map once per load and
   index it by agent name rather than scanning rows per lookup; run
   tests/perf/bench_artifacts_first_paint.py before and after and report both numbers on
   this bead. If the target cannot be held, say so here rather than quietly moving it.
   AgentsSnapshot already carries the ArtifactLinksSnapshot (agents_data.py:24-56), so
   no new load is needed — but after sase-tt.3 the pane paints from a bounded head
   slice, so check whether the bounded pass populates that field before depending on it.

3. Assert the acceptance counts on a complete snapshot. The stated acceptance —
   "linked:false returns agents with no rows and its count plus linked:true's equals the
   unfiltered total" — is a whole-corpus invariant. After sase-tt.3 the pane paints from
   a bounded head slice with AgentsSnapshot.complete False and builds the full-corpus
   index in a background extension worker, and _filtered_agents_snapshot intersects
   result.matched_row_ids with the loaded snapshot.rows (agents_query.py:452-460).
   Assert that invariant only against complete=True. Otherwise this phase reproduces,
   from a different cause, exactly the "confidently wrong answers" failure it cites
   index-durability as the fix for: a lossy index is not the only way the corpus can be
   incomplete.
```
