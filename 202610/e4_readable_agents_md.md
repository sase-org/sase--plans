---
tier: epic
title:
  "E4 (instruction migration): Keep a readable AGENTS.md and retire redundant
  instruction copies"
goal: "The sase repo keeps one full, CI-checked AGENTS.md (plus its CLAUDE.md copy) that
  people, GitHub readers, and non-SASE tools can read. That file names where it comes
  from and how to see an agent's real bundle. The GEMINI.md, QWEN.md, and OPENCODE.md
  copies are retired, and the nested guides reach every provider. The TUI opens the
  bundle an agent actually received. A new decision record makes this the durable shape
  of E4.

  "
decisions:
  nested_guides:
    ask:
      Should the hand-written tools/, src/sase/ace/, and demos/tapes/ guides move into
      memory reference notes?
    choices:
      fold:
        Every provider sees their rules through the bundle; six nested files go and the
        repo tracks 2
      keep:
        The nested guides stay as files; Codex, Grok, and Muse still never see them; the
        repo tracks 8
    default: fold
    why: About 78% of runs (Codex, Grok, Muse) never load the nested files today
    answer: fold
  files_deck:
    ask:
      Should E4 also list each run's bundles in the Agents-tab Files deck and add the
      inspection commands to help?
    default: true
    why:
      It is the cheap in-product transparency fix the research recommends, on the same
      helper
    answer: true
  memory_decision_records:
    ask:
      Should E4 add the readable-AGENTS.md decision record and mark
      instructions-per-invocation superseded in part?
    memory:
      - decisions:readable-agents-md
      - decisions:instructions-per-invocation
    corrects: Native files remain committed until E4/E5
    default: true
    answer: true
  memory_vocabulary:
    ask:
      Should E4 rewrite the agent-instruction-file glossary term and add Instruction
      Bundle and Manifest terms?
    memory:
      - glossary:agent-instruction-file
      - glossary:instruction-bundle
      - glossary:instruction-manifest
    corrects:
      "sase supports one agent instruction file per supported agent CLI (ex: `CLAUDE.md`
      for claude, `GEMINI.md` for antigravity, etc...)"
    default: true
    answer: true
  memory_nested_notes:
    ask:
      If nested_guides is fold, should E4 add the tools, ace, and demos reference notes
      and merge Symvision text?
    memory:
      - tools.md
      - ace.md
      - demos.md
      - symvision.md
    requested:
      Can you help me implement the E4 epic for the "Memory-Built Instruction" work that
      is described in the memory_built_instruction_migration_epics.md and
      instruction_bundles_vs_readable_agents_md.md files?
    default: true
    answer: true
phases:
  - id: readable-projection
    title:
      Projection header, neutral contract wording, parity repo check, and a CI drift
      gate
    depends_on: []
    size: medium
    description:
      "readable-projection: add a short project-only header to the generated AGENTS.md,
      make the packaged contract wording true for files and bundles, make the parity
      repository check run on numbered headings, and stop CI regenerating committed
      instruction files before it checks them."
  - id: shim-retirement
    title: Retire the GEMINI.md, QWEN.md, and OPENCODE.md copies at project roots
    depends_on:
      - readable-projection
    size: medium
    description:
      "shim-retirement: canary-probe agy, Qwen, and OpenCode on AGENTS.md alone, shrink
      project and nested provider copies to CLAUDE.md with safe retirement in sase
      memory init, update every validator, inventory, history, snapshot, and Rust
      content-layout consumer (with the sase-core pin move), and delete the twelve
      copies."
  - id: nested-guides
    title: Nested tools/, ace/, and tapes/ guides reach every provider
    depends_on:
      - shim-retirement
    size: medium
    description:
      "nested-guides: per decision nested_guides, either fold the three hand-written
      nested guides into reference notes and delete the six nested files, or keep them
      with stale lines fixed."
  - id: launch-row
    title: The TUI opens the instruction bundle an agent received
    depends_on: []
    size: medium
    description:
      "launch-row: add a public launch-bundle locator, point the Memory lane's
      as-launched row at the recorded root bundle for explicit runs while old and native
      runs keep the AGENTS.md snapshot, and (decision files_deck) list bundles in the
      Files deck with help-text commands."
  - id: record
    title: Decision record, glossary, docs sweep, and acceptance record
    depends_on:
      - nested-guides
      - launch-row
    size: medium
    description:
      "record: write the readable-agents-md decision record and mark
      instructions-per-invocation superseded in part, rewrite and add glossary terms,
      correct the remaining E3-era docs, verify every exit criterion, and attach the
      acceptance record to the epic."
proposed_by: bbugyi200.athena.10r
decided_by: reviewer
decided_via: tui
create_time: 2026-10-16 15:15:51
status: wip
---

- **PROMPT:**
  [prompts/202610/e4_readable_agents_md.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/e4_readable_agents_md.md)

# Plan: E4 — Keep a readable AGENTS.md and retire redundant instruction copies

## Context

### Where this comes from

