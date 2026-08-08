---
tier: epic
title: XPrompt target mode for the prompt input stack
goal: 'Editing an existing xprompt definition from the ACE TUI is a first-class, obvious,
  reliable loop: loading a definition from any surface puts the prompt input widget
  stack into a visibly distinct "targeting" state, <enter> offers a single-keypress
  "save to the targeted xprompt" option, and saving writes the correct file (chezmoi
  source included) and then offers exactly the follow-up actions that file needs —
  commit/push, a scoped chezmoi apply, or the matching `sase memory init` / `sase
  skill init`.

  '
phases:
- id: target
  title: Targeting model, reference identity, and chezmoi-aware write paths
  depends_on: []
  size: medium
  description: 'target: harden XPromptBinding into a full targeting record (canonical
    reference form, resolved chezmoi write path, apply target, cheap stat-only staleness),
    add the single target/clear choke point on PromptInputBar, and route bound writes
    through the resolved write path.

    '
- id: surfaces
  title: Every definition-load surface targets its xprompt
  depends_on:
  - target
  size: medium
  description: 'surfaces: audit every path that loads an xprompt definition into the
    prompt stack, add the missing "edit definition here" action to the Select XPrompt
    panel, stop the external-editor round trip from silently dropping the target,
    and lock the audit down with an invariant test.

    '
- id: menu
  title: Target-aware <enter> chooser with a single-key save
  depends_on:
  - target
  size: medium
  description: 'menu: open the submit chooser whenever the stack is multi-pane or
    targeted, and add the single-key "save to the targeted xprompt" and "save as a
    new xprompt" rows with dirty/clean-aware copy.

    '
- id: followup
  title: Post-write follow-up actions with memory and skill init integration
  depends_on:
  - target
  size: medium
  description: 'followup: replace the single commit/push confirmation with a classifier-driven
    follow-up chooser that offers commit/push, a scoped chezmoi apply, `sase memory
    init`, or `sase skill init` — whichever actually apply to the written file — and
    runs them through the tracked task queue.

    '
- id: visual
  title: Visual language for the targeting state
  depends_on:
  - target
  - menu
  size: medium
  description: 'visual: give the targeting state a deliberate, theme-safe look — distinct
    bar border, a reference chip with clean/dirty/read-only/stale states, matching
    frontmatter panel tint, subtitle and footer hints — and pin it with PNG snapshot
    goldens.

    '
- id: docs
  title: Help modal, footer, and documentation
  depends_on:
  - surfaces
  - menu
  - followup
  - visual
  size: small
  description: 'docs: bring the `?` help popup, the conditional keybinding footer,
    and the ACE / xprompt documentation in line with the new targeting keys and save
    flow.

    '
- id: verify
  title: End-to-end verification of the edit loop
  depends_on:
  - docs
  size: small
  description: 'verify: run the full verification gate and drive the real TUI through
    the complete loop for a plain xprompt, a memory note, and a skill source, including
    the chezmoi path.'
proposed_by: bbugyi200.athena.vy
create_time: 2026-08-08 15:51:55
status: wip
bead_id: sase-hp
---

- **PROMPT:** [prompts/202608/xprompt_target_mode.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/xprompt_target_mode.md)
- **BEAD:** [sase-hp](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hp/README.md)

# Plan: XPrompt target mode for the prompt input stack

## Context: what already exists

A substantial amount of this feature is already built and must be **extended, not
rebuilt**. Read this section before writing any code.

- `sase.ace.tui.widgets.prompt_stack` already defines `XPromptBinding`
  (`kind`/`path`/`target_format`/`entry_name`/`loaded_fingerprint`),
  `SourceFingerprint`, and `PromptStackState.bind()` / `unbind()` / `is_dirty` /
  `source_changed()` / `mark_written()` / `markdown_preserving_unchanged_body()`.
- `PromptInputBar._refresh_title()` already appends `· <name> · <path>` and a gold `●`
  when the bound stack is dirty.
