---
tier: epic
title: "`:` Command Line: run sase commands from the TUI"
goal: "`;` opens the Command Palette and `:` opens a new bottom-anchored Command Line
  panel. In the panel, users type `sase` commands with better completion than any shell
  offers: selection-aware, fuzzy, documented and grammar-checked. Output streams inline.
  Every command runs as a durable, tagged proc that keeps running after the panel is
  hidden, shows up in Admin Center → Procs, and returns to the transcript when the panel
  reopens.

  "
phases:
  - id: color-contract
    title: Output color contract
    depends_on: []
    size: medium
    description:
      "color-contract: add one shared color resolver (NO_COLOR, then
      FORCE_COLOR/CLICOLOR_FORCE, then isatty). Adopt it in the bead renderers and every
      other stdout color branch, so a piped `sase` child renders in color the way a
      terminal does."
  - id: spec-contract
    title: Command Line spec contract
    depends_on: []
    size: medium
    description:
      "spec-contract: extend the completion spec with
      required/metavar/default/value_hint and per-command run policy, writes and
      confirms data. Add `sase completion spec -d/--descriptions` and an identity-keyed
      on-disk spec cache that is built in a subprocess."
  - id: kind-coverage
    title: Value-kind coverage and ratchet
    depends_on:
      - spec-contract
    size: medium
    description:
      "kind-coverage: add GATE, TOOL_RUN and TASK_TYPE value kinds with providers,
      annotate the unkinded entity slots, and add a coverage ratchet test. The test
      requires every non-hidden value slot to be kinded, to have choices, or to carry a
      free-form value_hint."
  - id: proc-retention
    title: Command-line proc tag and retention bucket
    depends_on: []
    size: small
    description:
      "proc-retention: in sase-core, give finished procs tagged `command-line` their own
      retention bucket of 50 so they never evict operational proc history. Export the
      tag and limit through the binding and move sase's sase-core revision pin."
  - id: line-resolver
    title: sase-core CommandLineGrammar resolver
    depends_on:
      - spec-contract
      - proc-retention
    size: large
    description:
      "line-resolver: build a frozen `CommandLineGrammar` handle in sase-core that
      parses the spec JSON once. Per keystroke it returns tokens, slot, diagnostics,
      signature, run policy and fuzzy-ranked candidates. Add the Python adapter in sase
      and move the revision pin."
  - id: proc-plumbing
    title: Command-line proc plumbing
    depends_on:
      - proc-retention
    size: medium
    description:
      "proc-plumbing: add `submit_command_line_proc` (a tagged ordinary proc with the
      output env contract and no operation), an observer exit watch, ref-counted tails,
      an offset-based `ProcLogCursor`, `ObservedProc.tags` with a Procs `tag:` query
      field, and an Admin Center proc focus target."
  - id: panel-shell
    title: Command Line panel shell (beta flag)
    depends_on:
      - proc-plumbing
    size: medium
    description:
      "panel-shell: create the `ace_command_line` beta flag. Add `open_command_line`,
      the bottom-anchored `CommandLineScreen` and the app-held `CommandLineSession`,
      plus the working-context chip, a locked history store with ghost text, submitting
      to procs, and streaming transcript blocks. Add PNG goldens for the base states."
  - id: transcript-blocks
    title: Transcript block interactions and lifecycle
    depends_on:
      - panel-shell
    size: medium
    description:
      "transcript-blocks: add NORMAL-mode block navigation with expand, pager, kill,
      rerun, edit, copy, open-in-Procs and remove. Add completion toasts while hidden,
      unseen dots, restore after a TUI restart, pruned-record handling, and `⏎` from a
      Procs command-line row back to its block."
  - id: completion-popup
    title: Grammar-aware completion popup and signature line
    depends_on:
      - panel-shell
      - line-resolver
    size: medium
    description:
      "completion-popup: load the grammar at idle and wire the resolver into the input.
      This adds token highlighting, advisory diagnostics, the floating fuzzy popup fed
      by in-memory TUI entities and debounced providers, the zsh menu-select key rules,
      and the live signature line with its chips."
  - id: completion-extras
    title: Empty state, doc peek, and history search
    depends_on:
      - completion-popup
    size: medium
    description:
      'completion-extras: add the empty-state RECENT and derived "FOR <selection>" rows,
      a wide-terminal doc peek beside the popup, ctrl+r fuzzy history search in the
      popup, marked rows that fill variadic slots, and provider-unavailable footers.'
  - id: run-policies
    title: Run policies, confirmation-aware blocks, and built-ins
    depends_on:
      - transcript-blocks
      - completion-popup
    size: medium
    description:
      "run-policies: run foreground-policy commands in the real terminal through
      `app.suspend()` and refuse deny-policy commands with an alternative. Add
      confirmation-aware declined blocks with an explicit `R` rerun-with-`-y`, plus the
      `cd`, `clear`, `help` and `history` built-ins."
  - id: flip-and-land
    title: Flip `:` and `;`, remove the flag, and land
    depends_on:
      - color-contract
      - kind-coverage
      - completion-extras
      - run-policies
    size: medium
    description:
      "flip-and-land: bind `:` to the Command Line and `;` to the palette only. Add the
      palette-side hop key, the fallback row and the one-time tip. Update every help,
      onboarding, docs and test touchpoint, delete the flag's Off branch, close the flag
      bead, and regenerate the PNG goldens."
proposed_by: bbugyi200.athena.0qs
create_time: 2026-09-24 11:20:33
status: done
---

- **PROMPT:**
  [prompts/202609/command_line_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/command_line_panel.md)

# Plan: `:` Command Line — run `sase` commands from inside the TUI

## Context

Today `:` and `;` both open the Command Palette
(`open_command_palette: "colon,semicolon"` in `src/sase/default_config.yml`). The
palette only runs TUI actions. The CLI has 63 top-level commands and 365 leaf commands,
and most of them are unreachable without leaving the TUI. The only "run a command"
feature, `!!`, takes three modals to reach. It runs `sh -c` as a service oneshot, and
the Procs tab hides service oneshots.

This epic gives `:` to a new **Command Line**. It is a vim/Helix/k9s-style drawer where
you type `sase` commands (the `sase` prefix is implicit) and get completion that no
shell can match. The Command Line knows the current selection, holds live entities in
memory, ranks fuzzily with Rust, shows rich descriptions and a live signature line, and
gives advisory grammar diagnostics. Commands run as ordinary durable procs, so hiding
the panel never interrupts anything.

