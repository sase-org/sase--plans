---
tier: tale
title: Switch colon arguments to parentheses while typing
goal:
  Typing an opening parenthesis after an invocation's argument colon removes the colon
  in ACE and supported LSP editors while preserving pairing and cursor behavior.
size: medium
proposed_by: bbugyi200.athena.0kg
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0kg](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0kg.md)
  - [bbugyi200.athena.sase-10w.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-10w.1/README.md)
  - [bbugyi200.athena.sase-10w.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-10w.2/README.md)
  - [bbugyi200.athena.sase-10w.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-10w.3/README.md)
  - [bbugyi200.athena.sase-10w.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-10w.4/README.md)
  - [bbugyi200.athena.toobig-5e.agent_load_tiering_fixture.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5e.agent_load_tiering_fixture.0/README.md)
  - [bbugyi200.athena.toobig-5e.agent_load_tiering_harness.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5e.agent_load_tiering_harness.0/README.md)
  - [bbugyi200.athena.toobig-5e.test_axe_chop_agents.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5e.test_axe_chop_agents.0/README.md)
  - [bbugyi200.athena.toobig-5e.test_bare_git_workspace.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5e.test_bare_git_workspace.0/README.md)
  - [bbugyi200.athena.toobig-5e.test_enrich_agent_waiting.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5e.test_enrich_agent_waiting.0/README.md)
  - [bbugyi200.athena.toobig-5e.test_fleet_agents_projection.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5e.test_fleet_agents_projection.0/README.md)
  - [bbugyi200.athena.toobig-5e.test_gate_wait_dependency.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5e.test_gate_wait_dependency.0/README.md)
  - [bbugyi200.athena.toobig-5e.test_keymaps_display_help.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5e.test_keymaps_display_help.0/README.md)
  - [bbugyi200.athena.toobig-5e.test_notification_store.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5e.test_notification_store.0/README.md)
