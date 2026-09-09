---
tier: epic
status: done
title: A link-traversing SASE pager
goal: "SASE has one reading surface. `sase bead show`, the Agents-tab `v` keymap, `sase
  artifact read`, and the new `sase pager` command all render the same `PagerDocument`
  through the same Textual app; every artifact ref, file path, and URL in that document
  carries a pre-painted key; one keypress follows it into another pager document instead
  of dead-ending in `less`; `ctrl+n`/`ctrl+p` puts the next entity's header at row 0;
  and a visible breadcrumb trail walks back with `backspace`.

  "
phases:
  - id: primitives
    title: The allocator and the scanner
    depends_on: []
    size: medium
    description: "primitives: create the beta flag, extend `jump_hints` with prefix-free
      variable-width label allocation over a reserved-key alphabet, and add the one
      `link_scan` module that finds typed refs, URLs, paths, and origin-scoped bare
      tokens in plain text.

      "
  - id: document
    title: A document of sections, not a string
    depends_on:
      - primitives
    size: medium
    description: "document: add `PagerDocument`/`PagerSection`/`PagerOrigin`, ingest
      ANSI bodies through `Text.from_ansi`, accept caller-attached typed targets bound
      to spans, and build the two adapters that turn a bead show batch and a path list
      into sections.

      "
  - id: viewer
    title: The reading surface
    depends_on:
      - document
    size: medium
    description: "viewer: build the `SasePager` Textual app shell — the sticky chrome
      band, section rules, scrolling, `ctrl+n`/`ctrl+p` scroll-to-header, the
      availability-driven footer legend, and the re-hosted `VimSearchController` — wired
      to no caller yet.

      "
  - id: labels
    title: Keys painted ahead of time
    depends_on:
      - viewer
    size: medium
    description: "labels: paint one stable key capsule before every scanned link, match
      keypresses through `normalize_jump_key` and `match_jump_hint`, give pending
      prefixes visible feedback, and ship the window-scoped fallback dormant.

      "
  - id: follow
    title: A press that always lands
    depends_on:
      - labels
    size: medium
    description: "follow: define the `resolve_ref` interface with its CLI-backed
      implementation, open resolved targets as new pager documents rather than escaping
      to a viewer, remember and dim dead ends, delegate media to `graphics`, and add the
      one-shot `y`/`E` prefixes.

      "
  - id: trail
    title: Breadcrumbs you can see and walk back
    depends_on:
      - follow
    size: medium
    description: "trail: add the bounded pager-owned trail whose entries restore the
      view rather than just the document, render it as a kind-glyph breadcrumb strip
      that is absent at depth zero, and bind `backspace`/`ctrl+o`/`ctrl+i` with the
      exhausted-trail exit rule.

      "
  - id: cli
    title: The `sase pager` command
    depends_on:
      - trail
    size: medium
    description: "cli: add the `sase pager` top-level command over refs, paths, and
      stdin; route `page_or_print` in-process when the resolved pager is SASE's own so
      `sase bead show` gets the new surface; and document it.

      "
  - id: ace
    title: "`v` opens the pager"
    depends_on:
      - trail
    size: medium
    description: "ace: route the Agents-tab `v` keymap to `SasePager` under `suspend()`,
      carrying the selected files as sections and the typed commit and report specs as
      attached targets, within the measured open-to-first-paint budget.

      "
  - id: parity
    title: One ref, one destination, one glyph
    depends_on:
      - cli
      - ace
    size: small
    description: "parity: assert with one shared test that the pager and the `sase-ug`
      link rail resolve the same ref to the same target, glyph, accent, and dangling
      vocabulary, and swap the resolver onto the live `LinkIndex` when that epic has
      landed it.

      "
  - id: land
    title: Four pagers become one
    depends_on:
      - parity
    size: medium
    description:
      "land: delete `_print_or_page`, `_view_files_with_pager`, and the `less` back end
      of `page_or_print`, point the text mode of the artifact viewer at the pager,
      remove the flag, and close the flag bead — as a net deletion."
proposed_by: bbugyi200.athena.0ej
bead_id: sase-uk
create_time: 2026-09-09 19:50:46
---

- **PROMPT:**
  [prompts/202608/link_traversing_pager.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/link_traversing_pager.md)
- **BEAD:**
  [sase-uk](https://github.com/sase-org/sase--beads/blob/main/pages/sase-uk/README.md)

# Plan: A link-traversing SASE pager

## Problem

SASE has four pagers and none of them is a pager.

| Path                                                                      | Used by                             | Mechanism                                                        |
| ------------------------------------------------------------------------- | ----------------------------------- | ---------------------------------------------------------------- |
| `src/sase/cli_pager.py` `page_or_print()`                                 | `sase bead show`                    | pipes rendered ANSI into `$SASE_PAGER`/`$PAGER`/`less`           |
| `src/sase/artifact_cli/read.py:338` `_print_or_page()`                    | `sase artifact read`                | its own `less -R -F` / `bat --paging=always`                     |
| `src/sase/ace/tui/actions/hints/_files.py:267` `_view_files_with_pager()` | ACE `v` on Agents and Patches       | `suspend()` + `bat --color=always … \| less -R` via `shell=True` |
| `graphics/_viewer_loop_media.py` `artifact_text_viewer_command()`         | ACE artifact viewer, text artifacts | `bat --paging=always`, else a Python dump                        |

Four call sites, four flag sets, zero shared contract — and every one of them ends in a
program that cannot paint a key, cannot own a history, and cannot hand a keypress back
to SASE.

That last property is the real cost. Today `v` on the Agents tab opens a plan file in
`less`. The plan cites `bead:sase-ua` and `src/sase/relations/artifact_links.py:201`.
Both are dead text. To follow either one the user quits `less`, returns to the TUI, and
starts over. **Reading a SASE document is where SASE's link graph stops being
navigable.** The `sase-ug` epic is fixing that for the _selected entity_ on every ACE
tab; nothing is fixing it for _the text you are reading_.

`cli_pager.py` (added 2026-08-25) is the newest and best of the four: a real `PagerMode`
(`auto`/`always`/`never`), a wide-character-aware row estimator via
`rich.cells.cell_len`, `SASE_AGENT` suppression, SIGINT isolation, `BrokenPipeError`
handling. **It is the right front door and the wrong back end.** Every one of those
behaviors survives this epic; only what it hands the text to changes.

## This is mostly consolidation

Six-sevenths of the requested pager already exists in this tree, tested and shipped.
That changes the risk profile and it decides the sequencing: phase one extends
primitives, it does not write a viewer.

| Requirement                                                                  | Existing implementation                                                                                                                                |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| The exact requested `0-9`/`a-z`/`A-Z` alphabet                               | `actions/navigation/jump_hints.py:14` `JUMP_HINT_CHARS`, `build_jump_hint_maps()`, `match_jump_hint()`                                                 |
| Textual's shifted-key trap for `A-Z` labels                                  | `jump_hints.py:69` `normalize_jump_key()` — already defused                                                                                            |
| `ctrl+n`/`ctrl+p` next/prev entity                                           | `ZoomPanelModal` (`zoom_panel_modal.py:110`), `CommitViewModal`                                                                                        |
| Single-key link following                                                    | `GlossaryPreviewModal` binds `1`–`9` to `follow_reference(n)` (`glossary_preview_modal.py:61`)                                                         |
| `backspace` over a history stack                                             | `GlossaryPreviewModal.action_go_back`; `travel_back: "backspace,h"` in `default_config.yml:374`                                                        |
| Bounded trail                                                                | `memory_panel_travel.py` — `_trail`, `_MAX_TRAIL_LENGTH = 32`, `_travel_forward`, `action_travel_back`                                                 |
| Breadcrumb rendering                                                         | `modals/trail_strip.py` `build_trail_strip()` — pure, already shared by two panels                                                                     |
| Vim `/` `n` `N` incremental search                                           | `widgets/vim_search_controller.py` — a controller with a documented host protocol, already re-hosted twice                                             |
| One-shot key prefix                                                          | `modals/numbered_link_keys.py`                                                                                                                         |
| Render budget with a visible truncation notice                               | `widgets/prompt_panel/_hint_caps.py` `HintContentBudget`, 128 KB / 5,000 lines                                                                         |
| Typed-ref scanning                                                           | `artifact_ref_operations.py:204` `scan_artifact_refs()` (Rust binding)                                                                                 |
| Path/URL scanning                                                            | `widgets/prompt_panel/_file_path_hints.py:172` `iter_file_path_matches()`                                                                              |
| CLI ref resolution outside ACE                                               | `artifact_cli/references.py:76` `resolve_cli_reference()`                                                                                              |
| Multi-entity divider                                                         | `bead/cli_show_batch.py:185` `_show_divider` — `── 2/3 ────` already renders                                                                           |
| Media handoff                                                                | `graphics/_viewer_launch.py:210` `view_artifact_files` — kitty graphics, mpv, tmux panes                                                               |
| Kind glyphs and accents                                                      | `_artifact_tab_model.py:52` `ARTIFACTS_ACCENTS`, `:64` `ARTIFACTS_ICONS`                                                                               |
| **A standalone Textual app run from a CLI subcommand _and_ from inside ACE** | `MemoryReviewTuiApp` — `memory/cli_review.py:338` calls `.run()`; `actions/agents/_notification_handlers.py:141` calls it inside `with app.suspend():` |

That last row is the load-bearing one. It means no subprocess, no session manifest, no
new handoff protocol: the pattern this epic needs is already in production in this
repository, in both directions.

Two measurements from `research:202608/link_traversing_pager.md` set the key budget,
taken over 2,424 local plan and research documents, 51,694 links, 522,288 sliding
45-line windows:

- Links per 45-line screen: p50 **3**, p90 9, p99 18, **max 47**.
- Reserving the ten house command keys costs **3.4 points** of single-key coverage (94.7
  % → 91.3 %) and leaves 52 label keys — more than the worst screen in the corpus ever
  needs.
- **20.4 % of screens have no links at all.** Absence is the common case, so it must
  cost nothing visually.

## Design

### The reading surface

```
 ◈ sase-ug · A link rail on every tab                            2/3 · 41% · ⌘ 88c
 TRAIL  ⬡ sase-ug.6.code  ›  ✎ link_rail_every_tab.md  ›  ◈ sase-ug
 ────────────────────────────────────────────────────────────────────────────────
 CHILDREN
   PHASES
     ✓ [0] sase-ug.1: One projection for the machine-local read model    [CLOSED]
     ✓ [1] sase-ug.2: A stale clone may not prove deletion               [CLOSED]
     ◐ [2] sase-ug.3: Projected edges from facts SASE already owns  [IN_PROGRESS]

 LINKS (1)
   ← implemented-by · [3] plan:202608/link_rail_every_tab.md

 PAGE
   [4] https://github.com/sase-org/sase--beads/blob/main/pages/sase-ug/README.md

 PLAN
   [5] plan:202608/link_rail_every_tab.md
   → [6] /home/bryan/.sase/plans/202608/link_rail_every_tab.md
 ────────────────────────────────────────────────────────────────────────────────
 0-9a-z follow · ⌫ back · ^N/^P entity · / search · y copy · E edit · ? keys · q
