---
tier: tale
title: Decode tmux CSI-u key encodings inside bracketed pastes
goal:
  Pasting multi-line text into ACE or the pager inside tmux (with modifyOtherKeys mode 2
  requested) inserts the pasted text intact, with real newlines and no `^[106;5u`
  garbage, while Ctrl+Shift chords keep working.
size: medium
proposed_by: bbugyi200.athena.0wg
create_time: 2026-10-04 13:56:26
status: wip
---

# Plan: Decode tmux CSI-u key encodings inside bracketed pastes

## Problem

Pasting multi-line text (Cmd+V on the Mac client, over SSH into tmux on athena) into the
ACE prompt input inserts garbage like `^[106;5u│ ` where every newline should be. The
`│ ` is just pasted content (text copied from a boxed TUI); the `^[106;5u` is a newline
that went through tmux and Textual and came out wrong.

## Root cause (diagnosed and reproduced)

This is a regression from commit `d8efa2a6e5` (2026-10-03, "land three-pane splits").
The failure takes three steps:

1. **SASE asks tmux for modifyOtherKeys mode 2.** `src/sase/tmux_driver.py` added
   `TmuxModifyOtherKeysDriver`. When `TMUX` is set, its `start_application_mode()`
   writes `\x1b[>4;2m` so that Ctrl+Shift chords reach SASE. Both ACE
   (`AceApp.get_driver_class` in `src/sase/ace/tui/app.py`) and the pager
   (`src/sase/pager/app.py`) select this driver through `maybe_tmux_driver_class`.
