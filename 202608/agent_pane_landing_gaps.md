---
tier: epic
title:
  Agent pane landing gaps — reachable navigation, a working CLI, and real visual
  coverage
goal: "Close the three defects epic sase-tj shipped: `sase agent search` cannot take its
  own options after a query, the Artifacts Agent pane declares entry navigation it binds
  no key for, and the fast-startup test stub hides the pane from every PNG golden so the
  land phase's required visual rebaseline silently never happened.

  "
phases:
  - id: cli_options
    title: Make `sase agent search` accept its options after the query
    depends_on: []
    size: small
    description: "cli_options: replace the positional QUERY's nargs=REMAINDER, which
      swallows -j/-l/-p into the query text, and add argv-level tests that exercise the
      real parser instead of a hand-built Namespace.

      "
  - id: navigation
    title: Bind j/k entry navigation on the Agent pane and guard the capability gap
    depends_on: []
    size: medium
    description: "navigation: add agents_next/agents_prev over the pane's existing
      move_selection(), register them everywhere the sibling panes register theirs, add
      the conformance check that fails a pane declaring a capability no pane-applicable
      action serves, and report j/k p95.

      "
  - id: visual
    title: Put the Agent pane in the fast-startup inventory and rebaseline the goldens
    depends_on:
      - navigation
    size: medium
    description: "visual: add the agents descriptor to _fast_artifacts_subtabs so the
      whole fast-policy corpus renders the shipped sub-tab strip, rebaseline every
      affected artifacts_* PNG golden, add the six Agent-pane snapshots the parent
      epic's land phase required, and give the mount test a bounded wait so it stops
      flaking under the parallel lane.

      "
proposed_by: bbugyi200.athena.sase-tj.land
parent_bead: sase-tj
status: done
bead_id: sase-tj.10
create_time: 2026-09-09 19:49:46
---

- **PROMPT:**
  [prompts/202608/agent_pane_landing_gaps.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agent_pane_landing_gaps.md)
- **PARENT:** [202608/artifacts_agents_pane.md](artifacts_agents_pane.md)
- **BEAD:**
  [sase-tj.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-tj/sase-tj.10.md)

# Plan: Agent pane landing gaps

## 1. Why this plan exists

Epic sase-tj (`plan:202608/artifacts_agents_pane.md`) shipped the Artifacts → Agent
pane, the `agents` query profile, the Textual-free catalog, and `sase agent search`. Its
final phase, `land` (sase-tj.9), was auto-closed by `sase stitch create` when commit
`e5989fd28` landed. That auto-close note says in its own words that "no verification is
implied", and the phase's plan-required evidence — a monitored `just check-full`, a PNG
golden refresh, six new pane snapshots, and a j/k p95 number on the bead — was never
produced.

Three defects survived into master because of it. Each was confirmed by running the
shipped code on this tree, not inferred:

1. `sase agent search 'kind:family' -l 3` exits 2 with
   `error: Unexpected character: - (at position 61)`. The same query with the flag
   _before_ it works.
2. `sase artifact pane show agents` reports `entry_navigation ON` and `entry_open ON`,
   and its Keys table contains neither `j`/`k` nor `enter`.
   `sase artifact pane show files` lists `files_next j`, `files_prev k`,
   `files_view_selected enter`.
3. `just test-visual -k "artifacts or tab_icon_glyphs"` passes 134 tests against goldens
   that render an Artifacts sub-tab strip with **no Agent pane in it**. The shipped
   strip has zero pixel coverage.

The parent epic bead sase-tj stays open until these land; its land agent resumes the
interrupted landing from this plan's `parent_bead` link.

**Out of scope.** Nothing in the parent epic's §12 "What not to do" list is reopened
here. No new query keys, no `content:`, no link filters, no main-Agents-tab migration,
no `DismissedAgentSelectModal` deletion. This plan repairs what sase-tj shipped and adds
the coverage that would have caught it.

## 2. `cli_options` — `sase agent search` cannot take its own options after a query

### 2.1 The defect

`src/sase/main/parser_agent_search.py:70` declares the positional query as:

```python
search_parser.add_argument(
    "query",
    metavar="QUERY",
    nargs=argparse.REMAINDER,
    help="Boolean agent-catalog query; may include a limit: token",
)
```

`argparse.REMAINDER` stops option parsing at the first positional and captures
everything after it verbatim, flags included. `_query_from_args`
(`src/sase/agents/cli_search.py:178`) then joins the whole list with spaces, so the
flags become query text:

