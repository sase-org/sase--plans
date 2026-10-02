---
tier: epic
title:
  Make the SASE pager much faster with a virtualized body, a light cold path, and
  bounded memory
goal: "`sase pager`, `sase bead show`, `sase artifact read`, and every pager embedded in
  `sase tui` open and respond in time proportional to what is on screen rather than to
  document size, start without importing the ACE TUI stack, and release their memory
  when closed. Rendered output, keys, and navigation stay byte-for-byte identical, and
  the work adds no disk caches or unbounded memory.

  "
phases:
  - id: pager-bench
    title: Pager benchmark and baseline
    depends_on: []
    size: small
    description:
      "pager-bench: add a subprocess-isolated pager benchmark over a synthetic corpus
      (open, keys, search, memory, leak, cold import, CLI wall time), a fast smoke test,
      and recorded baseline numbers."
  - id: scan-leak-fixes
    title: Quadratic scans, span memoization, and the dismissed-view leak
    depends_on:
      - pager-bench
    size: medium
    description:
      "scan-leak-fixes: fix the theme-watcher leak that keeps every closed pager alive,
      make link scanning and window-label row lookups near-linear, memoize per-section
      target spans and live-pin digests, and stop trail entries from retaining search
      copies."
  - id: cold-path-diet
    title: Cold-path import and startup diet
    depends_on:
      - pager-bench
    size: medium
    description:
      "cold-path-diet: make `sase.pager` imports lazy, move pure helpers out of
      `sase.ace.tui` into leaf modules, defer heavy imports to use sites, skip
      interactive-only work in plain mode, and add import-weight regression tests."
  - id: inventory-memo
    title: Repo inventory and config-key memoization
    depends_on:
      - pager-bench
    size: small
    description:
      "inventory-memo: memoize `repo_config_cache_key` by config identity and add a
      scoped per-command `collect_repo_inventory` memo that bead and artifact pager
      entry points enter, so one command builds the inventory once."
  - id: body-line-model
    title: Textual-free virtual body line model with a parity oracle
    depends_on:
      - scan-leak-fixes
      - cold-path-diet
    size: medium
    description:
      "body-line-model: build the pure per-row layout and render model (line index,
      exact wrap counts, bisect row lookup, per-row gutter/label/mark rendering) and
      prove it row-for-row identical to the current composer through a reference oracle
      kept in tests."
  - id: virtual-body-widget
    title: Swap the Static body for a Line-API ScrollView
    depends_on:
      - body-line-model
    size: large
    description:
      "virtual-body-widget: replace the one-giant-Static body with a ScrollView that
      renders only visible rows from the line model through a bounded strip cache, split
      invalidation into paint versus layout, and keep every PNG golden unchanged."
  - id: virtual-search-overlay
    title: Viewport-proportional incremental search
    depends_on:
      - virtual-body-widget
    size: medium
    description:
      "virtual-search-overlay: add an optional match-painting host hook to
      `VimSearchController` so the pager renders search rows lazily instead of
      rebuilding a styled copy of the whole corpus on every keystroke; ACE hosts keep
      today's path."
  - id: perf-gates-docs
    title: Final measurements, regression gates, and docs
    depends_on:
      - inventory-memo
      - virtual-search-overlay
    size: small
    description:
      "perf-gates-docs: rerun the benchmark against the baseline, add the memory-ceiling
      and leak gates, document the performance model in the pager docs and perf runbook,
      and record proposed follow-ups."
proposed_by: bbugyi200.athena.0v9
create_time: 2026-10-02 08:37:41
status: wip
---

- **PROMPT:**
  [prompts/202610/pager_performance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/pager_performance.md)

# Plan: Make the SASE pager much faster

## Why this is slow today (measured 2026-10-02 on athena)

Every number below came from the current master. Headless numbers use Textual's
`run_test` at 120x40; the harness's own floor is about 5 ms CPU per key, and a trivial
Textual app mounts in about 50 ms.

| Scenario                                                       | Today                           |
| -------------------------------------------------------------- | ------------------------------- |
| `sase pager --plain <500-line file>` wall time                 | 2.4 s (`sase --version`: 0.5 s) |
| `sase pager --plain bead:<id>` wall time                       | 4.5 s, 385 MB RSS               |
| `sase pager <519-line source file>`, real terminal first paint | 4.0 s                           |
| Open 2,000-line file, headless mount to first paint            | 14 s (34 s in a real terminal)  |
| Open 20,000-line file                                          | 145 s, 5.7 GB peak RSS          |
| Open 100,000-line log (one path per line)                      | did not finish in 15 min        |
| `scan_links` on 5k / 20k link-dense lines                      | 0.8 s / 21 s (quadratic)        |
| Dismissed pagers collected after closing                       | 0 of 3 (all leak)               |

