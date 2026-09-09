---
tier: epic
title: Show and set the current project from the Admin Center Projects tab
goal: "The Projects tab of the SASE Admin Center says which project is current — in the
  project's own accent color, the same one the top-bar `+<project>` chip uses — and one
  configurable keypress on any row makes that project current, everywhere, at once.

  "
phases:
  - id: core-set
    title: A verified write path for the current project
    depends_on: []
    size: medium
    description: "core-set: add `set_current_project()` to `sase.current_project` —
      eligibility check, provider-exact MRU prefix, single MRU promotion, post-write
      re-resolve — returning a typed outcome, and land `sase project set-current
      <project>` as its first real consumer.

      "
  - id: pane-display
    title: Render the current project in the Projects sub-tab
    depends_on: []
    size: medium
    description: "pane-display: give ProjectsPane a reusable off-thread current-project
      resolve (replacing the seed-only worker), cache it in session state, and render it
      as a `CUR` column marker plus accent-colored name, a `current: +<name>` segment in
      the summary line, and a `Current project:` block in the detail panel.

      "
  - id: keymap-scope
    title: Make every Projects-tab key configurable
    depends_on:
      - pane-display
    size: medium
    description: "keymap-scope: add the `ace.keymaps.projects` scope covering all three
      Projects-tab sub-tabs, following the statistics/glossary pattern (dataclass,
      defaults, schema, loader, binding builder, registry field, pane wiring), and
      render the hints line from the configured keys instead of hardcoded letters.

      "
  - id: set-action
    title: The set-current keypress
    depends_on:
      - core-set
      - keymap-scope
    size: medium
    description: "set-action: bind `set_current_project` (default `c`) in the Projects
      sub-tab, run core-set's write on a thread worker with cheap in-memory pre-checks,
      report every outcome honestly in the status line and a notification, and refresh
      the top-bar chip immediately instead of waiting out its poll.

      "
  - id: docs-visual
    title: Documentation and visual proof
    depends_on:
      - set-action
    size: small
    description:
      'docs-visual: rewrite the "Current project" and "Projects Tab" sections of
      docs/ace.md, document the new keymap scope in docs/configuration.md, add a PNG
      golden that shows the current-project row and detail block, and run the full
      verification lane.'
status: done
proposed_by: bbugyi200.athena.06w
bead_id: sase-qd
create_time: 2026-09-09 19:51:13
---

- **PROMPT:**
  [prompts/202608/projects_tab_current_project.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/projects_tab_current_project.md)
- **BEAD:**
  [sase-qd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-qd/README.md)

# Plan: Show and set the current project from the Admin Center Projects tab

## Context

Epic `sase-pw` gave SASE one **current project**: the first VCS xprompt MRU entry that
resolves to an enabled project. It ships a resolver (`src/sase/current_project.py`), an
18-color accent palette (`src/sase/ace/tui/project_styles.py`), a top-bar `+<project>`
chip, eight consumers that seed their project filter from it, and
`sase project current`.

It deliberately shipped **no way to set it**. Both the docs and the CLI help say so:

```text
$ sase project current -h
The current project is a pure read of the VCS xprompt MRU store
(~/.sase/vcs_xprompt_mru.json). Launch an agent on a project, or on a Patch owned by
that project, to make it current. There is no separate set command.
```

```text
docs/ace.md:3414
ACE has one **current project**: the project you last launched an agent on, derived from
the head of the VCS xprompt MRU store. There is no separate "set current project"
command — launching an agent is what moves it.
```

That is a real gap. The current project decides the first-open project scope of the
Artifacts pane, Statistics, the Repos/Workspaces inventories, the Glossary ring, and the
`+` launch picker. Today the only way to change all six is to launch an agent you did
not otherwise want. And the one screen dedicated to projects — the Admin Center
**Projects** tab — does not mention the current project at all: it renders name, VCS
kind, state, claims, workspace/repo counts, and warnings, and nothing about which of
those rows is the one every other surface is scoped to.

### Why this is not a large change

The MRU store already has a public, canonicalizing, self-pruning write path:
`record_vcs_xprompt_usage()` in `src/sase/history/vcs_xprompt_mru.py`. Promoting a
project's prefix to the MRU head is _exactly_ what launching an agent on it does. So
"set the current project" needs no new store, no new file format, and no new
invalidation logic — just a correct call to a function that already exists.