| Command                                   | `args.query`                    | Composed query fragment |
| ----------------------------------------- | ------------------------------- | ----------------------- |
| `sase agent search 'kind:family' -l 3`    | `['kind:family', '-l', '3']`    | `(kind:family -l 3)`    |
| `sase agent search 'kind:family' -j`      | `['kind:family', '-j']`         | `(kind:family -j)`      |
| `sase agent search 'kind:family' -p sase` | `['kind:family', '-p', 'sase']` | `(kind:family -p sase)` |

`_presentation_scoped_query` (`src/sase/agents/cli_search.py:130`) prefixes
`NOT hidden:true AND NOT kind:workflow-child AND `, the tokenizer reaches the `-`, and
the command exits 2. `args.limit`, `args.json`, and `args.project` all stay at their
defaults, so even a query that happened to tokenize would silently ignore the flags.

The reported error position is the arithmetic proof: for `'kind:family' -l 3` the prefix
is 48 characters and `(kind:family ` is 12 more, putting the `-` at 60 and the 1-based
report at 61 — exactly what the command prints.

### 2.2 Why no test caught it

`tests/test_agent_search_cli.py` has four tests. Three of them build the namespace by
hand:

```python
code = handle_agents_search(
    argparse.Namespace(json=True, limit=None, project="sase", query=[...])
)
```

That bypasses `register_agent_search_parser` entirely. The fourth,
`test_agent_search_parser_and_help_are_complete_and_sorted`, only asserts strings in
`format_help()` output — `-j`, `--json`, `-l LIMIT` — which are present regardless of
whether the option can ever be reached. No test has ever pushed an argv list through the
real parser.

### 2.3 The change

- Replace `nargs=argparse.REMAINDER` with `nargs="*"` so argparse interleaves options
  and positionals normally. `_query_from_args` already handles a list; a multi-token
  query still joins with spaces, and a single quoted query still arrives as a one-item
  list.
- Verify a query whose own text begins with `-` (a negated bare word) still reaches the
  parser. Under `nargs="*"` argparse treats a leading-dash token as an option, so if the
  boolean dialect can spell such a query, `--` must be documented as the separator in
  the `--help` epilog. Establish this by test, and only add the epilog line if a real
  spelling exists; the boolean dialect uses `NOT`, not `-key:x`, so this is expected to
  be a non-issue worth one test rather than a feature.
- Keep option order alphabetical and every long option short-aliased per `cli_rules.md`;
  no option becomes required.

### 2.4 Verification

- Argv-level tests through `register_agent_search_parser` (not a hand-built `Namespace`)
  covering `-j`, `-l`, and `-p` in **both** positions — before the query and after it —
  asserting the parsed `Namespace` fields and that `handle_agents_search` returns 0.
- A regression test asserting `sase agent search 'kind:family' -l 3` does not raise a
  tokenizer error, named so its intent survives.
- Keep the existing help-completeness test; add nothing that only checks help text for
  this behavior, because that is the check that already passed while the option was
  unreachable.

## 3. `navigation` — the pane declares entry navigation and binds no key for it

### 3.1 The defect

Every sibling Artifacts pane binds `j`/`k` to a pane-scoped next/prev action:
`stitches_next`, `plans_next`, `beads_next`, `files_next`
(`src/sase/ace/tui/bindings.py:126,142,150,169` and the matching
`src/sase/default_config.yml` keymap entries). The Agents sub-tab section
(`src/sase/ace/tui/bindings.py:181`, `src/sase/default_config.yml:472`) binds only
`agents_revive: "w"`. No `agents_next` or `agents_prev` action exists anywhere in
`src/`.

The pane is not missing the _capability_: `sase artifact pane show agents` reports
`entry_navigation ON` and `entry_open ON`, both derived from `has_inventory`. It is
missing every key that serves them. The widget layer is already there —
`AgentsNavigationMixin.move_selection(offset)`
(`src/sase/ace/tui/widgets/artifacts/agents_navigation.py:160`) is the same shape
`action_beads_next` drives — so nothing but the wiring is absent.

`tests/ace/tui/bench_artifacts_jk.py` documents the gap in a source comment and excludes
`agents.next` / `agents.prev` from its `expected_actions` tuple, pointing at "the filed
bug bead". No such bead exists; `sase bead list -s all -t task -S 2026-08-25` shows
none.

The user-visible consequence is worse than a missing key: the help modal's "Agent Pane"
section (`src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py:183`) splats
`*artifact_list_navigation`, so the pane advertises list navigation it does not have.