Root causes, ranked by impact:

1. **The whole document is one giant `Static` renderable.** `compose_body`
   (`src/sase/pager/_layout.py`) and `apply_gutter` (`src/sase/pager/_gutter.py`) build
   one Rich `Text` for the entire document, and `PagerBody(Static)` inside
   `PagerBodyScroll(VerticalScroll)` makes Textual render every row of it. A mount does
   two full composes and about six full Textual renders (cProfile: 486k
   `Text.__rich_console__` calls for 2,000 lines). Every label-prefix key, syntax
   publish, history-mark change, goto mark, window-label shift, resize, and search
   keystroke repeats that O(document) work. The `_layout.py` docstring assumes Textual's
   compositor virtualizes rows of a single big renderable. It does not: the Static
   renders and caches strips for every row.
2. **Quadratic link scanning and label placement.**
   - `scan_links` (`src/sase/pager/link_scan.py`) checks each span against every earlier
     span with a linear `_overlaps`.
   - `_byte_to_character_offsets` builds a dict entry per character (about 100 B/char,
     2.3 s for 6.5 MB).
   - `section_target_spans` (`src/sase/pager/document.py`) re-runs the Rust scan on
     every label-layer build, with no memo. Each link costs about 24 µs of wire
     conversion, so 100k links take 2.4 s per build.
   - Window-mode labels call `row_for_character_offset` (`src/sase/pager/_labels.py`)
     per occurrence, and it walks the text from offset 0 each time.
3. **Search rebuilds the world per keystroke.** `VimSearchController._render_overlay`
   (`src/sase/ace/tui/widgets/vim_search_controller.py`) asks the pager for
   `styled_search_base`, makes 3-4 full-document copies, styles every match, and paints
   the result into the Static, so Textual renders all rows again.
4. **Every dismissed pager leaks.** `PagerView.on_mount` (`src/sase/pager/view.py`)
   calls `self.watch(self.app, "theme", ...)`. Textual only prunes those watchers when
   the theme changes, so each closed view stays reachable from the App until then. That
   includes its document, up to 64 trail documents, the composed body, and the strip
   caches. In `sase tui`, every `V` metadata pager and every file-hint pager
   accumulates.
5. **The cold path imports the ACE TUI.**
   - `sase/pager/__init__.py` eagerly imports app, screen and labels.
   - `link_scan.py` imports `sase.ace.tui.widgets.prompt_panel._file_path_hints`, which
     runs the whole `prompt_panel` package init (1.0 s: agent models, gate_turn,
     notification_gates, xprompt, finalizers).
   - `_labels.py` imports `sase.ace.tui.actions.navigation.jump_hints` (the navigation
     package pulls in `actions.agents`).
   - `_resolve_artifact_refs.py` imports the eager `sase.artifact_cli` package.
   - `syntax_theme.py` imports `xprompt.highlight_theme` (0.55 s), and
     `_screen_actions_section.py` imports `_prompt_jump_target` (1.3 s).
   - Parser registration loads the merged config (`parser_pager.py` calls
     `markdown_print_width()`).
   - Together `import sase.main.pager_handler` costs 1.6 s.
6. **Repo inventory is rebuilt repeatedly.** One bead pager run calls
   `collect_repo_inventory` (`src/sase/repo_inventory.py`) three times (agent identity,
   agents remote, artifact context). `repo_config_cache_key`
   (`src/sase/_linked_repo_config.py`) re-freezes the merged config on every call, 293k
   `_freeze_config_value` calls per run. A plain file view still pays one inventory
   build just to compute `known_kinds`, which plain output never uses.

## Target architecture

The body becomes a **virtualized row model**. Layout is computed once per (document,
width, layout inputs) as cheap integer arrays. Only visible rows are styled and
rendered, on demand, through a bounded strip cache.

- **Line index per section:** logical line start offsets (compact `array`), built once
  per section object.
- **Layout per width:** wrapped row count per logical line, prefix sums, section offsets
  and rule rows, and removal-anchor rows. A row maps to (section, line, wrap index) by
  bisect.
  - Row counts must equal what Rich `Text.wrap(console, width, overflow="fold")` yields
    today for the displayed line, including tab expansion and label capsules (`[hint]`,
    icon, NBSP, and the dangling ` (missing)` suffix).
  - Fast path: an ASCII line with no tab and no label that fits the width is one row.
    Every other line goes through Rich's own `divide_line` on its display string.
  - Measured cost: 100k short lines in 14 ms; 100k lines needing a wrap in 1.3 s, which
    is the accepted worst case.