```

Four things in that frame are deliberate:

- `[3]` and `[5]` are the **same ref reached two ways** and `[6]` is its resolved path.
  All three are keyed. The pager labels occurrences, not destinations, because the user
  presses what they are looking at.
- The trail row occupies a line **only because there is a trail.** At depth zero the row
  is absent, not empty.
- The footer is a legend driven by availability, not decoration: with no trail there is
  no `⌫ back`.
- Bare bead ids `sase-ug.1`–`sase-ug.3` are keyed because the origin is a bead document.
  In a research document the same tokens would be plain text.

A file document, mid-scroll, showing a section rule in flight:

```
 ▤ 3 files · artifact_links.py                                   1/3 · 12% · ⌘ 88c
 ────────────────────────────────────────────────────────────────────────────────
    198  def _known_target_for_ref(ref: str) -> ArtifactEntryTarget | None:
    199      """Resolve a ref by scanning every known target."""
    200      for target in _all_targets():          # ← see [k] tui_perf.md rule 8
    201          if target.ref == ref:
 ━━ 2/3 ━ ▤ src/sase/ace/tui/relations/artifact_links.py ━━━━━━━━━━━━━━━━━━━━━━━━
```

### D1 — One Textual `App`, run in-process by both hosts

`SasePager(App[PagerExit])` in a new `src/sase/pager/` package. The CLI calls
`SasePager(document, …).run()`. ACE calls it inside `with self.suspend():`.

`MemoryReviewTuiApp` already proves both halves of exactly this pattern in exactly this
tree. It buys `VimSearchController`, Rich renderables, PNG snapshot testing via
`just test-visual`, and headless `Pilot` key tests for free.

**Rejected — a subprocess with a session manifest.** `sase version` from a warm shell
costs 288–318 ms; that is the floor for any `SASE_PAGER="sase pager"` process hop, and
it also forces a versioned serialisation format for typed targets that a direct call
passes as live objects.

**Rejected — keeping `less` with a `lesskey` map or a link sidecar.** `less` cannot
paint labels, cannot own a trail, and cannot dispatch a key back to SASE.

**Rejected — a modal inside the running ACE app.** It solves only ACE; `bead show` and
`sase pager` would need a second host, and it puts pager crashes inside the control
surface.

### D2 — The input is a document of sections

```python
@dataclass(frozen=True)
class PagerSection:
    identity: str                     # "bead:sase-ug", "file:/abs/path.py"
    title: str                        # rendered into the section rule
    kind: str                         # drives glyph + accent
    body: RenderableType | str        # Rich renderable, or ANSI/plain text
    subject_ref: str | None = None    # this section's own canonical ref
    targets: tuple[AttachedTarget, ...] = ()   # caller-attached, span-bound

