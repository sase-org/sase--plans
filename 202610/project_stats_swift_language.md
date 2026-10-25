---
tier: tale
title: Recognize Swift and other missing languages in Projects stats
goal:
  Swift (and other common languages) count as code in sase stats project, and uncounted
  files can no longer vanish silently.
size: medium
decisions:
  unclassified_note:
    ask:
      Flag large buckets of unrecognized, uncounted files so the next missing language
      is visible?
    choices:
      add:
        Show a dim "not counted" card row and table note when a repo's unrecognized
        lines pass a threshold
      skip: Fix recognition only; unrecognized files stay invisible, as today
    default: add
    why:
      The bug hid 165k lines without a trace; on today's data the threshold fires only
      for it
    answer: add
proposed_by: bbugyi200.athena.147.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-25 15:22:10
status: wip
---

# Recognize Swift (and the other missing languages) in Projects stats

## Root cause (diagnosed and verified)

`sase stats project -p bob-cli` shows no Swift because **sase-core does not recognize
`.swift` as a language**, so every Swift file is classified as role `other`. Role
`other` counts toward no code total and no language row, and it is shown nowhere.

The chain, in the linked `sase-core` repo (`sase repo open sase-core`):

1. `crates/sase_core/src/project_stats/classify.rs::detected_language` asks the shared
   `crate::source_language::resolve_source_language` for the file's language.
2. `crates/sase_core/src/source_language.rs::SUFFIX_MAP` is the pager's
   highlighting-oriented table (~45 suffixes). It has no `swift` entry, so the result is
   `language: None`, reason `unknown`. There is no shebang either.
3. `classify_file_evidenced` then returns `Included { role: Other, language: None }`
   (the final fallthrough). Code lines are source + tests only, so the file vanishes
   from `CODE`, from `by lang`, and even from the `?  no language detected` row. That
   row only covers source/test lines with no language, and the classifier never produces
   such lines.

Evidence from live data (`sase stats project -p bob-cli -j`, core 0.42.5):

| repo            | source+tests lines |      role `other` lines | languages reported |
| --------------- | -----------------: | ----------------------: | ------------------ |
| bob-mac-capture |              6,569 | **164,788** (356 files) | bash, python, sh   |
| bob-cli         |            510,534 |                     466 | rust, python, …    |
| bob-plugins     |            454,390 |                       3 | javascript, …      |

bob-mac-capture's default branch tracks **353 `.swift` files (≈178k raw lines)**:
`Sources/{BobMacCapture,CaptureCore,RefsCore,…}` and `Tests/*Tests`. That is nearly the
whole `other` bucket. Across every other code repo in every project, `other` is 0–5% of
code lines (justfiles, `.gitignore`, one `.vim` file in sase-nvim, …). So this is a
clean outlier, and the card's "3 of 3 code repos" made the report look complete.

The same gap applies more widely. The classifier's own shebang table already recognizes
`perl`, `php`, `fish`, `awk`, `r`, and `powershell`, but `foo.pl`, `foo.php`,
`foo.fish`, `foo.awk`, `foo.R`, and `foo.ps1` are all `other`. Kotlin, Objective-C, C#,
Vim script, SCSS, Terraform, `.mts`/`.cts` TypeScript, and others are missing too.
sase's renderer already has labels for several of these ids, but they are reachable only
by shebang.

**Cache trap:** the incremental code-snapshot cache stores every path's `classification`
(`project_stats/snapshot/cache.rs::CachedPathRow`). Unchanged `.swift` blobs would stay
`other` forever unless `PROJECT_STATS_CLASSIFIER_VERSION`
(`project_stats/wire/response.rs`, currently `2`) is bumped. That constant feeds
`CodeSnapshotCacheEntry::matches_repo`, the history `cache_key`, the code source tokens,
and the response fingerprint, so one bump invalidates all of them. Old and new installs
key their entries differently, so they do not thrash each other. tokei already counts
`.swift` by extension, so the SLOC counter needs no change and `COUNTER_VERSION` stays.

## Design

- **One shared language identity.** Keep `source_language::SUFFIX_MAP` as the single
  table for the pager and Projects stats; do not fork a stats-only table. Every new id
  is a valid Pygments lexer alias (verified against Pygments 2.19.2). The pager passes
  unknown ids straight to `get_lexer_by_name`, so it gains Swift/Kotlin/… highlighting
  for free.
- **Guard the class, not just the instance.** Add invariant tests:
  - every language reachable by shebang is also reachable by suffix;
  - every suffix the classifier can count as source resolves to a real tokei language,
    not `Text`.
