---
tier: tale
title: Add a word to the aspell personal dictionary from the spellcheck panel
goal:
  The spellcheck correction panel gains a `d` action that writes the word to the user's aspell personal dictionary,
  verifies the write actually landed by re-checking in a fresh aspell process, and clears the sticky squiggle only once
  that verification succeeds.
size: medium
proposed_by: bbugyi200.athena.qx.f0
create_time: 2026-08-01 07:57:07
status: done
---

- **PROMPT:** [prompts/202608/aspell_dictionary_add.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/aspell_dictionary_add.md)

# Plan: Add to the aspell personal dictionary from the spellcheck panel

## Problem

`SpellcheckPanelModal` currently offers exactly one way out of a word `aspell` rejected but the user knows is fine: `a`,
which writes to SASE's own `allowed` list in `~/.sase/prompt_misspellings.json`. That stops the squiggle inside ACE and
nothing else. `aspell` still rejects the word, so:

- Every _other_ `aspell` consumer (`sase` elsewhere, the user's editor, `git commit` hooks, anything) keeps flagging it.
- `K` on that word still spends a subprocess proving what the user already decided, and still opens the panel.
- The `allowed` list grows into a shadow dictionary that duplicates, badly, the thing `aspell` already has.

The user is sitting in front of a panel that names the exact word and already knows `aspell` rejected it. That is the
one moment where "teach my spell checker this word" costs a single keypress. Today it costs leaving ACE and editing
`~/.aspell.en.pws` by hand.

## Design

### Two escape hatches, two scopes

After this change the panel offers two ways to stop fighting a word, and the only thing a user needs to understand is
_how far the decision reaches_:

| Key | Reach                                   | Reversible by                              |
| --- | --------------------------------------- | ------------------------------------------ |
| `a` | SASE only. `aspell` still rejects it.   | Editing `~/.sase/prompt_misspellings.json` |
| `d` | The whole machine. `aspell` accepts it. | Editing `~/.aspell.en.pws`                 |

That distinction is carried in three places so it cannot be missed: the footer says `d add to aspell` (naming the tool
that leaves SASE's control), the success notification says `Added 'Bugyi' to your aspell personal dictionary`, and the
docs spell out both scopes side by side. `a` keeps its current wording and behavior — it is shipped, documented, and
tested, and it stays the right choice for a word the user does not want in a global dictionary.

**This deliberately reverses a decision in the previous plan.** `sticky_misspellings.md` rejected "writing to the user's
`aspell` personal dictionary on accept" on the grounds that an accept inside ACE should not mutate global user state.
That objection was about `a` doing it _implicitly_. It does not apply to a separate, explicitly-named key whose label
says which file it writes. `a` continues to touch nothing outside SASE.

### The reliability core: aspell lies about success

This is the finding that shapes the whole implementation. Measured directly against `aspell 0.60.8.1`:

```
$ printf '*well-formedd\n#\n' | aspell -a --encoding=utf-8 --lang=en_US
exit=0
stdout: @(#) International Ispell Version 3.1.20 (but really Aspell 0.60.8.1)
stderr: Error: The word "well-formedd" is invalid. The character '-' (U+2D) may not
        appear in the middle of a word.

$ HOME=<read-only dir> printf '*zzqqword\n#\n' | aspell -a --encoding=utf-8 --lang=en_US
exit=0
stderr: Error: The file ".../.aspell.en.prepl" can not be opened for writing.
```

**`aspell` exits `0` on both a rejected word and an unwritable dictionary file.** The exit code carries no signal, and
stdout carries only the banner. Parsing stderr for `Error:` works for the two cases above but is a guess about every
other failure mode (disk full, a dictionary the running user cannot write, a language config that routes the write
somewhere our checks never read).

So the add is **verified, not assumed**: after the write, run `check_spelling(word)` in a **fresh process** and require
`status == "correct"`. A fresh process is essential — a same-process `*word\n#\nword\n` returns `*` because `*` mutates
the in-memory dictionary, which proves nothing about whether `#` reached disk. The verify covers every failure mode
uniformly with no error-string guessing, and stderr is then used only to _explain_ a failure the verify already caught.

The cost is one extra subprocess on an explicit, human-paced keypress. Measured: the add is ~6 ms, so add + verify is
~15 ms, off-thread. This is not on any render, keystroke, or startup path.

### Consequence: the squiggle clears after the verify, not before

The sticky-misspelling store elsewhere uses optimistic mutation (mutate in memory, publish, persist off-thread). That is
wrong here. The entire point of the verify is to never claim a write that did not happen, and un-squiggling
optimistically would show exactly that claim for ~15 ms and then take it back on failure. So `d` dismisses the panel,
the worker runs add + verify, and only a verified `added` calls `forget_misspelling(word)`. A 15 ms gap before the
underline clears is imperceptible; a squiggle that flickers off and back on is not.

On failure the word simply stays flagged — which is correct, since it _is_ still misspelled as far as `aspell` is
concerned — and an error notification carries `aspell`'s own explanation.

### Consequence: `forget`, not `allow`

A verified add makes `aspell` itself accept the word, so a future `K` returns `correct` and the existing self-heal path
never re-records it. Adding to `allowed` on top of that would be redundant state. `forget_misspelling` is the exact
right call: drop the squiggle, add no override.

If the word was _already_ in `allowed` (the user pressed `a` earlier, then came back and pressed `d`), leave `allowed`
alone. Removing it would silently destroy the user's SASE-local decision, which is the only thing still protecting them
if they later prune their `.pws` file by hand.

### aspell case and punctuation semantics (measured, and they matter)

```
add "Bugyi"  ->  Bugyi ✓   BUGYI ✓   bugyi ✗
add "teh"    ->  teh   ✓   Teh   ✓   TEH   ✓
add "Bugyi's" -> accepted (apostrophes are fine)
add "well-formedd" -> rejected: '-' may not appear in the middle of a word
```

Two things follow:

1. **Add the word exactly as the user typed it.** A lowercase entry matches every capitalization; a capitalized entry
   matches only capitalized and all-caps forms. That is `aspell`'s deliberate proper-noun handling and preserving the
   user's casing is both the honest and the more useful behavior. Do not lowercase, do not use the `&` (add-lowercase)
   pipe command.
2. **Hyphenated words can never be added.** `extract_lookup_word` accepts interior hyphens, `aspell -a` reports the
   misspelled _sub_word so the panel does open for them, and `d` on such a word will always fail. This is handled by
   surfacing `aspell`'s own message verbatim —
   `The word "well-formedd" is invalid. The character '-' (U+2D) may not appear in the middle of a word.` — which is
   clearer than anything this codebase could invent, and does not hardcode a language rule that varies by dictionary.

Also measured: adding a word already in the dictionary is a silent no-op with no duplicate entry, so `d` is idempotent
and needs no pre-check.

### Do not try to report the dictionary file path

`aspell --lang=en_US dump config personal` prints `.aspell.en_US.pws`, but the file `aspell -a --lang=en_US` actually
creates and reads is `.aspell.en.pws`. `dump config` disagrees with runtime behavior, so resolving the path that way
would print a filename that does not exist. The notification says "your aspell personal dictionary" and the docs give
`~/.aspell.en.pws` as the usual location with an explicit hedge that `aspell` config can move it.

### The footer is broken today, and this fixes it

The shipped footer is one string, `1-9 apply | j/k move | a accept | Enter apply | Esc cancel` — 58 characters in a
50-column content box (container `width: 56`, minus a 1-cell double border each side, minus `padding: 1 2`). It wraps
mid-list and leaves a dangling trailing `|`, which is visible in
`tests/ace/tui/visual/snapshots/png/spellcheck_panel_modal_120x40.png`. Adding a sixth hint to that string makes it
worse.

Replace the accidental wrap with a deliberate two-line footer, grouped by intent — line one fixes the word, line two
leaves it alone:

```
1-9 apply | j/k move | Enter apply          (34 cols)
a accept | d add to aspell | Esc cancel     (39 cols)
```

The no-suggestions variant collapses to the second line only:

```
a accept | d add to aspell | Esc cancel      (39 cols)
```

This also drops the misleading `j/k move` hint from the no-suggestions footer, where `_select_relative` returns early
and the keys do nothing.

One `Static` containing a `\n` renders this; `height: auto` and `text-align: center` already handle it and no new CSS
selector is needed.

**Raise `max-height` from 18 to 20** on `SpellcheckPanelModal > Container`. A full nine-suggestion panel needs
`2 (border) + 2 (padding) + 1 (title) + 1 (title margin) + 9 (rows) + 5 (footer: margin 1 + border-top 1 + padding 1 + 2 text lines) = 20`.
At 18 the footer clips. This is a latent bug today — the wrapped footer is already two lines — that the deliberate
two-line footer would otherwise make deterministic. A nine-suggestion visual snapshot proves the fix.

### Why `d`

`d` for "dictionary" is free in the modal's binding table (`escape`, `q`, `enter`, `j`, `k`, `ctrl+n`, `ctrl+p`, `a`,
and digits are taken) and has no destructive reading in a panel with no editable content. The footer label avoids the
bare word `dict` on purpose: in this codebase `dict` is the DICT-protocol tool `K` uses for _definitions_, so `d dict`
would collide with an existing concept. `add to aspell` names the actual thing being written.

### Where `d` is reachable

Only from `SpellcheckPanelModal`, and the panel only opens after a `check_spelling` call returned `misspelled` — which
means `aspell` existed moments ago. `unavailable` is still handled (the binary can vanish between the two calls) but is
not the common path. No new entry point is added anywhere else.

### Rust core boundary

Stays in Python, for the same reason `sticky_misspellings.md` gave: `src/sase/core/word_lookup.py` already owns every
`aspell`/`dict` subprocess in this repo, and this is one more invocation of a tool this module already drives. Nothing
here is domain state another frontend would need to mirror — it writes to a file `aspell` owns, not to SASE's model.

## Implementation

### 1. `src/sase/core/word_lookup.py` — the aspell write

Extract the shared invocation so the add and the check cannot drift:

```python
_ASPELL_PIPE_ARGS = ("aspell", "-a", "--encoding=utf-8", "--lang=en_US")
```

`check_spelling` passes `list(_ASPELL_PIPE_ARGS)` so its call shape is byte-identical to today's. The add must use the
same flags, because the language selection is what decides which `.pws` file the write lands in and the check reads.

Add:

```python
_AddToDictionaryStatus = Literal["added", "unavailable", "error"]


@dataclass(frozen=True)
class AddToDictionaryResult:
    """The parsed outcome of one aspell personal-dictionary add."""

    status: _AddToDictionaryStatus
    detail: str = ""


def add_to_personal_dictionary(
    word: str,
    *,
    runner: _Runner = subprocess.run,
) -> AddToDictionaryResult:
    """Add *word* to the user's aspell personal dictionary, verifying it stuck."""
```

Sequence:

1. `shutil.which("aspell") is None` → `unavailable`.
2. Reject a word that is empty, whitespace-only, or contains a newline or carriage return, without spawning anything —
   `\n` would be read as a second pipe-protocol command. Detail: `word is not addable`. Unreachable through the panel
   (`extract_lookup_word` cannot produce such a word), but this is a public function and the failure mode is silent
   corruption of the user's dictionary.
3. Run
   `runner(list(_ASPELL_PIPE_ARGS), input=f"*{word}\n#\n", capture_output=True, text=True, timeout=_ASPELL_TIMEOUT_SECONDS, check=False)`.
   `*` adds to the personal dictionary; `#` flushes it to disk. `subprocess.TimeoutExpired` and `OSError` map to
   `error`, mirroring `check_spelling`.
4. `completed.returncode != 0` → `error` with `_command_error_detail("aspell", completed)`. Defensive only, per the
   measurements above, but a genuinely broken invocation would exit nonzero.
5. Capture `detail = _first_aspell_error_line(completed.stderr)` for use in step 6.
6. Verify: `check_spelling(word, runner=runner)`. `correct` → `AddToDictionaryResult(status="added")`. Anything else →
   `error` with `detail` if stderr explained it, otherwise the verify's own `detail`, otherwise
   `aspell did not accept the word`. Threading the same injected `runner` through both calls is what lets one fake drive
   the whole sequence in tests.

New helper beside `_first_nonempty_line`:

```python
_ASPELL_ERROR_PREFIX = "Error: "


def _first_aspell_error_line(stderr: str) -> str:
    """Return aspell's first ``Error:`` explanation, or its first stderr line."""
```

Strip the `Error: ` prefix (the notification already supplies the "could not add" framing) and normalize through
`_one_line`. Fall back to `_first_nonempty_line(stderr)`.

Export `AddToDictionaryResult` and `add_to_personal_dictionary` from `__all__`. Keep `_AddToDictionaryStatus` private,
matching `_SpellCheckStatus` and `_DefinitionStatus`.

### 2. `src/sase/ace/tui/modals/spellcheck_panel_modal.py` — the `d` action

- Widen the choice: `action: Literal["apply", "accept", "dictionary"]`.
- Add `("d", "add_to_dictionary", "Dictionary")` to `BINDINGS`.
- Add an `elif event.key == "d":` arm to `on_key`, before the printable catch-all, calling `_stop_key` then
  `action_add_to_dictionary`.
- `action_add_to_dictionary` → `self.dismiss(SpellcheckChoice(action="dictionary"))`. Unlike `action_apply_selected` it
  has no `if not self._suggestions: return` guard — accepting a word `aspell` has no ideas about is the case that needs
  this most.
- `_build_footer` returns the two-line string above, or the single second line when `self._suggestions` is empty.
- Update the class docstring to describe both escape hatches and their scopes.

### 3. `src/sase/ace/tui/styles.tcss`

`SpellcheckPanelModal > Container`: `max-height: 18` → `max-height: 20`. No other rule changes; the two-line footer
needs no new selector.

### 4. `src/sase/ace/tui/widgets/_prompt_word_lookup.py` — wiring

Import `add_to_personal_dictionary` and `AddToDictionaryResult` at module top, beside `check_spelling`, so tests can
monkeypatch `sase.ace.tui.widgets._prompt_word_lookup.add_to_personal_dictionary` the way they already patch
`check_spelling`.

In `_apply_spelling_suggestion`, add a branch before the apply path:

```python
if choice.action == "dictionary":
    self._add_word_to_personal_dictionary(span.word)
    return
```

New methods:

```python
def _add_word_to_personal_dictionary(self, word: str) -> None:
    """Schedule the off-thread aspell dictionary add for *word*."""
    self.run_worker(
        self._add_to_personal_dictionary_async(word),
        name="prompt-dictionary-add",
    )


async def _add_to_personal_dictionary_async(self, word: str) -> None:
    result = await asyncio.to_thread(add_to_personal_dictionary, word)
    if not self.is_mounted:
        return
    self._notify_dictionary_add_result(word, result)
```

`_notify_dictionary_add_result`:

- `added` → `self._forget_prompt_misspelling(word)` then
  `self.notify(f"Added '{word}' to your aspell personal dictionary")`.
- `unavailable` → `self.notify(f"aspell is no longer available; '{word}' was not added", severity="warning")`.
- otherwise → `self.notify(f"Could not add '{word}' to aspell: {detail}", severity="error")` where `detail` falls back
  to `"aspell did not accept the word"`.

The worker is **not** exclusive and carries **no** request-id staleness check. Two `d` presses on two different words
must both complete; unlike `_resolve_word_lookup_async` there is no single modal slot they race for. `is_mounted` is the
only guard needed, and it is what keeps `notify` off a torn-down widget.

`_forget_prompt_misspelling` already exists and already degrades to a no-op when the app has no provider.

### 5. `docs/ace.md` — "Word definitions & spellcheck"

Extend the paragraph that currently documents `a`. It must say, concretely:

- `d` adds the word to the user's `aspell` personal dictionary (usually `~/.aspell.en.pws`, though `aspell` config can
  relocate it), so it stops being flagged everywhere on the machine, not just in ACE.