- `PromptInputBarStashActionsMixin.request_write_xprompt()` (the `gw` / `^G w` keymap)
  posts `WriteXpromptRequested`; the app handles it in
  `sase.ace.tui.actions.agent_workflow._prompt_bar_save_xprompt`, resolving external
  changes through `XPromptWriteConflictModal` (overwrite / reload / save-as) before
  writing via `write_binding_sync`.
- `_offer_git_commit()` in `_prompt_bar_save_xprompt_git` already pushes a
  `ConfirmActionModal` commit/push confirmation and runs `run_git_commit_push_sync`
  (add, commit, pull --rebase, push, then an unscoped `apply_chezmoi()`), all inside the
  tracked task queue.
- Two surfaces already bind on load: the Admin Center XPrompts tab
  (`xprompt_browser_actions.action_edit_xprompt`) and the jump panel
  (`_prompt_jump._load_jump_target_into_prompt`).

So this epic is about closing the gaps between that machinery and a loop a person
actually wants to use. Each phase below states its gap explicitly.

## Naming

The user-facing noun is **target**: the prompt stack _targets_ an xprompt definition.
Prefer "target" in every string, docstring, and doc the user can read. The internal type
keeps the existing `XPromptBinding` name so the diff stays reviewable; do **not** rename
it.

## Cross-cutting constraints

- **Rust core boundary.** Per `sase/memory/rust_core_backend_boundary.md`, shared domain
  behavior belongs in the sibling Rust core. The new write-path resolver and
  written-file classifier are domain logic a CLI would also need, so they must live in
  plain, non-TUI Python under `src/sase/xprompt/` (importable without Textual), built on
  the existing `sase.content_layout` / `sase.config.targets` adapters — not inside
  `src/sase/ace/tui/`. If a phase agent finds the corresponding contract already modeled
  in the Rust core, extend it there and adapt here instead of duplicating it.
- **Never edit memory files.** No phase may add, edit, or remove anything in
  `sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, or
  `QWEN.md`. That includes the glossary. If a phase concludes a memory note should
  change, it records a `PROPOSED FOLLOW-UP:` note on its own bead instead.
- **Do not hand-edit `CHANGELOG.md`** — release-please generates it from commit
  messages.
- **Verification.** Run `just install` first (workspaces are ephemeral), then
  `just check` while iterating. The `verify` phase runs `just check-full`.
- **No blocking IO on the Textual event loop.** Every `stat`, read, write, `git`, and
  `chezmoi` call goes through `asyncio.to_thread` or the tracked task queue, matching
  the existing code in `_prompt_bar_save_xprompt.py`. Before touching anything that runs
  on a refresh path, read `sase/memory/tui_perf.md` via the `/sase_memory_read` skill.
- **TUI conventions.** `src/sase/ace/CLAUDE.md` requires that any conditional keymap
  (available only in some states) appear in the keybinding footer, that the `?` help
  popup stay in sync, and that help modal boxes keep their fixed width.

---

## Phase `target`: Targeting model, reference identity, and chezmoi-aware write paths

### Gap

Three problems in the current model:

1. **`XPromptBinding` has no canonical reference.** `binding.name` is
   `entry_name or Path(path).stem`, so a memory note renders as `foo` when the user
   would type `#memory/foo`, and a skill renders as its file stem when the user would
   type `/name`. The UI cannot show the user the string they actually use.
2. **`binding.path` is both the read path and the write path.** With `use_chezmoi`
   enabled, home xprompt/skill/memory discovery reads the _applied_ copy under
   `~/sase/...`, but the editable file is the chezmoi source under
   `~/.local/share/chezmoi/home/sase/...`. A bound write today lands on the applied
   copy, which (a) is not in a git repo, so no commit is ever offered, and (b) is
   silently clobbered by the next `chezmoi apply`. This is the "good support for
   chezmoi" the feature needs most.
3. **Staleness detection is expensive.** `SourceFingerprint.from_path` reads and hashes
   the whole file, so nothing can afford to check for external changes except at write
   time.