- **Paint versus layout invalidation:**
  - Pending label prefix, syntax publish, goto/emphasis accent, change marks, rail
    styles and theme are paint-only. They bump an epoch and refresh visible rows.
  - Document swap, width, the label set (which lines carry capsules) and removal anchors
    are layout inputs. A label-set change relayouts only the lines whose labels changed.
- **Per-row rendering:** gutter cells, plus the wrapped piece of the line's styled
  `Text` (prepared syntax text or body text), sliced in O(log spans + spans in line),
  plus label capsules and accents. Section rules and removal-anchor rows render exactly
  as today. Rows reach Textual through the same Rich-to-Strip conversion Textual applies
  to Static content, so every PNG golden stays byte-identical.
- **Bounded caches:** a per-view strip LRU of about max(512, 4 x viewport) rows, and
  bounded per-section caches. All of them are cleared on document swap and unmount.

## Guardrails for every phase

These apply to all phases. Each phase restates the ones it is most likely to trip.

- **No visible behavior change.**
  - Rendering, keys, navigation, trail, history, split, search, goto, and labels behave
    and paint exactly as before.
  - Do not update any PNG golden unless a phase proves the old pixels were a
    pre-existing bug and says so explicitly in its notes. None is expected.
  - Existing pager tests must pass. Edit one only where it reached into a removed
    internal, and keep an equivalent assertion.
- **Memory stays bounded and released.**
  - Every new cache is in memory, per view (or per section object), size-bounded with
    LRU eviction, and dropped on document swap and unmount.
  - Process-global caches need a hard bound and an explicit reset hook in
    `tests/_conftest_runtime.py`.
  - Never retain a second full copy of a document's text beyond what exists today.
- **No new disk footprint.**
  - Add no on-disk caches, temp files, or always-on logs.
  - Any new perf logging is opt-in through an environment variable and reuses existing
    sinks (`tui_trace`).
- **Event-loop safety.**
  - Follow the `tui_perf.md` reference memory (read it with `/sase_memory_read`). In
    particular: no I/O or O(document) work in keystroke paths; use the existing
    pump-free task pattern with generation checks for off-loop work.
  - Cancel all work at unmount, and add no new long-lived threads.
- **Fail soft.**
  - A row that fails to render paints that row's plain text and never raises out of
    `render_line`.
  - CLI entry points keep their existing plain-output fallback.
- **Boundaries.**
  - The pager's layout and rendering are presentation code and stay in Python. Do not
    move wrap or layout logic into `sase-core`.
  - Rust link grammar is unchanged; only Python glue around it changes.
  - Do not add a feature flag: each phase lands complete, behind parity proof.
- **Verification.**
  - Read the `lint_and_test.md` and `symvision.md` reference memories before finishing.
  - Run `sase tool run check`.
  - Phases that change rendering also run the pager PNG suites in check mode
    (`just test-visual -- tests/pager/visual`), through `/sase_monitor` if they would
    outlast the turn.
  - Do not run `just check-full` unless the bead says so.
- **Measure, don't guess.** Each phase from `scan-leak-fixes` on reruns the relevant
  `pager-bench` cases and records before/after numbers in its bead notes. If a numeric
  target is missed, record why and propose a follow-up. Never trade away a guardrail to
  hit a number.
- **Follow-ups.** Epic phases record discovered work as `PROPOSED FOLLOW-UP:` notes on
  their own bead and do not create beads.

## Phase pager-bench: Pager benchmark and baseline

Add `tests/perf/bench_pager.py` (marked `slow`), a corpus generator
`tests/perf/_pager_bench_corpus.py`, and a non-slow smoke test that runs the smallest
case end to end so the harness cannot rot. The pattern is `test_bench_epic_launch_smoke`
in `tests/perf/README.md`.

**Corpus (deterministic, generated in a temp dir):**

| Name          | Contents                                               |
| ------------- | ------------------------------------------------------ |
| `code-sparse` | Python-like lines with a file-path link every 50 lines |
| `log-dense`   | One path link per line                                 |
| `wide`        | CJK, emoji and combining characters                    |
| `long-wrap`   | About 200-column lines that need wrapping              |
| `ansi`        | SGR-colored stdin-style text                           |
| `multi`       | Six sections, bead-like                                |
| `markdown`    | Within syntax caps, so the syntax overlay runs         |

The line-count ladder is 500, 2k, 20k and 100k, configurable with `--max-lines` and
`--cases`.

**Metrics.** Each case runs in a fresh subprocess with a per-case timeout, and reports
`TIMEOUT` rather than hanging; today's 20k and 100k cases take minutes or never finish.
Collect:

- Document build time.
- Mount to first paint: wall and CPU, headless `run_test` at 120x40.
- Time until the syntax overlay publishes, where applicable.
- Per-key CPU median and max for `j`, `k`, `ctrl+d`, `G`, `g`, one label-prefix key, a
  three-character search typed then `enter`, `n`, and one resize.
- Peak RSS from `resource.getrusage` in the child.
- A leak probe: push and pop a `PagerScreen` three times in a host `App`, run gc, and
  count live `PagerView` weakrefs.
- A trivial-Textual-app floor, measured in the same harness, so numbers are reported
  both raw and above the floor.

**Cold-path probes:**

- `python -X importtime -c "import sase.main.pager_handler"` and
  `"import sase.pager.screen"` cumulative times.
- `sase pager --plain <file>` wall time.
- `sase pager --plain bead:<id>` wall time, against a disposable `SASE_HOME` fixture
  bead store if one is available; otherwise skip with a clear message.
- A pty-driven real-terminal first paint for one file (spawn under a pty sized 120x40,
  wait for a marker token, send `q`).

**Output.** Print one table (case, metric, value, value above floor). Record today's
baseline in the bead notes. Do not commit baseline numbers: shared-host timing is noisy.
Add a short "Pager" section to `tests/perf/README.md` with the run commands. A `just`
recipe is optional; if you add one, mirror the existing `bench-*` recipes.

**Acceptance:** the bench runs end to end on master (timeouts reported, not hung), and
the smoke test passes under `sase tool run check`.

## Phase scan-leak-fixes: Quadratic scans, span memoization, and the dismissed-view leak

Each item ships with focused tests. Results must be identical to today; property-style
tests compare old and new implementations on random inputs with fixed seeds.

1. **Fix the dismissed-view leak.** In `PagerView.on_mount`, replace
   `self.watch(self.app, "theme", ...)` with
   `self.app.theme_changed_signal.subscribe(self, ...)`, and call
   `theme_changed_signal.unsubscribe(self)` in `on_unmount` next to
   `cancel_pump_free_tasks`. Textual's `Signal` keeps callback lists as
   `WeakKeyDictionary` values, so the bound method would otherwise pin the key. Keep the
   existing `_on_app_theme_changed` behavior.
   - Test: push and pop a `PagerScreen` three times, including once after opening and
     closing a split, then gc; zero `PagerView`s may survive.
   - Also verify closed split panes and `SasePager` app exit.
2. **Make `scan_links` near-linear.**
   - Replace the linear `_overlaps` scan with an ordered-interval check (sorted starts
     plus bisect), keeping exact first-wins precedence: Rust links in Rust order, then
     bare tokens.
   - Replace `_byte_to_character_offsets` with an ASCII fast path (identity) and, for
     non-ASCII text, a conversion that encodes once and maps only the byte offsets
     actually needed.
3. **Memoize target spans.** Memoize `section_target_spans(section, origin)` per section
   object, keyed by the effective origin. A private
   `init=False, compare=False, repr=False` slot on the frozen `PagerSection` set lazily
   is acceptable. Dangling status is not part of the memo.
4. **Window-mode label rows without O(offset) walks.** Keep `row_for_character_offset`'s
   exact estimate semantics: character fold, cell widths, newline resets. Compute it
   from a per-(section, width) prefix of per-line estimated rows plus an in-line walk,
   so each occurrence costs O(log lines + line length). A property test against the old
   function proves identical results. Apply the same line-start approach to
   `_estimated_line_rows` in `_layout.py`.
5. **Bisect instead of scanning.** Convert the linear scans in `reading_anchor_at_row`
   and `current_section_index` (`_layout.py`) to bisect, with identical results.
6. **Memoize the live-pin digest.** `_syntax_key_for_section` (`_screen_syntax.py`)
   hashes the full text of a live history pin on every scroll and compose. Memoize the
   digest per section object.
7. **Trail entries stop pinning search copies.**
   - Trail entries (`_screen_trail.py`, `trail.py`) and `VimSearchController.exit`
     should no longer keep the search corpus and `line_starts` after search ends.
   - Keep the remembered search (query/direction) and its restore behavior exactly; the
     corpus is rebuilt on the next search start, as it already is.
   - Prove it with the existing trail and search tests plus one new retention test.

**Targets** (rerun the bench):

- `scan_links` on 20k link-dense lines: under 1.5 s (was 21 s), dominated by Rust
  conversion.
- The leak probe reports 0 live views.
- Window-mode label build on `log-dense` 20k: no longer quadratic.

