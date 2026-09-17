---
tier: epic
title: Agent screenshots of a real sase TUI (sase screenshot)
goal: "Agents can launch a real `sase tui` locally or on a remote machine, drive it with
  keypresses, and capture a canonical PNG of the live screen; TUI memory is restructured
  under a new tui.md reference note that inlines tui_screenshot.md and tui_perf.md.

  "
phases:
  - id: visual-render-promotion
    title: Promote the canonical rasterizer out of tests/
    depends_on: []
    size: small
    description:
      "visual-render-promotion: move render_svg_to_png and the bundled fonts from
      tests/ace/tui/visual/ into a runtime src/sase module with lazy resvg/pillow
      imports, repoint the visual suite at it, and verify goldens stay byte-identical."
  - id: live-screenshot-export
    title: Externally-triggerable live-app screenshot export
    depends_on: []
    size: medium
    description:
      "live-screenshot-export: add the per-window request-dir protocol and a
      SIGUSR2-triggered, settle-then-export SVG capture to the live TUI, with the tmux
      launcher injecting SASE_TUI_SCREENSHOT_DIR and printing the dir."
  - id: screenshot-cli
    title: sase screenshot local orchestration
    depends_on:
      - visual-render-promotion
      - live-screenshot-export
    size: medium
    description:
      "screenshot-cli: add the top-level command that launches the TUI in tmux with
      fixed geometry, sends keys with regex settle-waits, triggers the in-app SVG
      export, rasterizes to PNG, and cleans up the window unless --keep/--window."
  - id: screenshot-remote
    title: Remote capture via --host
    depends_on:
      - screenshot-cli
    size: medium
    description:
      "screenshot-remote: resolve enrolled machine aliases or raw SSH destinations, run
      the SVG capture leg remotely over SSH with a contract probe and cleanup modeled on
      sudo/ssh.py, rasterize locally, and report the remote sase version."
  - id: memory-inline-embeds
    title: Flat-note inline embedding in memory reads
    depends_on: []
    size: medium
    description:
      "memory-inline-embeds: make sase memory read/show render ![[target]] links of flat
      notes inline with depth caps, cycle guards, and no duplicate reference/children
      listings, updating docs and the test locking old behavior."
  - id: tui-memory-notes
    title: Author the TUI memory notes
    depends_on:
      - screenshot-remote
      - memory-inline-embeds
    size: small
    description:
      "tui-memory-notes: create tui.md and tui_screenshot.md, reparent tui_perf.md under
      tui.md, link tui_screenshot from lint_and_test.md, and regenerate agent
      instructions via sase memory init."
proposed_by: bbugyi200.athena.0m5
create_time: 2026-09-17 08:43:25
status: wip
---

- **PROMPT:**
  [prompts/202609/tui_agent_screenshots.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/tui_agent_screenshots.md)

# Agent Screenshots Of A Real `sase tui` (`sase screenshot`)

## Goal

Give sase agents one command that spins up a **real** `sase tui` instance on the machine
they run on (and, via `--host`, on a remote machine), emulates user keypresses, and
produces a **PNG** of the resulting screen that matches what a user would see — so
agents can verify the visual impact of their TUI changes. Also restructure TUI memory: a
new `sase/memory/tui.md` reference memory that inlines a new
`sase/memory/tui_screenshot.md` note and the existing `sase/memory/tui_perf.md` note.

Grounding: `research:202609/tui_agent_screenshot_automation.md` (read this turn). Its
recommendation — **Option B2** — is adopted: chain the four capture legs that already
exist in this repo instead of building a new stack.

## Architecture (decisions already made — do not relitigate in phases)

1. **Capture pipeline (local).** Launch the real TUI through the existing agent-facing
   tmux machinery (`sase tui --tmux`, `src/sase/main/ace_tmux.py`), drive it with
   `tmux send-keys` synchronized by screen-regex polling of `tmux capture-pane -p`, have
   the **live app export its own SVG** via a new externally-triggerable export
   (generalizing `_capture_screen` in `src/sase/ace/tui/repro/capture.py:458`, which
   already proves `app.export_screenshot()` works in the live production app), then
   rasterize with the **same resvg + bundled-Fira-Code renderer that defines the 674
   golden PNGs** (`tests/ace/tui/visual/png_diff.py`, promoted into `src/`). This gives
   exact grid fidelity and project-canonical pixels (Tier 2 in the research report:
   byte-comparable with the golden visual suite). Literal terminal-emulator pixels
   (Tier 3) remain the VHS demo lane's job and are out of scope.
