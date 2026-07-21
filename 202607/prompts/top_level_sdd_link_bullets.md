---
plan: '[../202607/top_level_sdd_link_bullets.md](../top_level_sdd_link_bullets.md)'
---
 %xprompts_enabled:false
# Previous Conversations

You are forking from 1 prior source. Source sections are independent parents, and section order carries no priority. Members inside an agent family section are sequential: each member continued the previous member's work. Carry forward relevant goals, constraints, decisions, and unfinished work with attribution when it matters. The New Query is the active request and takes precedence over conflicting source instructions.

## Source 1 of 1 — agent family `gz`

- **Members shown:** 2 of 2 (sequential chain, oldest first)

Family members ran as one sequential chain: each member continued the previous member's work, and the last member reflects the family's final state. These are transcripts of prior agents' conversations, not your own — attribute decisions to the named member when it matters.

### Member 1 of 2 — agent `gz--plan`

- **Outcome:** `completed` · **Model:** `codex/gpt-5.6-sol` · **Launch:** `20260721084340`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-gz__plan-260721_084340.md`

**User:**

I would like to start using a markdown link for the `prompt` (used in
the frontmatter of plan files) and `plan` (used in the frontmatter of prompt
files) properties. Can you help me implement this?

- The goal is for me to be able to jump from one page to another when viewing
these files on GitHub.
- For example, consider the shared_word_completion_min_length.md plan file
hosted at
https://github.com/sase-org/sase--plans/blob/main/202607/shared_word_completion_min_length.md
( shows what this page looks like on GitHub). After this change, I would
like the user to be able to click on
`202607/prompts/shared_word_completion_min_length.md` from that page to
navigate to the
https://github.com/sase-org/sase--plans/blob/main/202607/prompts/shared_word_completion_min_length.md
page. They should then be able to click on
`../202607/shared_word_completion_min_length.md` from that page (note that we
don't include the `../` currently, but you should add this) to navigate back
to the plan file page on GitHub (i.e. the page we were on before clicking on
`202607/prompts/shared_word_completion_min_length.md`).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

**Assistant:**

*Plan submitted for review.*

**Plan file:** `/home/bryan/.sase/plans/202607/clickable_sdd_frontmatter_links.md`

> # Plan: Make SDD prompt and plan frontmatter links clickable
> ## Context and outcome
> SASE currently stores `prompt` and `plan` frontmatter values as plain strings rooted according to the active SDD layout.
> GitHub presents those fields in its rendered frontmatter table, but plain strings are not navigable. The same values
> also serve as machine references in SDD validation, repair, prompt search, plan search, and the commit-time path that
> attaches a prompt to a copied plan, so wrapping them in Markdown without teaching those consumers about the new shape
> would turn a valid pair into a literal nonexistent filename.
> Make Markdown links the canonical write format while retaining historical plain paths as a read-compatible format. For
> the flat plans-sidecar example in the request, the stored values should be equivalent to:
> | Source artifact             | Field    | Markdown label shown by GitHub | Link target resolved from the source file |

*See full plan file for details.*

### Member 2 of 2 — agent `gz--code`

- **Outcome:** `completed` · **Model:** `codex/gpt-5.6-sol` · **Launch:** `20260721085335`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-260721_084340.md`

**User:**

@sase/repos/plans/202607/clickable_sdd_frontmatter_links.md

The above plan has been reviewed and approved. Implement it now.

**Assistant:**

I’ll open the approved plan through the repository workflow required for sidecar repos, then implement it in this checkout and run the mandated install/check verification.

The repository skill requires an audited `sase repo open` before reading the plan. I’m resolving the plan sidecar now; after that I’ll treat the approved document as the implementation contract.

The plan crosses the documented backend boundary: the link-value contract and plan projection belong in `sase-core`, while serialization, migration, CLI behavior, and docs belong here. I’m opening the linked core checkout through the same audited workflow before inspecting either implementation surface.

The existing code is clean and already has a centralized SDD link/repair module plus a Rust-backed plan reader, so the change can stay focused: one shared parser/renderer contract, one source-aware path adapter, then route writers, validation, repair, search projection, and docs through it. I’m now mapping the exact fixtures and serialization expectations before editing.

The shared contract is now fully specified from the approved plan: canonical values are exact single inline links; plain non-empty values remain legacy paths; Markdown-like but malformed values are explicitly classified invalid; and canonical targets must be relative POSIX paths. I’m applying that contract in Rust first so every Python consumer uses the same classification.

The Rust contract, Python adapter, writers, validator/repair path resolution, both search projections, commit-time inference, and documentation templates are wired. I’m doing an early Rust compile/test pass now; this catches API and binding mistakes before I update the broader Python regression expectations.

The focused Rust suite is green, including the PyO3 binding parity test. I’ve also raised SASE’s minimum core dependency to the next release line because the new Python adapter requires those newly added bindings. Next I’m rebuilding the local extension with `just install`, then I’ll run the focused Python tests and close any behavioral gaps they expose.

The local core checkout reports its unreleased source version as `0.8.0`, so the install emits the expected release-window warning while still building from source; release-plz owns that Rust version and the core repo explicitly forbids manual version edits. The build itself is continuing normally.

The first Python pass found four fixture-level issues, not implementation failures: one test selected the reverse-link diagnostic before the missing-href diagnostic, one helper emitted unquoted Markdown into YAML, and the installed-core validation tool maintained its own explicit binding list. Those are corrected; I’m rerunning the focused suite now.