- `a` is the SASE-only alternative — reversible from `prompt_misspellings.json`, and it leaves `aspell` untouched.
- The add is verified by re-checking the word afterwards; the squiggle clears only once `aspell` genuinely accepts it,
  and a failure leaves the word flagged with `aspell`'s own explanation.
- Case follows `aspell`: a word added capitalized stays flagged in lowercase.
- Hyphenated words cannot be added — `aspell` does not permit `-` inside a personal-dictionary entry.

No `docs/configuration.md` change: this adds no configuration.

### 6. Deliberately unchanged

- **No new config key.** `d` is an explicit, per-word, per-keypress action whose own label names the tool it writes to,
  and the write is reversible by editing one line of a text file. A `dictionary_writes: true|false` knob would add
  schema, parsing, docs, and tests to gate something the user must already ask for by name.
- **No help-modal entry.** `binding_common.py` documents the global `K` keymap; modal-local keys (`a`, `1`–`9`, `j`/`k`)
  have never been listed there, and the panel footer is their documentation. Adding `d` alone would be inconsistent.
- **No onboarding copy change.** `agent_onboarding.py:254` already carries one dim line about sticky squiggles; a second
  would grow the card for a key only reachable from a panel that shows it in its own footer.
- **No `src/sase/ace/testing/editors.py` change.** `_PromptTestApp` already implements `forget_misspelling`, and the
  aspell call is monkeypatched at the `_prompt_word_lookup` import site.

