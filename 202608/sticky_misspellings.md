---
tier: tale
title: Sticky misspelling highlighting in the prompt input widget
goal:
  Words that NORMAL-mode `K` proved misspelled are remembered durably and rendered with a red underline in every prompt
  input from that moment on, with a first-class way to accept a word and stop flagging it.
proposed_by: bbugyi200.athena.qx
create_time: 2026-08-01 07:04:20
status: wip
---

- **PROMPT:** [202608/prompts/sticky_misspellings.md](prompts/sticky_misspellings.md)

# Plan: Sticky misspelling highlighting in the prompt input widget

## Problem

`PromptTextArea` deliberately does not spell-check as you type: running `aspell` over every word on every keystroke is
expensive and unreliable. The result is that `K` (`_prompt_word_lookup.py`) is the only place a misspelling is ever
surfaced, and the knowledge evaporates the moment the correction panel closes. Decline a suggestion and the prompt looks
exactly as it did before — no trace that the word is wrong.

The fix does not need a live spell checker. Every `K` press that reaches `aspell` produces a verdict we already paid
for. Remember the misspelled verdicts and the prompt can highlight those words forever after, for free, with no
subprocess on any render or keystroke path.

## Design

### The mental model: a personal misspelling list

The user is teaching ACE their own misspellings, one `K` press at a time. That framing drives every decision below:

- **Only proven verdicts count.** A word is remembered only when `aspell` returned `status == "misspelled"`. A `dict`
  `no_match` is _not_ a misspelling (many correctly spelled words have no `dict` entry), and `unavailable`/`error` teach
  nothing.
- **The list is durable and global**, like `~/.sase/prompt_word_deletions.json`. Learning a misspelling in one ACE
  session must show up in the next one, and in every prompt input.
- **The list must be escapable.** Without an "accept this word" affordance, one `K` on `Bugyi` disfigures every future
  prompt with no way out. That is unacceptable for a feature the user cannot turn off per-word, so an accept action is
  part of the core feature, not a follow-up.
- **The list self-heals.** If `K` on an already-flagged word later returns `correct` (the user added it to their
  `aspell` personal dictionary), the entry is dropped silently.

### Store shape

`~/.sase/prompt_misspellings.json`, guarded by `~/.sase/prompt_misspellings.lock`:

```json
{
  "version": 1,
  "misspelled": ["recieve", "teh"],
  "allowed": ["Bugyi", "sase"]
}
```

Two lists in one file, because they are two states of the same decision and must be updated atomically relative to each
other. `allowed` exists solely so that pressing `K` on an accepted word does not immediately re-add it — without it,
"accept" would be undone by the next lookup.

Follow `src/sase/history/prompt_placeholders.py`, not `prompt_word_deletions.py`: several `sase ace` processes can run
at once, so every mutation is a read-modify-write inside an `fcntl` lock, followed by the atomic `tempfile` +
`os.replace` write both modules already use.

Words are stored with their first-seen spelling but compared **casefolded**, and only one entry per casefold key is
kept. `recieve` flagged at the start of a sentence must still squiggle as `Recieve` later. Because the list only ever
contains words `aspell` already rejected, the usual case-folding hazards (`US` vs `us`, `Polish` vs `polish`) cannot
arise.

### What gets highlighted

Reuse the exact word semantics `K` uses, so the invariant _"anything squiggled is something `K` can target"_ holds by
construction. `extract_lookup_word()` in `src/sase/core/word_lookup.py` already encodes those rules (Unicode letters,
interior `'` and `-` only, reject any run containing a digit or `_`, 64-character cap). Add a whole-text scanner beside
it that shares the same acceptance helper rather than re-deriving the rules.

Words inside code literals are skipped, using `code_literal_ranges()` exactly as `_todo_highlight.py` does — fenced
blocks and inline code are quoted material, not prose. Jinja/xprompt/placeholder token interiors are **not** excluded: a
token spelled with a word the user personally flagged is still worth flagging, and every structural overlay paints after
this one anyway (see MRO placement), so structural styling always wins on true overlap.

### What it looks like

Rich's `Style` exposes only `underline` and `underline2`; it cannot emit SGR `4:3` (undercurl) or SGR `58` (underline
color), and injecting raw escapes into `Segment.text` would corrupt Textual's cell-width accounting. A literal wavy line
is therefore off the table, and this plan does not attempt one.

The chosen treatment is a **red underline**: `Style(color="#FF8787", underline=True)`. `#FF8787` is the exact salmon
`SpellcheckPanelModal` already paints the misspelled word in, so the color in the correction panel and the color in the
prompt are the same color — the squiggle reads as "this is the thing that panel was about". It is deliberately softer
than the theme `error` red (which `jinja.error` owns) so flagged prose stays comfortable to read, and it carries no bold
or italic. Promote the literal to a shared constant so the modal and the overlay cannot drift.