- **COMMITS:**
  - [107a003](https://github.com/sase-org/sase/commit/107a00334193fd4c9057096e429f5b442f213563)
    — test(perf): split agent load tiering fixture
  - [20d4782](https://github.com/sase-org/sase/commit/20d4782c96804fa8ea79810be86ec412a7b10ad4)
    — test(notification-store): split test_notification_store.py into focused files
  - [3fc45ec](https://github.com/sase-org/sase/commit/3fc45ec01a164f6b163428bf104df8b29b1e3549)
    — test(enrich-agent-waiting): split test_enrich_agent_waiting.py into focused files
  - [5024571](https://github.com/sase-org/sase/commit/5024571a3254393d56c2c9d5cf45fac996d18128)
    — fix(monitor): repair store_lane and monitor __init__ imports
  - [526df13](https://github.com/sase-org/sase/commit/526df13e48b81e8128b37552e76233e362d75775)
    — fix(scope): recalibrate scoped lane budget
  - [59e6670](https://github.com/sase-org/sase/commit/59e6670f4bfa2ceb1b2c65ca41c55b8265f42bdf)
    — test(keymaps-display-help): split test_keymaps_display_help.py into focused files
  - [7dfbaea](https://github.com/sase-org/sase/commit/7dfbaea8bba90a16105d8d755a94f7db3a2d53a3)
    — test(tui): split fleet agents projection tests
  - [8823a85](https://github.com/sase-org/sase/commit/8823a856171253bde90440577d75070d22338f7f)
    — test(bare-git): split test_bare_git_workspace.py by function under test
  - [8f4bf9a](https://github.com/sase-org/sase/commit/8f4bf9a28050cfa3110cbe8707bdcf46dd887252)
    — test(chop-agents): split test_axe_chop_agents.py into focused files
  - [a29c19f](https://github.com/sase-org/sase/commit/a29c19fdf0ced56f4470a1f2f0992130ecaeb73f)
    — test(gate-wait-dependency): split test_gate_wait_dependency.py into focused files
  - [cc91c0a](https://github.com/sase-org/sase/commit/cc91c0aa435c225402a4598dc6adf998ef257510)
    — test: make git identity hermetic in tests
  - [d0a849d](https://github.com/sase-org/sase/commit/d0a849df74be36f030ec392f30e159b54a65cb36)
    — test(ace): rebaseline drifted ACE PNG goldens and fix shell-label squeeze
    truncation
  - [f86056c](https://github.com/sase-org/sase/commit/f86056c7fd08d94f6dcf0ce2ac094249fc9f54ea)
    — feat(ace): convert argument colons when typing parens
  - [faf37fe](https://github.com/sase-org/sase/commit/faf37fec2214a7568e5038663fd1ff42fd2ded02)
    — refactor(tests): split agent_load_tiering_harness into private submodules

# Switch an invocation's colon arguments to parentheses while typing

## Goal and scope

When the user types `(` immediately after the argument-opening `:` of an
xprompt/workflow reference or recognized `%` directive, remove that colon. In the ACE
prompt input, `Some prompt here. %q:|` becomes `Some prompt here. %q(|)` (`|` denotes
the cursor). Apply the same colon removal through the xprompt language server's standard
on-type formatting support, and enable that support in the maintained Neovim
integration.

This is a `tale`, size `medium`: one coding agent can implement and verify the bounded
shared rule, its Python and LSP adapters, and the Neovim client setup. The work crosses
three repositories but does not require independently landed phases. Deliver the
complete behavior together. No new CLI commands, keymaps, feature flags, or parser
syntax are required.

## Repository access and existing implementation

Before reading or changing another repository, use `/sase_repo` and the paths returned
by these commands; do not assume a sibling checkout location:

```sh
sase repo open gh:sase-org/sase-core -r "Implement shared colon-to-parentheses editing and LSP support"
sase repo open sase-nvim -r "Enable and test xprompt LSP on-type formatting"
```

Read each checkout's applicable agent instructions. File paths below are relative to the
named repository, so this plan is portable between workspaces.

Existing integration points:

- **sase:** `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py` routes normal
  and visual modes before insert-mode pairing.
  `_prompt_text_area_key_pairing.py::_try_prompt_text_pair_edit` applies planned edits
  and reopens argument assistance after `(`. `_paired_text_editing.py` provides
  `TextEdit`, generic pair insertion, close-skip, and paired deletion.
  `_apply_planned_text_edit` uses the keyboard replacement path, restores the cursor,
  and refreshes completion state. `src/sase/ace/tui/util/editor_offsets.py` converts
  Python character positions to/from UTF-16. `model_alias_completion.py` and
  `_model_shortcut_marker.py` illustrate thin adapters around required Rust bindings.
- **sase-core:** `crates/sase_core/src/editor/` owns shared editor behavior.
  `token.rs::DocumentSnapshot` performs UTF-16/UTF-8 position conversion;
  `xprompt_args.rs` recognizes `#name`, `#!name`, namespaces, and invocation modifiers;
  `directive.rs` owns directive names, aliases, and syntax metadata.
  `prompt_literal_zone_ranges` covers fenced code, inline code, and disabled regions.
  `editor/model_alias_shortcut.rs` demonstrates additional frontmatter and Jinja
  exclusions. Public editor exports pass through `editor/mod.rs` and
  `crates/sase_core/src/lib.rs`; PyO3 exports and their tests currently live in
  `crates/sase_core_py/src/lib.rs`.
- **sase-core LSP:** `crates/sase_xprompt_lsp/src/server.rs` advertises `(` for
  completion but has no on-type formatting provider/handler. It stores full document
  snapshots, uses FULL text synchronization, and guards language features with document
  eligibility. `lsp_convert.rs` supplies range conversion. Existing JSON-RPC tests live
  in `crates/sase_xprompt_lsp/tests/`.
- **sase-nvim:** `lua/sase/lsp.lua` currently uses an `on_attach` callback only to
  enable native completion; that callback returns early when completion is disabled or
  cmp is selected. `tests/lsp_config.lua` verifies this wiring.
  `tests/lsp_queue_directive_smoke.lua` is a real-server headless test example. The
  plugin's existing editing helpers leave closing brackets to the user's pairing plugin.

Planning verified that the installed Neovim 0.12.5 runtime provides
`vim.lsp.on_type_formatting.enable(true, { client_id = client.id })`. Its implementation
schedules requests after insertion, applies UTF-16 edits using the client's encoding,
and discards results from older document versions. Consult the implementing runtime's
`:help vim.lsp.on_type_formatting.enable()` and `lua/vim/lsp/on_type_formatting.lua`
when verifying client behavior.

## Behavior contract

1. Trigger only on typing `(` in insert mode at a collapsed caret immediately after the
   invocation's first argument colon. Do not require that completion inserted the colon:
   manually authored text and moving back to a colon work.
2. Recognize the existing complete xprompt/workflow name grammar, including `#foo`,
   `#!foo`, `#ns/foo`, `#ns__foo`, aliases, and supported `!!`/`??` suffixes. Use syntax
   recognition, without catalog loading: an otherwise valid xprompt name need not
   already exist in the local catalog. Directives must resolve through canonical
   directive metadata and support both colon and parenthesized forms; include short
   aliases such as `%q`, `%w`, and `%m`.
3. Match valid invocation boundaries from the shared grammar, not a bare `endswith(':')`
   rule. Ignore ordinary colons, URLs, timestamps, mid-word marker fragments, unknown
   directives, escaped markers, and incomplete names. Reject double-colon syntax and
   colons inside an already-started argument.
4. Ignore launch-inert code/disabled regions and editor definition content such as YAML
   frontmatter and Jinja tags. Reuse existing Rust recognition and literal-zone helpers;
   if a helper needs to become shared, extract only that helper with regression
   coverage. Do not change general parsing or other completion classification while
   introducing this feature.
5. Only the colon is discarded; preserve all authored text to the right of the caret.
   For the TUI, compose the conversion with its existing safe-pair policy: insert `()`
   at EOF, before whitespace, or where generic pairing already permits it. Before a
   non-safe following token, remove the colon and insert just `(`, preserving that
   token. The rule never wraps or consumes existing argument text. For example,
   `#foo:|value` becomes `#foo(|value`.
6. The LSP receives text after insertion. `%q:(|` yields a deletion of only `:`,
   producing `%q(|`; `%q:(|)` yields the same deletion, producing `%q(|)`. It must not
   insert another `(` or `)`, emit snippets, or replace the document. Editor pairing
   supplies `)` when enabled. Standard on-type formatting must be enabled by external
   clients for this feature to run.
7. Programmatic loads, ordinary paste, and undo/redo must not initiate a new
   normalization pass. Existing TUI mode/selection behavior and generic pairing outside
   this invocation-specific case remain intact. A conversion is a single TUI keyboard
   replacement with its cursor between the pair.

## Implementation

### 1. Add one pure shared Rust edit planner and binding

Add a small module under `crates/sase_core/src/editor/`, for example
`argument_syntax_edit.rs`. Expose a planner taking a `DocumentSnapshot` and an
`EditorPosition` located immediately **after the candidate colon**, returning
`Option<EditorTextEdit>` containing precisely the colon's range and an empty `new_text`.
This pre-insertion text and coordinate are the common contract: the TUI has them
directly; the LSP verifies the inserted `(` and builds an in-memory view with that one
typed opener omitted. Positions before the opener, including the returned colon range,
are unchanged. This also makes the double-colon guard work when the user types between
the two colons.

Keep recognition and literal exclusions in Rust. Reject invalid/out-of-range positions
without panicking, including UTF-16 positions splitting a surrogate pair. Match the
argument-opening colon itself, not an arbitrary colon found later in the argument body.
Check that the colon is not the start or end of `::`. Reuse or narrowly extract the
existing name and invocation-boundary recognition, taking care that some completion
helpers intentionally accept incomplete tokens and therefore are too permissive for
destructive edits.

Export the planner through the usual core exports and add a required PyO3 binding
accepting text and the position dictionary, returning a plain edit dictionary or `None`.
Reuse `EditorTextEdit`/`EditorRange`; no new broad editor wire envelope is necessary.
Add binding serialization and invalid-input tests. Keep the operation in-memory, without
catalog refreshes, filesystem reads, subprocesses, or host-bridge calls on either typing
path.

### 2. Compose the edit into TUI pairing

Add a thin Python adapter for the Rust planner, using the existing binding loader and
UTF-16 conversion helpers. Validate/convert its returned colon range; do not reproduce
invocation matching in Python or add a Python fallback.

Within `_try_prompt_text_pair_edit`, after the existing collapsed-selection guard and
specifically for `(`, try this adapter before generic insertion. If eligible, compose
the colon deletion with `plan_pair_insert`'s decision about adding `)` into one
`TextEdit` against the original buffer. If generic pairing declines, the composed
replacement is just `(`. Apply it once through `_apply_planned_text_edit` and preserve
its completion refresh and `(` argument assistance path. Account for insert-capture
offsets when deleting a colon before the capture point; use the existing remapping
facility where needed.

Keep the original handling of other characters. After conversion, typing `)` must skip
the inserted closer, Backspace between an empty pair must remove both delimiters, and
ordinary undo/redo must restore/reapply the conversion without a transient second edit
for the removed colon.

### 3. Expose standard LSP on-type formatting

Advertise `documentOnTypeFormattingProvider` with `firstTriggerCharacter: '('` in
`initialize`, and implement the corresponding `LanguageServer` method. Require
`ch == '('`, an open eligible document, and a valid position just after the typed
opener. Verify the actual text at that position; do not search backward through
arbitrary text for an older `:(`. Use the shared planner on the pre-insertion view
described above, at the position immediately before `(`, and convert its optional edit
to the standard `TextEdit[]` result. The virtual view is only for recognition: apply the
returned colon range to the actual post-insertion document and preserve any editor-added
closer. Return no edits for unsupported triggers, positions, documents, and contexts.
Preserve FULL synchronization and existing completion triggers; no new server-initiated
`workspace/applyEdit` behavior.

Use the existing document snapshot without a catalog/bridge request. Explicitly test
requests after `didChange`, an editor-added closer, non-BMP characters, and repeated
requests after deletion (which must return no edit). Subsequent completion at `%q(`
should offer the existing parenthesized keyword choices.

### 4. Enable the Neovim client feature

Refactor `lua/sase/lsp.lua`'s attachment callback to enable completion and on-type
formatting independently. For the SASE client supporting
`textDocument/onTypeFormatting`, feature-detect `vim.lsp.on_type_formatting.enable` and
enable it for that client ID only. This must also run with cmp and with
`native_completion = false`.

Use Neovim's native implementation for request scheduling, edit application, version
checks, and detach cleanup. Do not add a second `(` mapping or a Lua invocation parser,
and do not enable formatting globally for unrelated LSPs. Missing native APIs or an
older SASE server must leave attachment functional. Document that older Neovim runtimes
need a client implementation of standard on-type formatting; do not silently claim the
new editing feature works there or increase the plugin's overall minimum version for
unrelated features.

Verify real typing with pairing enabled as well as bare insertion. Confirm caret
placement, editor undo behavior, and no duplicate request/edit path. Exercise typing
more text, moving the caret, and switching buffers before a delayed response. Reuse
native stale-result rejection; never force an old edit onto newer text. If testing
exposes an integration defect on the supported runtime, fix the narrow client
integration and its regression test rather than duplicating the grammar or applying
unchecked scheduled mutations.

### 5. Documentation and coordinated verification

Add the user-facing shortcut beside argument syntax/editor assistance in the sase repo's
`docs/xprompt.md`. Update `sase-nvim/README.md` with the same example, native on-type
enablement, client capability/version requirements, and the existing editor pairing
dependency. No default keymap or configuration change in `src/sase/default_config.yml`
is needed unless implementation actually adds a configurable option; a new option is not
part of this plan.

## Tests and acceptance

Use table-driven Rust cases for shared eligibility and range semantics, thin binding
tests for transport, and focused TUI/LSP/Neovim tests for actual editor behavior. Prefer
existing test harnesses; do not duplicate the full grammar test matrix independently in
three languages.

Required cases:

| Input/context (`<cursor>` is the caret)                                                                                      | Expected conversion behavior                       |
| ---------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| `Some prompt here. %q:<cursor>` + `(`                                                                                        | TUI becomes `Some prompt here. %q(<cursor>)`       |
| `%queue:<cursor>`, `%w:<cursor>`, `%model:<cursor>`, `%m:<cursor>`                                                           | Alias and long-form conversion                     |
| `#foo:<cursor>`, `#!foo:<cursor>`, namespaced and supported modifier forms                                                   | Conversion without catalog I/O                     |
| A second line, accented text, emoji before the invocation                                                                    | Correct range and caret in each encoding           |
| `#foo:<cursor>value`, `#foo:<cursor> trailing prose`                                                                         | Preserve suffix; retain generic TUI pairing policy |
| `Note:<cursor>`, URL/fragment, time, `%unknown:<cursor>`, `word%q:<cursor>`, `word#foo:<cursor>`                             | No colon deletion                                  |
| `%q::<cursor>`, `#foo::<cursor>`, caret between the two colons                                                               | No colon deletion                                  |
| `%q:1<cursor>`, `%q:1:<cursor>`, `%q: <cursor>`, `%q(foo:<cursor>`                                                           | No colon deletion                                  |
| Escaped invocation, inline/fenced code, disabled span, frontmatter, Jinja                                                    | No colon deletion                                  |
| Active selection, normal/visual mode, paste/programmatic load                                                                | Existing input behavior                            |
| LSP `%q:(<cursor>` and `%q:(<cursor>)`                                                                                       | Exactly one colon-only deletion                    |
| LSP `%q:(<cursor>:` (typed between colons), wrong trigger, invalid position, unopened/ineligible URI, already converted text | No edits                                           |

Extend `tests/ace/tui/widgets/test_prompt_pair_editing.py` or add a focused neighbor
test module. Drive real key events for the user's example, text and cursor assertions,
paired Backspace, closer skip, undo/redo, and completion refresh. Verify
insert-capture/dot-repeat behavior when the colon predates entering insert mode. Run the
existing Jinja/alt pairing and directive completion tests that cover adjacent dispatch
paths.

Add a Rust LSP JSON-RPC test exercising initialize, didOpen/didChange, onTypeFormatting,
application of its edit, and completion on the updated text. Unit-only calls to the
planner are insufficient evidence of LSP wiring.

Extend `sase-nvim/tests/lsp_config.lua` with API-present/API-absent, wrong-client,
unsupported-server, and completion-disabled cases. Add a headless smoke test using the
real rebuilt LSP and real insert-mode key input; cover both an editor pairing mapping
and no pairing, Unicode, stale responses, and buffer lifecycle. Explicitly select the
newly built server using `SASE_XPROMPT_LSP_CMD` rather than relying on a globally
installed old binary.

Before declaring implementation complete:

1. Read `lint_and_test.md` through `/sase_memory_read`. In sase-core run its required
   `just check` (or `./scripts/check.sh`), including PyO3 and LSP tests. Do not
   substitute `cargo test -p sase_core` for workspace checks.
2. Rebuild/install the changed binding into the active sase environment using
   `SASE_CORE_DIR=<opened-core-path> just rust-install`; use `just install` first if the
   environment needs setup. Do not hand-edit Rust release versions or path dependency
   pins; release-plz owns them. Ensure both the Python tests and the LSP smoke test use
   this implementation.
3. Run the targeted editor tests and `just check` in sase. Follow the existing two-speed
   verification guidance if it escalates to `just check-full`. Use `/sase_monitor` for
   checks that require a long-running handoff.
4. In sase-nvim run `nvim --headless -u NONE -c 'set rtp+=.' -l tests/lsp_config.lua`
   and the new on-type smoke test, plus the existing queue completion and adjacent
   spacer/alt editing checks. Use the same headless test invocation convention for each
   test file.
5. Review all three repository diffs and the acceptance matrix. Include all changed
   repositories in the host-owned final declaration, and report any client-version
   limitations and verification gaps explicitly. No direct git commits, branches, or PR
   creation by the coding agent.

Success means the user's example works in the prompt widget and through a configured
real LSP client, the shared detector has no Python/Lua duplicate, ordinary colon text
remains intact, and the required checks pass.