## Tests

| File                                                                    | Coverage                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tests/test_word_lookup.py` (extend)                                    | The add call gets `list(_ASPELL_PIPE_ARGS)` and stdin exactly `*<word>\n#\n`; `check_spelling` still gets the identical arg list. `added` when the verify call returns `correct`. `error` carrying aspell's stderr `Error:` text (prefix stripped, whitespace normalized) when the verify still returns `misspelled` — the hyphen case. `error` with a generic detail when stderr is silent and the verify fails. `unavailable` when `shutil.which` returns `None`, with no subprocess spawned. `TimeoutExpired` and `OSError` map to `error`. A nonzero exit maps to `error` without running the verify. Empty, whitespace-only, and newline-bearing words are rejected without spawning. Casing is passed through verbatim (`*Bugyi`, never `*bugyi`). |
| `tests/test_word_lookup.py` (new, `skipif` no `aspell` on PATH)         | End-to-end against the real binary with `monkeypatch.setenv("HOME", str(tmp_path))`: `add_to_personal_dictionary("zzqqxword")` returns `added`, the created `.pws` file contains `zzqqxword`, and a following `check_spelling("zzqqxword")` returns `correct`. Adding the same word twice stays `added` and writes no duplicate. `add_to_personal_dictionary("well-formedd")` returns `error` with a detail mentioning the hyphen. This is the only test that proves the pipe-protocol string is actually right.                                                                                                                                                                                                                                         |
| `tests/ace/tui/widgets/test_prompt_normal_mode_word_lookup.py` (extend) | `d` in the panel dismisses with `action="dictionary"` and spawns no apply/replace. A stubbed `added` result clears the word from `misspelled_words()` and notifies `Added '<word>' to your aspell personal dictionary`. A stubbed `error` result leaves the word in `misspelled_words()`, leaves the prompt text untouched, and notifies at `error` severity with the detail included. A stubbed `unavailable` result notifies at `warning` severity and leaves the word flagged. `d` works on the no-suggestions panel. The prompt text is never modified by any `d` outcome.                                                                                                                                                                           |
| `tests/ace/tui/modals/` (new or extend)                                 | `_build_footer` returns two lines with suggestions and one line without; every advertised key in each variant has a live binding, and `j/k move` does not appear in the no-suggestions footer; both lines fit the 50-column content box.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `tests/ace/tui/visual/test_ace_png_snapshots_word_lookup.py`            | Regenerate `spellcheck_panel_modal_120x40` for the new footer (the dangling `                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | `must be gone). Add`spellcheck_panel_modal_full_120x40`with nine suggestions, proving the`max-height: 20`bump renders the whole footer. Add`spellcheck_panel_modal_no_suggestions_120x40`, proving the single-line footer variant. |

