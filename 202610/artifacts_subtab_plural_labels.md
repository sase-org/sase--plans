---
tier: tale
title:
  Pluralize the Artifacts sub-tab labels (Commits · Patches · Beads · Plans · Files)
goal:
  The Artifacts strip reads Commits · Patches · Beads · Plans · Research · Files, every
  label-derived string uses the right noun (no "planss", "researchs", or "stitche"), and
  ids, keys, refs, config, saved state, and digit shortcuts are unchanged.
size: medium
decisions:
  stitches_label:
    ask: What should the `stitches` sub-tab be labeled?
    choices:
      commits: Commits; the rows are VCS commits and ✦ marks the stitch-landed ones
      stitches:
        Stitches; keeps the SASE word in case this pane later lists commit-less stitches
    default: commits
    why:
      The report's pick; with sidecars shown ~80% of rows are auto commits, not stitches
    answer: commits
  decision_record:
    ask:
      Should the coder write a decisions:artifacts-tab-labels record with the label rule
      and reopen conditions?
    memory:
      - decisions:artifacts-tab-labels
    default: false
    answer: false
  nav_section_glossary:
    ask: Should glossary:nav-section name the Artifacts panes by their new labels?
    memory:
      - glossary:nav-section
    corrects:
      each pane's entry list (Agent, Stitch, Patch, Bead, a document provider such as
      Plan, or File) is one
    default: true
    answer: true
proposed_by: bbugyi200.athena.105
decided_by: auto
create_time: 2026-10-13 17:44:12
status: wip
---

# Plan: Pluralize the Artifacts sub-tab labels

## Context

The Artifacts sub-tabs currently show singular labels:
`Stitch · Patch · Bead · Plan · Research · File` (the Agent pane is retired behind
`artifacts_agent_pane_retired`). The research report
`research:202610/artifacts_subtab_naming/artifacts_subtab_naming.md` (also readable as
`sase artifact read file:explicit:2cbfd8c15c6ea92ec4dfc289 "<why>"`) recommends
`Commits · Patches · Beads · Plans · Files`. It rejects `PRs` and `Bugs`, and it
recommends fixing how labels are interpolated into prose so the change is safe.

This is presentation-only Python/Textual work in this repo. sase-core holds no tab-label
strings, and `ref.pane` is Python-only metadata: `_rust_ref_provider_spec()` strips it
before Rust validation. No sase-core change, no feature flag, and no migration are
needed.

**Why one tale, not an epic.** The report suggests landing the "plumbing" first with no
visible change. That doesn't work. Today, collection prose gets its plural by appending
`s` to a singular label (`f"No {label.lower()}s"`). Once that prose uses the label
verbatim, it reads "No plan" and "Refresh plan" until the labels flip. The plumbing and
the flip have to land together.

## Target labels

> [!decision] stitches_label = commits
>
> The `stitches` label is **Commits**. The Help section title is
> `Commits Pane (stitches)`, so typing `stitch` in the Help filter still finds the
> section. The empty-state title becomes "No commits". The palette prefixes
> `Stitches: …` become `Commits: …`. The binding descriptions `Show Stitches`,
> `Fetch Stitches`, and `Refresh Stitches` become `Show Commits`, `Fetch Commits`, and
> `Refresh Commits`.

> [!decision] stitches_label = stitches
>
> The `stitches` label is **Stitches**. The Help section title is `Stitches Pane`. Keep
> the empty-state title "No stitches", the `Stitches: …` palette prefixes, and the
> `Show/Fetch/Refresh Stitches` binding descriptions. Everything else in this plan still
> applies, including the description-summary fix and the item noun "commit".

| Pane id        | Today    | New label (collection)                    | Item noun (singular) |
| -------------- | -------- | ----------------------------------------- | -------------------- |
| `stitches`     | Stitch   | **Commits** (or Stitches, per decision)   | Commit               |
| `patches`      | Patch    | **Patches**                               | Patch                |
| `beads`        | Bead     | **Beads**                                 | Bead                 |
| `agents`       | Agent    | Agent (unchanged; retired pane)           | Agent                |
| `files`        | File     | **Files**                                 | File                 |
| `ref:plan`     | Plan     | **Plans** (builtin spec `ref.pane.label`) | Plan                 |
| `ref:research` | Research | Research (plugin-owned, unchanged)        | Research             |