- **Bump the classifier version once** so the cache trap cannot hide the fix.
- **Make the failure visible** (decision `unclassified_note`).

## Part A: sase-core (linked repo; open with `sase repo open sase-core`)

Read its `AGENTS.md` first. Never run bare `cargo`; use `just fast`,
`just test -p sase_core <filter>`, and finally `sase tool run check`.

### A1. `crates/sase_core/src/source_language.rs`: extend `SUFFIX_MAP`

Keep the table alphabetized by suffix. Add exactly these (suffix → id):

| family          | entries                                                                                                                                             |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Apple / mobile  | `swift`→`swift`, `m`→`objective-c`, `mm`→`objective-c++`, `kt`/`kts`→`kotlin`, `dart`→`dart`                                                        |
| JVM / .NET      | `scala`→`scala`, `groovy`→`groovy`, `cs`→`csharp`                                                                                                   |
| C++ gaps        | `cxx`/`hh`/`hxx`→`cpp`                                                                                                                              |
| TypeScript gaps | `mts`/`cts`→`typescript`                                                                                                                            |
| scripting       | `php`→`php`, `pl`/`pm`→`perl`, `ps1`/`psm1`→`powershell`, `fish`→`fish`, `awk`→`awk`, `r`→`r`, `jl`→`julia`, `bat`→`batch`, `rake`/`gemspec`→`ruby` |
| functional      | `ex`/`exs`→`elixir`, `erl`/`hrl`→`erlang`, `hs`→`haskell`, `ml`/`mli`→`ocaml`, `clj`/`cljs`/`cljc`→`clojure`, `elm`→`elm`                           |
| systems         | `zig`→`zig`, `nim`→`nim`                                                                                                                            |
| web             | `scss`→`scss`, `sass`→`sass`, `less`→`less`, `vue`→`vue`, `graphql`/`gql`→`graphql`                                                                 |
| infra / tooling | `tf`→`terraform`, `nix`→`nix`, `proto`→`protobuf`, `cmake`→`cmake`, `vim`→`vim`                                                                     |

Deliberately excluded because the suffix is ambiguous or has no Pygments lexer: `.d`,
`.v`, `.fs`, `.sc`, `.svelte`, `.cmd`, `.svg`.

Extend `tests::mapping_table_covers_declared_families` with one case per new suffix,
including an upper-case `Foo.R`.

### A2. `crates/sase_core/src/project_stats/classify.rs`

- **Apple project metadata is config, not "other".** Add `plist`, `entitlements`,
  `xcconfig`, `pbxproj`, `xcscheme`, `xcworkspacedata`, `xcstrings`, `strings`,
  `storyboard`, and `xib` to `is_config_data_path`'s extension list. Add
  `package.resolved` (the SwiftPM lockfile; components are already lower-cased) to its
  basename list.
- **Xcode test targets.** In `is_test_path`, also treat file names ending in
  `tests.swift` or `test.swift` (lower-cased) as tests. Xcode test targets often live in
  `AppTests/` with no `Tests/` parent. SwiftPM's `Tests/` dir is already covered.
- **Shebang table.** Refactor the interpreter→language `match` in `shebang_language`
  into a `const SHEBANG_INTERPRETERS: &[(&str, &str)]` table. Keep the python/pypy
  special case as code, and add `("swift", "swift")`. Behavior is otherwise unchanged.
- Expose `SUFFIX_MAP`'s language ids to the crate through a small `pub(crate)` accessor
  (for example `source_language::suffix_language_ids()`) for the invariant tests. Do not
  widen the public API.

### A3. Bump the classifier version

`crates/sase_core/src/project_stats/wire/response.rs`:
`PROJECT_STATS_CLASSIFIER_VERSION: u32 = 3`. Search both repos for a hard-coded `2`
paired with `classifier`. Today there is none; sase's Python fixtures use `1` as an
opaque value and need no change.

### A4. Tests (sase-core)

- `classify_tests.rs`:
  - a Swift package case: `Sources/App/Main.swift` → Source `swift`;
    `Tests/AppTests/MainTests.swift` → Tests `swift`; `AppTests/LoginTests.swift` →
    Tests `swift`; `Package.swift` → Source `swift`; `Package.resolved` and
    `Resources/Info.plist` → ConfigData.
  - an extensionless `#!/usr/bin/env swift` script → Source `swift`.
  - **invariant:** every language in `SHEBANG_INTERPRETERS` (plus `python`) appears in
    `suffix_language_ids()`.