@dataclass(frozen=True)
class PagerDocument:
    sections: tuple[PagerSection, ...]
    title: str
    origin: PagerOrigin               # what opened it; seeds bare-token rules
```

`ctrl+n`/`ctrl+p` needs entity boundaries; the trail needs stable identities; the chrome
needs a subject. A concatenated string has none of these, and `handle_bead_show()`
flattens all three away before it reaches `page_or_print` today.

**ANSI bodies are first-class.** `sase bead show` renders styled ANSI, not a renderable.
`rich.text.Text.from_ansi()` yields plain text for scanning _and_ style spans for
rendering, so labels insert at exact offsets with no bespoke SGR parsing.
`cli_pager._SGR_RE` becomes unnecessary.

### D3 — The pager owns scanning; callers may attach typed targets

One scanner, run per section's plain text, in precedence order:

1. **Typed artifact refs** — `scan_artifact_refs()`. Highest precedence; already the
   authority on ref spans.
2. **URLs** — `_HTTP_URL_PATTERN`, which `iter_file_path_matches` currently matches only
   in order to discard.
3. **File paths** — `iter_file_path_matches()`, minus artifact-ref ranges via the
   existing `_matches_outside_artifact_refs`.
4. **Origin-scoped bare tokens** — bead ids in a bead document, short shas in a diff.
   Declared per `PagerOrigin`, **never globally**, so a research document that says
   `sase-core` does not sprout a false link.

Rule 4 is worth its cost: live `sase bead show sase-ug` prints roughly thirteen bare
bead ids (`sase-ug.1` … `sase-ug.10`, `sase-ug.land`, `sase-ud.3`) that today's scanner
cannot see, because `_FILE_PATH_ALTERNATIVES` requires a `/` or an extension. In bead
output those are the highest-value links on the screen.

But the scanner cannot recover what `AgentHintRender` already holds: a `CommitViewSpec`,
a `SlowToolCallReportSpec`, a lazily-materialised memory or glossary report
(`_agent_display_state.py:55-75`). Those are objects, not substrings. So:

> The pager discovers every link it can find in text. A caller may additionally attach
> already-typed targets bound to explicit spans. Attached targets win over scanned ones
> on overlap; both share one label sequence in document order.

This is the only formulation under which `v` keeps its commit and report hints while
`git show --stat | sase pager` still works with no caller cooperation at all.

**No I/O to recognise a link.** A label's appearance derives from the ref string alone.
Existence is checked on press, never on paint. Reuse `HintContentBudget` and its 128 KB
/ 5,000-line caps, including the visible truncation notice; do not derive a second
budget.

### D4 — Keys

**The alphabet is `JUMP_HINT_CHARS` minus the reserved commands, in order.** Ten keys
stay commands, matching the house reading vocabulary already fixed by `ZoomPanelModal`
(`zoom_panel_modal.py:96-115`): `q` close, `j`/`k` scroll, `g`/`G` top/bottom, `y` copy,
`E` edit, `r` refresh, `n`/`N` search-next/prev. That leaves **52 single-key labels**,
costing 3.4 points of coverage against a corpus whose worst screen holds 47 links.
Non-alphanumeric commands (`/`, `?`, `backspace`, arrows, space, `PgUp`/`PgDn`, `[`/`]`,
`ctrl+*`) never collide with a label.

Digits still come first, as requested: label 1 is `0`, label 11 is `a`, and the reserved
characters are skipped in place.

**Overflow is prefix-free and never timed.** Where a document exceeds 52 links, reserve
the _last_ k alphabet characters as two-key prefixes; a reserved prefix is never itself
a label, so `a` can never be both "link 11" and "the start of `aQ`". k=1 → 103 labels,
k=2 → 154, k=3 → 205, above the corpus maximum of 187. **The first ~51 links — where the
eye is — always stay single-key.**

_This is a deliberate deviation from the literal `00`–`ZZ` spelling in the brief._
`build_jump_hint_maps` today is fixed-width: past 62 targets _every_ label becomes two
characters, so one extra link doubles every keystroke in the document. Prefix-free
variable width serves the brief's actual intent — "we shouldn't need to resort to
multiple keys often" — strictly better. The two-key labels are drawn from the tail of
the same alphabet (`Z0`, `Z1`, …) rather than starting literally at `00`. Add the mode
as a new parameter; leave existing callers (`jump_all_modal`, `models_panel_*`) on
current behavior.

**Assignment is document-scoped and stable.** Labels are assigned in document order at
load and never change while the document is open. Scrolling, resize, reflow, and search
never renumber anything. This is the opposite of Vimium's model and it is right here,
because the labels are _always painted_: churning them under a moving viewport would be
noise for no benefit.

**Route every keypress through `normalize_jump_key`.** Textual may report a shifted
letter as `event.key == "a"` with `event.character == "A"`. Any design that uses `A-Z`
as labels is broken without it, and it already exists.

**Window-scoped fallback, shipped dormant.** If a document exceeds two-key capacity —
nothing in the corpus does, but the agent detail document's 924–1,154 hints would —
label only the visible band plus one screen of overscan, re-anchored when the viewport
leaves the band. The worst screen in the corpus holds 47 links against 52 keys, so
single keys always win in that mode. Ship it now; it is what makes the deferred "`v`
opens the agent document itself" work possible later.

**A timeout-based input mode is rejected outright.** It mis-fires under load, cannot be
reasoned about, and needs a fake clock to test. Prefix-free allocation is unambiguous by
construction.

### D5 — `ctrl+n` / `ctrl+p` scrolls a section header to row 0

The pager is **one continuous scrollable document**. `ctrl+n` scrolls so the next
section's rule sits at row 0; `ctrl+p` does the reverse; at the last section `ctrl+n`
goes to the end of the document rather than beeping.

This follows the brief's own phrasing — "re-draw the contents with that file's header at
the top of the screen" — which is a scroll, not a mode. It also keeps `/` document-wide,
keeps `g`/`G` meaning the real top and bottom, and preserves the mental model of
`bat f1 f2 f3 | less` that `v` users already have.

This deliberately differs from `ZoomPanelModal` and `CommitViewModal`, which bind the
same chord to a _view swap_ because their entities load independently. Note the
divergence in the code so a later reader does not "fix" it.

The section rule is `_show_divider`'s shape with a glyph and an accent:

```
━━ 2/3 ━ ◈ bead:sase-ug ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### D6 — A press always lands, and the pager never dead-ends