### Work

**1. Write-target resolution.** Add a new non-TUI module (suggested:
`src/sase/xprompt/write_targets.py`) with a frozen dataclass and resolver:

```python
@dataclass(frozen=True)
class XPromptWriteTarget:
    read_path: Path      # where the definition was discovered/displayed
    write_path: Path     # where an edit must actually be written
    apply_target: Path | None  # home path to `chezmoi apply` after a commit, else None
    via_chezmoi: bool

def resolve_xprompt_write_target(read_path: Path | str) -> XPromptWriteTarget: ...
```

Resolution rules, in order — each guard matters:

- Default: `write_path == read_path`, `apply_target is None`, `via_chezmoi is False`.
- Remap to the chezmoi source **only when all** of these hold: `get_use_chezmoi()` is
  true; `read_path` is under `Path.home()`; `read_path` is _not_ already under
  `CHEZMOI_HOME`; and `chezmoi_source_path(read_path)` **exists on disk**. The existence
  guard is essential — without it, an ordinary unmanaged home file would be redirected
  into the chezmoi source tree and never appear at its original location again.
- When remapped: `write_path` is the chezmoi source, `apply_target` is the original home
  path, `via_chezmoi` is true.

Mirror the existing precedent in `sase.config.targets.resolve_write_path` /
`sase.content_layout.chezmoi_source_path`; reuse those helpers rather than
re-implementing the `dot_` encoding. Unit-test the resolver directly with a fake home
root and fake chezmoi root, covering: no chezmoi, chezmoi on but source missing, chezmoi
on with source present, a path already inside the chezmoi source tree, and a path
outside `$HOME`.

**2. Canonical reference identity on the binding.** Add a `reference: str` field to
`XPromptBinding` holding the string a user would type — `#foo`, `#memory/foo`,
`/skill-name`, `#project/foo`. Callers already know it: `xprompt_browser_actions` builds
`f"#{name}"` today, and `_prompt_jump` has `payload.title`, which is exactly the
reference. Derive it from `sase.content_layout.memory_reference_name` /
`skill_reference_name` when a caller only has a stem. Keep `name` working as-is for
existing callers (`UnifiedXPromptSaveModal`'s `initial_name` relies on it) but stop
using it for display.

**3. Split read and write paths on the binding.** `XPromptBinding` gains
`write_path: str` and `apply_target: str | None`, populated by
`resolve_xprompt_write_target` inside `for_file()` / `for_config()`. Critically:

- `loaded_fingerprint` must be taken over **`write_path`**, not `path`, because that is
  the file a later write would overwrite and therefore the file whose external change
  matters.
- `source_changed()` and `mark_written()` follow `write_path`.
- `write_binding_sync` in `_prompt_bar_save_xprompt_targets.py` writes `write_path`.
- `_reload_bound_xprompt` keeps reading from `path` (the discovered/applied copy is what
  discovery would serve). Add a test for the chezmoi case asserting read and write
  resolve to different files and both are correct.

**4. Cheap staleness.** Add a stat-only comparison to `SourceFingerprint` (e.g.
`SourceFingerprint.stat_signature(path) -> tuple[int, int] | None` returning
`(mtime_ns, size)`, and a `matches_stat()` on the record) so a caller can ask "did this
change?" with one `stat()` and no read. The full hash stays authoritative and stays in
the write-time conflict check — the stat-only path is strictly an optimization for
display, and a false "changed" reading is acceptable there because the write path
re-checks properly.

**5. One choke point on the bar.** Add to `PromptInputBar` (the stack-rendering mixin
alongside `load_stack_from_xprompt_markdown`):

```python
def target_xprompt(self, binding: XPromptBinding, *, source_markdown: str | None = None) -> None
def clear_xprompt_target(self) -> None
def xprompt_target(self) -> XPromptBinding | None
```

These wrap `self._stack.bind()` / `unbind()` and additionally refresh the title and the
targeting CSS classes. Every existing caller of `_stack.bind(...)` outside
`prompt_stack.py` must be converted to `target_xprompt(...)`. This is what makes the
`surfaces` phase auditable and what stops the next new surface from forgetting.

### Done when

The resolver, reference field, split read/write paths, stat-only staleness, and the bar
choke point all exist with unit tests; a bound write under `use_chezmoi` writes the
chezmoi source and records the home apply target; `just check` passes.

---

## Phase `surfaces`: Every definition-load surface targets its xprompt

### Gap

The user asked for targeting to activate "from any surface". Two surfaces are missing
and one actively destroys an existing target:

- **Select XPrompt panel** (`xprompt_select_modal.py`, the `#` completion picker) has
  `ctrl+e` "open definition in $EDITOR" and `ctrl+i` "expand inline", but no way to load
  the definition into the bar for editing. This is the surface the user named first.
- **External editor round trip.** `_load_editor_markdown_into_bar` in
  `_prompt_bar_mount.py` calls `load_stack_from_xprompt_markdown(markdown)` with no
  binding, so opening a targeted stack in `$EDITOR` (`^G g`) and coming back silently
  drops the target. The user's edits are then one keystroke away from going nowhere.

### Work

**1. Add "edit definition here" to the Select XPrompt panel.** Mirror
`xprompt_browser_actions._load_xprompt_definition` exactly: resolve the highlighted
entry's source to a file path, read markdown off-thread (`load_config_xprompt_markdown`
for YAML-backed sources, plain read otherwise), build the binding when the source is
editable, and call the app's `load_xprompt_definition_into_home_prompt_bar`. Do not
duplicate that logic — lift the shared body into a helper both modals call.