The design follows the consolidated research report. Read it with
`sase artifact read research:202609/tui_colon_command_line/tui_colon_command_line.md "<why>"`.
Its claims were re-verified against current `master` of sase and sase-core while writing
this plan. This plan is self-contained. Where it differs from the report, **this plan
wins**.

## Design decisions (final)

| Decision                 | Choice                                                                                                                                                                                       | Why                                                                                                                                                                                                                                          |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Name                     | **Command Line** (action `open_command_line`, package `sase.ace.tui.command_line`, flag `ace_command_line`, keymap scope `command_line`, proc tag `command-line`)                            | In this TUI "mode" already means a transient prefix mode (leader, `bang_mode`, custom modes), and vim calls `:` the command line                                                                                                             |
| Surface                  | A bottom-anchored `ModalScreen` drawer with a rounded `$primary` frame                                                                                                                       | Sets it apart at a glance from the centered, double-framed palette. Modals are the house pattern, and they avoid focus and `check_app_action` juggling                                                                                       |
| Execution                | **Ordinary procs**: `origin="ace"`, tag `command-line`, no `operation`, no `service` block                                                                                                   | `_submit_durable_proc` needs an operation, and a plain command would settle as `error / missing-result`. The Procs default query `-service` hides service oneshots. `origin="ace"` keeps the observer's cross-session relevance rule working |
| Not everything is a proc | Each command has a run policy: `proc` (default), `foreground` (suspend the TUI and run in the real terminal), or `deny` (refuse and suggest an alternative)                                  | Procs run with `stdin=/dev/null` and no TTY, so editors, fzf and pagers cannot work under them                                                                                                                                               |
| Safety                   | **Never inject flags silently.** A `⚠ writes` chip is advisory, and Enter always runs. Confirmation-aware blocks offer an explicit, visible `R` rerun with `-y`                              | The CLI owns its safety prompts, and they fail closed on `/dev/null`                                                                                                                                                                         |
| Completion engine        | A frozen Rust `CommandLineGrammar` in `sase_core`, fed by the Python-built spec                                                                                                              | Core-memory boundary rule: another frontend (such as sase-nvim or mobile) would need the same behavior. Fuzzy ranking and the frozen-catalog handle pattern already live in sase-core                                                        |
| Spec source of truth     | argparse → `sase completion spec -d -j`, built **in a subprocess** and cached by runtime identity plus source fingerprint                                                                    | Keeps about 0.4 s of GIL-heavy work off the event loop, and describes the on-disk code that procs will actually run (tui_perf rule 15)                                                                                                       |
| Working context          | Follow the TUI's current project and resolve to its primary checkout, falling back to the TUI launch cwd. `cd` pins a directory and `cd -` unpins                                            | The results of `sase bead list` depend on the cwd, so the context is always visible as a chip                                                                                                                                                |
| Retention                | Finished `command-line` procs get their own bucket of 50 and are **visible by default** in Procs                                                                                             | Heavy use must not push agent-launch and Patch-operation procs out of the shared 100-row bucket                                                                                                                                              |
| Transcript persistence   | App-held while the TUI runs. On the first open after a restart, restore `command-line` procs from the last 24 h (up to 20) below an `── earlier ──` divider                                  | Durable procs make this free                                                                                                                                                                                                                 |
| Rollout                  | The beta flag `ace_command_line` (epic scaffolding) gates the panel. `open_command_line` defaults to the `unbound` sentinel during beta. The final phase flips the keys and deletes the flag | `:` keeps opening the palette until the feature is complete                                                                                                                                                                                  |
| Non-goals                | `:!<shell>` / converging with `!!`, generated forms, remote/fleet execution, a warm supervisor for latency, running quick queries in-process                                                 | Each is a possible later follow-up. None is needed for a great v1                                                                                                                                                                            |

## UX specification (the contract every panel phase implements)

### Geometry and chrome

- `CommandLineScreen(ModalScreen)` uses `align: center bottom`. The frame is 96% wide
  (max 160 columns). Its height grows with content up to 65% of the screen, and `ctrl+t`
  toggles full height. The dimmed TUI stays visible above the frame.
- Layout from top to bottom: transcript (scrollable) → popup (floats **over** the
  transcript, anchored just above the input, as in Helix) → input row → signature/hint
  row.
- The top border has the title `❯ Command Line` on the left and the working-context chip
  on the right. The bottom border has the context key hints on the left and `N running`
  on the right. Pad and recompose the titles on resize, so both ends stay aligned.

Empty state (Agents tab, an agent selected):

```
╭─ ❯ Command Line ────────────────────────────── ⌂ +sase · ~/projects/github/sase-org/sase ─╮
│  RECENT                                                                                   │
│    bead show sase-17p                                                     2h ago  ✓       │
│    agent wait research.2h --timeout 10m                                   1d ago  ✗ 124   │
│  FOR research.2h.cld · selected agent                                                     │
│    agent show research.2h.cld                                                             │
│    chat show research.2h.cld                                                              │
│                                                                                           │
│ ❯ sase ▌                                                                                  │
│   63 commands · type to search · ⇥ complete · ; Command Palette                           │
╰─ ⏎ run · ⇥ complete · ↑↓ history · ^R search · esc hide ──────────────────── 0 running ───╯
```

Typing, with the popup, the signature line and a chip:

```
╭─ ❯ Command Line ────────────────────────────── ⌂ +sase · ~/projects/github/sase-org/sase ─╮
│ ✓ bead list --status open                           exit 0 · 0.9s · 14:02 · proc 470vtq   │
│ │ ◐ M sase-17p.4 · Stop, follow, and wait on a ToolRun by id ← sase-17p                   │
│ │ ⋯ 406 more lines · o expand                                                             │
│        ┌───────────────────────────────────────────────────────────────┐                  │
│        │ ◆ sase-17p     epic  in_progress  Tool-run hand-off      sel  │                  │
│        │   sase-17p.4   task  in_progress  Stop, follow, and wait…     │                  │
│        │   sase-17m     epic  open         Rename agent family…        │                  │
│        │ bead · 3 of 410 · fuzzy                            ⇥ accept   │                  │
│        └───────────────────────────────────────────────────────────────┘                  │
│ ❯ sase bead close sase-17▌                                                                │
│   bead close ‹ID…› [-n NOTE] [-r REASON] [-R canceled|done|superseded]          ⚠ writes  │
╰─ ⏎ run · ⇥ accept · ↑↓ move · esc normal ─────────────────────────────────── 0 running ───╯
```

