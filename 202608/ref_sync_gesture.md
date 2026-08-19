---
tier: tale
title:
  Add the `@<kind>::` ref-sync gesture so the `@` payload menu can refresh its backing
  sidecar on demand
goal:
  Typing a second `:` right after `@<kind>:` in the ACE prompt consumes that keystroke,
  syncs (or first-clones) the sidecar backing that kind, rebuilds the completion
  catalog, and reopens the payload menu with the newly arrived rows visibly badged —
  with a live, animated in-panel status the whole time and a bounded, non-blocking,
  never-interactive failure path.
size: medium
proposed_by: bbugyi200.athena.07i
create_time: 2026-08-19 08:09:08
status: wip
---

# Plan: the `@<kind>::` ref-sync gesture

## 1. Symptom

An agent commits a new Markdown file to the research sidecar. Seconds later the user
types `@research:` in the ACE prompt input and the new file is **not** in the payload
menu. The same happens for `@plan:` and `@bead:` after another agent writes to those
sidecars.

## 2. Root cause — three independent staleness sources

Each is real and each must be addressed by the gesture, not just the first one.

1. **The sidecar clone is behind.** `@<document-kind>:` payload rows are read from the
   workspace-local sidecar checkout at `store.kind_root(<role>)`
   (`src/sase/artifact_ref_context.py:47-79`). Nothing in the completion path fetches;
   the clone only integrates when some other code path calls
   `ensure_sdd_kind_clone(...)`, which in turn is TTL-gated: `_pull_sdd_clone` returns
   early when `integration_is_fresh(...)` is true, and the default TTL is **120 s**
   (`src/sase/sdd/_integration_marker.py:12`). A commit pushed by another agent is
   invisible until some caller pulls.

2. **The sidecar clone may not exist at all in this workspace.** Verified in a live
   workspace: `sase repo list` reports `research` as `CLONED ✗` in 22 of 24 workspaces,
   and `store.kind_root("research")` resolves to `sase/repos/research` with
   `is_dir() == False` and remote `git@github.com:sase-org/sase--research.git`.
   `@research:` is a **pointer** document kind, so `materialize_missing_document_roots`
   deliberately skips it at launch
   (`src/sase/artifact_ref_prompt_materialize.py:70-76`), and
   `docs/artifact_references.md` states that discovery surfaces "remain read-only and
   never clone a missing sidecar. Materialize the role explicitly when one of those
   surfaces needs a local inventory." Today the only way to do that is to leave the TUI
   and run `sase repo path research --ensure`. In such a workspace `@research:` shows
   **zero** rows, which reads exactly like staleness.

3. **The in-process completion catalog is cached for the session.**
   `ArtifactRefHighlightMixin` warms one `ArtifactRefCompletionCatalog` per project into
   `_artifact_ref_completion_catalogs_by_project` and never re-warms it while the app
   lives unless `invalidate_artifact_ref_completion_cache()` fires on a config change
   (`src/sase/ace/tui/widgets/_artifact_ref_highlight.py:249-289, 350-360`). Even after
   the clone is up to date, the menu keeps serving the first snapshot.

`tools/select_tests`-visible fact worth stating plainly: the fix must touch all three.
Pulling without dropping the catalog changes nothing on screen; dropping the catalog
without pulling re-reads the same stale files.

## 3. Design

### 3.1 The gesture

`@<kind>:` + `:` → **refresh this kind's sources now, then show me the menu.**

The second `:` is intercepted at the key event and **never enters the buffer**, so there
is no flicker, no transient invalid reference, no highlight churn, and no undo-stack
entry to clean up. This satisfies "the second colon is immediately deleted" more
strictly than deleting it after the fact would.

The gesture fires only when **all** of these hold. Everything else inserts a literal
colon exactly as today:

| Guard                                                                                            | Why                                                                                                                        |
| ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `event.character == ":"` and `self._vim_mode == "insert"`                                        | It is a deliberate keystroke, not a paste, macro, or normal-mode key.                                                      |
| The prompt bar's `_mode` is `"prompt"`                                                           | Feedback/approve bars do not own artifact refs.                                                                            |
| Rust reports a **payload-stage** `@` context at the cursor                                       | Rust owns the reference grammar, the `@` cursor policy, and literal-zone exclusion — do not re-derive any of it in Python. |
| The payload span is **empty** (`replacement_start == replacement_end`) and the cursor sits on it | `@research:                                                                                                                | foo`and`@file:default:`keep inserting a literal colon, so the two-colon`@file:<source>:<digest>` grammar is untouched. |
| `context.kind` is in the warm known-kind set for the target project                              | The gesture is only offered for a kind that actually exists; an unknown kind still types literally.                        |
| Feature flag `ref_sync_gesture` is enabled                                                       | See §3.6.                                                                                                                  |

Note what this design does **not** require: no new Rust binding, no wire-schema bump, no
change to `sase-core`. Detection is a _probe of the existing Rust-owned context_ — the
widget already calls `detect_artifact_ref_completion_context` (which delegates to
`at_reference_context`) on every keystroke, and that call already excludes inline code,
fenced code, and disabled xprompt regions. The trigger is one predicate over its result.
The Rust-core boundary rule is respected: grammar and cursor policy stay in Rust; only
the _action_ (which is sidecar-clone management, and that is Python-owned) is added
here.

Deliberately **not** supported: refreshing with a partial query (`@research:auth:`). The
payload grammar allows a colon (`@file:<source>:<digest>`), so a trailing colon after a
non-empty payload cannot be unambiguously claimed. The user's path is `<BS>`-to-empty
then `::`, or reopen the menu after the sync completes and type the filter — the sync
result is cached, so filtering after a sync is instant.

### 3.2 What "sync" means per kind

The mental model is uniform — **`::` always refreshes that kind's sources and always
reopens the menu** — while the work performed scales to what the kind is backed by.

| Kind                                                                                                | Backing                                     | Action                                                                                                                                                  |
| --------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Any document kind with a sidecar role + recorded remote (`plan`, `research`, plugin-provided kinds) | Workspace sidecar clone                     | `ensure_sdd_kind_clone(ws, n, role, strict=True, fresh=True)` — **clones** when the root is missing, **force-pulls** past the 120 s TTL when it exists. |
| `bead`                                                                                              | Beads sidecar                               | Same call with role `beads`.                                                                                                                            |
| Document kind with no recorded remote (in-tree/local store)                                         | Local files                                 | Rescan only.                                                                                                                                            |
| `file`, `agent`, `stitch`/`commit`, `chat`, `bug`                                                   | Local index / hidden clone / pane snapshots | Rescan only (drop the cached catalog and reload).                                                                                                       |

Rescan-only kinds are still worth the gesture: source 3 (the session catalog cache) is
the dominant staleness for `file` and `agent`, and a rescan fixes it.

`agent` deliberately stays rescan-only rather than calling `sync_agents(...)`: that
function _publishes_ the outbox (`src/sase/agents_sync/git_sync.py:50`), and a keystroke
must never trigger a write. Recorded here so a later reader does not "fix" it.

### 3.3 Why this is safe on the keystroke path

`sase/memory/tui_perf.md` rule 11 says completion paths must never spawn subprocesses
that can prompt interactively, and `docs/ace.md` currently promises that in the `@` menu
"typing never launches Git". This gesture is the single, explicit, user-initiated
exception, and it is made safe structurally:

- The keystroke itself does **zero** I/O. It sets state and submits work. Every guard in
  §3.1 is a pure predicate over data already in hand.
- The network leg runs as a tracked session proc (`_submit_session_worker`), per
  tui_perf rule 3 — off the event loop, visible in the proc indicator and Procs tab,
  deduped, and counted at quit.
- Non-interactive and bounded by construction: the reused `ensure_sidecar_sdd_clone`
  path already sets `GIT_TERMINAL_PROMPT=0` (`src/sase/sdd/_store_link.py:515`), runs
  git under `network_git_timeout()` (`src/sase/sdd/_git.py:113`), and takes the store
  write lock with a bounded timeout plus lock-retry (`src/sase/sdd/_git_contention.py`).
  No new git invocation is written for this feature.
- Catalog reload reuses the existing off-thread warm worker; the menu refresh on its
  completion already exists (`_artifact_ref_highlight.py:326-340`).