Bind it to `ctrl+o` ("open for editing") on both the modal and its `_XPromptFilterInput`
forwarder, matching how `ctrl+e` / `ctrl+i` are forwarded. **Verify no collision first**
against `OptionListNavigationMixin.NAVIGATION_BINDINGS` and the existing `ctrl+f`,
`ctrl+b`, `ctrl+d`, `ctrl+u`, `ctrl+e`, `ctrl+i`. If `ctrl+o` is taken, pick another
free chord and note the choice on the bead. Show the key in the modal's footer/hint
line.

**2. Preserve the target across the editor round trip.**
`_load_editor_markdown_into_bar` must carry the bar's current binding through the
reload. Preserving is correct — the user edited the same definition's text — and the
stack's dirty hash will simply reflect the edited content. Add a regression test.

**3. Audit and pin every whole-stack replacement.** Enumerate the paths that replace the
stack and assert the intended targeting outcome for each:

| Path                                                                  | Expected                                                                                   |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `xprompt_browser_actions.action_edit_xprompt` (Admin Center, `enter`) | targets (already)                                                                          |
| `_prompt_jump._load_jump_target_into_prompt` (`ctrl+]`, and `gd`)     | targets (already)                                                                          |
| Select XPrompt panel `ctrl+o`                                         | targets (new)                                                                              |
| `_load_editor_markdown_into_bar` (`^G g` return)                      | preserves the existing target (new)                                                        |
| `_write_xprompt_target` save-as                                       | targets the newly written file (already)                                                   |
| `_reload_bound_xprompt` conflict reload                               | re-targets with a refreshed fingerprint (already)                                          |
| `stash_all_and_load_xprompt_markdown` restore                         | clears the target, keeps the existing warning toast                                        |
| history load / `load_prompt_into_pane`                                | leaves the target alone (it is a pane edit, and dirty tracking covers it)                  |
| read-only source (`is_editable` false)                                | no target; the bar must show a persistent read-only state (see `visual`), not only a toast |

Write one test module that walks this table. Where a surface is presentation-only,
assert through the widget API (`xprompt_target()`), not through private state.