## Phase cold-path-diet: Cold-path import and startup diet

1. **Lazy package exports.** Turn `src/sase/pager/__init__.py` into lazy PEP 562
   exports. Reuse `src/sase/_lazy_exports.py`; the public names and `__all__` are
   unchanged.
2. **Move pure helpers into leaf modules** that import neither `textual` nor
   `sase.ace.tui`. Name them in the style of the existing `sase/pager` or neutral
   top-level helpers. The original modules re-import from the leaf, so ACE callers and
   behavior are unchanged.
   - From `sase/ace/tui/widgets/prompt_panel/_file_path_hints.py`:
     `iter_pager_file_path_matches` and the regexes it needs.
   - From `_hint_caps.py`: `HintContentBudget` and `bound_hint_content`, plus their pure
     dependencies from `lazy_syntax.py` (`PLAIN_RENDER_MAX_*`,
     `truncate_plain_content`).
   - From `sase/ace/tui/actions/navigation/jump_hints.py`: `JUMP_HINT_CHARS`,
     `PAGER_RESERVED_JUMP_COMMAND_KEYS`, `build_jump_hint_maps`, `match_jump_hint`,
     `normalize_jump_key` and `JumpHintMatchOutcome`. Its type-alias-only imports
     (`PanelKey`, `ArtifactEntryTarget`) move under `TYPE_CHECKING` or to
     `sase.core.artifact_entry_target`.
   - The `_artifact_tab_model` accent and icon dicts, if still needed after the above.
3. **Import from the defining module, or at the use site.**
   - `scan_artifact_ref_document` from `sase.artifact_ref_operations`, not the
     `sase.artifact_refs` facade.
   - `ResolvedArtifactReference` under `TYPE_CHECKING` in `owner.py` and `landings.py`.
   - Function-level import of `artifact_cli.references` in `_resolve_artifact_refs.py`,
     or make `sase/artifact_cli/__init__.py` lazy. Choose the smaller change that keeps
     `sase artifact` behavior identical.
   - `_resolve_skills` imported inside the function in `resolve.py`.
   - `sdd.files` deferred in `link_context.py`.
   - Light imports of `ace.tui.graphics` types (`_viewer_types`).
4. **Interactive-only heavy imports.** `syntax_theme.py` → `xprompt.highlight_theme` and
   `_screen_actions_section.py` → `_prompt_jump_target` move to their use sites, or
   their pure pieces move to leaves.
5. **`sase.bead.cli_show_batch`** must reach only light pager modules at import time
   (this follows from steps 1-3).
6. **Parser.**
   - Stop `register_pager_parser` from loading the merged config at parser build. A lazy
     default is fine only if `sase pager --help` text and parsed values stay identical;
     `tests/main/test_pager_command.py` asserts them. If that is not achievable, leave
     it and record why.
   - Import `wrap_width` without pulling the eager `sase.vcs_log` package.
7. **Plain mode skips interactive-only work.** When `handle_pager_command` will write
   plain output (`--plain`, non-TTY stdout, or no `/dev/tty`), a local file input must
   not trigger `collect_repo_inventory` through `known_kinds_from_link_context`; plain
   output never reads `known_kinds`.
   - Decide plain mode before building the document and thread an explicit "no link
     painting" option through resolution.
   - Interactive behavior is unchanged.
   - A test counts inventory calls for a plain file run.
8. **Import-weight regression tests**, using the subprocess pattern from
   `tests/memory/test_history_import_cost.py`:
   - `import sase.pager.document` and `import sase.main.pager_handler` must not load
     `textual`, `sase.ace.tui.widgets.prompt_panel`, `sase.ace.tui.actions`,
     `sase.xprompt`, `sase.notification_gates`, `sase.finalizers` or
     `sase.artifact_cli.doctor`.
   - `import sase.pager.screen` must not load `sase.ace.tui.widgets.prompt_panel`,
     `sase.ace.tui.actions.agents`, `sase.notification_gates` or `sase.finalizers`.
   - `import sase.bead.cli_show_batch` must not load `textual`.

Watch for circular imports when moving helpers. Keep moved symbols public only where a
non-test consumer exists (`symvision.md`).

**Targets:**

- Cumulative `import sase.main.pager_handler` at or under 0.6 s (was 1.6 s).
- `sase pager --plain <file>` at or under 1.1 s (was 2.4 s).
- Real-terminal first paint of a 500-line source file at or under 2.5 s before the body
  rewrite lands.

## Phase inventory-memo: Repo inventory and config-key memoization