### Performance shape

Everything here obeys `sase/memory/tui_perf.md`:

- **The render path never touches disk.** The overlay reads an app-held warm `frozenset`; the store is read once,
  off-thread, by a worker.
- **Zero cost when unused.** `_build_highlight_map()` returns immediately when the set is empty, which is every user who
  has never pressed `K` on a misspelling.
- **Spans are cached** per `(text, generation)`, mirroring `_todo_cache_text`, plus the standard `_MAX_OVERLAY_BYTES` /
  `_MAX_OVERLAY_LINES` guards.
- **Recording is optimistic then persisted.** The in-memory set is mutated synchronously, the generation bumped, the
  overlay refreshed — then the JSON write happens in a worker. This is what makes the squiggle appear the instant the
  panel is dismissed.

### Rust core boundary

This stays in Python. The adjacent behavior it extends is already Python (`src/sase/core/word_lookup.py` shells out to
`aspell`/`dict`; `src/sase/history/` owns the sibling JSON word stores), and this feature is a TUI-local presentation
memory rather than shared domain logic another frontend would need to mirror. Moving `word_lookup` as a whole to
`sase-core` is a separate question this plan does not open.

## Implementation

### 1. Durable store — `src/sase/history/prompt_misspellings.py` (new)

Model on `src/sase/history/prompt_placeholders.py` for locking and atomic writes.

```python
MisspellingsSourceToken = tuple[str, int, int]

@dataclass(frozen=True, slots=True)
class MisspellingSets:
    misspelled: tuple[str, ...]
    allowed: tuple[str, ...]

def load_misspellings() -> MisspellingSets: ...
def record_misspelling(word: str) -> bool: ...   # no-op if casefold in `allowed`
def allow_word(word: str) -> bool: ...           # remove from `misspelled`, add to `allowed`
def forget_misspelling(word: str) -> bool: ...   # remove from `misspelled` only
def misspellings_source_token() -> MisspellingsSourceToken: ...
```

- Every mutation runs inside the `fcntl` lock as read-modify-write, then atomic replace.
- Membership and dedupe are casefolded; the stored spelling is the first seen for that key.
- Cap each list at `ace.prompt_spellcheck.max_remembered_words` (default `5000`), trimming oldest-first. Ordering in the
  file is insertion order, so the trim is a slice.
- A missing, unreadable, corrupt, or version-mismatched file yields empty sets — never raises.
- `record_misspelling` never raises; failures log at debug like `record_prompt_placeholders`.

### 2. Word scanning — `src/sase/core/word_lookup.py`

Extract the acceptance rules currently inline in `extract_lookup_word()` into a shared private helper, then add:

```python
def natural_word_ranges(text: str) -> Iterator[tuple[int, int, str]]:
    """Yield ``(start, end, word)`` for every K-targetable word in *text*."""
```

Absolute character offsets over the whole document (word runs never cross newlines, since `\n` is neither alpha nor a
connector). `extract_lookup_word()` must be refactored to call the same helper so the two cannot diverge. Export the new
name from `__all__`.

### 3. App-held warm cache — `src/sase/ace/tui/actions/_startup_misspellings.py` (new)

`StartupMisspellingsMixin`, modelled directly on `_startup_common_placeholders.py`:

| Member                            | Behavior                                                                                                              |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `misspelled_words()`              | Warm casefolded `frozenset[str]`; `frozenset()` while cold. Never touches disk.                                       |
| `misspellings_generation()`       | Counter bumped on every publish; overlays fold it into their cache key.                                               |
| `warm_misspellings()`             | Off-thread staleness check against the source token, then reload and publish. Coalesces with in-flight/pending flags. |
| `record_misspelling(word)`        | Optimistic in-memory add + publish, then off-thread persist. No-op if already present or accepted.                    |
| `allow_word(word)`                | Optimistic in-memory remove + publish, then off-thread persist.                                                       |
| `forget_misspelling(word)`        | Same, without adding to `allowed`.                                                                                    |
| `_refresh_misspelling_overlays()` | Walk `self.query(PromptTextArea)` and refresh mounted ones, mirroring `_refresh_visible_common_placeholder_surfaces`. |

Wire it into `StartupMixin`'s bases in `src/sase/ace/tui/actions/startup.py`, declare the attributes there, and
initialize them in `src/sase/ace/tui/actions/_state_init_late.py` alongside the common-placeholder fields.

The first warm is scheduled from `PromptTextArea.on_mount` through `getattr(self.app, "warm_misspellings", None)` — the
same lazy "warm on first use" shape the history-word and placeholder caches use, keeping data-scaled work off the
startup stopwatch. It only _schedules_ a worker; the file read is off-thread. Cold means no squiggles for a beat, then
the publish repaints them.

