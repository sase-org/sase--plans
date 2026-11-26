---
tier: epic
title: Question copy panel
goal: 'From the questions gate, the user can copy any agent-declared snippet, the
  question text, the highlighted option, or the whole Q&A with two keypresses (`y`
  then one key), exactly byte-for-byte, with visible progress through multi-paste
  workflows. The same structured copy contract reaches Telegram and agent guidance.

  '
decisions:
  prose_highlight:
    ask: Underline declared snippets inside the question text (teal, turning green
      once copied)?
    default: true
    why: Shows what y can copy without adding glyphs, so wrapping and mouse selection
      stay intact
    answer: true
phases:
- id: copy-contract
  title: Core copy contract and creation-time validation
  depends_on: []
  size: medium
  description: 'copy-contract: add the strict and tolerant `copy` normalizers, wire
    types, and PyO3 bindings to sase-core, then route `sase questions` and question-gate
    creation through them in sase while read paths stay tolerant.'
- id: palette-foundation
  title: Generalize the Copy as palette
  depends_on: []
  size: medium
  description: 'palette-foundation: extend CopyAsModal and CopyAsContext with custom
    sections, title, footer, marker gutter, aligned label column, badges, initial
    highlight, and an opt-in verbatim preview pane, leaving every existing palette
    pixel-identical.'
- id: copy-panel
  title: Question copy panel behind y
  depends_on:
  - copy-contract
  - palette-foundation
  size: medium
  description: 'copy-panel: rebind `y` in UserQuestionModal to open a QuestionCopyModal
    with SNIPPETS and QUESTION sections, stable keys, ordered exact delivery, copied
    checkmarks, a `yy` copy-all path that adds Copyable items, and tests, goldens,
    and TUI docs.'
- id: pane-cues
  title: Discoverability cues on the question pane
  depends_on:
  - copy-panel
  size: small
  description: 'pane-cues: show copy progress on the question pane border, underline
    declared snippets in the prose, and list declared items missing from the prose
    in a muted "also copyable" block so no copiable value is hidden.'
- id: found-in-text
  title: Conservative found-in-text detection
  depends_on:
  - pane-cues
  size: medium
  description: 'found-in-text: add a pure sase-core detector for fenced code, filtered
    backtick spans, scheme URLs, and SASE artifact refs, and render its results as
    a dim FOUND IN TEXT section in the copy panel.'
- id: telegram-copy
  title: Telegram copy buttons
  depends_on:
  - copy-contract
  size: small
  description: 'telegram-copy: render declared snippets on live Telegram question
    messages as CopyTextButton rows, and list long or out-of-prose values in the message
    text so Telegram shows every declared value.'
- id: agent-guidance
  title: Agent guidance, contract docs, and skill rollout
  depends_on:
  - pane-cues
  - telegram-copy
  size: small
  description: 'agent-guidance: teach agents the `copy` field through the sase_questions
    skill source with a worked example and authoring rules, document the contract,
    and request the post-landing skill deploy.'
proposed_by: bbugyi200.athena.1gp
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 10:49:23
status: wip
bead_id: sase-1vd
---

- **PROMPT:** [prompts/202611/question_copy_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/question_copy_panel.md)
- **BEAD:** [sase-1vd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vd/README.md)

# Plan: Question copy panel

## Why

Agents often ask questions that the user can only answer after pasting exact values
somewhere else: paths into a macOS folder picker, a `chrome://` URL, a command, an ID.
Today `y` in the questions gate copies the whole ballot as Markdown, and that is the
only copy option. Selecting text with the mouse breaks on exactly the hard cases: the
motivating question's `/Users/bbugyi/Library/Application Support/sase` wraps across two
rendered lines over SSH and tmux. The research report
`research:202611/question_copy_items/question_copy_items.md` measured the demand: about
one in three retained questions contains at least one obvious copy candidate, and one in
five contains two or more. Read it with `sase artifact read` before starting any phase.
This plan implements that report's consolidated recommendation, and the user has
accepted every requirement in it.

### Why a structured field instead of inline `copy[label]: text`

The user floated an inline `copy[<label>]: <text>` syntax. All five researchers rejected
it, the user accepted that finding, and this plan uses an optional structured `copy`
array on each question instead. The inline syntax has four problems:

- **Its delimiters are ambiguous.** Spaces, commas, and periods all occur inside real
  payloads (`Application Support`, `.build.`, `chrome://`), and an inline marker cannot
  hold a multi-line command.
- **It leaks onto other surfaces.** The `question` string is rendered verbatim by the
  TUI, successor prompts, notifications, the QUESTIONS deck, Telegram, and mobile.
- **It breaks an exact-string invariant.** `question_response_schema()` pins
  `"question": {"const": …}`, so the question text cannot be rewritten to strip markup.
- **It costs agents nothing to switch.** Agents already emit JSON to `sase questions`,
  so one more field adds no new escaping rules.

An inline form can later become sugar that compiles into the same array at parse time,
but it is out of scope here.

## Design

### The contract (owned by sase-core; see `decisions:rust-core-required`)

Each question object may carry an optional `copy` array. An entry is either a bare
string or an object with a `text` and an optional `label`:

```json
{
  "header": "Mac GUI",
  "question": "… Spotlight: System Settings → Spotlight → Search Privacy → + → paste each folder … Chrome: open chrome://settings/performance … What did you do?",
  "copy": [
    { "label": ".sase", "text": "/Users/bbugyi/.sase" },
    {
      "label": "App Support/sase",
      "text": "/Users/bbugyi/Library/Application Support/sase"
    },
    "/Users/bbugyi/.cargo",
    "/Users/bbugyi/.rustup",
    {
      "label": "bob-cli/target",
      "text": "/Users/bbugyi/projects/github/bobs-org/bob-cli/target"
    },
    {
      "label": "sase-core/target",
      "text": "/Users/bbugyi/projects/github/sase-org/sase-core/target"
    },
    {
      "label": "mac-capture/.build",
      "text": "/Users/bbugyi/projects/github/bobs-org/bob-mac-capture/.build"
    },
    { "label": "Chrome Memory Saver", "text": "chrome://settings/performance" }
  ],
  "options": [{ "label": "Done, both" }, { "label": "Skip for now" }]
}
```

The **strict normalizer** runs when questions are created: at `sase questions` time,
before the handoff stages (`decisions:handoffs-end-through-host-completion`), and again
at gate creation. Its rules:

- `copy` must be an array. An empty array is accepted and the key is dropped.
- At most 20 entries. Array order is authoritative and means paste order.
- An entry is either a string (shorthand for `{"text": …}`) or an object whose keys are
  a subset of `text` and `label`. Any other key is an error, so typos like `value` or
  `name` fail loudly. A `null` label means no label.
- `text` rules:
  - Trailing `\n` characters are stripped, so a pasted one-line command never
    auto-executes. Everything else is kept byte-for-byte: no trimming, quoting, tilde
    expansion, or Unicode normalization.
  - The text must be non-empty, must contain at least one non-whitespace character, and
    must be at most 4096 UTF-8 bytes.
  - No control characters (Unicode `Cc`) except `\n` and `\t`. This rejects ESC (no ANSI
    or OSC escapes reaching the terminal or clipboard), `\r`, NUL, DEL, and C1 controls.
  - No bidi embedding, override, or isolate controls (U+202A–U+202E, U+2066–U+2069), so
    the displayed value always matches the copied bytes.
- `label` rules: trimmed, non-empty, at most 48 characters, no control characters at all
  (single line), and the same bidi rule.
- Duplicate `text` values are rejected, and the error names both indexes.
- Every error names its location, for example
  `question[0].copy[3].text: contains control character U+001B; only newline and tab are allowed`.
- The output is canonical: objects only, with `label` omitted when absent. Normalizing
  twice gives the same result as normalizing once.

The **tolerant display parser** reads stored bundles and never breaks the modal. It
applies the same per-entry rules, skips invalid entries and counts them as
`unavailable`, and never guesses, truncates, deduplicates, or reorders. It has no item
cap. A non-array `copy` counts as one unavailable entry, and an absent or `null` `copy`
yields nothing.

Copy metadata is **excluded from successor prompts**: the asking agent wrote these
values and does not need them echoed back. It is **included in the human `yy` export**.

### Interaction

`y` behaves as follows in the questions gate:

- **Outside input mode,** `y` always opens the copy panel, even when the agent declared
  no snippets, so the key behaves the same every time.