Transcript blocks, with a block selected in NORMAL mode:

```
│ ✗ bead close sase-zz                                exit 2 · 0.7s · 14:05                 │
│ │ error: no such bead: sase-zz                                                            │
│ ⊘ agent restart foo                                 declined · exit 2 · R rerun with -y   │
│ │ Restart would stop foo and wipe 3 related agents.                                       │
│▌⠹ agent wait research.2h --timeout 10m             running 0:42 · proc 3aacsa             │
│▌│ waiting for 3 agents (research.2h.mus, research.2h.gem, research.2h.cld) …              │
│ ↗ prompt edit                                       ran in terminal · exit 0              │
╰─ K kill · r rerun · e edit · o expand · v pager · y copy · p Procs · i input ─ 1 running ─╯
```

### Visual language

Reuse the palette's exact color values; read them from
`modals/command_palette_modal.py`.

| Element                              | Style                                                                                                                                                                   |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Frame                                | `border: round $primary`, not the palette's `double`. The `❯` in the title is bold `#FFD700`                                                                            |
| Implicit prefix                      | A dim, non-editable `❯ sase ` before the input. A leading `sase ` that the user types or pastes is stripped at once                                                     |
| Command-path tokens                  | bold `#00D7AF`                                                                                                                                                          |
| Options                              | `#87D7FF`                                                                                                                                                               |
| Quoted strings                       | a warm, opaque accent. Follow `test_prompt_bar_palette_safety.py`: opaque colors, and avoid xterm-256 slots 16–21                                                       |
| Diagnostics                          | a red undercurl on the token, with the message shown in the signature row                                                                                               |
| Ghost suggestion                     | Textual's `text-area--suggestion`, dim italic                                                                                                                           |
| Chips (signature row, right-aligned) | `⚠ writes` dim amber · `↗ terminal` (foreground) · `⊘ not runnable` (deny) · `asks to confirm · -y`                                                                     |
| Block gutter                         | gold braille spinner (running) · green `✓` · red `✗` plus the exit code · dim `⊘` (killed, declined or denied) · `↗` (ran in the terminal) · `›` (built-in)             |
| Block header right                   | dim `exit 0 · 0.9s · 14:02 · proc 470vtq`, or `running 0:42 · proc …` while running                                                                                     |
| Selected block                       | a `▌` bar in `#00D7AF` on the left, as in the Agents list                                                                                                               |
| Popup rows                           | entity glyph, value (match runs highlighted), kind badge and description columns. Selected-entity rows get `◆ … sel`. Footer: `<kind> · N of M · fuzzy` plus a key hint |
| Unseen block                         | a small `•` dot next to the gutter until the block is viewed                                                                                                            |

### Keys (panel-scoped, configurable under `ace.keymaps.command_line`)

| Context   | Key                        | Action                                                                                                                             |
| --------- | -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| INSERT    | `⏎`                        | Run. If the popup menu is **active** (the user tabbed into it), accept the highlighted item instead (zsh menu-select rule)         |
| INSERT    | `⇥` / `shift+⇥`            | Tab inserts the unique candidate or the longest common prefix. Otherwise it activates the menu, and later presses cycle through it |
| INSERT    | `ctrl+n` / `ctrl+p`        | Activate the menu and move within it                                                                                               |
| INSERT    | `↑` / `↓`                  | With the menu active, move in it. Otherwise walk history filtered by the typed prefix                                              |
| INSERT    | `→` at the end of the line | Accept the ghost text (native `TextArea.suggestion`)                                                                               |
| INSERT    | `ctrl+r`                   | Fuzzy history search, shown in the popup                                                                                           |
| INSERT    | `esc`                      | Menu active → leave the menu and restore the typed text. Empty line → hide the panel. Otherwise → NORMAL                           |
| INSERT    | `;` on an empty line       | Hop to the Command Palette                                                                                                         |
| INSERT    | `ctrl+t`                   | Toggle full height                                                                                                                 |
| INSERT    | `ctrl+l`                   | Clear the transcript (same as the `clear` built-in; procs are untouched)                                                           |
| NORMAL    | `esc`                      | Hide the panel. Running commands continue and the draft is kept                                                                    |
| NORMAL    | `k` / `↑`                  | Move into the transcript (block navigation)                                                                                        |
| Block nav | `j` `k` `g` `G`            | Select a block, or jump to the first or last block                                                                                 |
| Block nav | `o` / `⏎`                  | Expand or collapse the block                                                                                                       |
| Block nav | `v`                        | Open the full output in `PagerScreen`                                                                                              |
| Block nav | `K`                        | Kill, with confirmation, through the Procs pane's kill path                                                                        |
| Block nav | `r` / `R`                  | Rerun / rerun with `-y` (only on confirmation-declined blocks)                                                                     |
| Block nav | `e`                        | Load the command into the input for editing                                                                                        |
| Block nav | `y` / `Y`                  | Copy the output / copy the command                                                                                                 |
| Block nav | `p`                        | Open Admin Center → Procs with this proc selected                                                                                  |
| Block nav | `x`                        | Remove the block from the transcript (the proc record stays)                                                                       |
| Block nav | `i` `a` `:`                | Return to the input in INSERT mode                                                                                                 |

### Completion behavior

1. **Slot-aware.** The resolver classifies the cursor as a subcommand, option name,
   option value, positional, or remainder. It handles `--opt=value`, stacked short
   flags, `--`, remainder positionals such as `proc run -- CMD`, bare groups with a
   `default_child`, and argparse's unique long-option abbreviations. It never flags a
   unique abbreviation as unknown.
2. **Sources, in priority order.**
   - The TUI's in-memory entities: agents, procs, projects and Patches, which are
     instant and fresh.
   - Static spec data: subcommands, options and choices.
   - `candidates_for(kind, "", project=…, limit=2000)` in a debounced (~80 ms) worker,
     cached per `(kind, project)` with a short TTL. The provider limit is applied before
     any ranking, so fetch wide and rank in Rust.
   - Paths.