### 3.2 Why conformance cannot see it

`check_declared_keys_resolve_to_named_actions`
(`tests/ace/tui/artifacts_contract/harness.py:241`) builds its `declared` tuple as:

```python
declared = tuple(
    action
    for capability in contract.capabilities
    for action in CAPABILITY_HOST_ACTIONS[capability]
    if action_applies_to_contract(contract, action)
)
```

`CAPABILITY_HOST_ACTIONS[ENTRY_NAVIGATION]`
(`src/sase/ace/tui/_artifact_tab_actions.py:18`) holds the other panes' actions plus
`jump_to_entry`. The other panes' actions are filtered out by
`action_applies_to_contract`, `jump_to_entry` resolves, and the loop passes. A pane can
therefore declare any capability, contribute no action to it, and conform. This is the
structural hole, and closing it is the durable half of this phase.

`check_declared_actions_are_registered` (`harness.py:104`) has the same blind spot: it
asserts the capability's action tuple is non-empty globally, never that any entry in it
applies to _this_ contract.

### 3.3 The change

Mirror `files_next` / `files_prev` exactly; do not invent a generic navigator.

- `src/sase/ace/tui/actions/artifacts_agents.py`: add `action_agents_next` and
  `action_agents_prev` on `ArtifactsAgentsActionsMixin`, each resolving `_agents_pane()`
  and wrapping `pane.move_selection(±1)` in `_begin_artifacts_navigation("next"|"prev")`
  / `_finish_artifacts_navigation()`, the way `ArtifactsBeadsBrowseActionsMixin` does
  (`src/sase/ace/tui/actions/_artifacts_beads_browse.py:11`). The begin/finish pair is
  what emits the `SASE_TUI_PERF` paint sample, so it is required, not decorative.
- `src/sase/ace/tui/actions/artifacts_agents.py:21`: add both to
  `AGENTS_ARTIFACT_ACTIONS`.
- `src/sase/ace/tui/bindings.py:180`: add `Binding("j", "agents_next", ...)` and
  `Binding("k", "agents_prev", ...)` in the Agents sub-tab block.
- `src/sase/default_config.yml:471`: add `agents_next: "j"` and `agents_prev: "k"` under
  the Agents sub-tab comment, per the default-keymap gotcha.
- `src/sase/ace/tui/commands/availability.py:153`: add `app.agents_next` and
  `app.agents_prev` to `_AGENTS_ARTIFACT_COMMANDS`.
- `src/sase/ace/tui/_artifact_tab_actions.py:18`: add `agents_next` / `agents_prev` to
  `CAPABILITY_HOST_ACTIONS[PaneCapability.ENTRY_NAVIGATION]`.

**Decide `entry_open` explicitly rather than by omission.** `entry_open` is ON for this
pane and no `enter` binding exists. The Agent pane renders its detail in the split panel
on selection, so an `enter` preview modal may be genuinely redundant — but the contract
currently claims otherwise. Either add `agents_view_selected` bound to `enter` alongside
the sibling panes, or make the pane not declare `entry_open`. Record which was chosen
and why on the phase bead; do not leave the contract asserting a capability with no key,
because §3.4's new check will then fail.

### 3.4 The guard that makes this class of bug impossible

Add a conformance check to `tests/ace/tui/artifacts_contract/harness.py` — parametrized
over every descriptor like its siblings — asserting that for every ON capability that is
not in `_PRESENTATION_ONLY_CAPABILITIES`, **at least one** action in
`CAPABILITY_HOST_ACTIONS[capability]` satisfies
`action_applies_to_contract(contract, action)` and is available on that contract.