- **While typing an "Other" answer or a global note,** `y` stays ordinary typing.
- **While the app's `%` copy mode is active,** `y` goes to copy mode as it does today.
- **Copying never answers.** No row selects an option, submits, or writes a response.

Panel keys:

| Key                                                        | Row                                                                    | Copies                                                                                              |
| ---------------------------------------------------------- | ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `1`–`9`, `0`, then `b c d e f g h i l m n p r s u v w x z` | each snippet (declared first, detected later)                          | the exact text                                                                                      |
| `a`                                                        | All N snippets (shown only when there are 2 or more)                   | declared texts joined with `\n`                                                                     |
| `t`                                                        | Question text                                                          | the `question` string, exactly                                                                      |
| `o`                                                        | Highlighted option                                                     | `label — description` (or `label`); disabled with a reason on "Other…" or when there are no options |
| `y`                                                        | Questions & answers                                                    | today's Markdown export (so `yy` keeps working), plus a Copyable items section                      |
| `⏎` / click                                                | highlighted row                                                        |                                                                                                     |
| `j`/`k`, `↑`/`↓`, `ctrl+n`/`ctrl+p`                        | move                                                                   |                                                                                                     |
| `q` / `esc`                                                | close; nothing is copied and the form is untouched                     |                                                                                                     |
| other printable                                            | "Unknown copy key — use 1–8, a, t, o, y" warning; the panel stays open |                                                                                                     |

Keys are stable: a row's key never changes with scrolling, ✓ state, or selections. The
key pool holds 29 keys. That covers 20 declared snippets plus 5 detected ones with room
to spare, so every row gets a key. Only a damaged historical bundle can exceed the pool,
and its extra rows stay reachable with `j`/`k` and `⏎`.

After a delivery succeeds, the copied snippet gets a ✓ (and `a` marks every snippet).
Each time the panel reopens, it highlights the first uncopied declared snippet, or the
`y` row when none remain. The motivating workflow then becomes
`y ⏎ · paste · y ⏎ · paste …`, and the panel always shows how far you have gotten.
Progress is kept in memory per question bundle for the life of the TUI process, so
closing the question modal to check something does not reset it.

### Layout (wide, dark)

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                                 Copy from question                                   ║
║                        Q1/1 · Mac GUI · 3 of 8 snippets copied                       ║
║ ┌──────────────────────────────────────────────────────────────────────────────────┐ ║
║ │ SNIPPETS                                                                         │ ║
║ │  1  ✓ .sase                 /Users/bbugyi/.sase                                  │ ║
║ │  2  ✓ App Support/sase      /Users/bbugyi/Library/Application Support/sase       │ ║
║ │  3  ✓ /Users/bbugyi/.cargo                                                       │ ║
║ │▌ 4    /Users/bbugyi/.rustup                                                      │ ║
║ │  5    bob-cli/target        /Users/…/bobs-org/bob-cli/target                     │ ║
║ │  6    sase-core/target      /Users/…/sase-org/sase-core/target                   │ ║
║ │  7    mac-capture/.build    /Users/…/bobs-org/bob-mac-capture/.build             │ ║
║ │  8    Chrome Memory Saver   chrome://settings/performance                        │ ║
║ │  a    All 8 snippets        one per line · 8 lines                               │ ║
║ │ QUESTION                                                                         │ ║
║ │  t    Question text         Mac busy-process fixes are measured and recorded…    │ ║
║ │  o    Highlighted option    Done, both — Added all 7 Spotlight folders and…      │ ║
║ │  y    Questions & answers   Markdown · 1 question · current picks                │ ║
║ └──────────────────────────────────────────────────────────────────────────────────┘ ║
║ ╭─ 4 · /Users/bbugyi/.rustup · 21 chars ───────────────────────────────────────────╮ ║
║ │ /Users/bbugyi/.rustup                                                            │ ║
║ ╰──────────────────────────────────────────────────────────────────────────────────╯ ║
║                1–8 copy · a all · ⏎ highlighted · j/k move · esc close               ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

Visual language:

- **Inherited from Copy as.** The panel keeps the `double $primary` frame, the
  `$surface` background, dim bold uppercase section headings, the bold `#00D7AF` key
  column, bold labels, dim previews, `$warning` text for disabled reasons, and
  `$success` checkmarks.