### 4. Overlay — `src/sase/ace/tui/widgets/_misspelling_highlight.py` (new)

`MisspellingHighlightMixin`, modelled on `_yank_highlight.py` / `_todo_highlight.py`:

- Theme registration for `spell.misspelled` via `on_mount`, `_app_theme_changed`, and
  `_register_jinja_text_area_theme(..., apply=False)` chaining, so the style survives theme switches and lands on the
  shared `sase-jinja-prompt` theme.
- `_build_highlight_map()` calls `super()`, then bails immediately when the warm set is empty, when
  `ace.prompt_spellcheck.highlight` is off, or when the text exceeds `_MAX_OVERLAY_BYTES` / `_MAX_OVERLAY_LINES`.
  Otherwise it appends one `"spell.misspelled"` span per matching word from `natural_word_ranges()`, skipping ranges
  inside `code_literal_ranges(text)`.
- Spans cached per `(text, generation)`; the generation is read through
  `getattr(self.app, "misspellings_generation", None)` so non-Ace hosts degrade to no-op.
- `_refresh_misspelling_overlay()` → `self._build_highlight_map(); self.refresh()`.

**MRO placement:** insert between `PlaceholderHighlightMixin` and `JinjaHighlightMixin` in `PromptTextArea`'s base list.
Bases listed _earlier_ append later and therefore win; this slot puts the squiggle above base Markdown but below every
structural overlay (xprompt, artifact, alt, placeholder, code block) and below the transient search/yank feedback.

### 5. Recording on `K` — `src/sase/ace/tui/widgets/_prompt_word_lookup.py`

In `_handle_spelling_terminal_result`:

- On `status == "misspelled"`, record **before** notifying or pushing the panel, in both the has-suggestions and
  no-suggestions branches. Recording first is what makes an `Esc` from the panel land on an already-squiggled word.
- On `status == "correct"`, self-heal: if the word is in the warm set, call `forget_misspelling`. Guard on the cheap
  membership test so the common path does nothing.

Add small `_record_prompt_misspelling` / `_forget_prompt_misspelling` helpers that resolve the app provider with
`getattr` and no-op when it is absent.

### 6. Accept affordance — `src/sase/ace/tui/modals/spellcheck_panel_modal.py`

- Change the dismiss type from `str | None` to `SpellcheckChoice | None`:

  ```python
  @dataclass(frozen=True, slots=True)
  class SpellcheckChoice:
      action: Literal["apply", "accept"]
      suggestion: str = ""
  ```

- Add `a` → accept: dismisses with `action="accept"`, and the callback calls `allow_word` and notifies
  `"'<word>' will no longer be flagged as misspelled"`.
- **Allow zero suggestions.** Today the modal raises `ValueError` on an empty tuple and the caller falls back to a bare
  notification — which leaves no way to accept a word `aspell` has no ideas about, exactly the proper-noun case that
  needs accepting most. Render a single dim `no suggestions` row instead, with `Enter` inert and `a`/`Esc` live, and
  delete the `no suggestions` notify branch in `_prompt_word_lookup.py`.
- Footer becomes `1-9 apply | j/k move | a accept | Enter apply | Esc cancel`; the no-suggestions variant drops the
  `1-9` and `Enter` hints.
- `_apply_spelling_suggestion` handles the union: `apply` keeps the existing changed-before-apply guard, `accept` calls
  `allow_word`, `None` does nothing (and the word stays flagged, which is the point).
- Add a `.spellcheck-empty-row` rule near the existing `SpellcheckPanelModal` block in `src/sase/ace/tui/styles.tcss`.

### 7. Configuration

New `ace.prompt_spellcheck` section — it is not a completion setting and does not belong under `prompt_completion`:

```yaml
prompt_spellcheck:
  highlight: true
  max_remembered_words: 5000
```

- Add the block to `src/sase/default_config.yml`.
- Add the section to `src/sase/config/sase.schema.json`; `ace` is `additionalProperties: false`, so an unlisted key is a
  validation error.
- Add a `PromptSpellcheckSettings` frozen dataclass plus `parse_prompt_spellcheck_settings()` beside
  `PromptCompletionSettings`, parse it in `_state_init_late.py`, and expose `get_prompt_spellcheck_settings()` on the
  app.
- The store reads `max_remembered_words` through the cached merged config, the way `_common_placeholder_limit()` does,
  so no extra disk I/O reaches the record path.
- `highlight: false` suppresses painting only; `K` still records, so flipping it back on restores the full list.

### 8. Test harness — `src/sase/ace/testing/editors.py`