3. **Selection first.** The selected entity and the entities linked to it (a selected
   agent's or Patch's bead) rank first and are marked `◆ … sel`.
4. **Fuzzy, not prefix.** Ranking uses sase-core's `fuzzy_match` tiers with match runs
   highlighted. Exact-prefix hits outrank fuzzy ones.
5. **Grammar-aware suppression.** Non-repeatable options that are already used are
   hidden, as are the other members of a mutex group that is already used.
6. **Live signature.** The active slot is highlighted. When the popup highlights an
   option, the row shows that option's summary, choices, default and repeatable/mutex
   notes instead.
7. **Advisory, never gatekeeping.** The resolver reports unknown subcommands and
   options, invalid choices, missing required arguments, unterminated quotes, and "asks
   to confirm". **Enter always runs.** argparse is the authority and reports its own
   errors in the block.

### Leaving, returning, and reliability

- Hiding never interrupts anything, and the unsent line is kept as a draft. Reopening
  with `:` is instant because it restores from app-held state.
- A block that finishes while the panel is hidden raises a toast, such as
  `✓ bead list · exit 0 · 0.9s — : to view`. Use the live key display name, and omit the
  hint while unbound. Failures toast at error severity.
- Quitting the TUI never kills command-line procs.
- `⏎` on a `command-line` row in Admin Center → Procs opens the Command Line with that
  block selected, adding the block if the transcript lacks it.
- Output rendering:
  - Collapse `\r` overwrites and drop non-SGR CSI and OSC sequences, then apply
    `Text.from_ansi`. Cache the result per `(proc_id, byte offset)`.
  - A collapsed block shows its last 12 lines plus `⋯ N more lines · o expand`. An
    expanded block renders at most 2,000 lines. `v` opens the pager for anything longer.
- A double-Enter guard drops a submission of the same line within 300 ms. A deliberate
  `r` is always allowed.
- If submission fails, the block turns red with the error and the line goes back into
  the input.
- A block whose proc was pruned keeps its cached tail and shows `record pruned`.

### Performance budgets (tui_perf rules 1, 2, 4, 11)

- The keystroke path is synchronous and in memory: the grammar handle, in-memory sources
  and Rust ranking. It never spawns a process, never calls `resolve_ref` and never takes
  a lock.
- Async provider results re-capture the line and cursor after the await, and are dropped
  if either changed.
- Tailing and rendering run in pump-free tasks (`spawn_pump_free_task`), cancelled at
  teardown. A running block repaints only its own widget.
- Targets:
  - first paint under 50 ms
  - keystroke to popup p95 under 16 ms
  - async kinds under 150 ms
  - zero `tui_stalls.jsonl` rows during a stress run
  - `resolve` p95 under 1 ms on the full spec.

## Architecture and shared contracts

```
            ┌─────────────────────────────── TUI process ─────────────────────────────────┐
 keystroke ─▶ CommandLineScreen (ModalScreen, view only)                                  │
            │   input · popup · signature · transcript widgets                            │
            │        │ sync                                                               │
            │   CommandLineSession (app-held: blocks, draft, cwd pin, history cursor)     │
            │        │ resolve/complete ──▶ sase_core_rs.CommandLineGrammar (frozen)      │
            │        │ in-memory sources (agents/procs/projects/Patches)                  │
            │        │ async: candidates_for(kind) worker · debounced · last wins         │
            │        ▼                                                                    │
            │   submit_command_line_proc() ──worker──▶ submit_proc_request()              │
            │        ▲                   (no operation · origin=ace · tag=command-line)   │
            │   ProcObserver exit watch ◀── store rows (status, exit_code)                │
            │   ProcLogCursor (offset tail; visible running blocks only)                  │
            └─────────────────────────────────────────────────────────────────────────────┘
 spec cache: `sase completion spec -d -j -o <sase_home>/completion/cache/command_line_spec-<key>.json`
             built in a subprocess · key = runtime_identity_key() + source_fingerprint()
```

**Package layout.** The panel-shell phase creates `src/sase/ace/tui/command_line/`.
Later phases own separate modules so that parallel phases rarely touch the same file:

- `session.py`: `CommandLineSession` and `CommandLineBlock`. panel-shell creates them.
- `screen.py`: `CommandLineScreen`, composition and the action dispatch table.
  panel-shell creates it and later phases add small hooks.
- `input.py`: the input widget (a `SingleLineVimTextArea` subclass plus a highlight
  overlay). Owned by panel-shell and completion-popup.
- `transcript.py` and `block_render.py`: block widgets and output rendering. Owned by
  panel-shell and transcript-blocks.
- `popup.py`, `signature.py` and `sources.py`: owned by completion-popup and
  completion-extras.
- `context.py`: working-context resolution. `history.py` is the TUI adapter over the
  `src/sase/history/command_line.py` store.
- `policies.py` and `builtins.py`: owned by run-policies.

**Proc contract.**
`submit_command_line_proc(argv_tokens, *, cwd, project, width, session_id)` builds a
`ProcSubmitRequest` with these fields:

- `argv=sase_command_argv(*tokens)`, from `ace/tui/durable_ops.py`, so the TUI's own
  interpreter runs it
- `command=["sase", *tokens]`
- `label=": " + shlex.join(tokens)`
- `tags=(COMMAND_LINE_PROC_TAG,)`
- `origin="ace"`
- `env={"PYTHONUNBUFFERED": "1", "FORCE_COLOR": "1", "COLUMNS": str(width)}`
- `operation=None`, `service=None`, and no concurrency keys.

**Spec wire additions.** These are additive fields in `sase completion spec` JSON:

- `OptionSpec`: `required: bool`, `metavar: str | null`, `default: str | null`,
  `value_hint`.
  - `default` is display-safe: a scalar str/int/float/bool of at most 40 characters.
    Otherwise it is null, and it is also null for `SUPPRESS`.
  - `value_hint` is one of `"text" | "int" | "number" | "duration" | "path" | null`.
- `PositionalSpec`: `required: bool`, derived from `nargs`, and `value_hint`.
- `CommandSpec`:
  - `run_policy`: a list of rules
    `{"policy": "proc"|"foreground"|"deny", "when": {"absent": [dest…]} | {"equals": {dest: value}} | null, "note": str | null}`.
    The first matching rule wins, and a leaf with no rules is `proc`.
  - `writes: bool`
  - `stdin: bool`, meaning the command reads stdin, so the signature notes "reads stdin;
    pass input via flags or files".

  `confirms` is derived by the resolver: the leaf has a `-y/--yes` option.