- `counter_tests.rs`, **invariant:** for every `SUFFIX_MAP` entry whose sample filename
  `src/sample.<suffix>` classifies as `Role::Source`, `code_language(path, Some(id))` is
  not `tokei::LanguageType::Text`. Keep an explicit, commented allowlist for ids tokei
  cannot count; expect only `diff`, which already counts 0 SLOC today.
- `snapshot/tests.rs`: an end-to-end snapshot of a committed repo with
  `Sources/App/Main.swift` (code plus a `//` comment) and
  `Tests/AppTests/MainTests.swift`. Assert `languages` contains `swift` with SLOC lines
  (comment excluded), role partition holds (`assert_partition`), and `other` is 0.
- Update any test asserting the classifier version value or role counts that the new
  config-data extensions change.

Commit subject (Conventional Commits, not breaking):
`feat(project-stats): recognize Swift and other mainstream languages`.

## Part B: sase (this repo)

### B1. Labels and colors: `src/sase/stats/render/languages.py`

- Add `_LANGUAGE_LABELS` (keep the dict's existing order style):
  - swift→Swift, kotlin→Kotlin, objective-c→Objective-C, objective-c++→Objective-C++,
    dart→Dart, scala→Scala, groovy→Groovy, csharp→C#
  - elixir→Elixir, erlang→Erlang, haskell→Haskell, ocaml→OCaml, clojure→Clojure, elm→Elm
  - zig→Zig, nim→Nim, julia→Julia, batch→Batchfile
  - scss→SCSS, sass→Sass, less→Less, vue→Vue, graphql→GraphQL
  - terraform→Terraform, nix→Nix, protobuf→Protocol Buffers, cmake→CMake, vim→Vim script

  perl, php, fish, awk, r, and powershell already exist.

- Colors: the 8 categorical slots are full, and the dataviz rules forbid a generated 9th
  hue. So Swift and every other new id keep the neutral `#8a8984`, by design. One
  exception follows the shell-family precedent: map `scss`, `sass`, and `less` to CSS's
  existing violet `#9085e9`. That is not a new slot, so no re-validation is needed.

### B2. Unclassified-files guard (only if `unclassified_note = add`)

> [!decision] unclassified_note = add

- `src/sase/stats/project/languages.py`:
  - Add a frozen dataclass `UnclassifiedRepo` (project_key, project_label, repo_id,
    repo_name, lines, files) and a
    `LanguageBreakdown.unclassified: tuple[UnclassifiedRepo, ...]` field. Default it to
    `()` so the early-return constructions stay valid.
  - Fill it from each measured code repo's `snapshot.role_counts` row `other`. Include a
    repo when `other_lines >= _UNCLASSIFIED_MIN_LINES` (1,000) **and**
    `other_lines >= _UNCLASSIFIED_MIN_SHARE` (0.10) × that repo's code lines
    (`repo_code_lines`). A repo with 0 code lines qualifies on the line floor alone.
  - Skip snapshots whose `role_counts` is None. In portfolio mode, dedupe by `repo.id`.
    Sort by lines desc, then project label, then repo name.
  - Calibration on today's data: only pre-fix bob-mac-capture qualifies. After the fix,
    nothing does. The shared render fixture's `other` of 500 lines stays below the
    floor, so existing goldens do not churn.
  - Compute it even when the subject's code total is 0. A pure-Swift repo before this
    fix had 0 code lines, which is exactly when the guard matters.
- `src/sase/stats/render/languages.py`:
  - **Card** (`render_language_card_block`): after the language and `?` rows, add one
    row per subject in the grid's existing columns:
    - name `not counted` in `UNKNOWN_STYLE`;
    - count = compact sum of unclassified lines;
    - an empty bar cell and an empty share cell. These lines are not part of the code
      total, so never show a share or bar.
    - a dim note: `unrecognized files · <repo>` (one project) or `<project · repo>`
      (portfolio), with up to 2 names plus `+N more`.

    When the guard has entries, return a block even if `total_lines == 0` or there are
    no language rows. In that case the block is only that row.

  - **Language table** (`render_language_table`): append a dim note after the existing
    `not measured` note:
    `not counted as code: 165k lines in files with no recognized language (bob-mac-capture 165k)`.
    Use full `format_count` numbers and the same project prefixes as `not measured` in
    portfolio mode. Also append it on the `No code lines measured.` path.

- `src/sase/stats/render/markdown.py`: append the same text to `_card_language_line` (as
  ` · not counted: …`) and as a note in `_language_markdown`. Like the card, emit them
  even when the subject has 0 code lines.
- `src/sase/stats/registry.py`: extend the `By language` legend with
  `; not counted = lines in files with no recognized language, shown when a repo has many`.
  It flows into `-h` definitions and the TUI `?` help. Regenerate the CLI help goldens.

> [!decision] unclassified_note = skip

Skip B2 entirely; B1, B3, and B4 still apply.

### B3. Tests (sase)

- `tests/core/test_source_language_facade.py`: add one parametrized case per new suffix
  from A1. This runs against the real binding built from the linked core.
- `tests/stats/test_render_languages.py`:
  - extend `test_every_core_id_has_label` with every new id;
  - assert `language_label("swift") == "Swift"`, `"csharp"` → `C#`, and `"vim"` →
    `Vim script`;
  - assert `_language_color("swift")` is the neutral color and `scss` is the CSS violet.
- A new test, for example `tests/pager/test_syntax_language_ids.py`: for each new
  suffix, resolve `sample.<suffix>` through the real facade. Then assert that
  `pygments.lexers.get_lexer_by_name(normalize_language(id))` succeeds and that the id
  has an explicit entry in `_LANGUAGE_LABELS`. This is the cross-repo parity guard.
- If `unclassified_note = add`:
  - add an `other_lines` parameter to the snapshot builder in
    `tests/stats/_project_render_fixtures.py` (default 500, unchanged);
  - add a fixture view, for example "one project with a 40k-line unrecognized repo",
    plus goldens at widths 50 and 120 for the model card and `-g language`, plus
    Markdown;
  - in `tests/stats/test_project_languages.py`, unit-test the threshold edges (999
    lines; 1,000 lines at 9% vs 10%; a 0-code repo), portfolio dedupe, and
    `role_counts=None`;
  - regenerate goldens with `SASE_UPDATE_PROJECT_RENDER_GOLDENS=1`, inspect each diff,
    and confirm existing goldens did not change.
- With the new core, pager tests may now highlight `.swift` fixtures
  (`tests/pager/_rendered_link_tree.py` uses `src/lined.swift`). If a styled snapshot
  changes because of Swift highlighting, inspect it and regenerate it. That change is
  intended.

### B4. Docs

Read the `docs_writing` reference memory first.
`docs/reference/tui/machines-and-tools.md` (Projects sub-tab): add one sentence on the
`not counted` row (if `add`). No other doc lists recognized languages.

## Landing and the core pin

Make both repos' changes in this one turn. The host commits the pinned `sase-core`
sibling first and writes its SHA into `sase-core-revision.txt` before the sase commit,
so do not hand-edit the pin. sase's setup step rebuilds `sase_core_rs` from the linked
checkout when its source changes, so local tests see the new core.

## Verification

1. sase-core: `just fmt`, then `sase tool run check` (≈5 min) until green.
2. sase: `just fix`, then `sase tool run check` until green. Hand both to
   `/sase_monitor` if they would outlive the turn. Treat UNKNOWN triage as yours.
3. Real data, using this checkout's `.venv/bin/sase`, not the global install:
   - `sase stats project -p bob-cli -r 90d`: the card's `by lang` shows **Swift** as the
     third row, roughly 130–160k SLOC, about 12–15%, with note `bob-mac-capture`. The
     `CODE` total rises by the same amount, and no `not counted` row appears.
   - `-g language`: Swift → bob-mac-capture.
   - `-g repo`: bob-mac-capture's LANGUAGE column reads Swift and its CODE LINES jump
     from 6.6k.
   - `sase stats project -g language` (portfolio): still renders cleanly.
   - Verify that `sase stats project -p sase` is unchanged apart from the sase-nvim
     `.vim` file now counting as Vim script.
   - Evidence that the guard works in real data (if `add`): before rebuilding, or by
     pointing a test at the pre-fix JSON shape, show the
     `not counted 165k … · bob-mac-capture` row.
4. TUI: following the `tui` reference memory, open Admin Center → Statistics → Projects,
   filtered to bob-cli, with this checkout's code. Capture the default card and `g` → By
   Language at 120 columns and about 90 columns with `sase screenshot`, then look at
   each PNG: Swift's neutral bar aligns with the others, and nothing wraps badly.
5. The user's global `sase` keeps the old core until it is reinstalled. Global installs
   are human-only, so propose `just install-dev` through `/sase_gate` instead of running
   it.

## Out of scope

- Merging sase's other Python suffix maps (`mentor_review_models.py`,
  `code_injection.py`) into the Rust table.
- Linguist-style content heuristics for ambiguous suffixes (`.m` MATLAB vs Objective-C,
  `.pl` Prolog vs Perl). This plan picks the dominant meaning.
- New palette slots or per-language test splits.