Resolution goes through one narrow interface with two implementations:

```python
def resolve_ref(ref: str) -> LinkTarget | None: ...
```

satisfied inside ACE by `sase-ug.5`'s live `LinkIndex` and outside it by
`resolve_cli_reference()`. Resolution happens **on keypress, not on paint**, which is
what makes the CLI implementation's cost acceptable and what keeps paint time flat.

What a press does, by target kind:

| Target                                        | Behavior                                                                                                                                        |
| --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Artifact ref, file, plan, bead, patch, stitch | opens as a **new pager document**                                                                                                               |
| File path with `:line`                        | opens as a pager document scrolled to that line                                                                                                 |
| Directory                                     | opens as a pager document whose one section is the sorted entry listing — which the scanner then labels for free, so no new renderer is needed  |
| Image, video, PDF                             | suspends and hands the spec to `graphics.view_artifact_files`, then resumes at the same trail entry and scroll offset                           |
| URL                                           | copies the URL and toasts. Launching a browser from a keypress inside a TUI that is usually reached over SSH or tmux is deferred, not forgotten |
| Unresolvable                                  | toasts the dead end, **does not navigate, does not touch the trail**, and permanently marks that label `⊘` dim for the rest of the session      |

That last row is the reliability rule that makes the surface trustworthy: **the document
learns.** A dead end is shown once and then rendered as one, so the user is never
invited twice into the same failure. Dangling presentation borrows
`relation_panel.py:236`'s existing vocabulary — dim, `⊘`, `(missing)` — so a missing
link looks the same everywhere in SASE.

The general contract: **a text target opens inside the pager.** `E` is the only escape
to `$EDITOR`. This is the whole difference from the `bat | less` dead end.

### D7 — The trail

A bounded 32-entry trail, mirroring `_MAX_TRAIL_LENGTH` and
`_travel_forward`/`action_travel_back` in `memory_panel_travel.py`, but with richer
entries:
`(document identity, section identity, scroll offset, search state, label anchor)`.
`backspace` restores the **view**, not just the document.

- `backspace` back, plus `ctrl+o` back and `ctrl+i` forward for vim symmetry.
- **History is pushed only after a target loads successfully**, so a crumb never lies
  about where the user has been. While a target loads, show a transient footer status —
  never a provisional crumb.
- A failed or dangling jump does not mutate the trail.
- **When the trail is exhausted, `backspace` exits the pager**, returning to ACE where
  `ctrl+o` continues walking the rail's own trail (see the alignment section).

Rendered by extending `build_trail_strip()` with per-entry kind glyph and accent from
`ARTIFACTS_ICONS`/`ARTIFACTS_ACCENTS`, collapsing past three entries to
`⟨ …3 › ✎ plan ⟩` to match the rail's vocabulary. The strip is **absent, not empty**,
when the trail is empty, so a first-open pager wastes no row. `?` shows the full
numbered trail alongside the key legend, so an elided crumb is never information the
user cannot reach.

### D8 — `y` and `E` are one-shot prefixes

The current `v` flow already teaches `3@` to edit and `3%` to copy. Preserve that muscle
memory in a form that scales past nine:

- `y` then a label → copy that link's canonical ref or path.
- `y` then `y` → copy the current section's own ref.
- `E` then a label → open that link in `$EDITOR` via the existing `build_editor_args`,
  at its line when the ref carries one.
- `E` then `E` → open the current section.

Unambiguous, prefix-free, no timeout, and vim-shaped. Reuse
`modals/numbered_link_keys.py`'s one-shot prefix mechanism. The footer shows the armed
prefix while it is pending, and any other key cancels it.

### D9 — Search is the existing controller, re-hosted a third time

`widgets/vim_search_controller.py` already provides `/`, `n`, `N`, incremental
highlight, wrap feedback, and a documented host protocol; `zoom_panel_search.py` and
`_metadata_search.py` already host it. Host it a third time. Copy
`_STRUCTURAL_SEARCH_EXIT_KEYS`'s idea so structural keys (`ctrl+n`, `ctrl+p`,
`backspace`, `q`) exit search mode rather than typing into it. Because `n`/`N` are
reserved from the label alphabet, a label can never shadow a search-navigation key —
that is not a coincidence, it is why they are reserved.

### D10 — `sase pager`

**Yes, add the command.** Three capabilities justify the help text, completion, docs,
and tests it costs:

1. **It is the thing you put in `$PAGER`.** `_resolve_pager_argv()` already reads
   `SASE_PAGER` then `PAGER` and execs it with the body on stdin. The moment
   `sase pager` exists, `export SASE_PAGER="sase pager"` routes every paging SASE
   command through the new surface using plumbing that already ships — no config field,
   no flag, no new mechanism. **That also decides the name**: a `$PAGER` value must
   honestly be a pager. `sase view` and `sase open` read better as verbs but collide
   with `sase artifact open` and `sase artifact read`, which already exist with
   different jobs.