- The spinner timer is a thin synchronous callback that bumps an integer and repaints
  one `Static`, started only while a sync is live and stopped the moment none is (rule
  2).

### 3.4 Visual design

Everything happens **inside the completion panel**. No toast on success — the panel is
where the user is looking. Layout never jumps: the panel stays open for the whole
gesture, existing rows stay visible and usable, and the status occupies exactly one
pinned, non-selectable row at the top.

**Running** (spinner advances at 10 fps through `⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏`):

```
┌─ @ research · syncing ───────────────────────────────────────┐
│ ⠹  syncing sase--research …                                  │
│ [D] 202608/agentic_ci.md          Agentic CI        · 2d      │
│ [D] 202607/sase_sites_hub.md      Sase Sites Hub    · 3w      │
└──────────────────────────────── ~ fuzzy · 2 of 2 ────────────┘
```

First-ever sync of a missing sidecar says `cloning sase--research …` instead — the wait
is longer and the user should know why.

**Settled, succeeded** (row auto-dismisses after 2.5 s, then the panel is an ordinary
menu):

```
│ ✓  sase--research synced · 3 new                             │
│ [✦] 202608/ref_sync_notes.md      Ref Sync Notes    · 1m      │
│ [✦] 202608/agentic_ci_followup.md Agentic CI II     · 4m      │
│ [D] 202608/agentic_ci.md          Agentic CI        · 2d      │
```

Rows that arrived in this sync swap their source badge `[D] ` for `[✦] ` in the theme
accent colour — **the same 4 cells**, so nothing shifts, and the payoff of the gesture
is immediately legible. The badge persists until the panel closes.

**Settled, failed** (row persists until the panel closes — failures must not vanish):

```
│ ✗  sase--research sync failed · could not reach origin        │
```

Stale rows stay listed and usable underneath. If the panel is already closed when a
failure lands, fall back to one `severity="warning"` toast so the failure is never
silent. Success with the panel closed is silent — the refreshed catalog is the reward.

Subtitle and title composition follow the existing width ladder in
`_prompt_input_bar_completion_panel_labels.py`: the sync segment is prepended to the
normal `~ fuzzy · N of M · ⚠ K not scanned` subtitle and dropped first when narrow.

### 3.5 State machine

One entry per `(project, kind)` on the text area, alongside the existing per-project
catalog caches:

```
idle ──(gesture)──> running ──(worker ok)──> reloading ──(catalog warm)──> settled_ok
                       │                                                      │
                       └──(worker error/timeout)──────────────> settled_error │
                                                                     │        │
                        (2.5 s timer, success only) ─────────────────┴────> idle
```

- A gesture while `running` for the same `(project, kind)` consumes the colon and is
  otherwise a no-op — no second worker, no duplicate-proc toast.
- A gesture while `settled_*` restarts at `running`.
- Panel closed / context changed: the state is dropped for display purposes, but the
  worker runs to completion and the catalog reload still lands (the value persists).
- On completion the originating text area is re-resolved and checked for `is_mounted`
  before any UI mutation (tui_perf rule 4).

### 3.6 Feature flag

`sase flag new ref_sync_gesture --kind sunset` — default **on**, so the user gets the
behaviour immediately, with a one-key rollback for a change to the most-used input
surface.

- `--when-enabled`: a second `:` after `@<kind>:` is consumed and refreshes that kind's
  sources before reopening the payload menu.
