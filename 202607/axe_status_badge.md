---
create_time: 2026-07-10 09:56:38
status: done
prompt: 202607/prompts/axe_status_badge.md
tier: tale
---
# Plan: Label the AXE Status Pill in the TUI Footer

## Problem / product context

The `sase ace` TUI docks a status pill in the **bottom-right** of the keybinding footer. It shows one of `RUNNING`
(green), `STOPPED` (red), `STARTING` (yellow), `STOPPING` (orange), `RESTARTING` (blue), or, during TUI startup, an
animated `◴ starting 2.3s` stopwatch.

The pill communicates _state_ but never says **what** is running or stopped. A user glancing at a lone red `STOPPED`
chip has no way to know it refers to the **axe** background daemon. The connection is implicit — you have to already
know the footer's right slot is "the axe indicator." This is the exact ambiguity to fix: the status must self-identify
as belonging to AXE, and it should look beautiful doing so.

Relevant facts discovered during exploration:

- The pill is the `#keybinding-status` Static, the right child of `KeybindingFooter`
  (`src/sase/ace/tui/widgets/keybinding_footer.py`). It is `width: auto`, right-pinned; the left `#keybinding-content`
  (`1fr`) holds the conditional keybinding chips.
- All pill text is built in one place: `KeybindingStatusMixin._get_status_text()` in
  `src/sase/ace/tui/widgets/_keybinding_status.py`. The state words and their Rich inline styles live in the `if/elif`
  chain there (lines ~176–199); background-command badges `[*N]`/`[✓N]` are appended after.
- The state is driven by four booleans (`_axe_running/_starting/_stopping/_restarting`) plus the startup stopwatch.
  `_status_signature()` short-circuits repaints; it already captures every input.
- "AXE" is the established, consistent name for this daemon everywhere the user already sees it: the third TUI tab is
  literally **AXE**, the CLI namespace is `sase axe …`, and docs/blog capitalize it as **AXE**. So an `AXE` label reuses
  vocabulary the user has already learned — it will read as "this is the AXE tab's daemon," reinforcing the tab
  relationship rather than introducing a new term.

## Design goal

Make it _unmistakable_ that the footer status belongs to AXE, and make the result look like a polished, first-class
status badge — without regressing the state colors users have already learned, and without depending on glyphs that the
pinned test font may not render.

## The design: a segmented "AXE" status badge

Adopt the universally-recognized **status-badge** pattern (think shields.io): a **stable label segment** immediately
followed by a **colored value segment**, rendered as one seamless pill.

```
 AXE  RUNNING       AXE  STOPPED       AXE  ◴ starting 2.3s
└───┘└─────────┘   └───┘└─────────┘   └───┘└────────────────┘
label   state       label   state      label     startup
(slate) (green)     (slate) (red)      (slate)   (tiered)
```

- **Label segment** — a constant `AXE` chip on a neutral dark-slate background with bold near-white text. It never
  changes color or position, so the eye learns "this two-part chip = AXE" at a glance and can always find the word in
  the same place.
- **State segment** — the existing state word with its existing color, completely unchanged (`RUNNING` green, `STOPPED`
  red, `STARTING` yellow, `STOPPING` orange, `RESTARTING` blue, and the animated startup stopwatch with its escalating
  tier colors).
- The two segments sit flush against each other (no gap), so they read as a single cohesive pill while the color break
  at the seam visually parses them as "label | value."
- The background-command badges `[*N]`/`[✓N]` continue to trail the pill, unchanged.

The label is prepended in **every** state, including the startup stopwatch, so the cold-start indicator also
self-identifies (`AXE  ◴ starting 2.3s` instead of a bare, unlabeled stopwatch).

### Concrete visual spec

Proposed constants (Rich inline styles, matching how this file already encodes color):

- `_AXE_LABEL_TEXT = " AXE "`
- `_AXE_LABEL_STYLE = "bold white on rgb(68,71,90)"` — a mid-dark slate that is clearly raised above the near-black
  footer surface yet darker/less saturated than every state color, so it pairs cleanly with green **and** red **and**
  yellow next to it.

State segments are untouched:

| Segment    | Text            | Style (unchanged)                        |
| ---------- | --------------- | ---------------------------------------- |
| Label      | `AXE`           | `bold white on rgb(68,71,90)` (constant) |
| Restarting | `RESTARTING`    | `bold black on rgb(0,191,255)`           |
| Starting   | `STARTING`      | `bold black on rgb(255,255,0)`           |
| Stopping   | `STOPPING`      | `bold black on rgb(255,165,0)`           |
| Running    | `RUNNING`       | `bold black on green`                    |
| Stopped    | `STOPPED`       | `bold white on red`                      |
| Startup    | `◴ starting Ns` | existing escalating tier palette         |

### Why this design (leading the call)

- **Clarity first.** A constant, high-contrast white `AXE` word is maximally legible and always in the same spot — that
  directly serves the user's core ask ("make it clear this is axe"). I deliberately did **not** tint the label with the
  state hue, even though it looks marginally more cohesive, because tinting trades away legibility of the identifying
  word.
- **Neutral slate, not brand-teal.** I considered giving the label the app's signature teal (`#00D7AF`) to tie it to the
  brand. Rejected: a teal label sitting flush against the green `RUNNING` state would blend at the seam (two cool
  greens), muddying the very "label | value" split that makes the badge readable. Neutral slate is the one background
  that contrasts crisply with _all_ state colors.