**4. Grep guard.** After conversion, `_stack.bind(` must not appear outside
`prompt_stack.py` and the new choke point. Add a Symvision-style or plain test assertion
enforcing that, so a future surface cannot bind behind the bar's back. Read
`sase/memory/symvision.md` via `/sase_memory_read` if a lint pragma is needed.

### Done when

All three code changes land, the audit table is a passing test, no surface loads an
editable definition without targeting it, and `just check` passes.

---

## Phase `menu`: Target-aware `<enter>` chooser with a single-key save

### Gap

`PromptSubmitChoiceModal` offers only `a` (all) and `c` (current), and
`_prompt_text_area_key_handling` only opens it when `bar.is_multi_pane()`. A targeted
single-pane stack therefore sends on `<enter>` with no way to save, and a targeted
multi-pane stack's menu has no save row. Saving is reachable only through `gw` / `^G w`,
which is undiscoverable.

### Work

**1. Widen the trigger.** In `_prompt_text_area_key_handling`, open the chooser when
`(bar.is_multi_pane() or bar.xprompt_target() is not None)` and the stack has at least
one non-blank pane. Keep the current behavior exactly when neither holds: an untargeted
single-pane stack still sends immediately on `<enter>`. Do not regress that — it is the
most-used keystroke in the app.

**2. Rebuild the modal's option set.** `PromptSubmitChoiceModal` takes the target and
the pane count and renders only applicable rows, in this order:

| Key            | Row                    | Shown when               | Subtitle                                                                                            |
| -------------- | ---------------------- | ------------------------ | --------------------------------------------------------------------------------------------------- |
| `s`            | Send                   | single-pane and targeted | Launch this draft as one agent.                                                                     |
| `a` (`ctrl+s`) | Launch all N           | multi-pane               | Launch all N prompts as one xprompt swarm.                                                          |
| `c`            | Launch current         | multi-pane               | Launch only the selected prompt as a single agent.                                                  |
| `w`            | Save to `<reference>`  | targeted                 | dirty: `Overwrite <display path> with your edits.` clean: `No unsaved changes since the last save.` |
| `X`            | Save as a new xprompt… | targeted                 | Fork this definition to a new location.                                                             |
| `esc` / `q`    | Cancel                 | always                   | —                                                                                                   |

`w` dispatches `bar.request_write_xprompt()`; `X` dispatches
`bar.request_save_as_xprompt()`; `s` dispatches the existing single-pane submit. The
untargeted multi-pane menu must render **exactly** as it does today (`a` / `c` only) so
existing muscle memory is untouched.

**3. Presentation.** Follow the existing modal's idiom: `_render_choice`, the
`duration-choice-*` class family, the trailing dim key-summary line. The `w` row gets
the same reference chip treatment the `visual` phase defines for the border title, so
the two read as the same object. The dim summary line must list only the keys actually
rendered.

**4. Purity.** The modal does no IO. `is_dirty` is a hash comparison over in-memory text
and is safe; `source_changed()` is not, and must stay in the existing off-thread
`_handle_write_xprompt_requested` conflict check. Do not add a disk check to the modal.

**5. Subtitle.** `insert_mode_subtitle()` currently returns `[Enter] send …` for a
single-pane bar. When targeted, it must return the `[Enter] submit…` form so the hint
matches what `<enter>` does.

**6. Tests.** Cover: untargeted single-pane sends directly; targeted single-pane opens
the menu with `s`/`w`/`X`; untargeted multi-pane menu is byte-identical to today;
targeted multi-pane shows `a`/`c`/`w`/`X`; `w` posts `WriteXpromptRequested`; the
clean/dirty subtitle differs; `esc` dismisses without launching.

### Done when

`<enter>` on a targeted stack offers a one-keypress save, the untargeted paths are
unchanged, and `just check` passes.

---

## Phase `followup`: Post-write follow-up actions with memory and skill init integration

### Gap