2. **tmux re-encodes pasted control characters as extended keys.** On tmux 3.5a with
   `extended-keys on` + `extended-keys-format csi-u` (the user's live config), once a
   pane requests mode 2, tmux re-encodes every key it parses from the outer terminal.
   That includes bytes inside a bracketed paste. tmux reads a pasted LF as `C-j` and
   forwards it inside the paste as `\x1b[106;5u`. I reproduced this with an isolated
   tmux server: a pty-attached client pasted the same bytes into a raw-mode reader that
   had enabled bracketed paste.
   - without mode 2: `\x1b[200~line1\n│ line2\r\n\tline3\x1b[201~`
   - with mode 2: `\x1b[200~line1\x1b[106;5u│ line2\r\x1b[106;5u\tline3\x1b[201~`
   - Further probe (paste ``Ab Z!@#~`{}|\^_\x01x\x08y\x7fz\x1bq\x1b\x00w\x0bv\x0cu\n``):
     - Every C0 control except HT and CR becomes a ctrl chord: `\x01`→`\x1b[97;5u`,
       `\x08`→`\x1b[104;5u`, `\x0b`→`\x1b[107;5u`, `\x0c`→`\x1b[108;5u`,
       `\n`→`\x1b[106;5u`.
     - ESC+`q` becomes an alt chord `\x1b[113;3u`, and ESC+NUL becomes `\x1b[32;7u`.
     - HT, CR, DEL, printable ASCII (including uppercase) and non-ASCII (`│`) pass
       through raw.
     - With `extended-keys-format xterm`, the same keys arrive as `\x1b[27;<mod>;<cp>~`,
       e.g. `\x1b[27;5;106~`.
3. **Textual's parser corrupts any ESC inside a bracketed paste.** In Textual 8.0.1
   (pinned `textual==8.0.1`), `XTermParser.parse()` (`textual/_xterm_parser.py`) appends
   only the ESC to the paste buffer when it meets one inside a paste. It then reads
   ahead into a `sequence` until the next ESC or 32 chars, and re-issues that chunk as
   individual `Key` events via `reissue_sequence_as_keys(..., process_alt=False)`. That
   call maps ESC to `circumflex_accent` (`^`). The keys are emitted **before** the
   `Paste` event, and the final `Paste` text is truncated and can contain stray `\x1b`.
   Feeding the tmux bytes above to a bare `XTermParser` yields key events
   `^ [ 1 0 6 ; 5 u │ ␠ l i n e 2 …` followed by `Paste('line1')`. That is exactly the
   visible `^[106;5u│ `.

The bug is not specific to the prompt widget. It hits every Textual input (prompt
TextArea, `Input`s, modals, pager) whenever SASE runs inside tmux with extended keys
configured as documented.

## Rejected alternatives

- **Drop modifyOtherKeys mode 2, or tell users to set `extended-keys off`.** Either way
  loses the Ctrl+Shift+F/B/D chords that `d8efa2a6e5` deliberately enabled. Mode 1 does
  not help: tmux mode 1 sends Ctrl+letter as the legacy C0 byte, so Ctrl+Shift chords
  collapse back into Ctrl chords.
- **Fix it in the prompt widget's `_on_paste`.** Too late: the parser has already split
  the paste into out-of-order keystrokes and truncated the `Paste` text.
- **Globally monkeypatch `textual.drivers.linux_driver.XTermParser`.** That is a
  process-wide mutation of a third-party module, which is worse than one scoped override
  in SASE's own driver subclass.

## Fix

Normalize bracketed-paste bodies in the driver's input path before Textual's parser sees
them, so the parser only ever receives plain text between `\x1b[200~` and `\x1b[201~`.

### 1. New pure module `src/sase/bracketed_paste.py` (stdlib only, no Textual import)

- `normalize_paste_body(body: str) -> str` turns a re-encoded paste body back into the
  text that was pasted:
  1. **Decode extended-key encodings.** Handle both CSI-u
     `\x1b[<cp>[:<alt codes>][;<mod>[:<event>]]u` and xterm modifyOtherKeys
     `\x1b[27;<mod>;<cp>~`. Let `bits = mod - 1` (shift=1, alt=2, ctrl=4; a missing
     `mod` means 1). Decode as follows:
     - Any bit beyond shift|alt|ctrl, or `cp > 0x10FFFF`: drop the sequence. Such
       sequences cannot come from pasted text.
     - ctrl with `cp` in `0x40..0x5F` or `a..z`: `chr(cp & 0x1F)`, so `106;5` → `\n`.
     - ctrl+space, ctrl+`2` or ctrl+`@`: `""` (NUL).
     - ctrl+`?`: `\x7f`.
     - Otherwise: `chr(cp)`. The alt bit is ignored, which means the ESC prefix tmux
       folded into "alt" is discarded, consistent with step 2.
  2. **Strip every remaining escape.** Remove any other complete CSI sequence
     (`\x1b\[[0-?]*[ -/]*[@-~]`, e.g. pasted ANSI colours), then any leftover lone
     `\x1b` and any NUL. No ESC may reach Textual's paste path, because every ESC there
     corrupts the paste.
  3. **Leave everything else untouched**: CR, HT, DEL, printable text and Unicode.
- `BracketedPasteNormalizer` is a streaming pre-filter. `feed(data: str) -> str` returns
  the text to forward to Textual:
  - **Outside a paste:** forward input immediately and never hold anything back. A bare
    ESC keypress must reach Textual's own `ESCAPE_DELAY` timeout logic untouched. To
    detect a `\x1b[200~` that is split across reads, keep the last
    `len("\x1b[200~") - 1` forwarded chars as a lookbehind tail and search
    `tail + data`.
  - **On the start marker:** forward everything up to and including the marker, so
    Textual enters its own paste mode. Then buffer the body internally.
  - **Inside a paste:** append to the buffer and search it for `\x1b[201~`. Because the
    buffer accumulates, a split end marker is handled for free. When the marker is
    found, forward `normalize_paste_body(body) + "\x1b[201~"`. Then keep scanning the
    remainder as outside-paste data, so several pastes or keys in one read work.

Reference sketch (validated during diagnosis against the captured tmux bytes, every
2-way split, and 1-char chunking):

```python
START, END = "\x1b[200~", "\x1b[201~"
_KEY = re.compile(r"\x1b\[(?:27;(\d+);(\d+)~|(\d+)(?::\d*)*(?:;(\d+)(?::\d+)?)?u)")
_CSI = re.compile(r"\x1b\[[0-?]*[ -/]*[@-~]")

def feed(self, data: str) -> str:
    out = []
    while data:
        if not self._in_paste:
            scan = self._tail + data
            i = scan.find(START)
            if i < 0:
                out.append(data)
                self._tail = scan[-(len(START) - 1):]
                break
            cut = i + len(START) - len(self._tail)
            out.append(data[:cut])
            data, self._in_paste, self._tail, self._buf = data[cut:], True, "", ""
        else:
            self._buf += data
            data = ""
            j = self._buf.find(END)
            if j < 0:
                break
            out.append(normalize_paste_body(self._buf[:j]) + END)
            data, self._buf, self._in_paste = self._buf[j + len(END):], "", False
    return "".join(out)
```

### 2. `src/sase/tmux_driver.py`

- Add `PasteSafeXTermParser(XTermParser)`. In `__init__`, call `super().__init__(debug)`
  and create a `BracketedPasteNormalizer`. Override `feed(data)` as follows:
  - If `data == ""`, delegate to `super().feed(data)`; that is EOF semantics.
  - Otherwise compute `forwarded = normalizer.feed(data)` and
    `yield from super().feed(forwarded)` **only when `forwarded` is non-empty**.
    Textual's `Parser.feed("")` means EOF, so a chunk that is entirely held back must
    never reach it.
- Override `TmuxModifyOtherKeysDriver.run_input_thread()` with a verbatim copy of
  Textual 8.0.1 `LinuxDriver.run_input_thread`, changing only the parser to
  `parser = PasteSafeXTermParser(self._debug)`. Its imports (`selectors`,
  `codecs.getincrementaldecoder`, `textual._loop.loop_last`,
  `textual._parser.ParseError`, `textual._xterm_parser.XTermParser`) are already loaded
  by `textual.drivers.linux_driver`. Keep them inside the existing `try`/`LinuxDriver`
  guard so the pager cold-path import diet described in the module docstring is
  unchanged.
- Use the paste-safe parser for this driver **unconditionally**, not gated on `TMUX`. It
  only changes paste bodies that contain ESC, and Textual corrupts those anyway. Keep
  the modifyOtherKeys on/off writes gated on `TMUX` exactly as now.
- Update the module and class docstrings: non-tmux sessions now also get the paste-safe
  parser. Add `PasteSafeXTermParser` to `__all__`. Keep the class name
  `TmuxModifyOtherKeysDriver` to avoid churn.

### 3. Tests (no fixed sleeps: the repo's test-waits lint rejects them)

New `tests/test_bracketed_paste.py`:

- **Captured tmux fixtures.** Use the byte strings from the root-cause section as
  literals.
  - The csi-u body `line1\x1b[106;5u│ line2\r\x1b[106;5u\tline3` normalizes to
    `line1\n│ line2\r\n\tline3`.
  - The csi-u C0 probe body
    ``Ab Z!@#~`{}|\\^_\x1b[97;5ux\x1b[104;5uy\x7fz\x1b[113;3u\x1b[32;7uw\x1b[107;5uv\x1b[108;5uu\x1b[106;5u``
    normalizes to ``Ab Z!@#~`{}|\\^_\x01x\x08y\x7fzqw\x0bv\x0cu\n``.
  - The xterm-format equivalents (`\x1b[27;5;106~`, …) normalize to the same output.
- ANSI colour sequences inside a paste are stripped, a lone ESC is dropped, and
  super/hyper-modified sequences are dropped.
- **Streaming.** Build a stream such as
  `"ab\x1b" + paste + "\x1b[102;6u" + paste + "z"`. For every 2-way split and for 1-char
  chunking, the concatenated forwarded output equals the one-shot output, and the
  one-shot output equals the plain (pre-regression) bytes.
- **Never hold back outside a paste.** `feed("\x1b")` returns `"\x1b"`, and
  `feed("x\x1b[20")` returns its input unchanged.

Extend `tests/test_tmux_driver.py`:

- **`PasteSafeXTermParser` with the reproduction bytes.** Feeding the stream yields
  exactly one `events.Paste` whose text is `line1\n│ line2\r\n\tline3`, and no
  `events.Key`.
- **Event order is preserved.** `"a" + paste + "\x1b[102;6u"` yields `Key a`, then
  `Paste`, then `Key ctrl+shift+f`. Feeding the paste one char at a time gives the same
  single `Paste`.
- **Driver integration, deterministic and threadless.** Build the driver with
  `TmuxModifyOtherKeysDriver.__new__`. Set `fileno` to the read end of an `os.pipe()`
  that already holds the reproduction bytes, `exit_event` to an already-set
  `threading.Event()`, and `_debug = False`. Monkeypatch `process_message` to collect
  events, then call `driver.run_input_thread()` synchronously. With `exit_event` set,
  the loop body is skipped and the final `select(0.1)` pass reads and parses the pending
  bytes. Assert one `Paste` with the expected text and no `Key` events. Close both pipe
  ends.
- **Upstream drift guard.** Assert that
  `sha256(inspect.getsource(LinuxDriver.run_input_thread))` equals the value computed
  for Textual 8.0.1. The failure message should say the override in `sase.tmux_driver`
  must be re-synced with upstream before bumping Textual.

### 4. Docs

In `docs/ace.md` (the "The `Ctrl+Shift` chords need the kitty → tmux CSI-u chain"
paragraph, about line 5802) and `docs/pager.md` (same paragraph, about line 281), add a
sentence. It should say that in mode 2, tmux re-encodes pasted control characters
(newlines arrive as `CSI 106;5u`), and that SASE's input driver decodes them inside
bracketed pastes so pasted text arrives intact. `CHANGELOG.md` is release-generated, so
do not edit it.

## Verification

1. Run `just fmt`, then `just check`, as the `lint_and_test` memory requires.
   `bracketed_paste.py` and `tmux_driver.py` must stay well under the `toobig` limit.
2. Re-run the Textual-level reproduction: feed the tmux mode-2 bytes through
   `PasteSafeXTermParser` and confirm a single clean `Paste` (the new tests cover this).
3. Optional manual check inside tmux: launch `sase ace`, copy several lines of text
   (including box-drawing characters), and Cmd+V into the prompt. Lines should arrive
   with real newlines and no `^[106;5u`. Ctrl+Shift+F/B/D must still work on the Agents
   deck.