2. **Multi-ref reading in one view does not exist today.**
   `sase pager bead:sase-ug plan:202608/link_rail_every_tab.md src/sase/cli_pager.py` —
   three heterogeneous artifacts, one document, `ctrl+n`/`ctrl+p` between them, every
   mention in all three keyed. `sase artifact read` takes one ref and shells to `less`;
   `sase bead show` takes many, but only beads.
3. **It makes foreign output navigable.** `git show --stat | sase pager` and
   `gh pr diff 1421 | sase pager` turn every path in a diff into a keyed jump. The
   scanner works on plain text, so this needs no per-source integration.

Shape, following `cli_rules` (options sorted, a short alias for every public long
option, no required options, required values as positionals):

```
sase pager [-c auto|always|never] [-l auto|never] [-p] [-t TITLE] [-w WIDTH] [REF|PATH ...]

  positional  REF|PATH   Artifact references or file paths, one section each.
                         Omit (or pass -) to read the document from stdin.
  -c, --color            Color output: auto, always, or never (default: auto)
  -l, --links            Link scanning and painted keys: auto or never (default: auto)
  -p, --plain            Dump without paging; implied when stdout is not a TTY
  -t, --title TITLE      Document title for stdin input
  -w, --wrap WIDTH       Wrap prose at WIDTH columns (default: markdown.print_width)
```

Three implementation notes that are easy to get wrong:

- **Reopen the terminal for input.** When the document arrives on stdin, stdin is a pipe
  and Textual has no keyboard. Open `/dev/tty` for input before starting the app,
  exactly as `less` does, and fall back to `-p` behavior when there is no controlling
  terminal.
- **`page_or_print` must detect that the resolved pager is SASE's own and call it
  in-process** — to skip the ~300 ms hop, and more importantly to pass the structured
  `PagerDocument` instead of a flattened string. The subprocess form exists only for
  foreign callers and degrades honestly to scan-only.
- **The in-process path never consults `$SASE_PAGER`, and the subprocess path unsets it
  in the child env**, so `SASE_PAGER="sase pager"` cannot recurse.

With no TTY the command dumps plain, so it is safe in a pipeline and safe as an
unconditional `$PAGER`.

### Beauty rules

Beauty here comes from truthful structure, not ornament.

- **One sticky chrome band, one content surface, one contextual footer.** Nothing else
  competes for a row.
- **Absence costs nothing.** No links → no key capsules anywhere. No trail → no trail
  row. 20.4 % of screens have zero links; that case must render identically to a build
  with no label layer compiled in.
- **One label colour, many token colours.** The key capsule is always the same
  high-contrast style, so "this is a key" is learned once. The _token_ carries the
  destination kind's accent from `ARTIFACTS_ACCENTS` and its glyph from
  `ARTIFACTS_ICONS`, so the colour tells you where the key goes before you press it —
  the same contract the `sase-ug` rail chips use.
- **Kind colour lives in glyphs and capsules, never in large saturated panels.**
- **A label is never separated from its token by wrapping.**
- **Full legibility without colour.** Glyph and text differences alone must carry kind,
  media type, and missing state.
- **`?` is the cheat sheet.** Do not wrap every document in a permanent two-line legend;
  the footer names the verbs, `?` names every binding, the label legend, and the full
  trail.
- **Distinct small glyphs** for loading, missing (`⊘`), external, and media (`▨ ▶ ▤`)
  targets.

## Alignment with the `sase-ug` epic

`bead:sase-ug` — "A link rail on every tab", `plan:202608/link_rail_every_tab.md`,
phases 1–2 closed, 3–10 in progress — is the adjacent work. **Complementary surfaces,
shared substrate.**

| Dimension   | `sase-ug` link rail                            | This pager                                 |
| ----------- | ---------------------------------------------- | ------------------------------------------ |
| Surface     | one line above ACE's footer, on all three tabs | a full-screen document view, CLI and ACE   |
| Subject     | the **selected entity** in a list              | the **content being read**                 |
| Link source | the typed artifact-link **graph**              | **textual mentions** in the rendered bytes |
| Cardinality | p50 1, p90 2, max 26; ≤9 chips                 | p50 15, p90 48, max 187; inline            |
| Key grammar | `$` prefix + one key                           | one painted key per link                   |
| Trail       | app-level, `ctrl+o` / `ctrl+shift+o`           | pager-owned, `backspace` / `ctrl+o`        |

They answer different questions. The rail answers _"what is this entity linked to?"_;
the pager answers _"what does this text point at?"_ A bead's rail shows its
`implemented-by` edge; the pager over that same bead shows its thirteen child ids, its
plan path, and its GitHub page URL — none of which are graph edges. Two key grammars is
correct, because the two surfaces have genuinely different key budgets. **What must not
differ is where a key takes you.**

**Three seams are shared:**

1. **Ref → destination.** One `resolve_ref` interface, two implementations. The pager
   must never build a second resolution path, and must never reintroduce the O(n)
   `_known_target_for_ref` scan (`relations/artifact_links.py:200`) that `sase-ug.5`
   exists to retire.
2. **Ref → glyph, accent, and short label.** `ARTIFACTS_ICONS`/`ARTIFACTS_ACCENTS` and
   the rail's short-label rule (`bead:sase-u3` → `sase-u3`,
   `stitch:sase-org/sase@f4b827af6` → `sase@f4b827a`) must produce identical output in
   both surfaces. `◈` in `#D787FF` means "bead" everywhere.
3. **Dangling presentation.** Dim, `⊘`, `(missing)`, toast instead of navigate, do not
   mutate the trail.

**The trail is deliberately _not_ shared.** Two reasons, both structural. A `sase-ug`
trail entry records `(tab, ArtifactEntryTarget, pane query digest, fold state)` — an ACE
_selection_ restore; a pager entry records
`(document, section, scroll offset, search state)` — a _view_ restore. And that plan
states its trail "clears when the user navigates by any other means"; opening the pager
is exactly such a navigation, so under its own rule a shared object would clear on
entry. The synthesis keeps one _conceptual_ history without coupling two objects: the
pager owns its trail, and when it is exhausted `backspace` closes the pager and ACE's
`ctrl+o` resumes. Press `$1`, land in the pager, follow two links, press `backspace`
three times, and you are back where you started — with two truthful histories rather
than one fragile shared stack.