- **Roadmap (original E4):**
  `research:202610/memory_built_instruction_migration_epics/memory_built_instruction_migration_epics.md`,
  section "E4 Retire committed full instruction files in the sase repo", plus "Checklist
  for every epic plan". Read research with `sase artifact read`, never by opening files.
- **Re-scope (this plan follows it):**
  `research:202610/instruction_bundles_vs_readable_agents_md/instruction_bundles_vs_readable_agents_md.md`.
  It recommends **option C1**: keep E3's delivery, and keep a full, readable, CI-checked
  `AGENTS.md` in the repo. Read "Recommendation and next steps" (item 3 is this epic,
  item 4 is the `files_deck` decision) and "New findings from the lead".
- **E3** (epic `sase-1ko`, closed 2026-10-14; plan
  `plan:202610/e3_explicit_delivery_cutover.md`):
  - Claude, Codex, Grok, and Muse get the bundle once through explicit channels.
  - It has four default-on sunset flags: `claude_explicit_instructions` (`sase-1kv`),
    `codex_explicit_instructions` (`sase-1ks`), `grok_explicit_instructions`
    (`sase-1kt`), and `muse_explicit_instructions` (`sase-1ku`). Their beads are open,
    and **this epic does not touch them**.
  - Its acceptance record's "Notes for E4" assumed the stub-and-sync E4. This plan
    replaces those notes, as described below.
- **Already fixed after the research:**
  - `d070e6b2dc` fixed cdx Finding A (the Codex shadow home now follows the active
    delivery) and the Muse "bundle 2×" scoring.
  - `f2bedf6f4c` scores handoffs and foreign sessions from host records.
- **Still open, out of scope:** `sase-1l2` (READY). Export-mode renders leak the
  home-only `chezmoi` repo. Under C1 nothing publishes an export, so it stays separate.

### How this E4 differs from the roadmap's E4

| Roadmap E4 item                                                                                   | Disposition here                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sase instructions export` stub (≤80 lines), `export --check` in CI                               | **Dropped.** The full `AGENTS.md` stays. CI checks it with the existing `sase memory init --check` (`readable-projection`).                                             |
| `sase instructions sync`, gitignored projections, `AGENTS.override.md`, `just agent-instructions` | **Dropped.** There is nothing to sync, because the committed file is already readable.                                                                                  |
| Last-known-good store                                                                             | **Dropped.** The committed files remain the per-invocation fallback.                                                                                                    |
| Delete `GEMINI.md`, `QWEN.md`, `OPENCODE.md`                                                      | **Kept** (`shim-retirement`), for project and nested roots. Home roots wait for E5.                                                                                     |
| "Switch agy to prefix delivery in the same change"                                                | **Not done.** See design decision D4. agy reads `AGENTS.md` natively and has no switch to stop that, so a prefix bundle would double-load the contract.                 |
| Nested sets → path-scoped reference notes, merge Symvision text                                   | Decision `nested_guides` (default **fold**).                                                                                                                            |
| Memory History "stub era", retire `capture_instruction_snapshot`                                  | **Mostly dropped.** No stub era exists. The snapshot still records native loads for legacy and fallback runs. The "as launched" row moves to the bundle (`launch-row`). |
| Glossary rewrite plus _Instruction Bundle_ and _Instruction Manifest_ terms                       | **Kept** (`record`), under decision `memory_vocabulary`.                                                                                                                |
| Exit: `git ls-files` instruction files 20 → 2                                                     | **20 → 2** with `fold`, or **20 → 8** with `keep`.                                                                                                                      |

### Facts verified while planning (2026-10-16; sase `3c4d340935`; sase-core `594faf31`)

- **Tracked files.** There are 20 instruction files: the root `AGENTS.md` plus
  byte-identical `CLAUDE.md`, `GEMINI.md`, `QWEN.md`, and `OPENCODE.md`, and the same
  five in `tools/`, `src/sase/ace/`, and `demos/tapes/`.
  - The root file is a memory render, 19,126 bytes, with numbered headings
    (`## 1. Core Memory`, `#### 1.1.3 Repositories`).
  - The three nested `AGENTS.md` files are **hand-written**. No memory feeds them;
    `sase memory init` only copies each one into its four sibling shims
    (`main/init_memory/root_planning_files.py::agent_doc_shim_plans`).
- **Shim generation.**
  - `amd/constants.py::PROVIDER_SHIM_FILES = ("CLAUDE.md", "GEMINI.md", "QWEN.md", "OPENCODE.md")`
    is consumed by `amd/_shared.py::provider_shim_specs`, `amd/inventory.py`,
    `main/init_memory/tracking.py`, `memory/history/scopes.py`, and
    `axe/launch_evidence.py::capture_instruction_snapshot`.
  - The same names are hard-coded in `memory/inventory_models.py`
    (`INSTRUCTION_ROOT_FILENAMES`), `memory/history/pager_provider_core.py`
    (`_INSTRUCTION_NAMES`), `finalizers/commit_memory_guard.py`
    (`_GENERATED_ROOT_BASENAMES`), and `patch_stitch_audit.py`.
  - sase-core `crates/sase_core/src/content_layout.rs::agent_document_paths` lists all
    five names for the project, home, and chezmoi layouts. Only
    `tests/test_content_layout.py` consumes `agent_documents` in Python.
  - If the constant shrinks, nothing deletes the old copies: only `.tmpl` and
    `@AGENTS.md` legacy shims are cleaned up.
  - There is no git hook. Init runs through `sase memory init`, the Memory panel's
    publish action, and CI.