Confirm the check is real by running it against the pre-fix tree state: it must fail for
`agents` on `entry_navigation` (and on `entry_open` until §3.3's decision lands) and
pass for every other descriptor. A guard that passes before the fix is not a guard.

The guard is safe to add because `agents` is the only outlier. Every other pane binds
`j`/`k` today, verified by `sase artifact pane show` on each: `patches` →
`next_patch`/`prev_patch`, `stitches` → `stitches_next`/`stitches_prev`, `beads` →
`beads_next`/`beads_prev`, `files` → `files_next`/`files_prev`, and both document
providers (`ref:plan`, `ref:research`) → `plans_next`/`plans_prev`.

### 3.5 Verification

- The new conformance check, plus the existing 12, green for every descriptor.
- A pane test pressing `j` and `k` on the mounted Agent pane and asserting the
  highlighted entry target changes — the assertion the contract could not make before.
- `tests/ace/tui/bench_artifacts_jk.py`: delete the two "no `agents_next` /
  `agents_prev` exists" comments (module-level and beside `expected_actions`), drop the
  "see the filed bug bead" reference, add `agents.next` and `agents.prev` to
  `expected_actions`, and press `j`/`k` in the Agent pane burst alongside the existing
  `g`/`G`/`ctrl+d`/`ctrl+u`.
- Capture j/k p95 with `SASE_TUI_PERF=1` at the full 12,525-row synthetic corpus the
  bench already builds and **report the number on the phase bead**. The parent epic's §5
  target is p95 < 16 ms; this is the measurement its `land` phase owed and never took.

## 4. `visual` — the fast-startup stub hides the pane from every golden

### 4.1 The defect

`_fast_artifacts_subtabs()` (`src/sase/ace/testing/_startup.py:74`) returns a fixed
inventory:

```python
return assign_artifacts_digit_shortcuts(
    (
        fixed_descriptor("stitches"),
        fixed_descriptor("patches"),
        fixed_descriptor("beads"),
        _plan_test_descriptor(),
        fixed_descriptor("files"),
    )
)
```

`_install_fast_startup_overrides` patches `resolve_artifacts_subtabs` to this in
`_artifacts_view`, `_commands_catalog`, and `_keymap_bindings`
(`src/sase/ace/testing/_startup.py:235`). `AcePage`'s startup policy defaults to
`"fast"` (`src/sase/ace/testing/ace_page.py:198`), and the PNG visual harness does not
override it. So the entire fast-policy corpus — including every `artifacts_*` golden —
renders an Artifacts tab whose sub-tab strip has no Agent pane.

The stub's own docstring says its purpose is to "expose a deterministic plan document
provider": it pins the _provider_ descriptors, which vary by machine, and lists the
fixed panes alongside them. `agents` is a fixed pane. Its absence is an oversight, not a
design choice, and the epic worked around it twice rather than fixing it —
`tests/ace/tui/test_agents_pane_mount.py` opts into `startup_policy="real"`, and
`tests/ace/tui/bench_artifacts_jk.py` spins up a whole second `AcePage` because
`page.artifacts_digit("agents")` raises under the fast policy.

The consequence is that the parent epic's `land` phase premise — "the sub-tab strip
gains a pane and Files moves from digit 6 to 7, so every `artifacts_*` snapshot changes
... goldens change exactly once" — was never true in the test harness. Reading
`tests/ace/tui/visual/snapshots/png/artifacts_split_even_120x40.png` shows the strip as
`1 ⊙ Stitch | 2 ⌁ Patch | 3 ◈ BEAD | 4 ✎ Plan | 5 ▤ File`. Production renders an
`⬡ Agent` entry before File.

### 4.2 The change

- Add `fixed_descriptor("agents")` to `_fast_artifacts_subtabs()`, immediately before
  `fixed_descriptor("files")`, matching production order in
  `resolve_artifacts_subtabs()` (`src/sase/ace/tui/artifact_tabs.py:86`).
- Rebaseline the affected goldens with
  `just test-visual --sase-update-visual-snapshots`. Review the regenerated PNGs: the
  only intended differences are the added `⬡ Agent` strip entry, the File digit shift,
  and any footer/hint text that moves with them. **Any other pixel change is a
  regression to investigate, not a diff to accept.**
- Simplify the two workarounds now that the stub is faithful:
  `tests/ace/tui/test_agents_pane_mount.py` and the second `AcePage` in
  `tests/ace/tui/bench_artifacts_jk.py` both exist only because the stub omitted the
  pane. Keep `startup_policy="real"` wherever the test's stated purpose is to exercise
  live resolution (the mount test says so explicitly); drop it where it was pure
  workaround. Update the docstrings and comments that explain the workaround so they do
  not outlive it.

### 4.2.1 Fix the mount test's unbounded wait

`test_agents_pane_mounts_activates_and_loads`
(`tests/ace/tui/test_agents_pane_mount.py`) is the parent epic's own test and it has
failed twice in a full parallel lane while passing every direct rerun — reported
independently as a `PROPOSED FOLLOW-UP` on sase-tj.8 and, by epic sase-ti's land agent,
as note #1 on sase-tj itself.

The cause is visible in the test body: it awaits `page.pause()` and then asserts
`pane.snapshot is not None`, with no bounded wait for the off-thread catalog build.
Under `startup_policy="real"` that build reads the live registry — 12,525 names on this
machine — and `bench_artifacts_jk.py` already documents that the same build "can take
well beyond the default 5s wait budget" under host contention. `page.pause()` yields to
the event loop; it does not wait for a worker thread. Commit `e5989fd28` removed the
test's flag plumbing but left this shape untouched, so the flake is still live.

Replace both `page.pause()`/assert pairs with an explicit `page.wait_for(...)` on the
condition being asserted, with a generous timeout, mirroring the bench's `timeout=30.0`.
Do not weaken the assertions and do not mark the test flaky — the condition is real,
only the waiting is wrong.

Because the failure needs a loaded lane to appear, prove the fix the way the repo
already proves this class — the oversubscribed default-lane harness, whose own Justfile
comment says "One pass is not evidence about a class whose base rate is under one node
per run":

```bash
just test-contention -- tests/ace/tui/test_agents_pane_mount.py
```

Report the per-node failure tally on the phase bead. One clean serial run is not
evidence.

### 4.3 The six snapshots the parent epic owed

Add the Agent-pane PNG snapshots the parent plan's `land` phase named, in a new
`tests/ace/tui/visual/test_ace_png_snapshots_artifacts_agents.py` modelled on
`test_ace_png_snapshots_artifacts_files.py`:

| Snapshot           | What it must show                                                |
| ------------------ | ---------------------------------------------------------------- |
| populated          | rows with kind/state/project/status columns and the detail panel |
| empty              | the `No agents` empty state from the adapter's `PaneEmptyState`  |
| family-grouped     | the default `by_family` banner over member rows                  |
| filter completion  | the profile filter bar mid-completion                            |
| filter parse error | the filter bar's parse-error presentation                        |
| narrow             | the 80x24 narrow layout                                          |

Use deterministic fixtures, not the live registry: the pane must render identically on a
machine with no agent history. Follow `_ace_png_snapshot_fixtures.py` for the pattern.

### 4.4 Verification

- `just test-visual` green, including `test_tab_icon_glyphs.py`'s audit of
  `ARTIFACTS_ICONS` (which already covers `⬡` and already passes — confirm it still does
  through the strip, not only in isolation).
- A flag-free inventory test asserting the fast stub and the production resolver agree
  on the fixed pane order, so the two cannot drift apart again silently.
- Confirm the six new goldens are committed and that `.pytest_cache/sase-visual/`
  artifacts were reviewed, not blind-accepted.

## 5. What not to do

- **Do not accept a regenerated golden without looking at it.**
  `--sase-update-visual-snapshots` will happily bless a real regression. The strip entry
  and the digit shift are the only expected deltas.
- **Do not "fix" the navigation gap by making the pane stop declaring
  `entry_navigation`.** The pane is a navigable list with `has_inventory=True`; the
  capability is correct and the keys are what is missing.
- **Do not add a generic cross-pane next/prev action.** Every other pane is pane-scoped
  for a reason; a unification is a separate change with its own justification.
- **Do not widen `_fast_artifacts_subtabs` into a live resolver.** It is a fixed
  inventory on purpose; it just needs to be the _right_ fixed inventory.
- **Do not test the CLI fix through `format_help()`.** That is the assertion that passed
  while the option was unreachable.
- **Do not file the j/k gap as a task bead.** It is parent-epic-caused and belongs here.

## 6. Verification summary

| Area       | Required evidence                                                                                                                                    |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| CLI        | `-j`, `-l`, `-p` parsed correctly both before and after the query, through the real parser via argv; the exact failing invocation now exits 0        |
| Navigation | `j`/`k` move the Agent pane selection; `sase artifact pane show agents` lists `agents_next`/`agents_prev`; the `entry_open` decision is recorded     |
| Guard      | the new capability-reachability check fails on the pre-fix tree for `agents` and passes for every other descriptor                                   |
| Perf       | j/k p95 at the 12,525-row corpus reported on the phase bead against the parent epic's < 16 ms target                                                 |
| Harness    | fast stub and production resolver agree on fixed pane order, asserted by test                                                                        |
| Visual     | `just test-visual` green; existing `artifacts_*` goldens show the `⬡ Agent` strip entry and shifted File digit; six new Agent-pane goldens committed |
| Flake      | `just test-contention -- tests/ace/tui/test_agents_pane_mount.py` tally reported on the phase bead, with no failing nodes                            |

Every phase runs `just install` first (ephemeral workspaces), then `just check`. The
`visual` phase additionally runs `just test-visual`, which `just check` and
`just check-full` both exclude.