- **Columns.** Labels sit in one aligned column so the values line up.
- **Long values.** A path-like value that is too long for its row is middle-truncated:
  it keeps its root and as many trailing segments as fit
  (`/Users/…/bobs-org/bob-cli/target`), because the tail is the distinctive part. Other
  long values are end-truncated with `…`.
- **Multi-line values.** The row shows the first non-empty line plus a dim `+N lines`
  badge, an honest warning before pasting into a shell.
- **Preview pane.** It shows the highlighted value verbatim and wrapped, up to 6 lines,
  then a dim `… +N more lines`. Its border title gives the key, the label or value, and
  the size (`N chars` or `N lines`).
- **Size.** The panel is 100 columns wide, capped at 92% of the screen. In `-narrow`
  mode (90 columns or fewer, or 32 rows or fewer) it uses 94% of the width and a 3-line
  preview. The key and label columns are never hidden.

### Question pane cues

These cues let the user discover copiable items before pressing `y`:

- **Footer.** The hint becomes `y=Copy…`; the ellipsis is the usual sign that a chooser
  opens.
- **Border subtitle.** A dim right-aligned subtitle on the question text box reads
  `8 copyable · y`, then `3/8 copied · y`, then `✓ 8/8 copied` in green. When there are
  unavailable items it appends `· 2 unavailable` in `$warning`. Questions without copy
  items show nothing new. Avoid glyphs of ambiguous width: no emoji and no `⧉`.
- **Underlines.**

  > [!decision] prose_highlight Every verbatim occurrence of a declared snippet in the
  > prose is underlined in teal (`#00D7AF`), and turns green once copied. This uses
  > `Text` spans only, so no characters are inserted.

  > [!decision] prose_highlight = no The prose renders exactly as today, and the
  > subtitle and the also-copyable block carry the cues.

- **Also copyable.** Declared items that do not appear verbatim in the prose are listed
  in a muted block under the question text, so the user can read every value before
  copying or answering:

```
  ── also copyable · y then key ──
   9  Reset command          launchctl kickstart -k gui/501/com.apple.Spotlight
```

### Reliability rules

These apply to every phase:

- **One delivery path.** Every clipboard write goes through the existing seam
  (`schedule_copy_delivery` / `deliver_copy` in
  `src/sase/ace/tui/actions/clipboard/_delivery.py`): the system clipboard, then OSC 52,
  then the selectable fallback modal. Keep its honest status distinction. Add no new
  clipboard code, never probe the user's clipboard, and never reconfigure tmux.
- **Ordered delivery.** Rapid successive copies are delivered in the order they were
  chosen, so a slow earlier delivery never overwrites a later choice. Add this as an
  ordering option at the delivery seam, not as a bypass around it.
- **Exact bytes end to end.** Space-containing paths, `chrome://` URLs, `[/]` and Rich
  tokens, quotes, Unicode, tabs, and indented multi-line text all arrive unchanged.
  Tests check this byte-for-byte.
- **Agent text is data, never markup.** Labels, previews, prose, and payloads render
  through `Text` or `Content` with no markup parsing (see
  `tests/ace/tui/test_user_question_modal_markup.py`).
- **Event isolation.** `⏎` in the panel never submits the form, `esc` closes only one
  layer, and focus returns to the options list. Opening, copying, or cancelling keeps
  the question index, option highlight, selections, drafts, and scroll position.
- **Tolerant history.** Damaged metadata shows as unavailable. The panel never copies a
  guessed or truncated value.

### Cut, deferred, and out of scope

The research reviewed these options:

- **Question JSON row:** cut, because `d` (gate debug) already copies it.
- **Snapshot row:** cut, because `%` forwarding already reaches it.
- **Draft, selection, and global-note rows:** cut, because the user just typed or picked
  them.
- **Per-question Q&A row:** deferred until multi-question ballots prove common.
- **Open URL or run command actions:** cut, because they would act on the wrong machine
  over SSH and cross the line between copying and doing.
- **Space and Tab stay-open modes:** deferred, because they conflict with the palette's
  dismiss-on-select contract and the ✓ loop already makes repeated copies fast.