Give `_PromptTestApp` an in-memory implementation of the five provider methods (`misspelled_words`,
`misspellings_generation`, `record_misspelling`, `allow_word`, `forget_misspelling`, `warm_misspellings`) and add a
`misspellings: Sequence[str] = ()` keyword to `PromptPage`. Widget tests then seed the set without touching `~/.sase`.

### 9. Documentation

- `docs/ace.md` "Word definitions & spellcheck": document that `K` on a misspelling remembers the word, that remembered
  words get a red underline in every prompt input, that `a` accepts a word, that `K` on a now-correct word clears it,
  and the store path.
- `docs/configuration.md`: add the `ace.prompt_spellcheck` section to the `ace` table and a subsection matching the
  `ace.prompt_completion` format.
- `src/sase/ace/tui/modals/help_modal/guide_view.py`: mention the sticky squiggle in the prompt section. Leave the
  `binding_common.py` `K` row alone — it is at the 32-character limit.

## Tests

| File                                                                          | Coverage                                                                                                                                                                                                                                                                                                                         |
| ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tests/history/test_prompt_misspellings.py` (new)                             | record / allow / forget round trips; `record` no-ops on an accepted word; casefold dedupe and first-seen spelling retention; cap trim; missing / corrupt / wrong-version file yields empty sets; atomic replace leaves no temp files; concurrent locked read-modify-write does not lose an entry; source token changes on write. |
| `tests/test_word_lookup.py` (extend)                                          | `natural_word_ranges` invariant — for every yielded range, `extract_lookup_word` at each interior column returns the same word and columns; digits/underscore runs rejected; interior `'`/`-` accepted and edge connectors trimmed; 64-character cap; multi-line offsets.                                                        |
| `tests/ace/tui/widgets/test_prompt_misspelling_highlight.py` (new)            | spans appear for a seeded word; case-insensitive match; no spans when the set is empty; words inside inline code and fenced blocks skipped; substrings of larger words not matched; cache invalidates on generation bump and on text change; `_MAX_OVERLAY_*` guards; `highlight: false` paints nothing.                         |
| `tests/ace/tui/widgets/test_prompt_normal_mode_word_lookup.py` (extend)       | `K` on a misspelling records it before the panel opens; `Esc` returns to a squiggled word; the no-suggestions path opens the panel instead of notifying; `a` accepts and clears the squiggle; `K` on a now-`correct` remembered word forgets it; `unavailable` / `error` verdicts record nothing.                                |
| `tests/ace/tui/actions/test_startup_misspellings.py` (new)                    | warm publishes and bumps the generation; a matching source token skips the reload; optimistic record is visible before the persist worker finishes; a failing persist leaves the in-memory set intact; overlay fan-out reaches mounted `PromptTextArea`s; a store read failure degrades to an empty set.                         |
| `tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py` (extend) | dark and light PNG snapshots of a prompt containing a remembered misspelling, proving the underline and salmon read correctly against both themes. Add the prompt constant to `_ace_prompt_png_snapshot_helpers.py`.                                                                                                             |
| Config schema tests                                                           | the new `ace.prompt_spellcheck` block validates and unknown keys under it still fail.                                                                                                                                                                                                                                            |

## Verification

```bash
just install
just fmt
just check
just test-visual   # accept the two new goldens with --sase-update-visual-snapshots
```

Manual pass in `sase ace`: press `K` on a misspelled word, `Esc` out of the panel, confirm the red underline is already
there; type the same word again on a new line and confirm it squiggles; press `K` and `a` and confirm the underline
disappears everywhere; restart ACE and confirm the remaining flags survive.

## Risks and rejected alternatives

- **A real wavy underline.** Rejected: Rich cannot emit SGR `4:3` or `58`, and smuggling raw escapes through
  `Segment.text` breaks cell-width accounting. Red underline is the honest ceiling here.
- **Writing to the user's `aspell` personal dictionary on accept.** Rejected: an accept inside ACE should not mutate
  global user state outside SASE. The `allowed` list is reversible and lives with the rest of SASE's prompt state.
- **Highlighting every misspelled word live.** Explicitly out of scope — that is the expensive, unreliable thing this
  design routes around.
- **Reusing `prompt_word_deletions.py`'s lock-free write.** Rejected: concurrent `sase ace` processes would lose
  entries; the `prompt_placeholders.py` `fcntl` pattern is the precedent that already handles this.
- **False positives from casefolded matching.** Bounded by construction: only words `aspell` personally rejected for
  this user are ever in the set.
- **Cold-cache flicker.** The first paint after launch may briefly lack squiggles until the warm worker publishes.
  Accepted: the alternative is disk I/O before first paint, which `sase/memory/tui_perf.md` rule 9 forbids.