**Unchanged (do not touch):**

- sub-tab ids, pane DOM ids, and CSS
- `show_artifacts_*` actions and keymap ids
- `default_config.yml` keys
- copy groups and query profiles
- persisted session state and saved queries
- `LEGACY_ARTIFACTS_SUBTABS`
- `@stitch:`/`@patch:`/`@bead:` refs and the `sase stitch|patch|bead` CLI
- tab order, icons, accents, and digit shortcuts
- text that names the domain concept: the `Patch:` detail header, "Focus persistent
  Patch filter", relation labels, the `✦ stitch` origin, and `Cycle stitches folding`

## Step 1: One label table, one item-noun table, and `item_noun` on the contract

1. In `src/sase/ace/tui/_artifact_tab_model.py`, add `ARTIFACTS_LABELS` and
   `ARTIFACTS_ITEM_NOUNS` beside `ARTIFACTS_ICONS`/`ARTIFACTS_ACCENTS`.
   - Key them by `stitches`, `patches`, `beads`, `agents`, `files`, and `ref:plan`,
     using the values in the table above.
   - The `ref:plan` entries are static fallbacks for paths that have no compiled
     contract.
   - Export both through the `artifact_tabs` facade.
2. Add `item_noun: str` to `ArtifactsPaneContract`.
   - Include it in `explanation_payload()` and in both presentation-digest payloads
     (`_presentation_digest` and `_presentation_digest_for`).
   - Mirror it onto `ArtifactsTabDescriptor.item_noun` in `attach_contract()`, the same
     way `label` is mirrored.
   - If `docs/artifacts_pane_contract.md` lists the `sase artifact pane show --json`
     keys, add `item_noun` there.
3. `compile_builtin_contract()` takes `item_noun` from
   `ARTIFACTS_ITEM_NOUNS[adapter.pane_id]`.
   - `fixed_descriptor()` in `_artifact_tab_descriptors.py` reads `ARTIFACTS_LABELS`
     instead of its inline dict.
   - The two internal callers that pass `label="Patch"` take
     `ARTIFACTS_LABELS["patches"]` instead: `actions/navigation/_tree.py` and
     `actions/patch/_relation_index.py`.
4. Providers: extract the title-cased-kind fallback from `_provider_label()` (with the
   `"Document"` fallback) into a shared `provider_item_noun(kind)`.
   - `compile_provider_contract()` sets `item_noun = provider_item_noun(kind)`. A spec
     cannot override it in this change.
   - `_provider_label()` keeps honoring a configured label and otherwise falls back to
     `provider_item_noun(kind)`, so unlabeled providers keep today's tab text.
   - Replace the duplicate `_provider_label` in `widgets/artifacts/plans_data.py` with
     the shared helper.
5. In `src/sase/artifact_providers/_builtin.py`, give `builtin_plan_ref_provider_spec()`
   a `"pane": {"label": "Plans"}` block under `ref`.
   - Do **not** add a top-level or `ref.label` key. Those keys reach Rust validation.
     `ref.pane` is stripped before it and compiled by
     `compile_provider_pane_presentation`.
   - Confirm that `sase artifact pane show ref:plan --json` reports `"label": "Plans"`
     and `"item_noun": "Plan"`, and that the provider spec digest is unchanged.
6. In `src/sase/ace/tui/artifact_tabs.py`, add `artifacts_pane_label(pane_id)` and
   `artifacts_pane_item_noun(pane_id)`. Each one tries these sources in order:
   1. map legacy ids through `LEGACY_ARTIFACTS_SUBTABS`
   2. the compiled contract, when one resolves
   3. the static tables
   4. the title-cased tail of the id

   Every id-derived site in Step 2 uses these two helpers.

## Step 2: Route every label interpolation through the right noun

**Rule:**

- A **collection** surface names the pane or its list. It uses `label` verbatim.
- An **item** surface talks about one row, or uses the word as a modifier. It uses
  `item_noun`, lowercased mid-sentence as the site does today.
- Never append `s` to either value, and never strip one.

**Collection sites (use the label):**