- `--when-disabled`: the second `:` inserts literally and no sync is ever triggered
  (today's behaviour exactly).
- `--remove-when`: the gesture has shipped for two minor releases with no report of a
  colon consumed when the user meant to type one.

The flag is read through `current_flags()`, which is a process-memoized snapshot
(`src/sase/feature_flags/snapshot.py:108`) — safe to consult per keystroke.

## 4. Implementation

### 4.1 `fresh` threading in the SDD store (3 files, additive)

`ensure_sidecar_sdd_clone` already accepts `fresh` and forwards it to `_pull_sdd_clone`
to bypass the TTL (`src/sase/sdd/_store_link.py:37-76`), but no caller can reach it. Add
a keyword-only `fresh: bool = False` and forward it, changing no default behaviour:

- `src/sase/sdd/_store_workspace.py`: `ensure_sdd_kind_clone(...)` → forward to
  `ensure_sidecar_sdd_clone(..., fresh=fresh)`, to `ensure_beads_sidecar_clone(...)`,
  and through both self-recursive calls (owner-anchor redirect) and the
  `ensure_workspace_sdd_clone` delegation.
- `src/sase/sdd/_store_workspace.py`: `ensure_beads_sidecar_clone(...)` likewise.
- `src/sase/sdd/store.py`: re-export the parameter on the public `ensure_sdd_kind_clone`
  / `ensure_beads_sidecar_clone` facades.

### 4.2 New domain module `src/sase/artifact_ref_sync.py`

Pure Python, no Textual import, blocking — callers must run it off the UI thread. Keeps
the "what does syncing this kind mean" policy out of the widget and available to any
future CLI surface.

```python
ArtifactRefSyncMode = Literal["clone", "pull", "rescan"]

@dataclass(frozen=True, slots=True)
class ArtifactRefSyncPlan:
    kind: str
    mode: ArtifactRefSyncMode
    role: str | None          # sidecar role, when remote-backed
    label: str                # display label: sidecar repo name, else the kind
    checkout: Path | None

@dataclass(frozen=True, slots=True)
class ArtifactRefSyncOutcome:
    plan: ArtifactRefSyncPlan
    ok: bool
    detail: str = ""          # short, already user-facing on failure

def plan_artifact_ref_sync(context: ArtifactRefContext, kind: str, *, workspace_dir: Path, workspace_num: int) -> ArtifactRefSyncPlan
def run_artifact_ref_sync(plan, *, workspace_dir: Path, workspace_num: int) -> ArtifactRefSyncOutcome
```

- `plan_artifact_ref_sync` resolves the role from
  `context.document_expansion_for(kind)`, plus the explicit `bead → beads` mapping;
  `mode` is `clone` when the role's `kind_root` is not a directory, `pull` when it is
  and `store.remote_url_for_kind(role)` is not `None`, else `rescan`. `label` is the
  remote's repository name (e.g. `sase--research` from
  `git@github.com:sase-org/sase--research.git`), falling back to the role, then the
  kind.
- `run_artifact_ref_sync` calls
  `ensure_sdd_kind_clone(workspace_dir, workspace_num, role, strict=True, fresh=True)`
  for `clone`/`pull` and returns immediately with `ok=True` for `rescan`. Every
  exception is caught and shortened into `detail` (first line, no traceback, ≤ 60 chars)
  — a failed sync must degrade to stale rows, never to a crash.

### 4.3 New widget module `src/sase/ace/tui/widgets/_artifact_ref_sync.py`

`ArtifactRefSyncMixin`, mixed into `PromptTextArea` next to `ArtifactRefHighlightMixin`.
Owns:

- `_artifact_ref_sync_states: dict[tuple[str | None, str], _RefSyncState]` where
  `_RefSyncState` carries `phase`, `plan`, `label`, `detail`, `started_at`,
  `new_payloads: frozenset[str]`, `before_payloads: frozenset[str]`.
- `_artifact_ref_sync_trigger() -> str | None` — the §3.1 predicate; returns the kind.
- `_start_artifact_ref_sync(kind)` — snapshot `before_payloads` from the warm catalog,
  set `running`, start the spinner timer, submit the proc, repaint the menu.
- `_on_artifact_ref_sync_complete(outcome, project, kind)` — UI thread: on success move
  to `reloading`, evict that project's catalog and context, and re-warm; on failure move
  to `settled_error` (or toast if the panel is gone).
- `_finish_artifact_ref_sync_reload(project, kind)` — called from the existing
  warm-worker success handler; diffs the new catalog's payload set for that kind against
  `before_payloads` into `new_payloads`, moves to `settled_ok`, and arms the 2.5 s
  dismiss timer.
- `_artifact_ref_sync_row(project, kind) -> CompletionCandidate | None` and
  `_artifact_ref_sync_new_payloads(project, kind) -> frozenset[str]`.
- Spinner: `set_interval(0.1, ...)` started on the first `running` state and stopped
  when none remains; the callback increments a frame counter and calls the existing
  panel update. Cancel it in the widget's unmount path.