`_offer_git_commit` asks one y/n question and then, inside the task, runs an
**unscoped** `apply_chezmoi()` — a whole-home apply as a side effect of saving one
xprompt, with no say from the user. And nothing knows that some written files need more
than a commit: a memory note under `sase/memory/` needs `sase memory init` to regenerate
`AGENTS.md` and the provider shims, and a skill source under a canonical `skills/`
directory needs `sase skill init` to regenerate and deploy the generated skill files.
Today those silently go stale.

### Work

**1. Classifier (non-TUI).** Add a pure function next to the write-target resolver:

```python
class WrittenFileKind(StrEnum):
    XPROMPT = "xprompt"
    MEMORY_NOTE = "memory_note"
    SKILL_SOURCE = "skill_source"
    CONFIG_ENTRY = "config_entry"

def classify_written_file(path: Path | str) -> WrittenFileKind: ...
```

Classify by path containment only — no file reads:

- `SKILL_SOURCE` when the parent is a canonical skill directory
  (`sase.xprompt.skill_locations.is_canonical_skill_directory`).
- `MEMORY_NOTE` when the path is under a memory source root from
  `sase.content_layout.resolve_memory_file_sources`.
- `CONFIG_ENTRY` for a `.yml` / `.yaml` target.
- `XPROMPT` otherwise.

Classify the **write path** (post-chezmoi-remap), and also check the pre-remap read
path, because a chezmoi source path such as
`~/.local/share/chezmoi/home/sase/memory/foo.md` will not match a memory source root
that is rooted at `$HOME`. Prefer classifying the read path and carrying the kind
alongside the write path. Unit-test both forms.

**2. Follow-up plan builder (non-TUI, off-thread).** A function that takes the write
target plus the kind and returns an ordered list of offered actions, each with a label,
subtitle, single key, and default-on flag:

- **Commit & push** — when `get_git_root(write_path)` is not `None` and
  `has_git_changes(...)`. Default on.
- **Apply chezmoi** — when `target.via_chezmoi`. Default on. **Scoped to
  `target.apply_target`**, never a bare `apply_chezmoi()`.
- **`sase memory init`** — when kind is `MEMORY_NOTE`. Default on.
- **`sase skill init`** — when kind is `SKILL_SOURCE`. Default on.

**Mutual exclusion, and state it in the UI.** `sase memory init` and `sase skill init`
each already run their own commit/push sequence (and `sase skill init` does its own
chezmoi deploy). So when either is offered, it **replaces** the generic commit/push and
apply-chezmoi rows rather than stacking with them — otherwise the two race for the same
index and produce double commits. The modal must say so plainly in that row's subtitle,
e.g. `Regenerates AGENTS.md and provider shims, then commits and pushes for you.` This
is a correctness requirement, not a style preference.

All of the probing (`git rev-parse`, `git status`, path resolution) runs off the event
loop, exactly like the existing `build_config_commit_offer` does.

**3. `PostWriteActionsModal`.** One modal instead of a y/n chain. Rows are the offered
actions with their single keys; a key toggles a row; `enter` runs everything still
selected; `esc` skips all. When exactly one action is offered, keep it feeling like
today's confirmation: `enter` runs it, `esc` skips. Reuse `ConfirmActionModal`'s visual
vocabulary (title, subject line, icon) so it does not read as a new dialect. The subject
line shows the repo-relative write path, and — when `via_chezmoi` — makes the chezmoi
redirect visible, because a user who edited `~/sase/xprompts/foo.md` deserves to see
that the write landed in the chezmoi source.

**4. Execution.** Each selected action becomes a tracked task with a distinct
`dedup_key`, run in the listed order, later actions skipped if an earlier one fails (a
failed commit must not be followed by a push or an apply). Reuse
`_submit_xprompt_commit_task`'s shape: `TrackedTaskResult`, `on_complete` notification,
`reload_on_complete=False`, `notify_on_complete=False`, plus the existing stale
`index.lock` warning. `sase memory init` and `sase skill init` run as captured
subprocesses through the same queue; on failure, surface the command's stderr rather
than a generic message, since these commands have real, actionable diagnostics.