**Resolver API.** These are frozen pyclass methods returning JSON-shaped dicts:

- `CommandLineGrammar(spec_json: str)`
- `.resolve(line, cursor) -> LineContext`, containing:
  - `tokens[{text,start,end,role,quoted,unterminated}]`
  - `argv[]`, the POSIX-shlex-split tokens with a leading `sase` stripped
  - `path[]` and `node_kind`
  - `slot{kind,dest,value_kind,choices,value_hint,prefix,replace_start,replace_end}`
  - `used_dests[]`
  - `diagnostics[{start,end,severity,code,message}]`
  - `signature{segments[{text,role,active,required}],summary}`
  - `run_policy{policy,note}`, `writes`, `confirms`, `confirm_flag_present` and `stdin`.
- `.complete(line, cursor, dynamic, selected, limit) -> {replace_start, replace_end, items[{insert_text, display, description, badge, source, match_runs, selected}], total, kind}`.
  - `dynamic` is `[{value, description, badge, source}]`.
  - `insert_text` is quoted when needed and ends with a trailing space once a token is
    complete.
- `.command_help(path) -> {usage, summary, positionals[…], options[…], children[…]}`,
  used by the doc peek and the `help` built-in.

A single tokenizer produces both the highlighted tokens and the submitted argv, so what
you see is what runs.

## Phase sections

### `color-contract`: Output color contract

Today `bead list` and `bead show` print no ANSI at all under a pipe, even with
`FORCE_COLOR=1`, because `bead/cli_dep_render.py` `resolve_color` checks only `NO_COLOR`
and then `isatty()`.

- Add one shared helper, for example
  `sase.core.term_color.should_colorize(stream, *, mode="auto")`. Its precedence:
  1. Explicit `-c always|never` flags win where they exist.
  2. `NO_COLOR` means off.
  3. `FORCE_COLOR` or `CLICOLOR_FORCE` (non-empty, not `0`) means on.
  4. Otherwise use `stream.isatty()`.

  Also make any Rich `Console(...)` constructed for stdout honor it: Rich already reads
  `FORCE_COLOR`, so verify it and do not double-handle.

- Sweep every **stdout/stderr color** `isatty()` branch in `src/sase` and route it
  through the helper. There are about 32 files that mix stdin and stdout checks. Leave
  stdin interactivity checks alone, because those are confirmation fail-closed semantics
  that must not change.
- Tests: a parametrized env matrix for the helper, plus a regression test that runs
  `bead list` and `bead show` rendering with `FORCE_COLOR=1` on a non-TTY stream and
  asserts SGR sequences are present. `NO_COLOR` must still win.

### `spec-contract`: Command Line spec contract

- Extend `src/sase/completion/model.py` and `build.py` with the spec wire additions
  above. Keep `from_json` tolerant of their absence, so older caches still load.
- Add `src/sase/completion/run_policy.py` next to `kinds.py`'s `PATH_OVERRIDES`. It
  holds:
  - a table of `(path) → [rules]`
  - a `writes` heuristic (a verb list on the leaf name: close, create, delete, kill,
    prune, remove, restart, update, …) with explicit per-path overrides both ways
  - a `stdin` set.

  Initial policy sweep:
  - `foreground`: `run` when its prompt positional is absent or equals `.`;
    `prompt edit`; `prompt run`; `artifact open`; `pager`; `tmux-agent`; the `gate act`
    editor actions; `sudo approve` when both `--approve` and `--deny` are absent.
  - `deny`, with notes:
    - `tui` → "You're already in the TUI"
    - `service run` / `scheduler run` / `mobile gateway start` / `lsp` → "Foreground
      server; use `sase service start`"
  - `stdin`: `notify create`, `comments`, and any other command the sweep finds reading
    `sys.stdin`.

  Re-verify every name against the live parser. A **drift test** asserts that every
  table path exists in `build_spec()`. Its failure message tells the author exactly
  where and how to classify a new command.

- Add `sase completion spec -d/--descriptions` to keep real summaries instead of
  digests, following `cli_rules`: a short alias, alphabetized options and excellent
  `-h`. Update the checked-in snapshot (`tests/completion/snapshots/cli_spec.json`) and
  the digest behavior only as the additive fields require.
- Add `src/sase/completion/command_line_spec.py`, the identity-keyed cache:
  - `command_line_spec_path()` returns the cache path, keyed by `runtime_identity_key()`
    plus `source_fingerprint()`.
  - `ensure_command_line_spec(*, timeout)` returns the path of an existing spec. If none
    exists, it builds one via
    `subprocess.run([sys.executable, "-m", "sase", "completion", "spec", "-d", "-j", "-o", tmp])`,
    renames it into place atomically, and prunes stale siblings.

  It must be safe to call from a worker thread, it never prompts, and it records timings
  with the existing completion-cache conventions.

- Tests:
  - model round-trip with the new fields
  - `required` derivation for every `nargs` shape
  - display-safe defaults
  - the run-policy drift test
  - a cache hit/miss/rebuild test with a stubbed subprocess.

### `kind-coverage`: Value-kind coverage and ratchet

Today only 15% of value-taking options and 42% of positionals declare a kind.

- Add the `ValueKind` members `GATE`, `TOOL_RUN` and `TASK_TYPE`, and `SESSION` only if
  a real slot needs it. Add their `candidates_for` providers. They must be read-only and
  prompt-free, with no `textual`/`rich`/`sase.ace` imports, which is the catalog
  quarantine.
- Annotate the unkinded entity slots through `NAME_TABLE` / `PATH_OVERRIDES`:
  - `gate_ref`
  - ToolRun `run_id`
  - memory `selector` → `MEMORY`
  - `task_type`
  - `provider` → the existing `PROVIDER`
  - `path`-like slots → `PATH`/`DIR`
  - the `id`/`name` slots on paths where they name an entity.

  Do not add bare `id`/`name` to `NAME_TABLE`.

- Mark genuinely free-form slots with the spec's `value_hint` (`text`, `int`, `number`,
  `duration`), for example `reason`, `limit`, `timeout` and `description`. Use a
  declarative hint table beside `PATH_OVERRIDES`, with the same attribute mechanism
  `kinds.py` already uses.