2. **CLI shape.** A new **top-level** `sase screenshot` command. It is NOT nested under
   `sase tui` because `sase tui` has an optional positional `query`
   (`src/sase/main/parser_ace.py:38`), and mixing an optional positional with argparse
   subparsers is ambiguous (`sase tui screenshot` vs a query literally equal to
   `screenshot`). No top-level `screenshot` command exists today.
3. **Export trigger = SIGUSR2 + request directory, not a key binding.** SIGUSR1 is taken
   (`src/sase/ace/tui/actions/agents/_panel_artifact_pane.py`). A signal avoids
   consuming a key, avoids `default_config.yml` keymap changes, and works when a modal
   or text input has focus. The request/response protocol is a per-window directory
   (deterministic path derived from tmux session + window name, injected as
   `SASE_TUI_SCREENSHOT_DIR` by the tmux launcher) into which the app writes
   sequence-numbered `screen_<N>.svg` + `screen_<N>.done` (or `screen_<N>.error`) files.
   Deterministic derivation lets `sase screenshot --window <target>` reconstruct the
   directory for a previously-launched window without reading the remote process
   environment.
4. **Remote leg = SSH, provider-neutral, gateway deferred.** `--host` accepts either an
   **enrolled machine alias** (resolved through the dispatch config to
   `MachineRecord.effective_ssh_target`, `src/sase/dispatch/models.py:281` — this works
   for ANY dispatch provider, not just the tailnet one) or a **raw SSH destination**
   (`user@host`), so contributors without enrolled machines or with different remote
   providers can use any SSH-reachable box. The remote transport is modeled on the
   shipped `src/sase/sudo/ssh.py` pattern (contract probe → bounded remote command →
   fetch one file back with `cat` → `finally:`-guarded cleanup; see
   `resolve_sudo_target` in `src/sase/sudo/target.py:34`). Only the SVG leg runs
   remotely (remote needs just sase + tmux, no rasterizer/fonts); the **caller**
   rasterizes to PNG. A gateway-native typed operation was evaluated and rejected for
   v1: it needs ~15–20 files across two repos, a fleet contract/scope change, and
   re-enrollment of every machine. The future seam (`ContentHandleKindWire::Artifact` in
   sase-core's fleet contract, declared but never emitted) is recorded below as
   follow-up, not built now.
5. **tmux lifecycle.** The command kills the tmux window it created after capture, by
   default. `--keep` preserves the window (and prints the `sase_tmux_*` target lines) so
   an agent can keep driving the same live instance; `--window` captures an existing
   window. Together these support the iterate loop: screenshot → look → send more keys →
   screenshot again.
6. **Boundary and flags.** Everything here is TUI presentation/automation glue and
   Python CLI orchestration — **no sase-core changes** (per the rust-core boundary
   litmus test). No new feature flag: the command is purely additive, and the remote leg
   rides existing SSH authentication/authorization. No keymap changes.
7. **Memory mechanics.** Flat memory notes do NOT currently support `![[target]]` inline
   embedding — only web strands honor `link.inline` (`_resolve_note_links` docstring,
   `src/sase/memory/selector.py:319`; locked by
   `tests/memory/test_memory_selector.py:434`), even though `docs/memory.md` and the
   `sase_memory_write`/`sase_memory_read` skill sources already describe `![[...]]` as
   "rendered inline in the body". This epic implements flat-note inline embedding
   (making the documented semantics true), because the requested `tui.md` note depends
   on it. Listing suppression in generated agent instructions is controlled purely by
   `parent:` frontmatter (`src/sase/amd/_memory.py:404`; live precedent:
   `sase/memory/sase_sizes.md` with `parent: sase/memory/sase_beads.md`), so
   `tui_perf.md`'s "conversion" is: keep `type: reference`, change `parent: AGENTS.md` →
   `parent: sase/memory/tui.md`. Do NOT use `type: core` on the children (core notes are
   refused by `sase memory read` and would inline into every provider shim) and do NOT
   convert `tui.md` into a memory web (descriptors are always-loaded).

## Intended CLI surface (final flag set is the implementing phase's call, per CLI rules)

```
sase screenshot [-o OUT.png] [-s COLSxROWS] [-p KEY ...] [-w REGEX ...]
                [-d MS] [-S] [-k] [-W TMUX_TARGET] [-H ALIAS_OR_SSH]
                [-t SECONDS] [-- TUI_ARGS...]
```

- `-o/--output` — output path; defaults to a generated path printed on stdout.
- `-s/--size` — pane geometry, default `120x40` (the visual suite's canonical size).
- `-p/--press` — repeatable; tmux `send-keys` key names, sent in order.
- `-w/--wait-for` — repeatable; regexes that must match `tmux capture-pane -p` output
  before capture (the VHS `Wait+Screen` idiom). Complex interleavings of press/wait are
  explicitly NOT a v1 goal: agents iterate with `--keep` + raw `tmux send-keys` +
  `--window` instead.
- `-d/--settle-ms` — extra settle delay before capture.
- `-S/--svg` — stop after the SVG leg (used by the remote side; skips rasterization).
- `-k/--keep` — do not kill the tmux window after capture.
- `-W/--window` — capture an existing `sase_tmux_*` window instead of launching one.
- `-H/--host` — run the capture legs on a remote machine; rasterize locally.
- `-t/--timeout` — overall deadline.
- Trailing `-- TUI_ARGS` are forwarded to `sase tui` (e.g. `-t axe` for the startup
  tab). The command itself always appends the determinism defaults `-x -r 0` and env
  pins (`TEXTUAL_ANIMATIONS=none`, `COLORTERM=truecolor`, `TERM=xterm-256color`);
  golden-suite-only byte-stability pins (TZ/clock/version) are NOT applied — the
  screenshot should show this machine's real state, which is the feature.
- Machine-readable stdout: `key=value` lines (`png=`, `svg=`, `sase_tmux_window=`, …)
  matching the existing `--tmux` print contract (`src/sase/main/ace_tmux.py:232`).
- A hidden `--contract` flag prints a one-line JSON `{"schema_version": 1}` so the
  remote leg can probe "does this sase know `sase screenshot`?" (mirrors
  `sase sudo exec --contract`).

## Non-goals / recorded follow-ups (phase workers: leave as `PROPOSED FOLLOW-UP:` notes, do not build)

- Gateway-native screenshot operation over the fleet API (seam:
  `ContentHandleKindWire::Artifact`).
- Pushing uncommitted working-tree changes to a remote machine before screenshotting
  (deployment problem; remote screenshots show that machine's deployed reality, and the
  command must print the remote `sase --version` so agents don't misattribute diffs).
- `--gif`/VHS animated evidence lane; `tmux capture-pane -e` + `freeze` diagnostic
  recipe (document as a troubleshooting aside only if trivial).
- Windows support (no SIGUSR2).

## Phases

### Phase 1: Promote the canonical rasterizer out of `tests/` {#visual-render-promotion}

- size: small
- depends on: nothing

Move the SVG→PNG rasterization leg from `tests/ace/tui/visual/png_diff.py` into a new
runtime module `src/sase/ace/tui/visual_render.py`: `render_svg_to_png()` plus the
bundled-font machinery (`_bundled_font_files`, `skip_system_fonts=True` semantics) and
the four font files from `tests/ace/tui/visual/fonts/` relocated to a package-data
directory (e.g. `src/sase/ace/tui/fonts/`), including the fonts' README. The build
backend is hatchling with `packages = ["src/sase"]`, which ships package files by
default — verify the fonts land in the wheel (`python -m build` or a packaging test).
`resvg_py`/`pillow` imports must be lazy: importing `visual_render` must not fail when
the `visual` extra is absent; only calling `render_svg_to_png()` raises a clear,
actionable error naming the `visual` extra. Update `tests/ace/tui/visual/png_diff.py`
(and the visual conftest/`renderer_env.json` fingerprint code, which hashes font files)
to import/point at the promoted module and fonts so exactly one canonical rasterizer
exists. Pure refactor: golden PNGs must not change.

Verification: `just check` plus `just test-visual` (byte-identical goldens locally).

### Phase 2: Externally-triggerable live-app screenshot export {#live-screenshot-export}

- size: medium
- depends on: nothing

Read `sase memory read tui_perf.md -r "..."` before starting (event-loop/pump rules).

1. **Request-dir protocol module** (e.g. `src/sase/ace/tui/screenshot_export.py`):
   deterministic per-window directory derived from tmux session + window name under a
   SASE-home-scoped tmp root; helpers to compute the path shared by launcher, app, and
   CLI; sequence-file naming `screen_<N>.svg` / `.done` / `.error` (atomic
   write-then-rename).
2. **In-app trigger:** install a `SIGUSR2` handler when `SASE_TUI_SCREENSHOT_DIR` is set
   (guard `hasattr(signal, "SIGUSR2")`; follow the SIGUSR1 install/restore precedent in
   `src/sase/ace/tui/actions/agents/_panel_artifact_pane.py`, but prefer
   `loop.add_signal_handler` so the handler runs on the UI thread). On signal: settle
   (disable cursor blink for the frame, wait for pending visual work — reuse the
   convergence ideas from `tests/ace/tui/visual/_ace_png_snapshot_waits.py`), call
   `app.export_screenshot(simplify=True)` on the UI thread, write the sequence files.
   Keep the handler thin and pump-safe per `tui_perf.md` rule 2; on failure write
   `.error` with the message.
3. **Launcher injection:** `src/sase/main/ace_tmux.py` computes the request dir for the
   claimed window and injects `SASE_TUI_SCREENSHOT_DIR` alongside the existing
   `SASE_TUI_TRACE`/`SASE_TUI_PERF` env defaults, and prints a
   `sase_screenshot_dir=<path>` line after the existing `sase_tmux_*` lines.

Tests: unit-test the export function and protocol via the `AcePage` harness
(`src/sase/ace/testing/ace_page.py`) driving the handler body directly, plus one
real-PTY smoke test in `tests/ace/tui/terminal_smoke/` that spawns the TUI with the env
var set, sends a real `SIGUSR2`, and asserts a valid SVG + `.done` file appears.

Verification: `just check`.

### Phase 3: `sase screenshot` local orchestration {#screenshot-cli}

- size: medium
- depends on: #visual-render-promotion, #live-screenshot-export

Read `sase memory read cli_rules.md -r "..."` before starting.

1. New `src/sase/main/parser_screenshot.py` + handler module (e.g.
   `src/sase/main/screenshot_handler.py` with the orchestration logic in a testable
   module under `src/sase/` that accepts an injected command runner, like
   `src/sase/sudo/ssh.py` does). Register in `parser_full_registrars.py`. Implement the
   CLI surface above (minus `--host`, which is Phase 4; accept the flag but error "not
   yet supported" only if wiring it early is awkward — otherwise leave the flag entirely
   to Phase 4).
2. **Geometry:** always create the capture window in the dedicated detached
   `sase_ace_agents` session (never the caller's attached session, whose client size
   would win): set the session's `default-size`, create the window via the existing
   claim machinery, then `tmux resize-window -x <cols> -y <rows>` and verify with
   `tmux display -p -t <target> '#{window_width}x#{window_height}'`; clear error on
   mismatch or on tmux < 2.9 (match `_require_tmux_binary`'s error style,
   `src/sase/main/ace_tmux.py:51`). Reuse/refactor `ace_tmux.py` internals rather than
   duplicating window-claiming logic.
3. **Drive and capture:** startup wait (poll `capture-pane` for a non-blank, settled
   frame), send `--press` keys in order, honor `--wait-for` regex polling with the
   overall `--timeout`, optional `--settle-ms`, then `kill -USR2 <pane_pid>` and poll
   for the next `screen_<N>.done`/`.error`. On timeout or `.error`, include the last
   `tmux capture-pane -p` text in the error message (that is the agent's debugging
   view).
4. **Rasterize** via the promoted `render_svg_to_png()` unless `--svg`; actionable error
   naming the `visual` extra when it is missing.
5. **Lifecycle:** kill the created window in a `finally:` unless `--keep`; never kill a
   `--window` target the command did not create. Print the `key=value` stdout contract;
   add the hidden `--contract` probe flag.
6. Docs: agent-facing section next to the `--tmux` docs (`docs/` — wherever
   `sase tui --tmux` is documented today) covering the iterate loop
   (`--keep`/`--window`) and determinism defaults.
7. Tests: orchestration unit tests with an injected runner (no real tmux), plus one
   integration test that runs the full local pipeline when tmux is available (skip
   otherwise); regenerate `tests/completion/snapshots/cli_spec.json`.

Verification: `just check`; run the command manually end-to-end
(`sase screenshot -p j -p j -o /tmp/shot.png`) and eyeball the PNG.

### Phase 4: Remote capture via `--host` {#screenshot-remote}

- size: medium
- depends on: #screenshot-cli

1. **Target resolution:** enrolled alias → `effective_ssh_target` via the dispatch
   config; otherwise validate as a raw SSH destination. Reuse or lightly generalize
   `resolve_sudo_target` (`src/sase/sudo/target.py:34`) and `validate_ssh_target`
   (`src/sase/dispatch/models.py:443`) rather than re-implementing; if generalizing,
   move the shared helper somewhere both sudo and screenshot can import without domain
   cross-talk.
2. **Transport helper** modeled on `src/sase/sudo/ssh.py` (injected `CommandRunner`,
   `shlex.quote`, uuid tmp paths, `finally:` cleanup, uniform ssh error mapping, bounded
   timeouts): probe `ssh <target> -- sase screenshot --contract` (parse JSON, check
   `schema_version`; a non-zero exit or parse failure yields "sase on <host> is missing
   or too old for `sase screenshot`; upgrade it"), then run the remote
   `sase screenshot --svg -o <remote-tmp> <forwarded args>` (forward size/press/
   wait-for/settle/timeout/keep/TUI args; never forward `--host`), fetch the SVG with
   `ssh <target> cat <remote-tmp>`, capture the remote `sase --version`, clean up remote
   temp files.
3. Rasterize locally; print `remote_sase_version=` and `host=` lines alongside the
   normal contract. `--keep` on a remote host prints the remote tmux target plus a
   ready-to-copy `ssh <target> tmux send-keys ...` hint. Short `ConnectTimeout` and a
   clear offline error.
4. Docs: remote section including the **trust-boundary note** — SSH is full-shell access
   and is NOT implied by machine enrollment (gateway credentials are scoped;
   `ssh_target` is a convenience pointer), plus the "remote shows deployed reality, not
   your working tree" caveat, plus what other contributors need (any SSH-reachable host;
   enrollment optional).
5. Tests: transport unit tests with the injected runner covering probe-fail, fetch-fail,
   cleanup-on-error, version reporting; completion snapshot regen if flags changed.

Verification: `just check`; manual smoke against a reachable host if one is available
(document the invocation used).

### Phase 5: Flat-note `![[...]]` inline embedding in memory reads {#memory-inline-embeds}

- size: medium
- depends on: nothing

Make `sase memory read/show` honor inline links for flat notes, matching the semantics
web strands already have and the docs/skills already describe:

1. In `src/sase/memory/selector.py`, extend `_resolve_note_links` (and the flat-note
   render path) so a `![[target]]` link — or any link when the source note declares
   `link_rendering: inline` — renders the target note's body inline at the bottom of the
   read, honoring `-d/--depth` (depth 0 = list as reference, matching web-closure
   semantics) with a cycle guard across the whole read batch.
2. Keep `_always_reference_target` (`selector.py:363`) authoritative: `type: core` notes
   and web descriptors are never inlined — they stay reference rows with the existing
   "already in your context" label. Web strand targets of a flat note's inline link:
   follow whatever the existing strand-embedding renderer produces.
3. No duplicate listings: a target rendered inline must not also appear under
   `## Linked References`, and a parent note's `## Children` section must not re-list
   children whose bodies were just inlined in the same read
   (`render_long_memory_entries` / children rendering in
   `src/sase/memory/notes.py:614`).
4. Reachability validation (`src/sase/memory/inventory_reachability.py`) already accepts
   parent-edge reachability, so no change should be needed there — but add a regression
   test proving a child note reachable only via `parent:` + `![[...]]` produces no
   `sase memory init`/doctor blockers or warnings.
5. Update `docs/memory.md`'s inline-link wording so flat-note behavior is stated
   accurately, and flip/adjust the test that locked the old behavior
   (`tests/memory/test_memory_selector.py:434`). Check whether the
   `sase_memory_read`/`sase_memory_write` skill sources in `src/sase/xprompts/skills/`
   need wording changes; they currently describe inline rendering unconditionally, which
   this phase makes true — if no text change is needed, do NOT redeploy skills.

Verification: `just check`.

### Phase 6: Author the TUI memory notes {#tui-memory-notes}

- size: small
- depends on: #screenshot-remote, #memory-inline-embeds

This phase edits SASE memory; the plan's approval is the authorization, and workers must
still route the edits through their `/sase_memory_write` skill.

1. **Create `sase/memory/tui.md`** — frontmatter `type: reference`, `parent: AGENTS.md`,
   `description:` one line covering the whole TUI domain (something like: "TUI domain
   hub - screenshot automation for seeing your changes (`sase screenshot`) and
   performance gotchas; both render inline when this note is read."). Body: minimal by
   design — two or three orientation sentences (the TUI is Textual-based; read this note
   before TUI work; the two inlined notes below carry the substance) followed by:

   ```
   ![[tui_screenshot]]

   ![[tui_perf]]
   ```

2. **Create `sase/memory/tui_screenshot.md`** — frontmatter `type: reference`,
   `parent: sase/memory/tui.md`, one-line `description:`. This note lives with agents
   for a long time; keep it a durable workflow contract, not a flag dump (`-h` owns the
   exhaustive flags). Required content, roughly in order:
   - **When to use it:** after changing TUI code, to SEE the change in a real running
     `sase tui` against real `$SASE_HOME` state; complements (never replaces) the golden
     PNG suite — goldens gate regressions (`just test-visual`, see
     [[lint_and_test.md]]), screenshots answer "what does my change actually look like".
     Pixels are canonical (same resvg + bundled Fira Code renderer as the goldens), so
     they are directly comparable to the golden corpus.
   - **Core invocation examples:** a bare capture; keys + wait
     (`sase screenshot -p j -p j -w 'REGEX' -o out.png`); startup-tab forwarding via
     `-- -t axe`.
   - **The iterate loop:** `--keep` to leave the TUI running, raw `tmux send-keys` to
     keep driving it, `--window` to re-capture — and that the window is otherwise killed
     automatically, so no manual tmux cleanup is needed.
   - **Remote captures:** `--host <enrolled-alias-or-ssh-destination>`; remote shows
     that machine's installed sase + state (check the printed `remote_sase_version=`),
     not your working tree; remote needs only sase + tmux + SSH access.
   - **Determinism notes:** the command pins animations/truecolor itself; screenshots
     include live timestamps/state by design.
   - **Troubleshooting:** `visual` extra missing (install hint), tmux missing/too old,
     remote sase too old, and "the capture timed out — the error message includes the
     final `capture-pane` text; read it".
3. **Reparent `sase/memory/tui_perf.md`:** keep `type: reference` and the body
   unchanged; change `parent: AGENTS.md` → `parent: sase/memory/tui.md`.
4. **Link from `sase/memory/lint_and_test.md`:** in the "PNG Snapshot Tests" section,
   add one sentence pointing at [[tui_screenshot]] for seeing changes in a real TUI
   (decision already made: yes, add it).
5. Sweep other memory/doc references to `tui_perf.md` (e.g. reference lists that tell
   agents to read it) and retarget them to `tui.md` where they mean "the TUI domain".
6. Run `sase memory init`; verify: AGENTS.md/CLAUDE.md list `tui.md` exactly once and
   neither child; `sase memory read tui.md -r test` renders both child bodies inline;
   `sase memory read tui_perf.md -r test` still works directly; `sase doctor` reports no
   new memory warnings.

Verification: `just check` (memory init touches generated files tracked by git).

## Rollout / ordering notes

- Phases 1, 2, and 5 are independent and can run in parallel.
- Phase 3 needs 1 + 2; Phase 4 needs 3; Phase 6 needs 4 + 5 (its content documents the
  final command surface, including `--host`).
- Every phase: `just fix` before verify; `just check` inline or via monitor per
  `lint_and_test.md`; phases touching the visual pipeline (1, and 3 if it changes
  anything under `tests/ace/tui/visual/`) also run `just test-visual`.