**Sequencing:** this epic does not block on `sase-ug`. It lands against the narrow
`resolve_ref` interface with a `resolve_cli_reference`-backed adapter and swaps in the
live `LinkIndex` in the `parity` phase. Conversely, this epic must not touch
`artifact-links.json` writers or the relation read model — `sase-ug.1` owns that
convergence, and the stale-generated-file drift already noted on `sase-ug` is that
epic's to resolve.

**One overlap to declare so no reviewer reads it as redundant:** `sase-ug.10` retires
`beads_open_plan` and `plans_open_bead` as duplicates the rail generalises; this epic's
`land` retires `_print_or_page`, `_view_files_with_pager`, and the `less` back end of
`page_or_print`. Both are net deletions and they do not collide.

## Out of scope for v1

- **No editing beyond `E`, and no link authoring.** `sase-ug.9`'s `$0` panel owns
  `add`/`rm`.
- **No `read` graph edges.** `sase artifact read` records a `read` row only under
  `SASE_AGENT` with a resolved agent identity. The pager is a human surface; recording a
  read because a human scrolled past a chip would pollute the very read model
  `sase-ug.1` is converging. `page_or_print` already declines to page under
  `SASE_AGENT`; keep that.
- **No browser launch from a keypress.** URLs render and copy.
- **No transitive expansion.** One hop at a time; the trail is the history.
- **No graph `LINKS` rail inside the pager** until `sase-ug.6` lands one to borrow. Bead
  output already carries its own `LINKS` block and those refs become pressable for free.
  Shipping a second rail implementation to delete weeks later is the one place these two
  efforts could genuinely collide.
- **The agent detail document itself stays out of the pager.** It is the largest single
  UX win available — its 924–1,154 hints per render are where the current flow hurts
  most, and the dormant window-scoped labeller is the mechanism that makes it possible —
  but it changes a keypath the owner uses constantly and deserves its own proposal and
  its own goldens. Record it as a follow-up; do not ride it on this epic.
- **Migrating the four private selector alphabets** (`runners_modal.py:49`,
  `agent_neighbor_modal.py:25`, `property_picker_modal.py:24`,
  `agent_workspace_tmux_modal.py:72`) onto the shared allocator. Worth doing, unrelated
  risk; file it as a follow-up from `land`.

---

## Phase `primitives` — The allocator and the scanner

1. Create the `beta` feature flag with `sase flag new` covering the pager surface, its
   keys, and its trail, so `cli` and `ace` can land without exposing an unfinished
   surface. `land` removes it. Create it here, not later, because `cli` and `ace` run in
   parallel and must not race to define it.
2. Extend `build_jump_hint_maps` with a **prefix-free variable-width** mode as a new
   parameter: an ordered alphabet with characters excluded, and the last k characters
   reserved as two-key prefixes where k is the minimum that covers the target count.
   Leave every existing caller on today's fixed-width behavior. Export the reserved
   command set (`q j k g G y E r n N`) as one named constant so the viewer's bindings
   and the allocator cannot drift apart.
3. Add `src/sase/pager/link_scan.py`: one function from plain text plus a `PagerOrigin`
   to an ordered tuple of spans, applying the four precedence rules in D3 and reusing
   `scan_artifact_refs`, `_HTTP_URL_PATTERN`, `iter_file_path_matches`,
   `_matches_outside_artifact_refs`, and `HintContentBudget`. No I/O. Origin-scoped
   bare-token recognisers are registered per origin, never global.
4. Tests: the allocator is prefix-free and stable for every document size 0–250, with
   boundary cases at 51, 52, 53, 103, 154, and 205; `normalize_jump_key` round-trips
   every uppercase label; the scanner finds every ref kind in both endpoint positions;
   URLs and typed refs are no longer swallowed by the path regex; a bare bead id is
   found in a bead origin and _not_ in a research origin; the budget's truncation notice
   still appears.

## Phase `document` — A document of sections, not a string

1. Add `src/sase/pager/document.py` with `PagerSection`, `PagerDocument`, `PagerOrigin`,
   and `AttachedTarget` per D2. Frozen dataclasses, no Textual import, so the CLI can
   build one without paying for the TUI.
2. Ingest ANSI and plain bodies through `rich.text.Text.from_ansi()`, keeping plain text
   for scanning and style spans for rendering, so a label inserts at an exact offset
   without bespoke SGR parsing.
3. Merge scanned spans with caller-attached targets: attached wins on overlap, both
   share one label sequence in document order.
4. Add two adapters. `sase bead show`'s `ShowBatch` becomes one section per bead
   carrying its canonical `bead:` ref — `resolve_show_batch` already produces the
   per-bead entries and `_show_divider` already draws the boundary, so this is
   re-plumbing, not re-rendering. A path list becomes one section per file.
5. Tests: the same bead renders to the same section set from the CLI adapter and from a
   fixture; an ANSI body round-trips its styles; an attached target at an overlapping
   span suppresses the scanned one and keeps its own label position.

## Phase `viewer` — The reading surface

1. Add `SasePager(App[PagerExit])` in `src/sase/pager/app.py`: the sticky chrome band
   (subject line, conditional trail row, rule), a virtualised body — never one widget
   per line — and the footer legend.
2. Bind the house vocabulary exactly as `ZoomPanelModal` does: `q`/`escape` close,
   `j`/`k` scroll, `ctrl+d`/`ctrl+u` half-page, `g`/`G` top/bottom, `r` refresh.
3. Implement `ctrl+n`/`ctrl+p` as scroll-to-header per D5, with the section rule
   rendered from `_show_divider`'s shape plus glyph and accent. Comment the deliberate
   divergence from `ZoomPanelModal`'s view-swap semantics.
4. Re-host `VimSearchController` for `/`, `n`, `N`, with structural keys exiting search
   mode rather than typing into it.
5. Make the footer availability-driven: a verb appears only when it would do something.
   Add `?` as the full binding sheet.
6. Honour `tui_perf` rules 1, 2, 8, and 9: first chrome paint never waits on file
   content, syntax highlighting, or resolution; parse each section once per content
   identity and mtime; cache rendered layout by width; cancel stale workers when the
   active location changes. Add a `pager.open` `tui_trace` span.