1. **Memoize `repo_config_cache_key`** (`src/sase/_linked_repo_config.py`) by config
   object identity.
   - Use a tiny bounded map that holds a strong reference to the config so ids cannot be
     recycled.
   - Clear it in `reset_linked_repo_config_caches`.
   - Cover `_sdd_sidecar_repo_dirnames` (`src/sase/_linked_repo_paths.py`), which
     re-freezes per sidecar.
2. **Add an opt-in, context-scoped memo for `collect_repo_inventory`.** For example, a
   `repo_inventory_session()` context manager backed by a `ContextVar`.
   - Key: (resolved root, project, `include_disabled` and the other arguments,
     `current_config_token()`, stat signature of the projects root).
   - Outside a session, behavior is exactly today's. The long-lived TUI never holds a
     session open across user actions.
   - Enter the session around a single command in `handle_pager_command`, the
     `sase bead show` handler, and the `sase artifact read` handler.
   - Add a reset hook to `tests/_conftest_runtime.py`.
3. **Load the agent-names registry once.** Wrap the bead document build in the existing
   `name_registry_load_session` (`src/sase/agent/names/_registry.py`), so the registry
   JSON loads once per command instead of twice.
4. **Tests:**
   - The memo hits within a session.
   - Nothing is memoized outside one.
   - A config-token change or projects-root change misses.
   - Identical inventory results with and without the memo.

**Target:** `sase pager --plain bead:<id>` at or under 2.6 s (was 4.5 s; combined with
cold-path-diet). Record the remaining split; Rust bead detail and the artifact-link
neighborhood are out of scope.

## Phase body-line-model: Textual-free virtual body line model with a parity oracle

Build the pure model that later phases mount. Add no Textual imports, and do not change
the widget yet.

1. **Reference oracle.** Copy the current `compose_body` / `_paint_section_body` /
   `apply_gutter` pipeline (with `section_rule`, label rendering and mark rendering)
   into `tests/pager/_reference_compose.py`, unchanged, as a frozen oracle. It lives
   under `tests/`, so Symvision ignores it. Make it self-contained: copy every src
   helper it uses that a later phase might delete or change (gutter, wrap, label capsule
   rendering), so later refactors cannot silently move the oracle.
2. **New modules** under `src/sase/pager/`, each kept under about 600 lines for
   `toobig`. Suggested split: line index, layout, row render.
   - **Section line index:** logical-line starts in a compact `array`, and the
     `logical_line_count` semantics (phantom trailing newline dropped; an empty body has
     zero lines but one visual row). Built once per section object and memoized like the
     target-span memo.
   - **Styled line extraction:** given the section's source `Text` (prepared syntax text
     or body text), return line _i_'s styled `Text` in O(log spans + spans touching the
     line). Build a span index once per source `Text`; `Text.from_ansi` spans never
     cross newlines, but syntax spans may, so handle both.
   - **Layout** for (document, paint width, label layer, removal anchors, gutter digit
     width):
     - Per-line row counts using exactly `Text.wrap(..., overflow="fold")` semantics on
       the display line, with the ASCII fast path and Rich `divide_line` otherwise.
     - Prefix sums and section offsets, matching `_section_row_offsets`.
     - `total_height`, `section_line_counts`, and `section_line_rows`, which may be a
       lazy `Sequence[int]`.
     - `locate(row)` returning one of: rule row, removal-anchor row, or (section, line,
       wrap index).
     - An incremental `relabel(new_layer)` that recomputes only lines whose label
       content changed.
     - Expose the same field names `ComposedBody` has today, so goto, trail, history,
       diff and reading-anchor callers keep working.
   - **Row renderer:** `render_row(row, paint_state)` returns the row's Rich `Text`. It
     must equal the oracle row: gutter number, rails, emphasis range, change marks,
     removal anchors, rail styles, label capsules with pending-prefix styling, dangling
     suffix, target accents, and the section rule. Non-`Text` section renderables (rare)
     are pre-rendered once per width into row texts, exactly as `_paint_section_body`'s
     fallback does.
3. **Parity tests** (`tests/pager/test_body_model_parity.py`). For a corpus at widths
   such as 20, 37, 80 and 118, render every row through both paths with a fixed
   truecolor `Console` and assert identical segments (text and style) per row, plus
   identical `section_offsets`, `total_height` and `section_line_rows`. The corpus:
   - plain, ANSI, syntax-prepared, wide/emoji/combining, tabs, long unbreakable tokens,
     trailing spaces, empty and newline-only sections
   - multi-section documents
   - document-mode and window-mode labels, with and without a pending prefix, and
     dangling targets
   - goto emphasis ranges across wrapped rows
   - change marks, removal anchors at 0 and mid-section, rail styles
   - non-`Text` renderables

   Use fixed-seed randomized documents in addition to hand-written cases.