Verified against this workspace's real project records with an isolated MRU file:

```text
before: CurrentProject(project_key='gh_sase-org__sase', display_name='sase', ...)

record_vcs_xprompt_usage("#gh:actstat")            # alias/display spelling
entries: ['#gh:gh_bbugyi200__actstat', '#gh:gh_sase-org__sase', '#gh:gh_bobs-org__bob-cli']
after:  CurrentProject(project_key='gh_bbugyi200__actstat', display_name='actstat', ...)

record_vcs_xprompt_usage("#gh:gh_bobs-org__bob-cli")
entries: ['#gh:gh_bobs-org__bob-cli', '#gh:gh_bbugyi200__actstat', '#gh:gh_sase-org__sase']
after:  CurrentProject(project_key='gh_bobs-org__bob-cli', display_name='bob-cli', ...)
```

The alias form was canonicalized to the directory key, the entry moved to the head, and
`resolve_current_project()` returned the new project on the very next call. No cache to
bust: the resolver re-reads the file every time, and the top-bar chip's poll token is
`(mtime_ns, size)` of that same file.

### The one way this silently fails

The same probe, with a workflow tag that does not match the project's real provider:

```text
record_vcs_xprompt_usage("#git:gh_sase-org__sase")
entries:         ['#gh:gh_bobs-org__bob-cli', '#gh:gh_bbugyi200__actstat', '#gh:gh_sase-org__sase']
current project: bob-cli        # unchanged
```

`record_vcs_xprompt_usage` ran `_vcs_prefix_provider_mismatched`, found the project's
ProjectSpec detects as `gh` rather than `git`, and **dropped the write on the floor with
no error**. `_is_stale_known_project_prefix` does the same for a project that is not
launchable. A naive implementation that guesses the workflow tag, or that lets a user
target a disabled project, will therefore report success while nothing happened — the
single worst outcome for a feature whose entire job is to tell you which project is
current.

That failure mode drives two hard requirements in the design below: derive the workflow
tag from the project's _detected_ provider, and **verify the write by re-resolving**
rather than assuming it took.

## Design

### One idea

Making a project current is the same act as launching an agent on it, minus the launch.
It promotes that project to the head of the VCS xprompt MRU — one store, one truth, and
every existing consumer picks it up with no new plumbing.

### What this deliberately does not build

A separate pin (`~/.sase/current_project.json`) that overrides the MRU head was
considered and rejected. It would:

- create a second source of truth that `resolve_current_project` must merge, which is
  precisely what `sase-pw` designed the module to avoid ("this module adds no second
  store and no write path");
- make the chip lie: pin `sase`, then launch three agents on `bob-cli`, and the top bar
  still claims `sase` is current while every launch says otherwise;
- introduce an unpin concept, an unpin key, an unpin CLI flag, and a rule for what
  happens when the pinned project is later disabled or deleted.

Promotion has none of those problems, and it is strictly more honest: after pressing the
key, "the project you most recently chose to work on" and "the current project" are the
same sentence again.

The cost is real and worth stating: promotion also puts that project at the head of the
prompt bar's `<ctrl+p>` VCS-prefix cycle. That is the correct coupling — both surfaces
answer "what am I working on now?" — and it is documented rather than hidden.

### The write path

```python
@dataclass(frozen=True, slots=True)
class SetCurrentProjectOutcome:
    status: Literal["set", "unchanged", "ineligible", "unverified"]
    project: CurrentProject | None   # the resolver's post-write answer
    message: str                     # one user-facing sentence, always populated
```

`set_current_project(project_key, *, projects_dir=None) -> SetCurrentProjectOutcome`
runs, in order:

1. **Resolve the record.** Look the key up through the same alias map the resolver uses,
   so a display name, an alias, or a directory key all work.
2. **Eligibility.** The project must exist, be `enabled`, and be launchable, and its
   ProjectSpec must yield a workflow type from
   `sase.workspace_provider.detect_workflow_type`. Any failure returns `ineligible` with
   a message that names the specific reason and the fix
   (`"sase is disabled; enable it first"`, `"sase has no launchable ProjectSpec"`).
   Nothing is written. Using `detect_workflow_type` on the resolved ProjectSpec — rather
   than the wire record's `vcs_kind` — is deliberate: it is the identical call
   `_vcs_prefix_provider_mismatched` makes, so a prefix built from it can never be the
   silently-dropped write reproduced above. (They agree for every project in this
   workspace today; the point is that they cannot disagree by construction.)
3. **Short-circuit.** If `resolve_current_project()` already returns this project key,
   return `unchanged` and write nothing. This matters beyond politeness:
   `record_vcs_xprompt_usage` calls `_save_vcs_xprompt_mru` unconditionally, so a
   redundant "set" would rewrite the file and bump its mtime, invalidating the top-bar
   chip's poll token and forcing every ACE instance on the host into a needless resolve.
4. **Write.** One `record_vcs_xprompt_usage(f"#{workflow_type}:{project_key}")` call. No
   new persistence code, no second file, no lock: the MRU module already owns
   canonicalization, dedup, the 100-entry cap, and the `OSError`-swallowing save.
5. **Verify.** Call `resolve_current_project()` again. If it returns the requested key,
   `set`. If it returns anything else — a Patch entry that outranks the new head, a
   project whose workspace directory vanished between steps — return `unverified` with
   the project the resolver actually chose, and say so. The caller never claims a
   success the resolver does not agree with.

Every branch is total and every branch carries a sentence a human can act on. Callers
render `outcome.message`; they do not re-derive prose from `status`.

### What the Projects tab shows

Three surfaces, all keyed on one color:
`project_accent(project_key, among=enabled_keys)` — byte-identical to the accent the
top-bar chip and `sase project current` already use. That shared color is the whole
point. A user who has seen `+sase` in orange at the top of the screen finds the orange
row instantly.

**1. Summary line** (`#projects-summary`). Today:

```text
enabled:3  ·  disabled:0  ·  marked:0
```

Gains one segment, rendered as a miniature of the top-bar chip — `+` in `dim <accent>`,
the display name in `bold <accent>`:

```text
enabled:3  ·  disabled:0  ·  marked:0  ·  current:+sase
```

Before the first resolve lands it reads `current:…` in `dim`; when nothing resolves,
`current:none` in `dim`.

**2. A `CUR` column** in the fixed-width row table, three characters wide, inserted
between `MARK` and `NAME`:

```text
MARK CUR NAME                                VCS   STATE        CLAIMS  WS   REPOS  WARN
         actstat (gh_bbugyi200__actstat)     gh    ● enabled    0       2    3      -
[✓]      bob-cli (gh_bobs-org__bob-cli)      gh    ● enabled    1       1    2      -
     +   sase (gh_sase-org__sase)            gh    ● enabled    24      31   6      -
```

The current row gets `+` in `bold <accent>` and renders its **name** in `bold <accent>`
instead of plain `bold`; every other row is unchanged. Two reinforcing signals — a
column you can scan and a color you cannot miss — and the row total goes from 85 to 88
characters, comfortably inside the ~100 columns the pane has at 120x40.

A dedicated column rather than a glyph glued to the name because the table's alignment
is load-bearing, and separate from `MARK` because a project can be both marked and
current. `+` rather than `★` or `●` because `+<project>` is already this feature's icon:
the top-bar chip, the launch picker, and `sase project current`'s human output all use
it.

**3. Detail panel.** The header line gains a trailing `+CURRENT` badge in
`bold <accent>`:

```text
sase (gh_sase-org__sase)   ● enabled    VCS: gh    +CURRENT
```

and a dedicated line lands below the aliases block, in the pane's established `#87D7FF`
hint style:

```text
Current project: yes  ·  via #gh:gh_sase-org__sase
Current project: yes  ·  via Patch fix-flaky-retry (#gh:fix-flaky-retry)
Current project: no   ·  press c to make bob-cli current
Current project: no   ·  enable widgets first (a), then press c
Current project: no   ·  widgets has no launchable ProjectSpec
```

This line is where "intuitive" is actually won: for **every** row, whether it is current
or not, the panel states the fact and — when there is one — the exact key that changes
it, or the exact reason there is no such key.

### Data flow

`ProjectsPane` already runs a one-shot `resolve_current_project` worker, but only to
seed the two inventory filters, and it discards the result. That worker becomes a
reusable current-project resolve returning `(CurrentProject | None, accent)` computed
off-thread — the same shape and the same reasoning as `_resolve_snapshot` in
`src/sase/ace/tui/widgets/current_project_indicator.py`, because
`project_accent(..., among=...)` needs `get_known_project_workspaces()`, which is
disk-backed and must never touch the UI thread.

It runs on mount, on `R`, and after a successful set. Filter seeding still happens
exactly once and still honors `ace.current_project.seed_filters`; the display state
updates every time. When seeding is off, the pane now resolves once anyway — one bounded
off-thread call per pane open, against a top-bar chip that already peeks every five
seconds.

`ProjectsSessionState` gains `current_project_key`, `current_project_name`, and
`current_project_accent` so a second open of the Admin Center paints the last known chip
immediately and then corrects it, matching the cached-first-then-revalidate rule the
rest of the TUI follows.

The write itself never runs in the key handler. The handler does only in-memory checks
against the already-loaded `ProjectRecordWire` (state, `launchable`, selection), sets a
status, and starts a thread worker; the worker calls `set_current_project()` and the
completion handler applies the outcome. Rule 1 of the TUI performance memory, and the
reason this action does not follow the synchronous shape the neighboring enable/disable
actions still use.

### Keeping the top bar honest

The MRU write bumps the file's mtime, so `CurrentProjectIndicator` would notice within
its five-second poll. Five seconds of a top bar contradicting the panel you just used is
exactly the kind of thing that makes a feature feel unreliable, and the 0.5-second stat
floor in `peek_current_project_change_token` means even an immediate `refresh()` can
miss the change.

So the indicator gains one public method — `invalidate()` — that clears its cached token
and schedules its existing off-thread resolve unconditionally, and the pane calls it
(guarded, best-effort, from the worker's completion handler on the UI thread) after a
`set` outcome. One hop, no new message type, and the caller lands in the same phase as
the method so there is never an unconsumed public symbol.

What deliberately does **not** happen on a set: the Repos and Workspaces filters do not
re-scope, and no other open surface re-seeds. That is the existing documented rule for a
mid-session launch, and a set is a launch minus the launch.

### The keymap

The default binding is **`c`** — "current" — in the Projects sub-tab. It is free in
`ProjectsPane.BINDINGS`, free in `ConfigCenterModal.BINDINGS`, and it fits the pane's
existing mnemonic grammar (`m`ark, `e`dit, `a`liases, `d`isable, `r`epos, `w`orkspaces).

`+` was considered, for its match with the chip, and rejected: it is a shifted key, and
at app level `plus` already means "open the launch picker", so the same keystroke would
do two different things depending on which surface has focus.

**The binding is configurable**, via a new `ace.keymaps.projects` scope built exactly
like the existing `statistics` and `glossary` scopes. This is a deliberate widening of
the literal request and the reviewer should weigh it:

- In SASE, "a new keymap" means an `ace.keymaps.*` entry. A hardcoded `BINDINGS` tuple
  is a binding, not a keymap.
- A scope containing only `set_current_project` would be a trap. The immediate next
  question is "then why can't I rebind `d`?", and the answer would be "because nobody
  needed it yet".
- The hints line has to become keymap-driven the moment _any_ of its keys is
  configurable, or it starts telling users to press keys that no longer do anything.
  Doing that for one key while hardcoding nineteen others produces a line that is only
  partly true.

The scope covers all three Projects-tab sub-tabs, not just the list. `focus_filter`,
`jump_to_entry`, `reload`, and the sub-tab cycle keys exist on the Projects list _and_
on the Repos/Workspaces inventory panes; configuring them for one and not the others
would ship a genuine inconsistency — `/` on two sub-tabs and something else on the
third.

**If the reviewer would rather not take the scope**, the alternative is small and clean:
drop the `keymap-scope` phase entirely, add
`("c", "set_current_project", "Set Current")` to `ProjectsPane.BINDINGS`, and hardcode
`c current` in the hints string. The `set-action` phase then depends only on `core-set`
and `pane-display`, and the epic shrinks from five phases to four. Nothing else in this
plan changes.

### Feature flag

None. Per `sase/memory/sase_flags.md`, a flag routes behavior that reaches users _before
it is ready_. Every phase here is user-complete on its own: `core-set` ships a working
CLI with tests, `pane-display` ships correct read-only rendering, and `set-action` ships
the key only once the write path behind it is landed and green. There is no half-wired
state for a flag to shield, and nothing users are meant to choose forever.

### Out of scope

- **Setting the current project to a Patch.** The resolver distinguishes
  `origin="project"` from `origin="patch"` and this feature always writes a project ref.
  Making a specific Patch current belongs to the Patch surfaces, not the Projects tab.
- **A "clear the current project" action.** There is no empty state to write — the MRU
  is a history, not a setting, and truncating it to un-set the current project would
  destroy unrelated cycle order.
- **Migrating the remaining Admin Center panes** (Config, Logs, Procs, Updates,
  XPrompts) to configurable keymap scopes. Same argument as the Projects tab, different
  tabs, and each is independently sized work.
- **The `docs/ace.md` claim that the Projects tab opens with `3`** while
  `config_center_catalog.py` assigns it number 4. Pre-existing, unrelated, and left
  alone so this epic's docs diff stays reviewable.

## Phases

### Durable notes for every phase

- Run `just install` first; these workspaces drift.
- Do **not** edit `CHANGELOG.md`. It is generated by release-please from conventional
  commit subjects, and `tools/validate_changelog` fails the `just check` lane on hand
  edits. Put the description in the commit subject/body.
- Do not land a public symbol whose only callers are tests. Every phase below is
  arranged so its new public symbols gain a real consumer in the same phase; no
  `--epic-symbol` entry in the `Justfile` should be needed, and none should be added.
  (The `sase-pw` epic broke `just check` for every agent on this host four separate
  times on exactly this, and then broke `master` when a later cleanup privatized a
  symbol whose real consumer had been masked by a stale whitelist.)
- `just check` is the per-phase gate. The `docs-visual` phase owns `just check-full`,
  which must be run through `/sase_monitor`.

### Phase `core-set` — A verified write path for the current project

In `src/sase/current_project.py`:

- Add `SetCurrentProjectOutcome` (frozen, slots) with the four-status shape from the
  Design section, and `set_current_project(project_key, *, projects_dir=None)`
  implementing the five ordered steps: resolve-through-aliases, eligibility,
  already-current short-circuit, one `record_vcs_xprompt_usage` call, re-resolve verify.
- Reuse the module's existing private helpers (`_project_snapshots` builds the records,
  alias map, and known-project map in one pass) rather than re-reading project records.
- The module docstring currently says "this module adds no second store and **no write
  path**". Update it: it now has exactly one write path, and that path goes through the
  MRU store rather than around it.
- Export both new names in `__all__`.

In `src/sase/main/parser_project.py` and `src/sase/main/project_handler.py`:

- Add the `set-current` subcommand with a positional `project` argument and a
  `-j/--json` flag. A positional, not an option, per `sase/memory/cli_rules.md`: "a
  value that is required for the command to execute belongs in a positional argument".
  Read `cli_rules.md` with `/sase_memory_read` before writing the parser — help text
  quality, alphabetical ordering, and short aliases for every long option are all gated
  there.
- Keep the subcommand list alphabetical everywhere it is spelled out: the `metavar` at
  `parser_project.py:34`, the `_HANDLERS` map, and the usage string at
  `project_handler.py:657` all become
  `{alias,current,disable,enable,list,set-current,set-state,show}`.
- Human output reuses `_print_current_human`'s vocabulary so `set-current` and `current`
  render the same project the same way; print `outcome.message` above it. `--json` emits
  `{"status": ..., "message": ..., "project": <the same payload _current_json_payload builds, or null>}`.
- Exit codes: `set` and `unchanged` exit 0; `ineligible` and `unverified` exit 1 with
  the message on stderr. A caller scripting this must be able to tell "it is current
  now" from "it is not".
- Fix all three places that assert no set command exists — `parser_project.py:90` (the
  `current` subcommand's help epilog), `docs/cli.md:190-192`, and
  `docs/ace.md:3414-3416`. Each should now point at `sase project set-current` and,
  forward-referencing since `set-action` lands it, note that ACE binds the same
  operation in the Projects tab. `docs/ace.md`'s full rewrite belongs to `docs-visual`;
  this phase only retires the false claim, so the tree is never self-contradictory
  between phases.
- Add a `sase project set-current` row to the subcommand table in `docs/cli.md` (line
  166 onward), keeping it alphabetical with the surrounding rows.

Tests (`tests/test_current_project*.py`, `tests/main/test_project_handler_current.py` or
a sibling), all against a temp MRU via the `sase.history.vcs_xprompt_mru._MRU_FILE` hook
and a temp projects dir:

1. Promotion moves the resolver: seed a two-entry MRU, set the tail project, assert the
   outcome is `set` and `resolve_current_project()` returns it.
2. Alias and display-name spellings resolve to the same directory key and produce one
   canonical entry.
3. Already-current returns `unchanged` **and leaves the MRU file's mtime untouched** —
   assert on `st_mtime_ns`, since this is the property that keeps every ACE instance
   from re-resolving on a redundant press.
4. A disabled project returns `ineligible`, names the reason, and writes nothing.
5. A project whose ProjectSpec detects a provider other than its `vcs_kind` still
   produces a prefix that survives `record_vcs_xprompt_usage` — the regression test for
   the silently-dropped write reproduced in Context.
6. `unverified`: make `resolve_current_project` return a different project after the
   write (a monkeypatched resolver, or a Patch-named entry that outranks the head) and
   assert the outcome reports the project the resolver actually chose.
7. CLI: `set-current` on an enabled project exits 0 and its output names the project; on
   a disabled one exits 1; `--json` round-trips through `json.loads` with the documented
   keys; `sase project set-current -h` renders.

### Phase `pane-display` — Render the current project in the Projects sub-tab

No behavior changes; this phase is read-only rendering plus the state that feeds it.

In `src/sase/ace/tui/modals/project_management_rendering.py`:

- Add `_CUR_WIDTH = 3` and thread a `current_project_key: str | None` and
  `current_project_accent: str` through `column_header_text()`, `record_label()`,
  `summary_text()`, and `detail_text()`. Keep them keyword-only with defaults so the
  functions stay callable from the existing tests while they are updated.
- `column_header_text()` gains a `CUR` column between `MARK` and `NAME`.
- `record_label()` emits `+` in `bold <accent>` in that column for the current row and
  three spaces otherwise, and styles the name `bold <accent>` for the current row.
- `summary_text()` appends the `·  current:` segment described in Design, with the `dim`
  `…` and `none` states.
- `detail_text()` appends the `+CURRENT` badge to the header line and the
  `Current project:` line below the aliases block, covering all five cases from Design
  (current-via-project, current-via-patch, eligible, disabled, not launchable). Take the
  origin and MRU ref from the `CurrentProject` value, not from a re-derivation.
- `hints_text()` is untouched here; `keymap-scope` rewrites it.

In `src/sase/ace/tui/modals/projects_pane.py`:

- Replace `_maybe_start_current_project_seed`'s worker body with a module-level function
  returning a small frozen snapshot of `(CurrentProject | None, accent)`, resolving both
  off-thread — `resolve_current_project()` then
  `project_accent(key, among=get_known_project_workspaces())`, wrapped so any exception
  degrades to "unknown" rather than killing the worker. Mirror
  `current_project_indicator._resolve_snapshot`.
- Split the completion handler in two: `_apply_current_project_display` (always, every
  resolve) and the existing `_apply_current_project_seed` gating on
  `project_filter_seeded` and `seed_filters`. Start the worker unconditionally on mount,
  and again from `action_reload_projects`.
- Hold `_current_project`, `_current_project_accent`, and `_current_project_loaded` on
  the pane; feed them to the four rendering calls; re-render through the existing
  `_refresh_options` / `_update_summary` / detail-debouncer paths rather than adding a
  new refresh route.

In `src/sase/ace/tui/modals/config_center_session.py`, add `current_project_key`,
`current_project_name`, and `current_project_accent` to `ProjectsSessionState`; seed the
pane's state from them in `__init__` so a reopen paints instantly, and write them back
on every successful resolve.

The Projects-tab display intentionally ignores `ace.current_project.indicator`. That
setting hides the ambient top-bar chip; the Projects tab is the surface you open when
you want to know. Note it in the `pane-display` commit body; `docs-visual` documents it.

Tests: extend `tests/ace/tui/test_projects_pane_current_project_seed.py` and
`tests/ace/tui/test_projects_pane.py`, plus the rendering module's own tests, for the
header/row/summary/detail output in all five detail cases, the `dim` pre-resolve state,
the session-state round trip, and the case where the resolved project is not in the
pane's record list (system-managed, or filtered out) — the summary segment must still
render, and no row is marked.

The row layout change drifts the four checked-in Projects goldens
(`config_center_projects_tab`, `_detail`, `_marked`, `_disabled`). Regenerate them here
with `just test-visual --sase-update-visual-snapshots` so the tree stays green; the new
current-project golden belongs to `docs-visual`.

### Phase `keymap-scope` — Make every Projects-tab key configurable

Follow `statistics` end to end; it is the closest analogue and every seam already
exists.

- `src/sase/ace/tui/keymaps/app_keymaps.py`: add `ProjectsPaneKeymaps` with a field per
  Projects-tab action. From `ProjectsPane.BINDINGS`: `next_option`, `prev_option`,
  `focus_filter`, `cycle_subtab`, `cycle_subtab_reverse`, `toggle_project_mark`,
  `clear_project_marks`, `edit_project_spec`, `edit_project_aliases`, `enable_project`,
  `disable_project`, `delete_project`, `force_current_state_change`,
  `default_project_action`, `reload`, `show_project_repos`, `show_project_workspaces`,
  `jump_to_entry`. From `ProjectInventoryPaneBase.BINDINGS`: `pick_project`,
  `clear_project_filter`. Plus the new `set_current_project: "c"`. `focus_filter`,
  `jump_to_entry`, `reload`, and the two cycle keys are shared across all three sub-tabs
  — one field each, not one per widget. Preserve today's multi-key alternates
  (`j`/`down`/`ctrl+n`) using the scope loader's comma-separated form.
- `src/sase/default_config.yml`: an `ace.keymaps.projects` block with every field, keyed
  identically, with a comment matching the `statistics`/`glossary` ones ("Focused Admin
  Center Projects-tab bindings. These keys are inactive everywhere else, even when they
  overlap app-level actions.").
- `src/sase/config/sase.schema.json`: a `projects` object under
  `properties.ace.properties.keymaps.properties`, one described property per field,
  matching the `statistics` block's shape.
- `src/sase/ace/tui/keymaps/defaults.py`: `load_builtin_projects_defaults()`.
- `src/sase/ace/tui/keymaps/scopes.py`: `load_projects_keymaps()` via
  `_load_scope_keymaps`.
- `src/sase/ace/tui/keymaps/bindings.py`: `_PROJECTS_BINDING_META` and
  `build_projects_bindings()`, plus `projects_help_bindings()` **only if** the hints
  line consumes it — do not land an unconsumed helper.
- `src/sase/ace/tui/keymaps/types.py` / `registry.py` / `__init__.py`: a `projects`
  field on `KeymapRegistry`, loaded in `load_keymap_registry`, exported.
- `src/sase/ace/tui/modals/projects_pane.py` and `project_inventory_pane_base.py`:
  accept a `keymaps` argument, build `BindingsMap(build_projects_bindings(...))` in
  `__init__` the way `StatisticsPane` does, and pass the same instance down to both
  inventory panes so all three sub-tabs share one configured set.
  `_PROJECT_ONLY_ACTIONS` must keep gating by action name, which does not change.
- `src/sase/ace/tui/modals/config_center_catalog.py`: `_projects_pane_factory` reads
  `registry.projects` off `modal.app._keymap_registry`, exactly as
  `_statistics_pane_factory` does, and tolerates its absence.
- `_ProjectsFilterInput.on_key` currently hardcodes `left_square_bracket` /
  `right_square_bracket` to forward sub-tab cycling out of the filter box. Drive it from
  the configured `cycle_subtab` / `cycle_subtab_reverse` keys, honoring comma-separated
  alternates, or rebinding the cycle keys silently breaks cycling while the filter has
  focus.
- `hints_text()` takes the keymaps and renders every key through
  `sase.ace.tui.keymaps.key_display_name`, and gains `set_current_project`'s key with
  the label `current`. The line already overflows 120 columns; keep the new segment
  short and place it next to the other single-row actions rather than at the end.

Tests: a default-config round trip asserting every field matches today's hardcoded key
(the migration must be a no-op for anyone who has not configured anything); an override
that rebinds `set_current_project` and asserts the new key fires and the hints line
shows it; an invalid-key and a duplicate-key override reverting to default with a
warning (the shared `_load_scope_keymaps` behavior, asserted once for this scope); a
rebound `cycle_subtab` still cycling from inside the filter input; and whatever
`tests/test_keymaps_display_help.py` and the config-schema tests require for a new
scope.

### Phase `set-action` — The set-current keypress

In `src/sase/ace/tui/modals/project_management_actions.py`, add
`action_set_current_project`:

- Read the selected record. No selection → `"No project selected"` status, no worker.
- Pre-check in memory, from the record already loaded: not `enabled` → status and a
  `warning` notification naming `a` to enable; `launchable` false → status naming the
  reason. No disk reads, no worker. These duplicate `core-set`'s eligibility check on
  purpose: the point is to answer instantly and never spawn a worker for a press that
  cannot succeed. `core-set` remains the authority; the pane is an optimization.
- Already the current project (compare against `pane-display`'s `_current_project`) →
  `"<name> is already current"`, no worker.
- Otherwise: set `"Making <name> current…"`, then
  `run_worker(..., thread=True, exclusive=False, group="current-project-set")` calling
  `set_current_project`.
- On completion, on the UI thread: apply `outcome.message` to the status line, notify
  with severity `information` for `set` / `unchanged`, `warning` for `ineligible`,
  `error` for `unverified`; on `set`, restart the `pane-display` resolve worker so the
  row marker, summary segment, and detail block all follow, and call the indicator's
  `invalidate()`. Re-read the selection before applying — a pump-free interval can move
  it while the worker runs.

In `src/sase/ace/tui/widgets/current_project_indicator.py`, add `invalidate()`: clear
`_cached_token`, then `_schedule_resolution_if_needed()`. It must bypass the token
comparison (that is the entire point — the 0.5-second stat floor can still be serving a
pre-write token), and it must be a no-op while a resolve is already in flight.

The pane reaches the indicator with a guarded
`self.app.query_one(CurrentProjectIndicator)` inside `try/except`; the chip is absent in
tests and when `ace.current_project.indicator` is false, and a missing chip must never
surface an error from a successful set.

Bind `set_current_project` in `keymap-scope`'s dataclass — the field already exists by
this point, so nothing needs rebinding here.

Tests: each of the four outcomes driving the right status text, notification severity,
and re-render; the disabled and non-launchable pre-checks proving no worker starts; a
set that lands updating the row marker, the summary segment, and the detail block;
`invalidate()` forcing a resolve that a plain `refresh()` would have skipped because the
token had not aged past the stat floor; and the guarded call surviving a missing
indicator.

### Phase `docs-visual` — Documentation and visual proof

- `docs/ace.md`, **Current project** section: it currently opens by asserting the
  feature this epic adds does not exist. Rewrite it around the actual model — the
  current project is the head of the VCS xprompt MRU; launching an agent moves it; the
  Projects tab's `c` and `sase project set-current` move it the same way, by promoting
  the project to that head — and state the visible consequence, that the promoted
  project also becomes the head of the prompt bar's `<ctrl+p>` cycle. Note that the
  Projects-tab display ignores `ace.current_project.indicator`, and that a set does not
  re-scope already-open surfaces, matching the existing rule for a mid-session launch.
- `docs/ace.md`, **Projects Tab** section: add `c` to the key table with a one-line
  description; describe the `CUR` column, the accent-colored name, the summary segment,
  and the detail block; state the enabled-and-launchable precondition.
- `docs/configuration.md`: document `ace.keymaps.projects` alongside the other scopes,
  and tighten the `ace.current_project.indicator` description to say it governs the
  top-bar chip only.
- Visual: add `test_config_center_projects_current_png_snapshot` to
  `tests/ace/tui/visual/test_ace_png_snapshots_config_center_projects.py` with the
  current project selected, so the golden captures the `CUR` marker, the accent name,
  the summary segment, and the `+CURRENT` detail badge in one frame. The Projects
  fixtures stub project records via `_patch_project_records`; stub
  `resolve_current_project` the same way so the golden is deterministic and does not
  read the developer's real MRU. Confirm `pane-display`'s regenerated goldens are still
  current.
- Run `just check-full` through `/sase_monitor` per this repo's two-speed rule, with a
  `--next` action that acts on the result. Confirm `sase bead epic-symbols <epic>`
  reports no leftover entries before the epic closes.