- `_artifact_tab_contract.py:238-251` (plan adapter and generic empty states).
  - Title: `f"No {label.lower()}"`.
  - Plan-adapter body:
    `f"No {label.lower()} match the current project scope and filters."` This keeps
    today's "No plans" text.
  - Generic-provider body: `f"No {item_noun.lower()} documents match …"`.
  - The degraded `f"{label} unavailable"` stays as it is.
- `_artifact_tab_presentation.py:343-370` `_compile_empty_state`. Default it from the
  _resolved_ pane label (the `ref.pane.label` override when present) plus the item noun,
  using the same forms as the generic provider. Thread `item_noun` into
  `compile_provider_pane_presentation`.
- `modals/help_modal/patches_artifact_bindings.py:298`: `f"Refresh {label.lower()}"`.
  Line 301: the plan-adapter title stays `f"{contract.label} Pane"`.
- `widgets/artifacts/plans_rendering.py:40` scope chip, `:157` heading `f"# {label}"`,
  and `:159` `f"# {label} unavailable"`. Also the `plans_pane.py:137` non-plan border
  title.
- `widgets/artifacts/shell.py:116` identity chip and `:233` degraded hero.
- `actions/clipboard/_artifacts.py` `_copy_label_for_artifacts_subtab`: use
  `artifacts_pane_label` and delete `_ARTIFACTS_COPY_LABELS` along with the
  `"File"`/`"Plan"` fallbacks.
- `actions/refresh_panel.py:213-215`: `f"Artifacts › {artifacts_pane_label(pane)}"`.
  This also fixes today's singular "Artifacts › Plan".
- These sites already read `descriptor.label`. They need no code change, only test
  updates:
  - tab strip: `widgets/artifacts/view.py:201`
  - palette `Show Artifacts: …`: `commands/catalog.py`
  - `Show …` bindings: `keymaps/bindings.py:46`
  - quickstart `Jump:`/`Cycle Artifacts:` rows: `widgets/tab_quickstart.py`
  - link-follow toasts: `_link_follow_helpers.pane_label`
  - `artifact_cli/pane.py:123`

**Item sites (use the item noun):**

- `patches_artifact_bindings.py`:
  - `:264` "Open selected {noun}"
  - `:267` "Open inline {noun} filter bar"
  - `:296` "Copy {noun} reference"
  - `:303` non-plan provider title `f"{item_noun} Documents"` ("Research Documents" is
    unchanged)
- `_artifact_tab_descriptions.py:232`: the fallback provider summary
  `f"{noun} documents contributed …"`.
- `plans_rendering.py`:
  - `:85` "{noun} docs"
  - `:157` "Loading {noun} documents…"
  - `:163` "No {noun} documents match…"
- `plans_list.py:168-172` and `:332`.
- `plans_navigation.py:290-291`: the reveal chip. The fallback noun is "Plan".
- `shell.py:203` "under the active {noun} filter" and `:213` "No {noun} entries yet."
- `actions/clipboard/_palette_artifacts.py`:
  - Replace `_artifact_context_label` with a label and a noun.
  - Use the noun in "No {noun} entry to copy", "No marked {noun} entries are visible",
    and "Selected {noun}".
  - Use the label in "{count} marked {label}" and in `unknown_context`.
  - Delete the `f"{…}s"` pluralizer and the `label[:-1]` singularizer, which today
    prints "Selected stitche".
- `actions/artifacts_navigation.py:175`: "No {noun} entry to mark". Today it prints "No
  Ref:Plan entry to mark".
- `actions/clipboard/_artifact_references.py:53,59`,
  `actions/clipboard/_artifact_target_support.py:31`,
  `actions/clipboard/_artifacts.py:247`, and `actions/artifacts_files.py:167`.

**Plans pane plumbing:**

- `PlansSnapshot` (`plans_data_models.py:92`), `plans_pane.py:88`, and the
  `plans_rendering.py`/`plans_list.py` defaults change to `provider_label="Plans"`.
- Add `provider_item_noun="Plan"` next to it, and pass it through to the item sites
  above.
- `widgets/artifacts/view.py:262,276` passes `descriptor.item_noun` alongside
  `descriptor.label`.

**Hard-coded label copies:** read these from `ARTIFACTS_LABELS` or the descriptor.

- Scope chips:
  - `commits_rendering.py:68` (`" Stitch "`)
  - `beads_rendering.py:56` (`label="Bead"`)
  - `files_rendering.py:69` (`" File "`)
  - `agents_options.py:304` and `agents_pane.py:102` (Agent, so no visible change)