Submission uses the app-level
`_submit_session_worker(proc_type="ref-sync", body=..., on_complete=..., display_name=f"sync @{kind}", dedup_key=f"ref-sync:{project}:{kind}")`.
Because §3.5 already refuses a second gesture while one is running, the duplicate path
is unreachable in practice; pass `duplicate_message` anyway so a race degrades to a
clear message rather than the generic one.

The workspace used for the sync must be resolved **identically** to the catalog warm, or
the sync will pull a different clone than the menu reads. Extract the resolution now
inlined in `_load_known_artifact_ref_kinds` (`_artifact_ref_highlight.py:57-77` —
target-project namespace, else caller workspace, else cwd) into a shared helper and call
it from both.

### 4.4 Wiring

- `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`: in `_on_key`,
  immediately before the existing `'#@'` pre-insert trigger block (which is the
  precedent for this shape), add the `:` branch. On a trigger: `event.stop()`,
  `event.prevent_default()`, `self._start_artifact_ref_sync(kind)`, return. The flag
  check and every §3.1 guard live in `_artifact_ref_sync_trigger()`, so this block stays
  three lines.
- `src/sase/ace/tui/widgets/_artifact_ref_completion_models.py`: add
  `ArtifactRefSyncCompletionMetadata(phase, label, detail, frame)` (frozen, slots) and
  `is_new: bool = False` on `ArtifactRefPayloadCompletionMetadata`.
- `src/sase/ace/tui/widgets/_file_completion_base.py`
  (`_artifact_ref_completion_result`): pass the new-payload set down so
  `_artifact_ref_completion_menu.payload_rows` can set `is_new`, and prepend the sync
  row when one exists for `(project, kind)`. Prepending here means every caller — open,
  refresh-from-cursor, tab — gets it with no further changes.
- `src/sase/ace/tui/widgets/_file_completion_open.py` (`_try_artifact_ref_completion`):
  when a sync row exists, do **not** `_clear_file_completion()` on an empty candidate
  list — the sync row is the panel's content in the not-yet-cloned case, which is
  exactly when the user needs feedback most.
- `src/sase/ace/tui/widgets/_artifact_ref_completion_context.py`
  (`at_reference_leading_match_count`): treat sync metadata like loading metadata
  (returns 0), so `force`-completion never auto-accepts the status row.
- `src/sase/ace/tui/widgets/_file_completion_accept.py` (`_move_file_completion`): when
  `_completion_kind == ARTIFACT_REF_COMPLETION_KIND`, skip non-selectable rows (sync and
  loading metadata) with a bounded loop, and seed `_file_completion_index` at the first
  selectable row. This also closes the pre-existing hole where the "loading files…" row
  could be highlighted and dismissed with `Enter`.
- `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows_artifacts.py`: render the
  sync row (glyph + label + detail, dim, accent glyph while running) and the `[✦]` badge
  for `is_new` payload rows.
- `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel_labels.py`: title
  `@ <kind> · syncing|synced|sync failed` and the subtitle segment, both derived from
  the visible rows so `show_file_completions` needs no new parameters.
- `src/sase/ace/tui/proc_producer_sites.py`: register the new `session_worker` site
  (`ace.ref_sync`, proc type `ref-sync`, classification `ui_only`, identifiers
  `("project", "kind")`, concurrency key `ace:ref-sync:{project}:{kind}`, optimistic UI
  "pinned sync row + spinner", restart recovery "not durable; session-local refresh").
  `tests/ace/tui/test_proc_producer_inventory.py` fails until this exists.
- `src/sase/feature_flags/registry.py`: the entry printed by `sase flag new`.

## 5. Tests

Mirror the existing split-by-focus layout under `tests/ace/tui/widgets/` and `tests/`;
add new focused modules rather than growing existing ones.

**Trigger predicate** (`tests/ace/tui/widgets/test_artifact_ref_sync_trigger.py`) —
table driven over `(text, cursor, expected)`:

- fires: `@research:|`, `@plan:|`, mid-prompt `see @research:|`, after a newline.
- does not fire: `@research:f|oo` and `@research:|foo` (non-empty payload);
  `@file:default:|` (two-colon file grammar); `` `@research:|` `` and a fenced block
  (literal zones, enforced by Rust); `@nosuch:|` (unknown kind); normal/visual vim mode;
  feedback-mode bar; cold known-kind set; flag disabled.