- **Ratchet test:** every non-hidden, value-taking option and positional in
  `build_spec()` must have a `kind`, `choices` or `value_hint`. Its failure message
  names each offending `(path, dest)` and tells the author the three ways to fix it.
  Regenerate the spec snapshot and the shell-completion goldens this touches. Shell
  completion improves as a side effect.

### `proc-retention`: Command-line proc tag and retention bucket

Work in the linked sase-core checkout (`sase repo open sase-core`), then in sase.

- In `crates/sase_core/src/procs/store.rs`:
  - Add `pub const COMMAND_LINE_PROC_TAG: &str = "command-line"` and
    `pub const COMMAND_LINE_PROC_HISTORY_LIMIT: usize = 50`.
  - In `apply_retention`, route terminal procs that carry the tag and are not named
    service procs into their own bucket pruned against that limit. This mirrors the
    per-name `SERVICE_PROC_HISTORY_LIMIT` buckets.
  - Unit tests:
    - 60 finished command-line procs plus 100 generic ones keep 50 plus 100.
    - Running procs are never pruned.
    - Generic retention is unchanged.
- Export both constants through `sase_core_py` and re-export them from a thin sase
  module, for example `src/sase/procs/command_line.py`. Add a sase test asserting the
  tag value.
- Move `sase-core-revision.txt` past the sase-core commit (see `docs/rust_backend.md`).
  Document the bucket beside `procs.history_limit` in `default_config.yml` comments and
  in the procs docs.

### `line-resolver`: sase-core CommandLineGrammar resolver

Work in sase-core first, then add the sase adapter. Read the sase-core `AGENTS.md`.