7. Not wired to any caller yet. Tests: headless `Pilot` key tests for every binding; PNG
   goldens at 120×40 and 60×30 for a zero-link document and a three-section document
   mid-rule.

## Phase `labels` — Keys painted ahead of time

1. Paint one key capsule immediately before each scanned or attached token, per the
   beauty rules: one constant label style, the token in the destination kind's accent
   with its glyph. Wrapping must never separate a label from its token.
2. Dispatch keypresses through `normalize_jump_key` and `match_jump_hint` against the
   document-scoped map. A pending prefix dims non-matching labels and brightens matching
   ones, and shows in the footer; any invalid key clears it with a quiet toast.
3. Implement the window-scoped fallback and ship it **dormant** — reached only past
   two-key capacity, labelling the visible band plus one screen of overscan and
   re-anchoring when the viewport leaves the band, not per line.
4. Enforce the invisibility contract: a zero-link document paints no capsules and
   reserves no gutter.
5. Tests: label assignment is stable across scroll, resize, reflow, and search;
   deterministic for the same document across runs; every uppercase label activates
   under a `Pilot` shifted keypress. PNG goldens for a 3-link document, a 60-link
   document with a two-key tail, and — as the strongest form of the invisibility
   assertion — **pixel equality between a zero-link document and the same document
   rendered with the label layer disabled**.

## Phase `follow` — A press that always lands

1. Define `resolve_ref(ref) -> LinkTarget | None` as the single narrow interface, with a
   `resolve_cli_reference`-backed implementation now. Resolution runs off the pump in a
   managed worker; every completion re-checks the active generation before updating. Per
   `tui_perf` rule 11, VCS subprocesses run with terminal prompting disabled and bounded
   waits, and a resolver that would need interactive credentials returns an actionable
   error card rather than borrowing the pager's tty.
2. Implement the press table in D6, including the `:line` offset, the directory listing
   section, and the media handoff to `graphics.view_artifact_files` with resume at the
   same scroll offset.
3. Implement dead-end memory: an unresolvable press toasts, does not navigate, and marks
   that label `⊘` dim for the session using the shared dangling vocabulary.
4. Add the one-shot `y` and `E` prefixes per D8 on `numbered_link_keys.py`'s mechanism,
   with the armed prefix shown in the footer and any other key cancelling.
5. Tests: every ref kind in both endpoint positions; a dangling ref renders, does not
   navigate, and marks itself; `y`/`E` in both the label and the doubled form; a media
   target suspends and resumes at the same offset; a `:line` target lands on the right
   row.

## Phase `trail` — Breadcrumbs you can see and walk back

1. Add `src/sase/pager/trail.py` on `memory_panel_travel.py`'s shape: bounded at 32,
   entries recording document identity, section identity, scroll offset, search state,
   and label anchor.
2. Push only after a target loads successfully; show a transient footer status while
   loading, never a provisional crumb. A failed or dangling jump leaves the trail
   untouched.
3. Bind `backspace` and `ctrl+o` back, `ctrl+i` forward. When the trail is exhausted,
   `backspace` exits the pager with a `PagerExit` that tells the host it was a
   trail-exhausted back, so ACE can resume its own trail.
4. Extend `build_trail_strip()` with per-entry kind glyph and accent, collapsing past
   three entries to `⟨ …3 › ✎ plan ⟩`. The strip is **absent, not empty**, at depth
   zero. Extend `?` to show the full numbered trail.
5. Tests: `backspace` restores scroll offset and search state, not just the document;
   the strip is absent at depth zero, asserted by pixel equality with a build that
   mounts no strip; elision at depth 4 and 10; the exhausted-trail exit returns the
   right `PagerExit`; both existing `build_trail_strip` callers still render unchanged.

## Phase `cli` — The `sase pager` command

1. Add `src/sase/main/parser_pager.py` and its handler, per the D10 shape and the
   `cli_rules` memory: options sorted, a short alias for every public long option, no
   required options, refs and paths as positionals, excellent `-h` output. Register it
   in `parser_full_registrars.py` and dispatch it in `entry.py`.
2. Positionals resolve through `resolve_cli_reference` into one section each; no
   positionals (or `-`) reads the document from stdin as a single section titled by
   `-t`. **Reopen `/dev/tty` for input when stdin is a pipe**, and fall back to plain
   dump when there is no controlling terminal.
3. Route `page_or_print` in-process when the resolved pager is SASE's own, passing the
   `PagerDocument` rather than a flattened string, behind the feature flag. Preserve
   `--pager auto|always|never` semantics exactly, including the `auto` row estimation,
   `SASE_AGENT` suppression, the no-pager-configured warning, and byte-stable direct
   output. The in-process path never reads `$SASE_PAGER`; the subprocess path unsets it
   in the child env.
4. **A pager startup failure falls back to the already-rendered direct output.** Never
   lose command output because a UI could not start.
5. Change `handle_bead_show` to build a `PagerDocument` from the `ShowBatch` instead of
   only a concatenated body, keeping the concatenated body for the direct and
   foreign-pager paths.
6. Add `docs/pager.md` and its `mkdocs.yml` nav entry; cross-link it from `docs/cli.md`
   and `docs/beads.md`.
7. Tests: `tests/test_cli_pager.py` and `tests/test_bead/test_cli_show_pager.py` stay
   green except where behavior intentionally changed; non-TTY dumps plain; a foreign
   `$SASE_PAGER` still wins; `SASE_PAGER="sase pager"` does not recurse; direct output
   is byte-identical to today; both flag states are covered.

## Phase `ace` — `v` opens the pager

1. Replace `_view_files_with_pager`'s call site in `_finish_view_request` with a
   `SasePager` launch under `with self.suspend():`, behind the flag. Selected files
   become sections; the typed `CommitViewSpec`, `SlowToolCallReportSpec`,
   `GlossaryReadReportSpec`, and `MemoryReadReportSpec` values already carried by
   `AgentHintRender` attach as `AttachedTarget`s so `v`'s existing commit and report
   hints keep working.
2. Media selections still route to `_view_files_with_artifact_file_viewer`; `E` and the
   copy paths keep their current behavior. Keep the existing off-pump hint render and
   its session/identity revalidation after the await untouched.
3. This removes today's `shell=True` `quoted_files | less` construction in favour of a
   direct call.