**5. Rework `run_git_commit_push_sync` carefully.** It is shared: `config_commit.py`
calls it for config edits, and `xprompt_browser_actions.action_external_edit_xprompt`
calls `_offer_git_commit`. Keep the existing entry points working. The specific change
is that the chezmoi apply moves **out** of `run_git_commit_push_sync` and becomes its
own opt-in, scoped action; audit every caller and give each one the equivalent behavior
it has today (config surfaces already do their own scoped `apply_chezmoi(target)` — see
`config_edit_modal.py` and `axe_config_actions/_backend.py` — so the pattern exists).

**6. Tests.** Classifier and plan-builder unit tests (memory note, skill source, plain
xprompt, config entry; chezmoi on and off; git repo and not). Modal tests for row
rendering, toggling, single-action fast path, and skip. A task-sequencing test asserting
that a failed commit halts the chain, and that memory/skill init suppresses the generic
commit and apply rows.

### Done when

Saving a targeted memory note offers `sase memory init`, a skill source offers
`sase skill init`, a chezmoi-backed file offers a scoped apply, an ordinary xprompt
offers commit/push, nothing double-commits, and `just check` passes.

---

## Phase `visual`: Visual language for the targeting state

### Gap

Today the only signal is text appended to the border title. There is no way to tell at a
glance that `<enter>` now means something different, no persistent read-only indicator
(only a toast that scrolls away), and no indication that the file changed underneath you
until the write fails.

### Work

Read `sase/memory/tui_perf.md` via `/sase_memory_read` before starting: nothing here may
add work to a hot refresh path.

**1. Bar silhouette.** The bar is `border: solid $accent`; feedback mode is
`solid $warning`. Give targeting a _border type_ change, not just a color, so it
survives every theme and is unmistakable in peripheral vision:

```
PromptInputBar.xprompt-target        { border: double $secondary; }
PromptInputBar.xprompt-target.dirty  { border: double $warning; }
PromptInputBar.xprompt-target.readonly { border: double $text-muted; }
```

Use theme variables only — no hardcoded hex — so light and dark both read correctly.
Classes are applied by the `target_xprompt()` / `clear_xprompt_target()` choke point and
refreshed wherever the dirty flag is recomputed.

**2. Reference chip in the border title.** Replace the current
`Prompt · <name> · <path>` text with a real chip plus a state marker:

- Chip: `✎ <reference>` on a `$secondary` background, using the **canonical reference**
  from the `target` phase (`#foo`, `#memory/foo`, `/skill-name`) — the string the user
  would actually type, not the file stem.
- Path: `display_path()`-shortened, then **middle-elided** to fit the available border
  width with the tail preserved (a deep path currently overruns the title). Reuse the
  repo's existing width helpers (`sase.markdown_width` / `sase.content.py`) rather than
  hand-rolling.
- State marker, always present so the state is read rather than inferred:
  - clean → dim `✓`
  - dirty → the existing gold `●`
  - read-only → a `🔒 read-only` chip (this replaces the transient "Read-only source —
    gw will save-as" toast as the _persistent_ signal; keep the toast as the
    moment-of-load explanation)
  - changed on disk → an `⚠ changed on disk` chip in `$warning`
- Remember the existing constraint: border titles parse Rich markup, so any literal
  bracket must be escaped, as `_refresh_title` already does for the mode suffix.

**3. Stale detection without a timer.** Do **not** add a polling timer. Use the
stat-only signature from the `target` phase, checked opportunistically on events the bar
already handles (bar focus / app-level refresh), rate-limited to at most once every few
seconds, in a worker thread. A false positive is harmless — the write path re-checks
with the full hash and still shows `XPromptWriteConflictModal`. If the phase agent
cannot make this genuinely cheap, drop the stale chip and record a `PROPOSED FOLLOW-UP:`
note on the bead; the other three states are the required ones.

**4. Frontmatter panel affinity.** When targeting, tint `#frontmatter-panel`'s border to
match the bar's targeting border so the panel and the stack read as one "definition
editor" unit, and auto-show it on a targeted load even when the definition has no
frontmatter yet — editing a definition is exactly when its properties matter.