- **No new glyphs.** The visual-snapshot suite pins Fira Code + fontconfig for deterministic rendering, and Fira Code is
  not a Nerd Font. An axe/pick glyph (⛏/🪓) or powerline separator () risks rendering as tofu. A plain `AXE` text label
  is guaranteed to render everywhere and is unambiguous. (A leading glyph could be revisited later _if_ verified against
  the visual suite — noted as a future enhancement, not part of this plan.)
- **Minimal, low-risk change.** The label is a constant prefix, so it is one prepend at the top of `_get_status_text()`
  plus two module constants. Because the label never varies, `_status_signature()` needs no change — the
  repaint-suppression fast path keeps working exactly as before.
- **Reuses learned vocabulary.** "AXE" already names the tab, the CLI, and the docs, so the label adds no new concept —
  it just makes an existing, implicit association explicit.

### Considered and rejected

- **State-tinted label text** — prettier, less legible; clarity wins.
- **A leading axe/tool glyph** — font-rendering risk under the pinned test font; deferred.
- **Moving/expanding the pill or adding a separate labeled widget** — unnecessary churn to the footer layout; the
  segmented badge solves the problem in-place.
- **Tooltip / hover explanation** — not a natural terminal affordance and invisible at a glance; the whole point is that
  it should be clear _without_ interaction.

## Implementation outline

1. **`src/sase/ace/tui/widgets/_keybinding_status.py`** (the only rendering change)
   - Add the two module constants (`_AXE_LABEL_TEXT`, `_AXE_LABEL_STYLE`).
   - In `_get_status_text()`, prepend the label chip before the existing state `if/elif` chain so every branch
     (startup + all four boolean states) is preceded by `AXE`. Leave the state branches and the trailing bgcmd-badge
     logic untouched.
   - No change to `_status_signature()` (label is constant → no new signature input).

2. **`src/sase/ace/tui/widgets/keybinding_footer.py`** — re-export the new constants alongside the other
   `_keybinding_status` constants **only if** a test or docstring needs them; otherwise no change. (Layout already
   adapts: `_available_content_width()` derives the content budget from `_get_status_text().plain`, so the widened
   status automatically shrinks the bindings column — verified safe down to the 60-col constrained snapshot, which has
   ample slack.)

3. **Tests — `tests/test_keybinding_footer_status.py`**
   - Extend the existing per-state tests to also assert `"AXE"` appears in the rendered text for each state
     (starting/stopping/running/stopped and, ideally, a startup-active case).
   - Keep the existing state-word substring assertions (they still hold — the words are unchanged).

4. **Visual snapshot goldens (largest mechanical surface)**
   - The footer is docked on every base screen, so the bottom-right of essentially every full-screen PNG golden
     (`120x40`, the `60x30` axe views, etc.) changes. Regenerate with `just test-visual --sase-update-visual-snapshots`,
     then **review the diff to confirm only the footer's bottom-right region changed** and that narrow layouts (e.g.
     `axe_constrained_width_no_wrap_60x30`) still render without unwanted wrapping.
   - Spot-check representative states: a RUNNING footer, a STOPPED footer, and (if a fixture exercises it) the startup
     stopwatch, to confirm the `AXE` chip reads cleanly against each state color.

5. **Docs (keep in sync; these are regular docs, not memory files)**
   - `docs/ace.md` (~line 1371) and `docs/axe.md` (~line 461): note that the footer status now carries a leading `AXE`
     label chip so the indicator self-identifies. Small wording addition; the state table stays valid.

## Scope / non-goals

- **In scope:** the footer status pill only — the one ambiguous, unlabeled indicator.
- **Out of scope:** the AXE-tab dashboard/`AxeInfoPanel` status header (already unambiguous by virtue of living on the
  AXE tab), the bgcmd badges, and any change to state colors or state vocabulary.
- **Not doing:** glyphs/emoji, powerline separators, or footer layout restructuring.

## Boundary check (Rust core)

This is **presentation-only** Textual rendering. The status _value_ is produced by the existing Python axe process probe
(`src/sase/axe/_process_probe.py`); this change only restyles how an already-computed status is displayed. Per the
repo's core-backend boundary rule, presentation/widget rendering stays in this repo — **no `sase-core` (Rust) changes
are required.**

## Verification

- `just install` (ephemeral workspace may have stale deps), then `just check`.
- Targeted: `just test tests/test_keybinding_footer_status.py` and the footer/widgets tests.
- `just test-visual` after regenerating goldens; review the visual diff artifacts under `.pytest_cache/sase-visual/` for
  any snapshot that changed outside the footer's bottom-right.
- Known environment caveat: the full test phase can be SIGTERM-killed in the sandbox; rely on targeted pytest subsets +
  the visual suite + `just lint`/`mypy` gates when the full run is truncated.

## Risks & mitigations

- **Snapshot churn is broad.** Mitigation: it is mechanical and localized to one screen region; the diff review step
  confirms nothing else moved.
- **Narrow-width crowding.** Mitigation: the layout already computes the bindings budget from the status width; verified
  headroom at 60 cols. Confirm via the constrained-width snapshot.
- **Slate contrast in a non-default theme.** Mitigation: the chosen slate is a fixed RGB with strong contrast against
  both the footer surface and all state colors; the visual suite renders the exact result for review.