- **Digit shortcuts on the question modal itself:** rejected, because gate modals bind
  digits to decision branches.

This plan also defers carrying `copy_items` into the QUESTIONS deck projection
(`QuestionEntryWire` is `deny_unknown_fields`, and the deck shows each question on one
line with no copy interaction). Mobile-app rendering is deferred as well. Phase agents
should record a `PROPOSED FOLLOW-UP:` only if they find a concrete need.

No feature flag is needed. The schema change is additive, `yy` keeps the old behavior
reachable, and no landed phase exposes an unfinished feature: agents only learn about
`copy` in the last phase, after every rendering surface can show it.

## Phase: copy-contract

This phase spans sase-core and sase. Open the linked checkout with
`sase repo open sase-core` and follow its `AGENTS.md`. When one declaration commits both
repos, the host commits sase-core first and moves `sase-core-revision.txt`
automatically. Only if the sase change lands separately, bump the pin with
`just ratchet-core-revision`.

**sase-core**

- Add a pure module (for example `crates/sase_core/src/question_copy/` with `wire.rs`,
  `normalize.rs`, and `tests.rs`) implementing the contract above:
  - `QuestionCopyItemWire { text, label? }`, with `deny_unknown_fields` and `label`
    skipped when `None`.
  - `QuestionCopyDisplayWire { items, unavailable }`.
  - The strict
    `normalize_question_copy(&Value) -> Result<Vec<QuestionCopyItemWire>, QuestionCopyError>`.
    The error carries a stable code (`invalid_question_copy`), the entry index, the
    field, and a message.
  - The tolerant `display_question_copy(Option<&Value>) -> QuestionCopyDisplayWire`.
  - The limits as public constants.
- Add PyO3 bindings in `crates/sase_core_py`:
  - `normalize_question_copy(value)` returns `list[dict]`, or raises `ValueError` whose
    message starts with the code.
  - `display_question_copy(value)` returns `{"items": [...], "unavailable": n}`.
- Rust tests cover:
  - every rule and error location;
  - idempotence;
  - the 8-entry Spotlight fixture above, with byte-exact paths;
  - bidi and control characters;
  - duplicates;
  - 20 versus 21 entries;
  - 4096 versus 4097 bytes;
  - multi-byte UTF-8 at the boundary;
  - tolerant counting of mixed valid and invalid entries.

**sase**

- Add a thin facade, `src/sase/core/question_copy_facade.py`, following the
  `question_round_facade.py` pattern:
  - `normalize_questions_copy(questions)` returns question dicts with a canonical `copy`
    (the key is removed when empty). It raises an error whose location is
    `question[i].copy[j]…`.
  - `display_question_copy_items(question)` returns a frozen `QuestionCopyDisplay`
    (`items: tuple[QuestionCopyItem(text, label)]`, `unavailable: int`).
- Register both bindings with `tools/validate_sase_core_rs` (the required-bindings list
  plus a probe). Call them through `require_rust_binding` with string-literal names, so
  the static scan in `tools/check_sase_core_rs_bindings` picks them up.
- In `src/sase/main/questions_command_handler.py`, `_validate_questions` normalizes
  `copy` strictly, and the staged record carries the normalized questions. An invalid
  `copy` fails with `Error: Invalid questions schema: question[0].copy[3].text: …`
  before anything stages.
- In `src/sase/user_question_actions.py`, `user_question_gate_spec` normalizes strictly
  (`UserQuestionActionError("invalid_question_copy", "payload.questions[i].copy[j]…")`).
  In `notification_gates/kind_validation/question.py`, `validate_question_spec` rejects
  any `copy` that is invalid or not already canonical.
- `validate_user_questions` also runs on read paths (`_questions_from_request_root`,
  `automatic_question_response`). It must keep passing `copy` through untouched, so
  history stays tolerant.
- Tests:
  - CLI rejection messages for each rule class;
  - `copy` survives staging, then the gate payload, then the resolved request, in
    canonical form;
  - a hand-damaged stored bundle still resolves, and its display reports `unavailable`;
  - a question-gate follow-up prompt built from a question with copy items contains none
    of the out-of-prose values and no "Copyable items" text (`question_gate_turn`,
    `qa_prompt`).

## Phase: palette-foundation