- **What each legacy CLI reads** (from the installed binaries' embedded docs):
  - **agy 1.4.0** reads both `GEMINI.md` and `AGENTS.md` (plus `.agents/rules/*.md`),
    walking from cwd up to the repo root. It has no flag to suppress rules and no
    system-prompt flag. With two identical root files it probably loads the contract
    twice today.
  - **Qwen Code 0.25.2** reads `[QWEN.md, AGENTS.md]` by default.
  - **OpenCode 1.19.3** never reads `OPENCODE.md`. It reads the first of `AGENTS.md`,
    `CLAUDE.md`, `CONTEXT.md`.
  - **Claude Code 2.1.301:** since 2.1.296, `instructionFiles` falls back to `AGENTS.md`
    when a directory has no `CLAUDE.md` (E3 plan).
  - So deleting the three copies removes duplicates and loses nothing.
- **Usage, last 30 days** (`sase instructions readout --since 30d`): muse 2,234, claude
  965, codex 951, grok 643, agy 106, qwen 0, opencode 0.
- **Nested guides in explicit runs.**
  - Claude excludes only the root and home files
    (`llm_provider/instruction_delivery.py::sase_owned_instruction_paths`), so it still
    lazy-loads nested `CLAUDE.md`.
  - Codex (`project_doc_max_bytes=0`), Grok (untrusted, `--rules` carries only the
    bundle), and Muse (no `--trust-workspace`) never see them. That is about 78% of
    runs.
  - The bundle compiler collects only `sase/memory` units.
- **Fallback dependencies.** On a render failure:
  - Claude reads the native `CLAUDE.md`.
  - Codex reads `AGENTS.md` plus `~/AGENTS.md`.
  - Grok's `--rules` uses the root `AGENTS.md`.
  - Muse trusts the workspace and reads `AGENTS.md`.

  No fallback reads `GEMINI.md`, `QWEN.md`, or `OPENCODE.md`. So C1, the full root file,
  keeps every fallback complete.

- **CI.**
  - The lint job in `.github/workflows/ci.yml` (step "Initialize SASE home") and in
    `master-gate.yml` runs `./.venv/bin/sase init memory --no-commit` **from the repo
    root**, before `just validate`. That rewrites the project files, which hides drift
    from `validate`'s `sase init memory --check`.
  - The test job does not regenerate. `tests/main/test_init_memory_committed_drift.py`
    (`project_actions == []`) catches drift there, so the gap is the lint job's masked
    check, not total blindness.
- **Parity bug.** `instructions/parity.py` uses `^##\s+Repositories\s*$`, which never
  matches the committed `#### 1.1.3 Repositories`. The repository-name parity check
  therefore never runs against the committed file.
- **"As launched" row.**
  - `ace/tui/widgets/prompt_panel/_agent_memory_versions.py::resolve_launch_row`
    resolves the workspace `AGENTS.md` from `agent_meta.instruction_snapshot`, even for
    explicit runs that never loaded it.
  - Recorded bundles live at `<artifacts>/instructions/NN-<provider>.md`, with a sibling
    `.json` manifest. Sequence numbers start at `00`.
  - `instructions/manifests.py::read_run_manifests` is public. `show.py`'s `_latest_seq`
    and `_check_bundle_integrity` are private.
  - The pin opener `ace/tui/actions/hints/_view_materialize.py::_snapshot_section` reads
    any path, but hard-codes `logical_filename="AGENTS.md"`.
- **Files deck.** `ace/tui/widgets/file_panel/_file_list.py::desired_file_pages` builds
  pages from commit diffs, the live diff, linked deltas, and `agent.extra_files`.
  - Availability (`decks/availability.py`) must stay free of I/O.
  - `extra_files[0]` is treated as the plan path, so new pages must be **appended**.
- **Memory History.** No Rust change is needed.
  - `subjects.rs` derives `instructions:{scope}/{dir}` subjects from the paths Python
    passes in (`memory/history/scopes.py`).
  - When a shim name leaves that list, its rows leave the pathspec cleanly.
  - A deleted nested set stops being discovered, so its subject disappears.

## Design decisions (owned by this plan; phase workers do not re-litigate them)

- **D1. C1, the runtime projection, stays committed.** The root `AGENTS.md` stays what
  `sase memory init` renders today. It is not the `mode: export` render (C2), which
  would weaken the Grok and Muse fallbacks and is blocked by `sase-1l2`.
- **D2. `CLAUDE.md` stays a byte-identical copy**, not an `@AGENTS.md` import and not
  deleted.
  - Epic `sase-5b` deliberately moved from imports to full copies.
  - Memory History folds equal blobs as aliases, while an import would add a permanent
    "diverged" row.
  - Older Claude Code versions don't fall back to `AGENTS.md`.
- **D3. The retirement is for project roots only.** Project roots, and the directory of
  any nested `AGENTS.md`, keep only `CLAUDE.md` beside `AGENTS.md`.
  - Home and chezmoi roots keep their four copies **until E5**. Deleting them there
    needs a `.chezmoiremove` entry and `chezmoi apply` on every host, which is E5's job.
  - Production lists shrink. Recognition lists (commit guard, pager detection) keep the
    retired names, so leftover copies in other projects are still classified as
    generated.
- **D4. agy, Qwen, and OpenCode stay on native `AGENTS.md` discovery**
  (`provider_legacy`). The re-scope report said to switch agy to prefix delivery. That
  assumed agy reads only `GEMINI.md` (the comment in `amd/constants.py`). It reads
  `AGENTS.md` too, and nothing suppresses that, so with a committed full `AGENTS.md` a
  prefix bundle would load the contract twice (violating R6). Qwen could be suppressed,
  but it has zero runs.
  - The decision record names the reopen trigger: a legacy CLI gains a suppression
    switch or a system-prompt channel, or Qwen or OpenCode gain real usage.
  - Workers record an agy explicit-channel spike as a `PROPOSED FOLLOW-UP:` note, for
    example on custom agents' `CustomAgentSystemPromptConfig`.
- **D5. No feature flag.** E4 changes generated files, docs, and a TUI row, not runtime
  delivery. Rollback is a git revert plus `sase memory init`.
- **D6. The header lives only in committed project-root files, never in bundles or home
  files.** It is an optional template variable, so custom `AGENTS.template.md` overrides
  still validate.
- **D7. Rust/Python split.** sase-core changes only
  `content_layout.rs::agent_document_paths`: the project layout drops the three names,
  and home and chezmoi are unchanged. The `sase-core-revision.txt` pin moves in
  `shim-retirement`. Composition stays in Python
  (`decisions:instructions-per-invocation`).
- **D8. `capture_instruction_snapshot` stays.** It still proves what native and fallback
  runs loaded.
- **D9. CI initializes only the home root.** The "Initialize SASE home" step runs from
  outside the checkout, so the lint job's `just validate` checks the committed project
  files as they are.

## Phase details

Every phase:

- reads the memory its area requires (`lint_and_test`, plus `tui` for `launch-row`, and
  `symvision` when symbols move or die);
- finishes with `sase tool run check`;
- regenerates instruction files only through `sase memory init`, never by hand;
- records discovered follow-up work as `PROPOSED FOLLOW-UP:` notes on its own phase
  bead, never as new beads.

Phases that change generated `AGENTS.md` content are serialized (`readable-projection` →
`shim-retirement` → `nested-guides`), so their regenerated files never conflict.

### readable-projection (medium)

1. **Header (D6).**
   - Add an optional variable (for example `projection_note`) to
     `src/sase/amd/templates/AGENTS.template.md`, directly under the H1. Keep it out of
     `_MANAGED_TEMPLATE_VARS` in `src/sase/amd/_template.py`.
   - Fill it only for project roots, never for home or chezmoi roots. Thread the root
     kind through the renderer.
   - The text is a blockquote of at most 4 rendered lines. It must say that:
     - the file is generated from `sase/memory/` by `sase memory init`, and readers
       should edit the notes rather than the file;
     - SASE-launched agents get a per-invocation bundle built from the same notes, plus
       the operator's home memory layer and a provider directive;
     - `sase instructions render` previews that bundle, and
       `sase instructions show NAME` shows what agent NAME received.
   - Prove that:
     - the header survives `number_agent_document_sections`, the anchor validation in
       `amd/memory_units.py`, prettier (`just fmt-md-check`), `legacy_parity`, H1
       readers (`amd/inventory.py`, `instructions/run_index.py`), and the description
       fallback in `instructions/sections_units.py`;
     - **it never appears in a compiled bundle**: add a test that
       `sase instructions render` output contains no header text.
2. **Neutral contract wording.** This is a code change to the packaged template.
   `sase/memory/sase.md` is generated from it and refuses direct edits.
   - In `src/sase/main/init_memory/templates/memory-sase.template.md`, replace "render
     into this file" and "inlined here and into every provider instruction shim" with
     wording that holds both in the committed file and in a bundle. For example: "render
     into your agent instructions (this project's `AGENTS.md` and each SASE agent's
     per-invocation bundle)". Do the same for the nearby "listed here" and "inlined
     here".
   - Run `sase memory init`. This regenerates `sase/memory/sase.md`, the instruction
     files, and the home files through the normal chezmoi path, as any template change
     does. Confirm `sase memory init --check` exits 0.
3. **Parity repository check.**
   - Make `instructions/parity.py`'s repository-section regex accept numbered and deeper
     headings (`#### 1.1.3 Repositories`), so the repository-name check really runs
     against the committed file.
   - Extend `tests/instructions/test_render_parity.py::test_parity_committed_agents_md`
     with a numbered-heading fixture case. If the now-active check finds a real gap, fix
     the cause; don't weaken the check.
4. **CI drift gate (D9).**
   - In both `.github/workflows/ci.yml` and `.github/workflows/master-gate.yml`, run the
     "Initialize SASE home" commands from outside the checkout (for example
     `working-directory: ${{ runner.temp }}` with absolute `.venv/bin/sase` paths). Home
     still initializes, the committed project files are never rewritten, and the later
     `just validate` checks them as committed.
   - First confirm that `sase init memory --no-commit` outside a project plans only the
     home root (`inputs.is_project_dir` is false) and that `sase skill init --force` is
     unaffected.
   - If home-only init is impossible, add the smallest home-only option following
     `cli_rules` (short alias, sorted options, complete help) and regenerate the CLI
     docs with `just fix`.
   - Add a regression test that parses both workflow files. It asserts that no step runs
     `sase init memory` without `--check` inside the checkout.
   - **Local proof**, recorded in a phase note:
     1. With `HOME=$(mktemp -d)`, run the step's commands from outside the workspace;
        `git status --porcelain` in the workspace stays empty.
     2. Append a line to `AGENTS.md`; `sase init memory --check` from the workspace
        exits non-zero, and the drift test fails.
     3. Restore the file.
5. **Docs.** In `docs/reference/agents/memory-initialization.md`, describe the header
   and state that CI checks the committed instruction files without regenerating them.
   Correct facts in place; legacy pages don't grow.

### shim-retirement (medium; crosses sase-core)

1. **Canary probes first.** Record them in a phase note headed `E4 PROBES:`, with CLI
   versions.
   - In a throwaway git repo outside any workspace, write only an `AGENTS.md` that tells
     the agent to answer a canary phrase. Ask each CLI for the canary, mirroring the
     adapter argv (`agy --print … --add-dir .`, `qwen` on stdin, `opencode run --dir`),
     with the cheapest model.
   - **agy is required.** Also run `GEMINI.md`-only, and both files with different
     canaries, to record whether agy loads both today.
   - Run Qwen and OpenCode if they're authenticated on the host. Otherwise cite the
     version-pinned binary evidence above.
   - If a CLI does **not** load `AGENTS.md` alone, keep that provider's file in the
     project set, record the exception in the note, and continue with the rest.
2. **Constants (D3).** Split the list so one place owns each role:
   - project and nested production (`("CLAUDE.md",)`);
   - home and chezmoi production (today's four, with an "until E5" comment);
   - retired names (`GEMINI.md`, `QWEN.md`, `OPENCODE.md`).

   Rewrite the `amd/constants.py` comment so it no longer says agy reads only
   `GEMINI.md`. Route `provider_shim_specs` and `agent_doc_shim_plans` by root kind.

3. **Safe retirement in `sase memory init`.**
   - At project roots, and beside every nested `AGENTS.md`, plan a `delete` for each
     retired file that is byte-identical to its sibling `AGENTS.md` (on disk, or as
     about to be rendered) or is a recognized legacy `@AGENTS.md`/`.tmpl` shim.
   - Leave a non-identical (custom) file alone and report it as a warning, never as a
     blocker.
   - `--check` reports pending deletions as drift. `main/init_memory/tracking.py` stops
     treating retired names as managed.
   - **Idempotence test:** run init twice. The second plan is empty and no retired file
     comes back. Add tests for the custom-file case, the home-root case (nothing
     retired), and a nested-root case.
4. **Consumers.**
   - Use the role-appropriate list in:
     - `amd/inventory.py` (`sase instructions list`: show a pending `retired` state or a
       kept `custom` file, and stop reporting retired names as `missing`);
     - `memory/history/scopes.py` (project subjects stop listing retired paths; home
       keeps them);
     - `axe/launch_evidence.py::capture_instruction_snapshot` (workspace versus home).
   - Make the hard-coded copies in `memory/inventory_models.py`,
     `memory/history/pager_provider_core.py`, `finalizers/commit_memory_guard.py`, and
     `patch_stitch_audit.py` use the constants. Keep recognition of retired names where
     the code classifies files rather than producing them.
5. **sase-core (D7).**
   - Open it with `sase repo open sase-core -r "<why>"` and work only in the printed
     path.
   - Make the project layout in `content_layout.rs::agent_document_paths` list
     `AGENTS.md` and `CLAUDE.md`. Keep the five for home and chezmoi until E5. Update
     the Rust tests, run `sase tool run check` there, and commit per that repo's
     obligations.
   - Then move `sase-core-revision.txt` past that commit (see `docs/rust_backend.md`),
     update `tests/test_content_layout.py`, and confirm
     `tools/check_sase_core_rs_bindings` passes.
6. **Tests and fixtures.** Update the shim-set assertions:
   - `tests/test_memory_inventory.py`
   - `tests/main/test_instructions_list_inventory.py`
   - `tests/main/test_init_memory_agent_docs.py`
   - `tests/main/test_init_onboarding_reporting.py`
   - the `test_init_memory_*` suites
   - `tests/ace/tui/modals/test_memory_pane_instructions.py`
   - the macro-terminology allowlists (`tests/_macro_terminology_docs.py` checks that
     listed files exist)
   - `tests/test_gemini_active_surface_guard.py`

   If the memory-pane visual snapshots derive shim rows from the constant, regenerate
   only those goldens with `just fix-tui-screenshots -- <selectors>`.

7. **Apply.** Run `sase memory init`. It deletes 12 files (3 copies × 4 directories).
   - `git ls-files | grep -cE '(^|/)(AGENTS|CLAUDE|GEMINI|QWEN|OPENCODE)\.md$'` prints
     `8`.
   - `sase memory init --check` exits 0.
   - `sase instructions render -p` exits 0.
8. **Docs.** Correct the shim lists in place in:
   - `docs/reference/agents/initialization.md`
   - `docs/reference/agents/memory-initialization.md`
   - `docs/reference/memory/memory.md`
   - `docs/reference/providers/agent-clis.md` (the provider-file table)
   - `docs/reference/config/core-behavior.md`
   - `docs/configuration.md`

   State that agy, Qwen, and OpenCode read `AGENTS.md`, and that home roots keep four
   copies until E5.

### nested-guides (medium)

> [!decision] nested_guides = fold
>
> 1. **Audit before moving.** Check every rule in `tools/AGENTS.md`,
>    `src/sase/ace/AGENTS.md`, and `demos/tapes/AGENTS.md` against the code. Drop or
>    correct stale ones. For example, `help_modal.py`'s `_BOX_WIDTH = 57` no longer
>    exists, and the Vim syntax file lives in the chezmoi repo.
> 2. **New reference notes** (decision `memory_nested_notes`). Use `/sase_memory_write`
>    first. Create `type: reference` notes whose descriptions are path triggers:
>    - `sase/memory/tools.md`: "Read before changing, adding, or vendoring anything
>      under `tools/`". It covers the `-YYmmdd` vendored-script rule, the `sase_bead`
>      wrapper, the ToolRun smokes, and the triage backtest.
>    - `sase/memory/ace.md`: "Read before changing anything under `src/sase/ace/`". It
>      covers the Patch suffix styling sync list, the footer keybinding convention, and
>      help popup and modal upkeep.
>    - `sase/memory/demos.md`: "Read before editing VHS tapes under `demos/tapes/`"; run
>      `just demos -y`.
> 3. **Symvision.** Move into `sase/memory/symvision.md` only facts it lacks (for
>    example, that the former public API whitelist file is obsolete). Drop the duplicate
>    section.
> 4. **Delete** `tools/AGENTS.md`, `tools/CLAUDE.md`, `src/sase/ace/AGENTS.md`,
>    `src/sase/ace/CLAUDE.md`, `demos/tapes/AGENTS.md`, and `demos/tapes/CLAUDE.md`.
>    `grep` the repo, docs, `sdd/`, and tests for references to those paths, and fix
>    each one.
> 5. Run `sase memory init`. Then confirm:
>    - the instruction-file count prints `2`;
>    - `sase instructions render -s` lists the three new reference entries;
>    - `sase memory init --check` exits 0.

> [!decision] nested_guides = keep
>
> 1. Keep the nested `AGENTS.md` and `CLAUDE.md` files as hand-written, repo-owned
>    guides. Fix stale rules in each `AGENTS.md`, then run `sase memory init` to refresh
>    the `CLAUDE.md` copies.
> 2. Replace the Symvision section of `tools/AGENTS.md` with a one-line pointer to the
>    `symvision` reference note. Make no memory edits.
> 3. In `docs/reference/providers/agent-clis.md`, state that nested guides are
>    repo-owned and that only Claude (lazily) and native-discovery providers load them.
> 4. The instruction-file count prints `8`.

### launch-row (medium; TUI)

Read the `tui` memory and its `tui_perf` and `tui_screenshot` children first.

1. **Locator.** Add a public helper to `src/sase/instructions/manifests.py`, for example
   `launch_root_bundle(artifacts_dir)`.
   - It returns the lowest-sequence non-helper `RunManifest` that has a bundle path,
     plus its delivery status.
   - Promote `show.py`'s integrity check into a shared helper rather than duplicating
     it.
2. **The "as launched" row.** In `_agent_memory_versions.py::resolve_launch_row`, when
   the launch manifest's root delivery is `explicit`:
   - build an "Instructions as launched · NN-<provider>" row from the bundle;
   - give it a chip showing the channel, size, and `bytes match` or `MISMATCH`;
   - make its pin open the bundle read-only, reusing `revision="snapshot"` plus
     `snapshot_path`. Pass a logical filename through `_view_materialize.py` instead of
     the hard-coded `"AGENTS.md"`.

   Runs with native or fallback delivery, and old runs without manifests, keep today's
   `AGENTS.md` snapshot or head row unchanged.

   Extend `MemoryLaunchRow` and its summary cache. All I/O stays on the existing worker
   path (`enrich_memory_versions`).

3. > [!decision] files_deck
   >
   > **Files deck.** Append the root and helper bundle files as static pages after the
   > existing pages, using a separate model field rather than `extra_files`.
   >
   > - Fill the field in the done loaders, and for running agents from a cached,
   >   worker-filled list, like the linked-delta groups. Keep `desired_file_pages` and
   >   deck availability free of I/O.
   > - Add `sase instructions render`, `show`, `verify -a`, and `readout` to the
   >   Agents-tab `?` help popup, keeping its formatting rules.

   > [!decision] files_deck = no
   >
   > Skip the Files deck and help-text work.

4. **Tests.**
   - Add unit tests for an explicit bundle, a fallback run, a pre-E3 run, and an
     integrity mismatch.
   - Add visual snapshots for the new row states (and the Files deck page) with
     `just fix-tui-screenshots -- <selectors>`.
   - The `tui_perf` budgets must hold.
5. **Docs.** Update the row description in `docs/reference/memory/history.md` and
   correct `docs/ace.md` in place (legacy page; don't grow it).

### record (medium)

1. **Decision record** (decision `memory_decision_records`). Use `/sase_memory_write`.
   - Add the strand `decisions:readable-agents-md` (slug `readable-agents-md`), titled
     "The Committed AGENTS.md Stays A Readable Projection". Follow the strand format:
     Claim, Why with rejected alternatives, Cost, Reopens when.
   - **Claim.** The repo commits the full runtime projection (`AGENTS.md` plus a
     `CLAUDE.md` copy), checked by CI. It is the readable standing instructions and the
     per-invocation fallback. SASE runs still get bundles. agy, Qwen, and OpenCode stay
     on native `AGENTS.md` discovery.
   - **Why.**
     - Transparency for people, GitHub readers, and non-SASE tools. `AGENTS.md` is also
       a cross-vendor convention.
     - 28 of the 29 bundle benefits don't need the file untracked.
     - Churn is small: 89 of 3,699 commits in 60 days.
     - The Codex "can't suppress" premise was false.
     - The full file keeps every fallback complete.
   - **Rejected alternatives.**
     - The export stub with `sync` and a last-known-good store (the roadmap's E4).
     - C2, committing the export render.
     - Prefix delivery for agy, because it would load the contract twice.
   - **Cost.**
     - Two artifacts to keep aligned. The CI drift gate and the parity test mitigate
       this.
     - Generated-file churn continues.
     - agy lacks the home layer.
   - **Reopens when.**
     - Merge conflicts on generated files measurably slow landings.
     - E7 audiences ship and a single file becomes misleading.
     - Outside contributors need a purpose-built stub.
     - A legacy CLI gains a suppression switch or a system-prompt channel.

   Link `[[decisions/instructions-per-invocation]]`.

2. **Supersession mark.** In `decisions/instructions-per-invocation.md`, set
   `metadata.status: superseded-in-part` and
   `superseded_by: decisions/readable-agents-md`. Add the leading blockquote back-link,
   following `decisions/record-before-admit.md`. It states that the Cost clause's "until
   E4/E5" retirement of committed native files, and the "until E4" legacy status of agy,
   Qwen, and OpenCode, are retired. Do not edit the accepted body otherwise.
3. **Glossary** (decision `memory_vocabulary`).
   - Rewrite `glossary:agent-instruction-file`:
     - `AGENTS.md` is canonical, with a `CLAUDE.md` copy;
     - agy, Qwen, and OpenCode read `AGENTS.md`;
     - `sase memory init` renders it from `sase/memory/`, with no post-commit-hook claim
       (there is no hook);
     - SASE agents get a bundle instead.
   - Add `glossary:instruction-bundle` and `glossary:instruction-manifest`. Distinguish
     the manifest from **Receipt**.
   - Run `just fix` to regenerate `docs/reference/glossary.md`, then `sase memory init`.
4. **Docs.**
   - `docs/reference/memory/instruction-bundles.md`: change the provider row for
     Antigravity, OpenCode, and Qwen to "native `AGENTS.md` discovery", and link the new
     record.
   - `docs/reference/providers/agent-clis.md`: update the legacy row.
   - `docs/contribute/instruction-bundle-internals.md`: replace "stay `provider_legacy`
     until E4" with a link to the record.
   - `docs/reference/providers/codex.md` "Codex instruction delivery": the stale
     paragraph says shadow-home suppression follows the flag. Correct it, since
     `d070e6b2dc` keys it to the active delivery. Fix the matching "Codex fallback
     limitation" link in `agent-clis.md`.
5. **Acceptance.**
   - Check every exit criterion below.
   - Attach a Markdown acceptance record to the epic with
     `sase bead attach <epic-id> <file> -N e4_acceptance_record.md -n "<summary>"`. It
     contains:
     - the sase and sase-core revisions and the pin;
     - CLI versions;
     - the `E4 PROBES:` results;
     - the CI local proof;
     - before and after `git ls-files` counts;
     - `sase memory init --check` and `render -p` output.
   - Also include **Notes for E5**:
     - home roots still write four copies and carry no header;
     - other projects drop their three copies and gain the header on their next
       `sase memory init`;
     - `sase-1l2` matters only if an export is ever published.
   - Record any failed criterion as `DISCOVERED ISSUE:` on this phase bead, and leave
     the bead open.

## Exit criteria (checked at landing)

- [ ] `sase tool run check` passes in sase and in sase-core. `sase-core-revision.txt`
      includes the content-layout commit, and `tools/check_sase_core_rs_bindings`
      passes.
- [ ] `git ls-files | grep -cE '(^|/)(AGENTS|CLAUDE|GEMINI|QWEN|OPENCODE)\.md$'` prints
      `2` (`fold`) or `8` (`keep`). The root `CLAUDE.md` is byte-identical to
      `AGENTS.md`.
- [ ] `head -8 AGENTS.md` shows the header, and a test proves no compiled bundle
      contains it.
- [ ] `sase memory init --check` exits 0. Running `sase memory init -C` twice leaves
      `git status` clean and recreates no retired file. The retirement tests cover
      custom files, home roots, and nested roots.
- [ ] `sase instructions render -p` exits 0, and a test proves the repository-name check
      runs on the numbered committed file.
- [ ] The CI local proof is recorded. The workflow regression test passes, and no
      workflow step regenerates committed project files before `just validate`.
- [ ] The `E4 PROBES:` note shows agy answering the canary from `AGENTS.md` alone (or a
      recorded exception that kept its file). Qwen and OpenCode are probed or covered by
      version-pinned evidence.
- [ ] The Rust project layout lists only `AGENTS.md` and `CLAUDE.md`. The home and
      chezmoi layouts are unchanged.
- [ ] The launch-row tests and visual snapshots pass for explicit, fallback, and pre-E3
      runs (and the Files deck page if `files_deck`).
- [ ] `sase memory read decisions:readable-agents-md` prints the record, and
      `instructions-per-invocation` shows superseded-in-part. The glossary changes are
      present, and `just fix` leaves no diff.
- [ ] The acceptance record is attached to the epic bead.

## Watch metrics (after landing; not landing criteria)

- agy declaration misses and memory reads
  (`sase instructions readout --since <landing>`) against the 30 days before. agy should
  now load the contract once instead of twice.
- With `fold`: audited reads of `tools.md`, `ace.md`, and `demos.md` in
  `memory_reads.jsonl` for runs that edit under those directories.
- `sase instructions verify` rows for claude, codex, grok, and muse stay at E3's
  after-state.
- The first stale commit caught by the lint job's `just validate`.
- Merge conflicts on generated instruction files (the record's reopen trigger).

## Confirm it yourself (about 5 minutes)

```bash
git ls-files | grep -cE '(^|/)(AGENTS|CLAUDE|GEMINI|QWEN|OPENCODE)\.md$'   # 2 (8 with keep)
head -8 AGENTS.md                                    # header: source, bundle, render/show
sase memory init --check && sase instructions render -p > /dev/null   # both exit 0
sase memory read decisions:readable-agents-md -r "confirm E4 outcome"
```

In the TUI, select an agent launched after E4. Its Memory lane leads with "Instructions
as launched", which opens the bundle.

## Expected scoreboard after this epic

It is unchanged from E3 for claude, codex, grok, and muse. agy stays `legacy` and `◌`
(unverifiable). Per the probes, its native contract load drops to 1× if it loaded both
files before.

## Rollback

Revert the phase commits and run `sase memory init`, which recreates any retired copies.
Revert the pin move together with the sase-core layout commit. There are no flags. The
four E3 sunset flags are unaffected.

## Memory edits

- `memory_decision_records`: the new `decisions:readable-agents-md`, and the
  supersession mark on `decisions:instructions-per-invocation` (`record`).
- `memory_vocabulary`: `glossary:agent-instruction-file`, and the new
  `glossary:instruction-bundle` and `glossary:instruction-manifest` (`record`).
- No decision is needed for `sase/memory/sase.md`. It is generated from the packaged
  template, which `readable-projection` edits as code; `sase memory init` regenerates
  the note.
- `memory_nested_notes` (only with `nested_guides = fold`): new `tools.md`, `ace.md`,
  and `demos.md`, and an edit to `symvision.md` (`nested-guides`).
- Nothing else. Generated files change only through `sase memory init`.

## Out of scope

- `sase instructions export`/`sync`, gitignored projections, the bootstrap clause, the
  last-known-good store, and untracking `AGENTS.md`. The record names the reopen
  triggers.
- `sase-1l2`, the export-mode home leak.
- Home files and their four copies, the bob-cli and actstat repos, and `sase tmux-agent`
  (E5).
- Answering the four E3 flag beads (FlagTriage after the soak).
- Explicit delivery for agy, Qwen, or OpenCode (D4); deleting `CLAUDE.md` in favor of
  Claude's `AGENTS.md` fallback (D2).
- E6's MEMORY deck, chips, and ledger; E7 audiences.