4. Handle `PagerExit`: a trail-exhausted `backspace` returns to the Agents tab with the
   prior selection intact.
5. Measure and record open-to-first-paint. Budget **150 ms**; `MemoryReviewTuiApp` makes
   that number obtainable rather than estimated. If it misses, fix it in this phase
   rather than deferring — the whole value of `v` is that it is instant.
6. Tests: the `agents.view_files` trace span still closes; `SASE_TUI_PERF` shows no
   Agents-tab regression; a `bench_tui_jk.py` case covers the Agents tab with the flag
   both on and off; terminal restoration under normal quit, `ctrl+c`, child failure, and
   ACE shutdown.

## Phase `parity` — One ref, one destination, one glyph

1. Add one shared test asserting that the pager and the `sase-ug` link rail resolve the
   same ref to the same target, the same glyph, the same accent, the same short label,
   and the same dangling presentation, for every ref kind in both endpoint positions.
2. Swap the ACE-side `resolve_ref` implementation onto `sase-ug.5`'s live `LinkIndex` so
   the pager stops paying an O(n) scan inside ACE.
3. **This phase does not block on the other epic.** If `LinkIndex` has not landed when
   this phase starts, keep the `resolve_cli_reference` implementation, run the parity
   test against whatever the rail resolves with today, and record the swap as a
   `PROPOSED FOLLOW-UP:` note on this phase's bead. The epic stays closable either way.

## Phase `land` — Four pagers become one

1. Delete `_print_or_page` (`artifact_cli/read.py`) and route `sase artifact read`
   through `page_or_print`. The audited `read` row is still written by the command; the
   pager itself records nothing.
2. Delete `_view_files_with_pager` (`actions/hints/_files.py`).
3. Delete the `less` back end of `page_or_print`, keeping the front door — `PagerMode`,
   the row estimator, `SASE_AGENT` suppression, SIGINT isolation, `BrokenPipeError`
   handling — and keeping the foreign-`$SASE_PAGER` subprocess path for users who set
   one deliberately.
4. Point `artifact_text_viewer_command`'s text mode at the pager.
5. Remove the feature flag: delete the Off branch, make the On branch unconditional,
   remove the registry entry, and close the flag bead in this same change.
6. **This phase must be a net deletion.** If it is not, the boundary was drawn wrong;
   say so on the bead rather than padding it.
7. File the two deferred follow-ups as task beads through `/sase_new_task`: migrating
   the four private selector alphabets onto the shared allocator, and putting the agent
   detail document itself into the pager.
8. Tests: parity tests for every retired caller; `just check-full` through
   `/sase_monitor`; `tools/check_feature_flags` clean.

---

## Risks

| Risk                                                                     | Severity | Mitigation                                                                                                                            |
| ------------------------------------------------------------------------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Textual boot inside `suspend()` makes `v` feel slower than `bat \| less` | **High** | Measure in `ace` against a 150 ms budget, obtainable today from `MemoryReviewTuiApp`; fix in-phase rather than deferring              |
| The pager and the `sase-ug` rail diverge into two link systems           | **High** | One `resolve_ref` interface, one glyph/accent table, one dangling vocabulary; the `parity` phase exists only to assert it             |
| Labels shift or mis-fire, so users stop trusting the keys                | **High** | Document-scoped stable assignment, prefix-free overflow, no timeouts; stability tested across scroll, resize, reflow, and search      |
| `A-Z` labels silently fail on shifted keys                               | Medium   | Route every keypress through the existing `normalize_jump_key`; assert every uppercase label under `Pilot`                            |
| A false-positive bare token (`sase-core` read as a bead)                 | Medium   | Bare-token recognisers are per-`PagerOrigin`, never global; a resolution failure renders `⊘` rather than navigating                   |
| Replacing four pagers regresses one caller                               | Medium   | `land` is the only deletion phase; every earlier phase leaves the old path intact behind the flag                                     |
| `sase pager` has no keyboard when the document arrives on a pipe         | Medium   | Reopen `/dev/tty` for input; degrade to plain dump with no controlling terminal; test both                                            |
| A pager crash loses command output                                       | Medium   | Startup failure falls back to the already-rendered direct output                                                                      |
| `SASE_PAGER="sase pager"` recursing into itself                          | Low      | The in-process path never consults `$SASE_PAGER`; the subprocess path unsets it in the child env                                      |
| Two-key labels appear more often than measured                           | Low      | 8.2 % of a 2,424-document corpus, only past link ~51. Raise k or switch that document to window-scoped labelling — both already built |

## Verification

- **Allocator:** prefix-free for every k; stable across scroll, resize, and search;
  deterministic across runs; boundaries at 51, 52, 53, 103, 154, 205.
- **Scanner:** every ref kind in both endpoint positions; URLs and typed refs no longer
  swallowed by the path regex; bare-token recognisers fire only in their declared
  origin; attached targets win over scanned ones on overlap.
- **Parity:** one shared test — same ref, same target, same glyph, same accent, same
  short label, same dangling presentation, in both surfaces.
- **Trail:** `backspace` restores scroll offset and search state; a failed jump leaves
  it untouched; an exhausted trail exits the pager and leaves ACE's own trail intact.
- **Invisibility:** the zero-link and zero-trail cases asserted by **pixel equality with
  a build that has no label layer and no trail strip mounted at all**, not by "the row
  says empty".
- **PNG goldens** at 120×40 and 60×30: zero-link, 3-link, 60-link two-key tail, dangling
  link, deep trail, mid-section-rule, monochrome.
- **CLI:** `tests/test_cli_pager.py` and `tests/test_bead/test_cli_show_pager.py` green;
  non-TTY dumps plain; `SASE_AGENT` still suppresses paging; a foreign `$SASE_PAGER`
  still wins; direct output byte-identical.
- **Terminal restoration** under normal quit, `ctrl+c`, child failure, and ACE shutdown.
- **Performance:** measure first chrome paint, first body paint, label activation, and
  scroll p95 **separately** — one aggregate startup number would hide whether resolution
  or rendering regressed. `pager.open` `tui_trace` span; scroll and label handling do no
  I/O, target p95 under 16 ms.
- **Flags:** both states tested in every phase that lands behind the flag;
  `tools/check_feature_flags` clean.
- `just check-full` through `/sase_monitor` before landing.