- `widgets/patch_onboarding.py`:
  - Delete `_ARTIFACT_LABELS` and render `descriptor.label`.
  - Filter rows by `descriptor.id in _ARTIFACT_DESCRIPTIONS`.
  - Fix the "Stitch takes digit 1" comment.
- Help section titles in `patches_artifact_bindings.py`:
  - `:80` becomes `Patches Pane`.
  - `:102` follows the decision: `Commits Pane (stitches)` or `Stitches Pane`.
  - `:146` becomes `Beads Pane`.
  - `:193` stays `Agent Pane`.
  - `:212` becomes `Files Pane`.
  - Titles must stay at most 50 characters. Descriptions must stay at most 32
    characters; see `src/sase/ace/CLAUDE.md`.
- `patches_copy_bindings.py:27,69,111` become `Copy Mode · Commits`/`Stitches`,
  `· Beads`, and `· Plans`. Leave `· Other`.
- `help_modal/filter_model.py:58-59` docstring.
- `tab_quickstart.py:297`: `"Files: previous / next version."`.
- Per the decision: the `bindings.py:159,177-178` and `keymaps/metadata.py:82` binding
  descriptions, and the eight `Stitches: …` command labels in
  `commands/_app_metadata_nav.py:252-297`. The palette scores command ids, so typing
  `stitch` still finds them; add a test for that.
- Comments and docstrings that name the old labels (`artifact_tabs.py`,
  `_artifact_tab_model.py:57`, and the `commits_*` module docstrings), where they refer
  to the tab label rather than the concept.
- Leave the unused `widgets/artifacts/placeholder_pane.py` alone.

**Copy and discoverability:**

- `_artifact_tab_descriptions.py:36` stitches summary. It is false for `auto`/`manual`
  rows under either decision. Replace it with: "VCS commits across your projects; ✦
  marks the ones SASE landed as stitches inside a Patch."
- `_artifact_tab_contract_adapters.py:164-167`: the title becomes "No commits"
  (`commits` branch only).
- In `commands/catalog.py` `_iter_artifacts_subtab_commands`, add
  `f"artifact {item_noun}"` plus these extra aliases. Keep the existing `prs` and
  `provider_kind` aliases.
  - `stitches`: `stitch`, `commits`
  - `patches`: `patch`
  - `beads`: `bead`, `bugs`, `issues`
  - `files`: `file`
- No dual labels on the strip (for example, `Commits (stitches)`).

## Step 3: Tests

- `tests/ace/tui/test_artifact_tab_labels.py`:
  - Assert the new id→label and id→item-noun maps, and delete the "no label ends in s"
    assertion.
  - Assert that the `ref:plan` descriptor built from the builtin spec has label "Plans"
    and noun "Plan".
  - Assert that `ARTIFACTS_LABELS["ref:plan"]` agrees with that descriptor.
  - Keep the `_provider_label("plan", {}) == "Plan"` fallback assertion.
- New regression tests, parametrized over every fixed pane, `ref:plan`, and a fake
  unlabeled provider such as `research`:
  - Help rows and titles, empty states, palette Copy-as contexts, the plans
    loading/unavailable headings, and the refresh-panel label must contain no doubled
    plural (`planss`, `commitss`, `researchs`) and no `stitche`.
  - Item prose must use the singular noun.
- Update the label assertions in these files:
  - `tests/ace/tui/widgets/test_tab_quickstart.py`
  - `tests/ace/tui/test_artifacts_scaffold.py:308` (`"2 ◉ STITCH"`)
  - `tests/ace/tui/test_artifacts_copy_mode.py:62-63,84,96`
  - `tests/ace/tui/test_link_follow.py` and
    `tests/ace/tui/test_link_follow_hydration.py` (the `Bead` toasts)
  - `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_retired.py:74`
  - `tests/ace/tui/test_copy_as_palette_contexts.py:420-428`
  - `tests/ace/tui/widgets/test_changespec_onboarding.py:66-77`
  - `tests/ace/tui/modals/test_help_modal_guide.py:72,75`
  - `tests/ace/tui/modals/test_help_modal.py:29-38`
  - `tests/ace/tui/test_help_modal_filter.py` (the `Bead Pane` waits)
  - `tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py:103-104`
  - `tests/ace/tui/test_refresh_panel_dispatch.py:291` (`Artifacts › Plans`)