## Verification

```bash
just install
just fmt
just check
just test
just test-visual   # accept the two new goldens and the one regenerated footer golden
                   # with --sase-update-visual-snapshots
```

Manual pass in `sase ace`, with `~/.aspell.en.pws` backed up first:

1. Type a proper noun `aspell` rejects, press `K`, confirm the panel opens and the footer reads
   `a accept | d add to aspell | Esc cancel` on its own line.
2. Press `d`. Confirm the success notification names the aspell personal dictionary and the red underline clears.
3. Confirm the word now appears in `~/.aspell.en.pws`, and that `K` on it opens the definition panel (or reports no
   definition) rather than the correction panel.
4. Type a misspelled hyphenated word, press `K`, press `d`. Confirm the error notification carries aspell's own
   invalid-character explanation and the underline is still there.
5. Restart ACE and confirm the added word is still unflagged and the hyphenated one is still flagged.

## Risks and rejected alternatives

- **Trusting aspell's exit code.** Rejected — measured to be `0` for both a rejected word and an unwritable dictionary
  file. This is the single most important finding here; an implementation that checks `returncode` would report success
  for every real-world failure.
- **Verifying in the same aspell process (`*word\n#\nword\n`).** Rejected — returns `*` because `*` mutates the
  in-memory dictionary, so it confirms nothing about whether `#` reached disk. Only a fresh process is evidence.