- Add a `command_line` module in `crates/sase_core/src/editor/` (or a sibling such as
  `crates/sase_core/src/command_line/`, if that fits the crate's layout better). It
  contains:
  - The spec wire types (serde, tolerant of unknown fields).
  - A POSIX-shlex tokenizer with spans, quote state and `unterminated` detection.
  - A slot resolver implementing the argparse semantics in "Completion behavior" item 1.
  - Static candidates: subcommands with aliases and summaries, options, and choices.
  - Merge and rank over static plus caller-supplied dynamic candidates, using
    `editor::fuzzy`, with the prefix-tier and selected-first rules and the suppression
    rules.
  - Diagnostics.
  - The signature builder with active-segment marking.
  - Run-policy rule evaluation over parsed dests.
  - `command_help`.
- Add the `CommandLineGrammar` frozen pyclass in
  `crates/sase_core_py/src/editor_completion/` (or a sibling), following the
  `GlossaryCatalogHandle` / `AtReferenceInventory` pattern. The spec JSON is parsed
  once, and every method returns JSON-shaped dicts per the resolver API above.
- Tests in sase-core:
  - Golden cases against a spec fixture generated from sase's
    `sase completion spec -d -j` and checked into sase-core. Cover every slot kind,
    `--opt=v`, stacked shorts, `--`, remainder, `default_child` groups, abbreviations,
    mutex suppression, unterminated quotes, and each run-policy predicate.
  - A perf test: `resolve` p95 under 1 ms, and `complete` with 1,000 dynamic candidates
    under 2 ms.
- sase adapter, `src/sase/completion/command_line_grammar.py`:
  - `load_command_line_grammar(path) -> CommandLineGrammar` (read and construct; call it
    only from a worker thread).
  - Typed Python dataclasses or `TypedDict` views over the returned dicts.
  - A small contract test that builds the real spec, loads it, and resolves a few lines,
    which proves that the Python spec and the Rust wire agree.
- Move `sase-core-revision.txt` past the sase-core commit and document the handle in
  `docs/rust_backend.md`.

### `proc-plumbing`: Command-line proc plumbing

- Add `submit_command_line_proc(...)` per the proc contract, in
  `src/sase/procs/command_line.py` (TUI-independent, so another frontend could reuse
  it). Register the TUI producer site in `ace/tui/_proc_producer_sites_actions.py`;
  `tests/ace/tui/test_proc_producer_inventory.py` enforces this.
- `ProcObserver` (`ace/tui/proc_observer.py`):
  - Add `register_exit_watch(proc_id, *, placeholder_id)`. It settles from the store
    row's `status`, `exit_code` and `finished_at`, and never decodes a typed result. Its
    completions are delivered on the snapshot like the existing watch completions, but
    as a distinct `ProcExitCompletion` type.
  - Replace the single `set_detail_proc` slot with ref-counted tail subscriptions
    (`subscribe_tail(proc_id) -> token` / `unsubscribe_tail(token)`), so the Procs pane
    and the Command Line do not fight. Migrate the Procs pane to the new API.
- Add `ProcLogCursor` in `src/sase/procs/logs.py`:
  - It holds `(inode, offset)` and reads only new bytes, with a cap per read.
  - It detects rotation (an inode change or shrink), drains `.log.1`, and reports
    `lost_bytes`.
  - Tests cover appends, rotation, truncation, and missing files.
- Add `ObservedProc.tags`, plumbed from store rows in `_proc_observer_store.py`.
- Add a `tag` field to the Procs query schema (`ace/query_profile/profiles/_procs.py`)
  and to the row adapter in `ace/tui/_proc_query.py`, so that `tag:command-line` and
  `-tag:command-line` work. Also add `origin`. Update the query help or docs for the
  Procs dialect.
- Add a `proc_focus_target` keyword to `_open_config_center`
  (`ace/tui/actions/base.py`), in the style of `log_error_target`. It is threaded into
  `ConfigCenterModal` so the Procs pane opens with that proc selected, even when the
  proc is filtered out by the current query (the pane should then clear the filter and
  show a subtle notice).
- Tests:
  - the submission request shape (tag, origin, env, no operation)
  - exit-watch success, error and killed outcomes
  - ref-counted tails
  - the `tag:` query field
  - opening the focus target.

### `panel-shell`: Command Line panel shell (beta flag)

- Create the flag with `sase flag new ace_command_line -k beta` and the three authored
  sentences:
  - `--when-enabled`: "The TUI offers the Command Line panel through `open_command_line`
    and a Command Palette row."
  - `--when-disabled`: "`open_command_line` only shows a notice and the palette has no
    Command Line row."
  - `--remove-when`: "The epic's final phase flips `:` to the Command Line and `;` to
    the palette."

  Paste the printed registry entry. Follow the printed both-states test checklist.

- Keymaps and the action:
  - Add `open_command_line: "unbound"` to `default_config.yml`, plus the `AppKeymaps`
    field, the `_BINDING_META` row, the `_APP_COMMAND_META` / catalog row "Command Line"
    (available only when the flag is on), and `action_open_command_line`.
  - With the flag off, the action shows an info toast naming the flag.
  - Add a `command_line` keymap scope (`CommandLineKeymaps` dataclass, bundled defaults,
    and a loader in `keymaps/scopes.py`) for the panel keys in the Keys table.
- `CommandLineSession`, held on the app:
  - a bounded block list (max 200)
  - the draft text and cursor
  - the cwd pin
  - a history cursor
  - a `restored` flag.

  Reopening re-mounts the screen from the session, with no disk I/O on open.

- Working context (`context.py`):
  - Resolve the current project (the source of `widgets/current_project_indicator.py` /
    `sase.current_project.resolve_current_project`) to its primary checkout, off-thread
    and cached. Fall back to the TUI launch cwd.
  - Chip: `⌂ +<project> · <~-abbreviated path>`, middle-truncated to fit. A pinned
    context shows `(pinned)`.
- History store (`src/sase/history/command_line.py`):
  - The file is `<sase_home>/command_line_history.json`, using an `fcntl` lock and
    atomic writes as in `history/prompt_store.py`.
  - Entries: `{line, cwd, project, last_exit, count, last_used}`, capped at 1,000 by
    LRU.
  - Loading and writes happen off-thread.
  - `↑`/`↓` walk history filtered by the typed prefix. Ghost text is the most recent
    prefix match, preferring the same cwd.
- Input: a `SingleLineVimTextArea` subclass with the dim implicit `❯ sase ` prefix and
  leading-`sase ` stripping. It follows the Escape and `;`-hop rules from the Keys
  table. The hop dismisses the panel and calls `action_open_command_palette`.
- Submit:
  - Tokenize with `shlex.split` (the resolver's argv replaces this in completion-popup;
    keep the seam in one function).
  - Create an optimistic `submitting` block and an observer placeholder at once, then
    call `submit_command_line_proc` in a thread worker.
  - Register the exit watch, and handle submit failure by restoring the line.
  - `COLUMNS` is the transcript's inner width.
- Transcript blocks:
  - The header shows the gutter glyph, the command, and the right-aligned metadata.
  - The collapsed body is the last 12 lines, fed by `ProcLogCursor`. It is polled every
    200 ms by a pump-free task, only for visible running blocks while the panel is
    shown.
  - Output uses the sanitize → `Text.from_ansi` → cache pipeline.
  - The spinner advances on the same tick.
- PNG goldens, as a new `tests/ace/tui/visual/test_ace_png_snapshots_command_line.py`
  with fixture data and frozen time: empty, typed line with ghost text, running block,
  success block, error block, and submit-failed. Run
  `just fix-tui-screenshots -- <selector>` and inspect every created golden.
- Tests:
  - both flag states
  - Escape semantics
  - draft persistence across hide and reopen
  - history locking and LRU
  - leading-`sase ` stripping
  - the submit happy path and failure path (stub the submitter)
  - no pump stalls in the tail task.

### `transcript-blocks`: Transcript block interactions and lifecycle

- NORMAL-mode block navigation and every Block-nav key from the Keys table:
  - Kill reuses the Procs pane's durable kill path and `ConfirmKillModal`.
  - `v` pushes `PagerScreen` with the full sanitized log.
  - `p` uses `proc_focus_target`.
  - The bottom-border hints switch to the block key set.
- Toasts: a finish while the panel is hidden raises the toast (error severity on
  failure) and marks the block unseen. Viewing a block clears the dot.
- Restore after restart: on the first open per app session, add blocks for store rows
  with the `command-line` tag and `origin == "ace"` from the last 24 h (up to 20). They
  go below an `── earlier ──` divider, and their output loads lazily when visible or
  expanded.
- Pruned records: the cached tail stays and the block shows `record pruned`. Restore
  skips pruned records.
- Rotation: if `ProcLogCursor` reports `lost_bytes`, show `⋯ earlier output rotated`.
- In the Procs pane, `⏎` on a `command-line` row opens the Command Line focused on that
  block. Extend `procs_pane_agent_jump.py`'s selection routing; monitor rows keep their
  agent jump.
- PNG goldens: block selection, the expanded block, and the restored `── earlier ──`
  divider.
- Tests: every block action (stub the kill and pager), toast-when-hidden, the unseen
  dot, the restore window and limit, pruned handling, and the Procs `⏎` routing.

### `completion-popup`: Grammar-aware completion popup and signature line

- Grammar loading: after the startup stopwatch at idle, or on the first open, run
  `ensure_command_line_spec` and then `load_command_line_grammar` in a thread worker.
  Store the handle on the app. Until it lands:
  - The popup shows `indexing commands…`.
  - History and ghost text still work.
  - Submission falls back to `shlex.split`.
- Input: on every edit, call `resolve` synchronously.
  - Apply token-role highlight spans through the existing `_highlights` overlay
    approach, as in `widgets/_alt_syntax_highlight.py`.
  - Add a red undercurl for diagnostics.
  - Submit through `LineContext.argv`.
- `popup.py`: a floating `OptionList` of at most 8 visible rows, positioned above the
  input at the replace-span column. Rows follow the visual-language spec and highlight
  match runs. Obey tui_perf rule 12 (guard programmatic highlight echoes).
- `sources.py`:
  - In-memory entity sources for agents, procs, projects and Patches, read from app
    state without I/O.
  - A debounced, last-wins `candidates_for` worker cache per `(kind, project)` that
    re-captures the line and cursor after the await.
  - Selected-entity extraction from `extract_command_context` (tab, selected
    agent/Patch/axe item, and the linked bead through the agent or Patch).
- Key rules: Tab, menu activation, Enter-accepts-when-active, Escape-leaves-menu, and
  ctrl+n/ctrl+p, exactly as in the Keys table.
- `signature.py`: the live signature segments with the active slot emphasized, the
  option-summary swap when an option is highlighted, the first diagnostic message, and
  the right-aligned chips (`⚠ writes`, `↗ terminal`, `⊘ not runnable`,
  `asks to confirm · -y`, `reads stdin`).
- PNG goldens: typing with the popup open, the signature with an active slot, the
  `⚠ writes` chip, a diagnostic undercurl, and `indexing commands…`.
- Perf: add a keystroke → popup timing probe in the `SASE_TUI_PERF` style and a test
  asserting that the keystroke handler never awaits.
- Tests: the Tab/Enter/Escape state machine, stale async results dropped, selected-first
  ranking, and highlight spans matching the tokens.

### `completion-extras`: Empty state, doc peek, and history search

- Empty state, shown when the line is empty, reusing the popup widget:
  - `RECENT`: the last 5 history entries with a relative time and ✓/✗ exit.
  - `FOR <selection>`: derived, not curated. It lists leaf commands whose first required
    positional's `kind` matches the selected entity's kind, ranked by history usage (max
    5).
  - Enter on a row inserts the line and does not run it.
  - The hint row: `N commands · type to search · ⇥ complete · ; Command Palette`.
- Doc peek: on terminals at least 140 columns wide, when the highlighted item is a
  subcommand or an option, show a right-hand card from `command_help` with the summary,
  usage, arguments, choices and defaults. On narrow terminals this collapses into the
  signature row.
- `ctrl+r`: fuzzy history search in the popup. It ranks with the same Rust fuzzy
  matcher, and Enter loads the line.
- Marked rows: when the active slot is variadic and the TUI has marked entities of that
  kind, show a first row `‹N marked›` that inserts all of them.
- Provider health: an empty or failed provider shows a subtle `⚠ <kind> unavailable`
  popup footer and never stalls typing.
- PNG goldens: the empty state with RECENT and FOR rows, the doc peek (wide fixture),
  and history search.
- Tests: the FOR-row derivation, the doc-peek width threshold, ctrl+r ranking, and
  marked-row insertion.

### `run-policies`: Run policies, confirmation-aware blocks, and built-ins

- At submit time, read `LineContext.run_policy`:
  - `foreground`: `with app.suspend(): subprocess.run(argv, cwd=…, env=…)` in the real
    terminal, following `modals/xprompt_select_modal.py`. Then add a
    `↗ ran in terminal · exit N` block and record history. No proc is created.
  - `deny`: add a `⊘ not run` block containing the note, and record nothing.
- Confirmation-aware blocks: when `confirms` is true, `-y/--yes` was absent, and the
  exit code is non-zero, render `⊘ declined` with the command's own output. `R` reruns
  with `-y` appended **visibly**, as a new block. The pre-run signature already shows
  `asks to confirm · -y`.
- Built-ins, in `builtins.py`. They are recognized by the first token, run instantly,
  and render `›` blocks with no proc:
  - `cd <path|+project|->`, which pins, unpins and completes dirs and projects
  - `clear`
  - `help [command…]`, rendered from `command_help`
  - `history [query]`.

  A guard test asserts that built-in names never collide with top-level spec commands.
  `<cmd> -h` still runs argparse's real help as a proc.

- PNG goldens: a declined block with the `R` hint, a denied block, a foreground `↗`
  block, and `help bead close`.
- Tests: policy routing for every predicate (stub suspend and subprocess), the declined
  → `R` flow, each built-in, and cd pin/unpin effects on the chip and on submission cwd.

### `flip-and-land`: Flip `:` and `;`, remove the flag, and land

- Keys: `open_command_palette: "semicolon"` and `open_command_line: "colon"`. `:` in the
  Admin Center Config pane stays jump-to-path. Remove the palette's `":"` display alias.
- Palette side:
  - `:` typed into an **empty** palette filter hops to the Command Line.
  - A fallback row, "Run `sase <query>` in Command Line", opens the Command Line
    pre-filled (not run) when the filter has no strong match.
- One-time tip: the first time the Command Line opens after the flip, the hint row says
  "Command Palette moved to `;` (type `;` here to jump)". Record it with a `sase_home`
  marker, as in `ace/tui/_keymap_unification_notice.py`.
- Remove the flag:
  - Delete the Off branch and make the On branch unconditional.
  - Remove the `FeatureFlag.ace_command_line` registry entry.
  - Close the flag bead in the same change.
  - Keep one test per former flag state only where the behavior still exists.
- Touchpoints:
  - `keymaps/app_keymaps.py`, `keymaps/metadata.py`,
    `commands/_app_metadata_display.py`, `_APP_COMMAND_META`, `actions/artifacts.py`
    `NON_PRS_ARTIFACT_ACTIONS`, and the `commands/_availability_artifacts.py` allowlist.
  - `help_modal/{agents,patches,axe,patches_artifact}_bindings.py`,
    `widgets/tab_quickstart.py`, `widgets/agent_onboarding.py`, the palette hint's
    `key::` example, and the `action_open_command_palette` docstring ("bound to `:`").
  - `docs/ace.md`: the global key table, the Command Palette section, a new **Command
    Line** section (keys, completion, policies, built-ins, procs and retention), and the
    custom-mode example that uses `prefix: ";"`.
  - Tests: `test_keymaps_{defaults_panels,validation,app_bindings}.py`,
    `test_command_catalog{,_guards}.py`, `test_command_palette_{modal,wiring,e2e}.py`,
    and `test_keymaps_display_help_key_display.py`.
- Regenerate and inspect the PNG goldens this affects (`agents_onboarding`,
  `changespecs_onboarding`, `help_panel`, `agents_fleet`, and every Command Line suite)
  with `just fix-tui-screenshots`.
- Take a live `sase screenshot` walkthrough: press `:`, type `bead show `, Tab, run,
  hide, reopen, and check Procs. This confirms the real-app experience and catches
  anything the fixtures miss.

## Verification (every phase)

- Run `just check` through `sase tool run check` in sase, and `sase tool run check` in
  sase-core for the phases that touch it. Run `just fix` first.
- Phases that change rendered TUI output also run targeted
  `just fix-tui-screenshots -- <selectors>`, through `/sase_monitor` when long, and
  inspect every golden creation and update.
- Phases that touch sase-core move `sase-core-revision.txt` in the same sase change that
  first calls the new binding.
- Every flag-gated behavior has both-state tests until flip-and-land removes the flag.