4. **Work-counter hooks.** Add cheap counters (lines laid out, rows rendered) that later
   phases' tests assert on.
5. **Symvision.** The new public API has no non-test consumer until
   `virtual-body-widget`. Add `--epic-symbol <this epic's bead id>(<symbol>)` entries in
   the `Justfile` Symvision invocation, per `symvision.md`; the next phase removes them.

**Targets:**

- Layout for 100k short lines under 100 ms.
- Rendering any single row costs O(that line), independent of document size. Prove it
  with a counter test on a 100k-line document.

## Phase virtual-body-widget: Swap the Static body for a Line-API ScrollView

This is the riskiest phase. Plan it first; it is sized `large`. Keep every PNG golden
unchanged.

1. **The widget.**
   - Replace `PagerBodyScroll(VerticalScroll)` plus `PagerBody(Static)`
     (`src/sase/pager/_screen_widgets.py`, composed in `view.py`) with one
     `textual.scroll_view.ScrollView` subclass. It keeps the id `pager-body-scroll` and
     the class name `PagerBodyScroll` if practical.
   - It implements `render_line(y)` for row `scroll_y + y` from the line model, and sets
     `virtual_size` from the layout.
   - It reproduces today's CSS (padding `0 1`, scrollbar behavior, widget base
     style/background) so pixels match.
   - It caches row `Strip`s in a bounded `textual.cache.LRUCache` keyed by (row, paint
     epoch, paint width), and crops and styles per Textual's Line-API conventions.
   - Keep `scroll_relative`, `scroll_to`, `max_scroll_y`, `scrollable_content_region`,
     focus, mouse wheel and scrollbar dragging working as today.
2. **Invalidation.** Replace the `self._body_width = None; self._ensure_body()` idiom at
   every call site with an explicit API: `_invalidate_body_paint()` (epoch bump plus
   `refresh()`) and `_invalidate_body_layout()` (re-layout, preserving the reading
   anchor rules that `_ensure_body` has today: a width change keeps the top logical
   line; same-width relayouts never move the scroll). Call sites include:
   - `_screen_body.py`, `view.py` (`set_pane_role`, split seed)
   - `_screen_syntax.py` (publish and theme)
   - `_screen_actions_labels.py` (prefix keys)
   - `_screen_goto.py`
   - `_screen_history_swap.py`, `_screen_diff_view.py`, `_screen_diff_folds.py`,
     `_screen_diff_navigate.py`
   - `_screen_trail.py`, `_screen_actions_resolve.py` (dangling refresh)
   - `_screen_time_band.py`, `_screen_split.py`
3. **Specific paths.**
   - Syntax publish and pending label prefix become paint-only.
   - Window-scoped label refresh uses the model's incremental relabel and layout rows
     for occurrence placement.
   - `PagerBodyScroll.on_resize` relayouts only when the paint width really changed.
4. **Search in this phase.** Keep `VimSearchController` unchanged. The pager host's
   `vim_search_paint_overlay(content)` switches the widget to an overlay row source that
   splits the given `Text` into lines lazily and renders only visible rows.
   `vim_search_hide_overlay` switches back. Preserve today's crop behavior exactly: the
   overlay never scrolls horizontally, because the old container is `overflow-x` hidden
   with a full-width Static.
5. **Remove the dead path.** Delete `PagerBody(Static)`, `ComposedBody.renderable`, and
   src-side `compose_body` / `apply_gutter` pieces that no longer have a non-test
   consumer (the oracle stays in tests). Drop the previous phase's `--epic-symbol`
   entries.
6. **Tests.**
   - Update only tests that read removed internals (for example
     `screen._body.renderable.renderables[0]` in `test_app_navigation.py`,
     `test_app_goto.py`, `test_goto.py` and `test_rendered_link_navigation.py`), using a
     row-text accessor with equivalent assertions. Update the `body_scroll` helper's
     type in `tests/pager/_app_helpers.py`.
   - Add headless parity tests that compare visible rows (text and style) against the
     oracle at several scroll positions, after resize, after a label prefix, and after a
     syntax publish.
   - Add counter tests at 120x40. Opening a 50k-line document renders at most 2 x
     viewport rows before first paint. `j` renders only newly exposed rows. A label
     prefix key relayouts 0 lines and re-renders at most a viewport. A syntax publish
     relayouts 0 lines.
   - Add a test that a row-render exception paints the plain row instead of crashing.
7. **Visual check.** Run the pager PNG suites in check mode
   (`just test-visual -- tests/pager/visual`) with zero golden changes, and spot-check
   ACE flows that embed the pager (Agents-tab `V`, file hints).