Focused Python coverage is now fully green (114 tests), including exact YAML for all three layouts, mixed-format validation, broken href diagnostics, search/list projection, repair dry-run/write/idempotence, and physical target resolution. I’m moving to the full Rust formatting/clippy/test gate before the repository-wide `just check`.

The full Rust gate is green: formatting, clippy with warnings denied, all workspace tests, binding tests, and doc tests. The post-format focused Python suite remains 114/114 green. I’m starting the mandated repository-wide `just check` now, including SASE/committed-plan validation and the complete Python/visual suite.

`just check` reached Symvision and found one ownership issue: the public Python renderer is only used inside its adapter module. The project instructions require the audited Symvision memory procedure before fixing this class of lint failure, so I’m pausing the check to consult that guidance, then I’ll make the narrow visibility fix and rerun the full gate.

The Symvision memory confirms the proper fix is to make the renderer private because it has a real in-module caller but no production cross-module consumer; test-only imports must not keep a public API alive. I’m applying exactly that visibility change, with binding parity still covered in Rust and through the canonical Python builder.

The second full check now passes every lint stage and SDD link validation, but initialization detects two kinds of generated drift:

- In scope: the plans sidecar README must be regenerated from the template changed by this feature.
- Unrelated: five installed `sase_plan` provider skill files differ in the external chezmoi store.

I’ll update only the in-scope sidecar README through the already-audited plans checkout, then rerun validation. I will leave the unrelated provider-skill drift untouched.

The in-scope generated README is now current, and `plan links validate` passes. Initialization is blocked solely by the five unrelated chezmoi provider-skill files, so `just check` cannot proceed past that gate without expanding scope. I’m running the two remaining check stages directly—committed-plan validation and the full Python/visual test suite—to complete verification without touching that external drift.

The full 20,232-test run completed with two failures:

- One in-scope expectation still asserted the old core minimum (`0.11.0`); I’ll update it to `0.12.0`.
- One unrelated TUI concurrency test observed a duplicate deep-archive fetch under the 28-worker run; I’ll rerun it in isolation to determine whether it is the known timing flake rather than alter unrelated TUI code.

All 20,224 other tests, including the visual suite, passed.

The implementation is complete; I’m finishing the last full verification pass and checking that only the intended files changed. One repository-wide validation gate still has unrelated pre-existing drift in five installed `sase_plan` skill copies, which I’ll keep separate from this change.

No verification process is still running. I’m rerunning the full test suite after the final binding-version assertion update; after that I’ll run diff hygiene checks across the main, core, and plans repositories.

The full suite has reached 99% without a failure so far. I’m waiting for the final result, then I’ll inspect the three worktrees and run whitespace/error checks.

The full suite is clean: 20,226 passed and 7 skipped. I’m now auditing the final diffs—including the Rust core contract and generated plans README—and checking for whitespace or accidental changes.

The diff audit is clean across all three repositories, with only the intended main, Rust core, and plans README changes present. I’m running the mandated `just check` one final time; its code, type, Rust, and SDD gates should pass before it reaches the known unrelated installed-skill drift.

All implementation-specific checks passed: formatting, Ruff, mypy, script lint, Symvision, size lint, Rust checks, and SDD link validation. The umbrella check stops only at `init --check` because of the five unrelated `sase_plan` provider skill files; I’m doing one final status check now and will leave those external files untouched.

Implemented the approved plan across SASE, `sase-core`, and the plans sidecar.

Key outcomes:

- New Rust/Python contract for canonical `[label](relative-target)` frontmatter links.
- Writers and commit hooks now emit source-relative clickable links.
- Validation, repair, list, and search support both canonical and legacy links.
- `sase plan links repair --write` migrates legacy links idempotently.
- Updated schema guidance, documentation, templates, and plans README.
- Added comprehensive Rust and Python coverage.

Verification:

- `just test`: 20,226 passed, 7 skipped.
- `just rust-check`: passed.
- Committed-plan validation: 2,999 files, no errors or warnings.
- Formatting, Ruff, mypy, Symvision, and diff checks passed.

`just check` reaches SASE initialization validation, then stops only because five unrelated installed `sase_plan` provider skill files in chezmoi are stale. Those external files were left untouched. No commit was created.

The finalizer confirms these are the implementation changes from this session, so I’m using the required `sase_git_commit` workflow. I’ll commit the core contract, plans README, and main integration separately, checking each repository is clean afterward.

The diff audit matches the approved feature exactly, including the two new Python files and the new Rust module; all three repositories pass whitespace checks. I’m committing the dependency layer first so the main integration can reference a committed core revision.

The core feature and regenerated plans documentation are committed and pushed, and both external repositories are clean. I’m now committing the main SASE integration with the finalizer’s explicit file list; the wrapper will safely reconcile the two upstream commits reported by the initial status check.

Committed and pushed all implementation changes:

- Main: `8e7851ede` — `feat: add clickable SDD frontmatter links`
- sase-core: `b741b42` — `feat: add SDD frontmatter link contract`
- Plans: `21478511` — `docs: document clickable SDD frontmatter links`

All three repositories are clean and synchronized with their upstream branches.

---

%xprompts_enabled:true
# New Query

 It looks like markdown links inside of frontmatter are not supported by GitHub (see #sshot). Can you help me instead start defining these fields as bullets at the very top of the file instead of as frontmatter properties?

- This bullet should be placed above the H1 header if one exists and a single blank link should be inserted after this bullet.
- This bullet should have the following form, where `<type>` is either `PLAN` or `PROMPT` and `<link>` is the appropriate markdown link: `- **<type>:** <link>`

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