Generalize `src/sase/ace/tui/modals/copy_as_modal.py` and `copy_as_types.py` so that one
palette look serves both the Copy as palette and the question panel. Every existing
palette must stay pixel-identical: the goldens in
`tests/ace/tui/visual/test_ace_png_snapshots_copy_as_palette.py` must pass without being
regenerated.

- Optional `CopyAsContext` fields, each defaulting to today's behavior:
  - `title` (default `"Copy as…"`);
  - `sections` (an ordered tuple of `CopyAsSection(id, title, dim=False)`; empty means
    today's `_CATEGORY_ORDER` grouping);
  - `footer`;
  - `unknown_key_message`;
  - `initial_row_key`;
  - `key_width` (default 9);
  - `label_width` (`None` means unaligned);
  - `preview_pane` (default off).
- Optional `CopyAsRow` fields:
  - `section` (used when `sections` is set);
  - `marker` (rendered in a two-column gutter, shown only when some row in the context
    has a marker or the context asks for the gutter);
  - `badge` (dim, right of the preview);
  - `value` (the exact text shown in the preview pane).
  - `category` becomes optional for section-based rows. Registry rows keep passing it.
- Preview pane: a bordered widget under the list.
  - It shows the highlighted row's `value` as `Text`, wrapped, never markup.
  - It holds at most 6 lines (3 in `-narrow`), then a dim `… +N more lines`.
  - Its border title shows the key, the label, and the size.
  - A disabled row shows its reason in `$warning`.
  - It updates on `OptionList.OptionHighlighted`.
- Initial highlight: `initial_row_key` wins when the row exists and is enabled.
  Otherwise the first enabled row is highlighted, as today.
- Keep accelerator precedence, the unknown-key toast, and the
  `on_option_list_option_selected` handling exactly as they are.
- Unit tests for each new field: section grouping and order, the marker gutter, aligned
  labels, badges, the preview pane (verbatim brackets, multi-line overflow, disabled
  reason), and the initial highlight. One new golden: a section-based context with the
  preview pane, rendered with a synthetic context in a bare test app.

## Phase: copy-panel

Wire the panel into `UserQuestionModal`. `user_question_modal.py` is already 661 lines,
close to the `toobig` limit of 700, so new code goes into new modules:

- `src/sase/ace/tui/modals/question_copy_items.py`: pure presentation helpers.
  - The key allocator: `1234567890` followed by `bcdefghilmnprsuvwxz`. The pool never
    includes `a j k o q t y`.
  - Path-aware middle truncation, and the multi-line first-line summary.
  - Building the rows into a `CopyAsContext` for a question snapshot:
    - The title is "Copy from question".
    - The subtitle is `Q{n}/{total} · {header} · {k} of {N} snippets copied`, or
      `· no snippets`.
    - Sections are SNIPPETS and QUESTION.
    - The footer is dynamic.
    - When `unavailable > 0`, an informational disabled row reads
      `N snippets unavailable — malformed copy metadata`.
  - The toast labels: the item label (or the truncated value), `8 snippets`,
    `question text`, `option "Done, both"`, and `questions & answers`, so the `yy` toast
    is unchanged.
- `src/sase/ace/tui/modals/question_copy_modal.py`: `QuestionCopyModal(CopyAsModal)`, a
  thin subclass that gets its own CSS (100 columns wide, a `-narrow` variant) in
  `styles.tcss`, plus its Footprint catalog entry.
- `src/sase/ace/tui/modals/user_question_modal_copying.py`: a mixin, following the
  `artifact_files_modal_copying.py` pattern.
  - `action_open_copy_panel` returns immediately in input mode or while app copy mode is
    active.
  - Before building the snapshot, it calls `_save_current_answer()` (as `y` does today)
    and records the highlighted option.
  - It dispatches the dismissed row through the ordered delivery seam.
  - It marks ✓ only on a `VERIFIED` or `OSC52_ONLY` outcome, and guards against a modal
    that has since been dismissed.
- Copy progress: an in-memory map on the app, keyed by the question bundle identity, for
  the life of the TUI process.
  `src/sase/ace/tui/actions/agents/_notification_question_modal.py` passes the bundle's
  response path as the key, and a modal opened without one uses instance-local state.
- `user_question_modal.py`: bind `y` to `open_copy_panel`, change the footer hint to
  `y=Copy…`, and mix in the copying mixin.
- `src/sase/main/qa_markdown.py`: add `include_copy_items: bool = False` to
  `build_qa_markdown`, and set it to `True` only from the TUI.
  - Each question that has display items gets a `**Copyable items:**` list after its
    answer lines.
  - A value is rendered verbatim in a code span or fenced block whose fence is longer
    than any run of backticks in the value; multi-line values use the fenced form.
  - With no copy items, the output is byte-identical to today, and successor prompts
    never pass the flag.
- Tests:
  - The key map, including that reserved keys are never assigned and that 20 + 5 rows
    all get keys.
  - `yy` is byte-identical for questions without `copy`, and adds the section when
    `copy` is present.
  - The exactness matrix from the Reliability rules, with the system clipboard transport
    mocked.
  - `y` inside the Other and global inputs types a `y`.
  - With app copy mode active, `y` does not open the panel.
  - Event isolation: `⏎` copies without submitting, `esc` closes one layer, and focus
    returns to the options list.
  - Form state, drafts, and scroll survive the panel.
  - The `o` disabled reasons.
  - Initial highlight in all three states: some uncopied, all copied, none declared.
  - ✓ persists across reopening the modal for the same bundle, and is not set on a
    `FAILED` outcome.
  - Ordered delivery when an earlier transport is slow.
  - Markup safety, for example a label `[/]` or a value `[bold]x`.
- Goldens, following the existing palette-over-modal precedent:
  - the panel over the question modal, wide and dark, using the Spotlight fixture with 3
    copied;
  - narrow and light at 80×24;
  - the no-snippets state.
- Docs: describe the panel's keys in the UserQuestion section of
  `docs/reference/tui/notifications-modal.md`, following the `docs_writing.md` routing.
  Read `tui.md` and `tui_screenshot.md` before capturing goldens.

## Phase: pane-cues

These are presentation changes to the question pane, implemented in the copying mixin or
a small sibling module, never by growing `user_question_modal.py` past `toobig`. They
share the key allocator and the progress state from copy-panel.

- **Border subtitle** on `#user-question-text-scroll`, as specified in the Design
  section. It updates when the question changes and when a ✓ lands.
- **Prose rendering.** Render `#user-question-text` from a `Text` built from the plain
  question string. Under `prose_highlight = yes`, add the underline spans:
  - Take declared texts longest first, mark every non-overlapping occurrence in teal,
    and switch copied texts to green.
  - `Text.plain` must equal the question string exactly (a test checks this), so
    wrapping and mouse selection are unchanged.
  - Under `prose_highlight = no`, this phase skips the underlines.
- **Also copyable block.** List the declared items whose text is absent from the prose
  under the question text, inside the same scroll container.
  - Header: `── also copyable · y then key ──`.
  - Each item shows its key (teal, matching the panel), its label, and its full verbatim
    value, wrapped and with multi-line values kept.
  - Hide the block when every item appears in the prose.
- Goldens:
  - the Spotlight question modal with cues, wide and dark, after 3 copies;
  - narrow and light;
  - a question with an out-of-prose multi-line command.
  - Existing question-modal goldens for questions without `copy` must not change.

## Phase: found-in-text

This phase spans sase-core and sase, and follows the same two-repo commit and pin rules
as copy-contract.

**sase-core detector**

Add
`detect_question_copy_candidates(question_text, declared_texts) -> Vec<{text, kind}>`, a
pure function, plus a binding. The kinds are `code`, `url`, and `ref`. It detects:

- **Fenced code blocks,** reusing `fenced_code.rs`. The candidate is the inner text with
  trailing newlines stripped.
- **Inline backtick spans** that pass a noise filter. Keep a span only if it:
  - contains whitespace, `/`, `://`, `=`, or `--`;
  - or has a `.` between alphanumerics;
  - or is at least 12 characters long.
- **Explicit scheme URLs** (`[a-z][a-z0-9+.-]*://…`, so `chrome://` and `file://`
  count). Strip trailing sentence punctuation, but keep balanced parentheses.
- **SASE artifact refs,** reusing the `artifact_ref` parser.

It never detects bare paths in prose: the research showed that two of the seven
motivating paths come out wrong. Candidates are deduplicated against declared texts and
against each other, keep the order they appear in, and are capped at 5. Apply the same
safety rules as the strict normalizer, and drop a candidate that fails them. Build
fixtures from real retained question bundles, anonymized, alongside the Spotlight
question.

**sase panel section**

- Add a dim FOUND IN TEXT section between SNIPPETS and QUESTION. Its keys continue after
  the declared snippets, and each row has a `code`/`url`/`ref` badge.
- Copies of these rows get a ✓, but the initial highlight never lands on a detected row.
- Detected items are display-only: they never appear in the `yy` export, the
  also-copyable block, the prose underlines, or the CLI contract.
- Update the panel goldens and add one line to the TUI docs.
- If the section proves noisy in real use, remove it rather than adding a setting.

## Phase: telegram-copy

This phase works in the linked `sase-telegram` repo (`sase repo open sase-telegram`; run
`sase tool run check` there).

- In `src/sase_telegram/formatting.py` `render_question_message`, read the display items
  through `sase.core.question_copy_facade.display_question_copy_items`. Import it
  lazily. If an older `sase` lacks it (`ImportError` or `AttributeError`), render
  exactly as today.
- Buttons:
  - Every item of 256 characters or fewer gets an
    `InlineKeyboardButton("📋 <label or truncated value>", copy_text=CopyTextButton(text=item.text))`.
  - Put two buttons per row when both labels are short (about 20 characters or fewer),
    and one per row otherwise.
  - The copy rows go after the existing answer and Submit/Custom rows, so answering
    stays the first action. They survive the re-renders that multi-select toggles
    trigger.
- Message text: add a `Copyable` section that lists, verbatim in MarkdownV2 code
  entities (escape `` ` `` and `\`), every item longer than 256 characters and every
  item whose text does not appear in the prose. When `unavailable > 0`, add an italic
  note.
- `format_answered_question` stays unchanged.
- Tests cover:
  - buttons and row packing;
  - the 256 and 257 character boundary;
  - out-of-prose and long items in the text;
  - MarkdownV2 escaping of tricky values;
  - the fallback when the facade is absent;
  - unchanged output for questions without `copy`.

## Phase: agent-guidance

This phase lands last, so agents learn about `copy` only once the TUI and Telegram can
show it.

- Update the skill source, `src/sase/macros/skills/sase_questions.md`:
  - Add `copy` to the JSON schema.
  - Add the Spotlight worked example in a compact form.
  - Add authoring rules:
    - Declare `copy` for every path, URL, command, ID, or other exact value the user
      must paste.
    - Order items in paste order.
    - Label long or opaque values (48 characters at most).
    - Keep the question readable on its own: short values should also appear in the
      prose, while long multi-line commands may live only in `copy`.
    - Prefer one runnable command over N manual GUI steps.
    - Never split one decision into several questions to create copy targets.
    - Know the limits: at most 20 items of 4 KiB each, only newline and tab as control
      characters, no duplicates. `sase questions` rejects violations with an indexed
      error before staging.
  - Preview the rendered skill with `sase skill init --diff`, and update any
    generated-skill tests.
- Document the contract following the `docs_writing.md` routing. Improve the
  `questions_json` argparse help so it points to the schema, then run `just fix`. Add
  the `copy` field's reference entry where the question JSON is documented, and
  cross-link the panel docs from copy-panel.
- Deploy the generated skill only from a landed tree, per `generated_skills.md`: commit
  first, then run `sase skill init --force`. This phase cannot deploy its own unlanded
  change, so it ends by requesting the post-landing deploy through `/sase_gate`. If a
  gate is not possible, it records a `PROPOSED FOLLOW-UP:`.

## Verification (every phase)

- **sase phases:** run `sase tool run check` (`decisions:guarded-recipes`). Never run
  `just check-full` unless explicitly told to (`decisions:check-full-is-explicit`).
  Capture new or changed PNG goldens with `just fix-tui-screenshots` only after reading
  `tui_screenshot.md`.
- **sase-core:** use the recipes in its `AGENTS.md`.
- **sase-telegram:** run `sase tool run check`.
- **Byte-exact copies:** any phase that touches copying adds or extends tests proving
  it.