- consumed-key assertion: after the gesture the buffer is byte-identical to before it —
  no `::` ever exists, and the undo stack is unchanged.

**Sync planning/execution** (`tests/test_artifact_ref_sync.py`): `mode` selection for
missing root vs present root vs no remote; `bead → beads`; rescan kinds; label derived
from the remote URL; `ensure_sdd_kind_clone` called with `fresh=True, strict=True`
(assert on a fake); an exception becomes `ok=False` with a short one-line `detail`.

**`fresh` threading** (`tests/test_sdd_store_kind_clone_fresh.py`): `fresh=True` reaches
`ensure_sidecar_sdd_clone` through the direct path, the owner-anchor recursion, and the
beads path; default stays `False`.

**Widget state machine** (`tests/ace/tui/widgets/test_artifact_ref_sync_flow.py`, fake
worker): running → reloading → settled_ok; catalog evicted and re-warmed exactly once;
menu reopened at the same cursor; `new_payloads` diff correct; a second gesture while
running submits nothing; failure yields `settled_error`; failure with a closed panel
yields exactly one warning toast; a text area unmounted mid-sync applies nothing.

**Panel rendering** (`tests/ace/tui/widgets/test_artifact_ref_sync_panel.py`): sync row
content per phase, spinner frame advance, `[✦]` badge on new rows with the badge cell
width unchanged at 4, title and subtitle text, subtitle drop order at narrow widths,
sync row is not selectable and `Enter` on the menu never inserts it.

**Visual** (`tests/ace/tui/visual/test_ace_png_snapshots_at_reference_completion.py`):
one new golden, `at_reference_sync_panel_120x40.png`, of the running state with a pinned
spinner frame and two `[✦]` rows below. Pin the frame index in the fixture so the golden
is deterministic.

**Flag** (both states, as the flags memory requires): enabled → gesture fires; disabled
→ the colon inserts literally and no proc is submitted.

## 6. Docs

- `docs/ace.md`, the "**`@` reference completion**" bullet: document the gesture, the
  badge, the status row, and the flag. The sentence "typing never launches Git, contacts
  a tracker, or performs unbounded filesystem scans" must be amended, not left to rot —
  it stays true for typing, with the explicit `::` gesture named as the one
  user-initiated exception that may fetch.
- `docs/artifact_references.md`, under "On-demand document sidecars": `@<kind>::` is the
  in-prompt equivalent of `sase repo path <role> --ensure`, and is the "explicit
  materialization" that section already tells the reader to perform when a discovery
  surface needs a local inventory. Note that pointer kinds such as `@research:` are
  reachable this way even though they never auto-materialize at launch.
- Do not hand-edit `CHANGELOG.md` (release-please owns it).

## 7. Non-goals

- No `sase-core` change, no new Rust binding, no wire-schema bump.
- No refresh-with-a-partial-query variant (§3.1).
- No `@::` "sync everything" form — unbounded work behind one keystroke.
- No `@agent::` outbox publication (§3.2).
- No new user config key; the flag is the only switch (§3.6).
- No change to the automatic warm/refresh cadence — the gesture is explicit, and adding
  background fetching to the completion path is a separate decision.

## 8. Verification

- `just install` first (ephemeral workspace), then `just check`.
- `just test-visual` after adding the golden; accept it with
  `--sase-update-visual-snapshots` only after eyeballing the artifacts in
  `.pytest_cache/sase-visual/`.
- `just check-full` before landing, run through `/sase_monitor` with a `--next` action —
  it routinely outruns one agent turn.
- Manual smoke in a workspace where the research sidecar is **not** cloned (the common
  case — 22 of 24): open `sase ace`, type `@research::`, confirm the panel shows
  `cloning sase--research …` with a live spinner, that the clone lands, that the menu
  reopens populated with `[✦]` badges, and that `sase repo list` now reports the sidecar
  cloned for that workspace. Then immediately repeat `@research::` and confirm the
  second run reports `synced · 0 new` rather than skipping on the 120 s TTL.