- Tests that pass a label in as fixture input (`test_contract_compiler.py`,
  `test_artifacts_shell.py`, `test_link_follow_toast.py`, and similar) stay
  self-consistent. Update them only where the new `item_noun` argument or field is
  required.
- Add a Help-filter test showing that `stitch` still keeps the stitches section
  (`commits` branch).

## Step 4: Docs

- `docs/ace.md`:
  - `:124`, `:151` (the digits stay 1/2/3; say so), `:168`, `:301`, `:307`, `:815-823`,
    and `:923`.
  - Rename the headings `### Bead Pane` → `### Beads Pane` and `### File Pane` →
    `### Files Pane`.
  - Fix the anchors: `docs/ace.md:2123` and `docs/cli.md:854` (`#file-pane` →
    `#files-pane`).
  - Fix the stale digit at `:643`. Beads is `3`.
- `docs/getting_started.md:130,199`. Also fix the stale digit `2`; it should be `1`.
- `docs/artifacts_pane_visual_grammar.md:3,36`.
- `docs/artifacts_pane_contract.md`: state the convention that a provider's
  `ref.pane.label` is the plural or mass noun for its collection, and that the item noun
  is the title-cased ref kind.

## Step 5: Visual goldens

`AcePage` opens on the Artifacts tab, so the strip appears in a few hundred PNGs.

- Before running anything, read `lint_and_test.md` and `tui_screenshot.md` with
  `/sase_memory_read`.
- Run the full `just fix-tui-screenshots` through `/sase_monitor` with
  `TESTING`/`TESTED`.
- Inspect the report:
  - Expect no creations or removals.
  - Update groups should differ only in strip labels, chips, help or onboarding text,
    and the Commits description.
  - Expand any group that differs in anything else.
- Wider plural labels push the strip to a narrower tier at 140, 150, and 180 columns
  with six tabs. That cost is accepted.

## Step 6: Memory

> [!decision] nav_section_glossary
>
> Use `/sase_memory_write`. In the `glossary:nav-section` strand, change "each pane's
> entry list (Agent, Stitch, Patch, Bead, a document provider such as Plan, or File)" to
> name the panes by their new labels: Agent, Commits (or Stitches), Patches, Beads, a
> document provider such as Plans, or Files. Then run `sase memory init`.

> [!decision] decision_record
>
> Use `/sase_memory_write` to add the strand `decisions:artifacts-tab-labels`,
> "Artifacts Tabs Name Their Rows". Then run `sase memory init`.
>
> - **Claim:** a tab's label names what its rows are. Use the common word when it names
>   the rows precisely. Keep the SASE word when the common word is narrower, wrong, or
>   already means something else in SASE. Labels are plural or mass nouns. Prose uses
>   the label for the collection and the item noun for one row.
> - **Why:** rows are VCS commits; Patch and Bead are the precise identities. `PRs` is
>   one forge's word, and only 3.5% of beads are bugs.
> - **Cost:** "stitch" and "commit" both appear near this pane.
> - **Reopens when:**
>   1. this pane lists commit-less stitches (reopens Commits)
>   2. SASE adopts provider-supplied forge nouns (reopens PRs)
>   3. the Beads pane's "External issue" UI is renamed to "tracker issue" (reopens
>      Issues)

## Step 7: Follow-up

The report says that at 140+ columns `#artifacts-split-spacer` grows from 8 to 33 cells,
leaving the strip less room than at 130 columns. Confirm this, then file it with
`/sase_new_task` as a `bug` task bead. Do not fix it here.

## Verification

- Run `just fix` first, then `sase tool run check`. Run `just check-full` only if
  explicitly told to.
- `sase artifact pane show stitches --json` should report label `Commits` (or
  `Stitches`), item noun `Commit`, and an empty-state title of "No commits" (or "No
  stitches"). `ref:plan` should report `Plans`/`Plan`/"No plans". `ref:research` should
  still report `Research`.
- Optional: `sase screenshot -o /tmp/artifacts.png` at 120×40 to see the live strip.
- In the final summary, say that digit shortcuts are unchanged. The last relabel
  renumbered them silently.