**Targets** (bench, headless, above the floor):

| Case                                     | Target               | Baseline                                        |
| ---------------------------------------- | -------------------- | ----------------------------------------------- |
| 2k-line open                             | ≤ 250 ms             | 14 s                                            |
| 20k-line open                            | ≤ 600 ms             | 145 s                                           |
| 100k `code-sparse`                       | ≤ 2 s                | did not finish                                  |
| 100k `log-dense`                         | ≤ 5 s                | did not finish (Rust link conversion dominates) |
| `j` / `k` / `ctrl+d` / `G` per key       | ≤ 5 ms at every size |                                                 |
| Label-prefix key at 20k lines            | ≤ 10 ms              |                                                 |
| Peak RSS opening 20k lines               | ≤ 350 MB             | 5.7 GB                                          |
| Real-terminal first paint, 500-line file | ≤ 1.3 s              | 4.0 s                                           |

## Phase virtual-search-overlay: Viewport-proportional incremental search

1. **Controller hook.** Add an optional host capability to `VimSearchController`, for
   example `vim_search_paint_matches(match_spans, current_index)`, detected with
   `getattr` the same way `vim_search_styled_base` is.
   - When present, `_render_overlay` must not build or copy a full styled `Text`; it
     hands the host the sorted match spans and the current index.
   - Hosts without it, such as the ACE agent-metadata search
     (`src/sase/ace/tui/actions/agents/_metadata_search.py`), keep exactly today's path.
   - Controller unit tests cover both paths.
2. **`n` / `N`.** Reuse the cached match spans while the query and corpus are unchanged,
   instead of re-running the regex on every repeat.
3. **Pager implementation.** The search row source renders corpus line _y_ from a
   per-line styled base: the prepared syntax line or body line, target accent styles and
   dangling styling for spans on that line (the `style_target_accents` semantics), match
   styles for matches intersecting the line (bisect into sorted spans), and the
   current-match style.
   - Cache base lines per search session in a bounded LRU.
   - `refresh_styled_base` (late syntax or theme change) invalidates them.
   - The corpus and `line_starts` are built once per search start, and released at exit.
4. **Tests.**
   - Parity: visible search rows equal the old controller path for several queries,
     including no-match, wrap-around, and `n`/`N`.
   - Search PNG goldens are unchanged.
   - Counter tests: a typed character re-renders at most a viewport and builds no
     full-corpus `Text`.

**Target:** a search keystroke on a 20k-line document at or under 60 ms above the floor,
dominated by the regex.

## Phase perf-gates-docs: Final measurements, regression gates, and docs

1. Rerun the full bench and record a before/after table against the `pager-bench`
   baseline in the bead notes.
2. Make the leak probe and a memory-ceiling check regular (non-slow) tests if they run
   in a few seconds. The ceiling uses `tracemalloc` peak while opening a 20k-line
   document headless, with a generous bound. Otherwise keep them in the slow bench with
   asserted budgets.
3. Docs:
   - `docs/pager.md`: a short "Performance" section on virtualized rendering, what
     scales with document size (one-time layout, link scan) and what does not
     (scrolling, labels, search typing), and the unchanged syntax caps.
   - `docs/perf_runbook.md`: the pager bench recipe.
   - `tests/perf/README.md`: final pointers.
4. Record these `PROPOSED FOLLOW-UP:` notes on this phase's bead:
   - A `memory` task to add a `tui_perf.md` rule: "never paint a whole document into one
     `Static`; Textual renders and caches every row — use a Line-API `ScrollView`", and
     to correct the "one big renderable is virtualized" misconception.
   - ACE panels the old `_layout.py` docstring named as using the same
     one-big-renderable shape (`AgentFilePanel`, `AgentPromptPanel`) likely have the
     same scaling problem.
   - ACE deck widgets that call `theme_changed_signal.subscribe` without unsubscribing
     (`block_rail.py`, `panel_chrome.py`) may leak the same way.
   - Per-link wire conversion cost in the Rust document-link scan (about 24 µs/link),
     which would belong to `sase-core`.
   - `~/.sase/perf/tui_trace.jsonl` has no rotation when tracing is enabled.
   - Anything else measured but out of scope.

## Non-goals

- Changing any visible output, key, or behavior, including the search overlay's crop
  behavior and the unused `--wrap` option.
- A separate no-argparse plain fast path for `sase pager --plain`.
- Capping search match counts.
- Moving layout to Rust.
- Pruning `_history_states`; it is bounded by sections visited and is rediscovered only
  by design.
- Speeding up Rust bead detail or the artifact-link neighborhood.