- **Parsing stderr as the success signal instead of verifying.** Rejected — it works for the two failures measured here
  and guesses at every other one. stderr is used for the human-readable _reason_, never for the verdict.
- **Optimistically clearing the squiggle before the verify.** Rejected — it re-introduces exactly the false success the
  verify exists to prevent, and costs a visible flicker when the add fails. The ~15 ms wait is imperceptible.
- **Pre-disabling `d` for hyphenated words.** Rejected — it would hardcode a rule that lives in `aspell`'s per-language
  data file. Letting `aspell` refuse and quoting its own explanation is honest and survives dictionary changes.
- **Reporting the resolved `.pws` path in the notification.** Rejected — `aspell dump config personal` reports
  `.aspell.en_US.pws` while the runtime actually uses `.aspell.en.pws`, so the "helpful" path would be wrong.
- **Adding the word lowercased (the `&` pipe command) so every capitalization is accepted.** Rejected — it silently
  discards the user's casing and would make `bugyi` a correctly spelled word, which is precisely the distinction
  `aspell`'s proper-noun handling exists to preserve.
- **Replacing `a` with `d` instead of adding it.** Rejected — `a` is the right answer for a word the user does not want
  in a machine-global dictionary, and it is the only option that stays inside SASE's reversible state.
- **Also removing the word from SASE's `allowed` list on a successful add.** Rejected — it would destroy the user's
  earlier SASE-local decision, which is the only thing still protecting them if they later prune `.pws` by hand.
- **A config key to disable dictionary writes.** Rejected — the action already requires an explicit keypress on a key
  whose label names the tool it writes to, and it is undone by deleting one line from a text file.
- **Concurrent `.pws` writes from two `sase ace` processes.** Accepted risk: `aspell`'s `#` rewrites the whole file with
  no locking, so two adds within the same few milliseconds could lose one. Both are deliberate, human-paced keypresses
  in separate terminals; SASE's own `fcntl` lock cannot guard a file `aspell` owns, and the losing word simply stays
  flagged and can be re-added.
- **Casefold asymmetry between the two stores.** SASE's misspelling set is casefolded while `aspell`'s dictionary is
  case-aware, so adding `Bugyi` and then calling `forget_misspelling("Bugyi")` also un-squiggles `bugyi`, which `aspell`
  still rejects. Bounded and self-healing: the next `K` on `bugyi` re-records it. Documented rather than engineered
  around, since making the SASE store case-aware would be a much larger change to a design that is casefolded on
  purpose.