**5. Hints.** `insert_mode_subtitle()` / `normal_mode_subtitle()` gain a
`[^G w] save <reference>` hint when targeted, elided sensibly on a narrow terminal. The
`gw` hint-panel label (`_g_prefix_label_write_xprompt`) already switches on the binding;
update it to name the reference (`save #foo`) instead of the generic `write xprompt`.

**6. PNG snapshot goldens.** Per the repo's visual-suite conventions, add goldens under
`tests/ace/tui/visual/snapshots/png/` for: targeted-clean, targeted-dirty,
targeted-read-only, and the target-aware `<enter>` menu. Run `just test-visual`; use
`--sase-update-visual-snapshots` only to accept the new goldens deliberately. These are
what make "beautiful" checkable by the next agent instead of a matter of opinion.

### Done when

The targeting state is unmistakable at a glance in both themes, all four chip states
render, no new timer runs on a hot path, the PNG goldens pass, and `just check` passes.

---

## Phase `docs`: Help modal, footer, and documentation

### Work

- **Keybinding footer.** `src/sase/ace/CLAUDE.md` requires that a keymap appear in the
  footer if and only if its availability is sometimes true and sometimes false. The save
  action is exactly that, so add the targeted-stack save key to `keybinding_footer.py`,
  following the module's sorting and lowercase angle-bracket conventions (`<enter>`
  before named keys).
- **`?` help popup.** Add the new Select XPrompt panel key, the target-aware `<enter>`
  menu, and the save key. Respect `_BOX_WIDTH = 57` / `_CONTENT_WIDTH = 50` and the
  32-character description cap.
- **Docs.** Update `docs/ace.md` and `docs/xprompt.md` with a short "editing an existing
  xprompt from the TUI" section: how to enter the targeting state from each surface,
  what the chip states mean, what `<enter>` offers, and what the follow-up actions do —
  including that memory notes and skills run their own `sase memory init` /
  `sase skill init` rather than a bare commit. Mention the chezmoi redirect explicitly.
- **Do not touch memory files or the glossary.** If a glossary entry for "xprompt
  target" seems warranted, record it as a `PROPOSED FOLLOW-UP:` note on the bead.

### Done when

Footer, help popup, and docs describe the shipped behavior with no stale keys, and
`just check` passes.

---

## Phase `verify`: End-to-end verification of the edit loop

### Work

1. `just install`, then `just check-full` — every lint gate plus the full suite. Also
   run `just test-visual` for the new goldens.
2. Drive the real TUI (see the project's run conventions) through the complete loop
   three times, confirming the visual state at each step:
   - **Plain xprompt** — load from the Select XPrompt panel's new key, edit, `<enter>`,
     save, confirm the commit/push offer appears and the file on disk matches.
   - **Memory note** — load a `#memory/...` definition, edit, save, confirm the
     follow-up offers `sase memory init` and **not** a bare commit/push, and that
     running it regenerates the derived instruction files.
   - **Skill source** — load a `/skill` definition, edit, save, confirm the follow-up
     offers `sase skill init`.
3. Exercise the chezmoi path. If `use_chezmoi` is not enabled in the verification
   environment, cover it with an integration test against a fake home + fake chezmoi
   source tree instead, asserting the write lands on the source and the apply is scoped
   to the home target — and say plainly on the bead which of the two was done.
4. Exercise the conflict path: modify the file externally after loading, then save, and
   confirm `XPromptWriteConflictModal` offers overwrite / reload / save-as and that each
   behaves.
5. Confirm the untargeted paths are unregressed: a plain single-pane prompt still sends
   on `<enter>` without a menu, and an untargeted multi-pane stack's menu is unchanged.

### Done when

`just check-full` and `just test-visual` pass, all three loops and the conflict path are
confirmed by observation, and any gap found is either fixed or filed via
`/sase_new_task`.
