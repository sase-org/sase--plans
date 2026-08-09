---
tier: tale
title: Document Meta Muse Code as a supported agent CLI
goal:
  All current supported-agent CLI documentation includes Muse Code without implying
  auto-detection.
size: medium
proposed_by: bbugyi200.athena.vr
create_time: 2026-08-08 11:36:53
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.vr](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.vr.md)
- **COMMITS:**
  - [125b5c3](https://github.com/sase-org/sase/commit/125b5c31b23a147d1e65445a7c8a1ff862b4f543)
    — docs: document Muse Code provider support

# Document Meta Muse Code as a supported agent CLI

## Goal

Make every current, user-facing enumeration of SASE's supported coding-agent CLIs
include Meta's Muse Code, while preserving the operational distinction that Muse is an
explicit-only provider and is never auto-detected from the generic `muse` executable
name.

## Context and constraints

- Muse Code is already a built-in provider. The dedicated provider, LLM, plugin,
  configuration, ACE, XPrompt, commit-workflow, and getting-started references already
  document core Muse behavior.
- The canonical provider guide names the CLI `muse`, links Meta's Muse Code
  documentation, documents installation through `sase agent-cli install muse`, and
  explains explicit selection through `llm_provider.provider: muse`,
  `%model:muse/<model>`, or `SASE_MUSE_PATH`.
- High-level examples must not suggest that a Muse-only installation will work with an
  unqualified `sase run`: Muse is intentionally excluded from auto-detection because its
  executable name is generic.
- Do not alter scenario-specific copy that accurately describes the existing
  three-provider fan-out demo, the historical five-command tmux menu, or other
  non-exhaustive examples. Do not rewrite changelog history.
- Do not edit canonical memory notes or generated provider instruction shims; the user
  did not request a memory update.

## Implementation

1. Update the repository landing and installation paths.
   - In `README.md`, add Meta's Muse Code to the opening supported-agent sentence, the
     Quick Start prerequisite list, and the **Works with your agents** table. Link the
     Muse row to the same canonical Meta documentation used by
     `docs/agent_providers.md`.
   - Add concise copy near the Quick Start/table explaining that Muse must be selected
     explicitly, and show an appropriate `%model:muse/muse-spark-1.2` variant of the
     first-run command so a Muse-only reader is not sent down a failing auto-detection
     path.
   - In `INSTALL.md`, add `muse` (Muse Code) to the required coding-agent CLI inventory
     and state that it requires explicit provider/model selection. Leave the
     `node`/`npm` row unchanged because Muse does not use npm.

2. Make the guided onboarding path safe for Muse-only users.
   - Keep the existing Muse entries in `docs/getting_started.md`, but add the
     explicit-only caveat before the first launch and provide a Muse-qualified form of
     the Step 3 command alongside the auto-detected-provider form.
   - Point readers to `docs/agent_providers.md` for installation, authentication, and
     the complete selection options rather than duplicating the detailed provider
     reference.

3. Correct exhaustive provider lists in the documentation/blog corpus.
   - In `docs/blog/posts/structured-agentic-software-engineering.md`, preserve the
     historical description of the author's five-command tmux menu, but revise claims
     that SASE itself supports only those five. Add `muse`/Muse Code to the bundled CLI,
     provider badge, skill deployment target, and installation enumerations; explain
     that the normal auto-detection order still contains only the five unambiguous CLI
     names and that Muse is selected explicitly.
   - In `docs/blog/posts/hello-sase-your-first-15-minutes.md`, add Muse Code to the
     prerequisite and provider-boundary lists and include the same short explicit-only
     first-launch guidance used by the primary getting-started guide.
   - In `docs/blog/posts/commit-workflows-plugins.md`, add Muse Code wherever the post
     exhaustively describes runtimes that share commit finalization and generated Git
     commit skills.
   - In `docs/blog/posts/why-coding-agents-need-orchestration.md`, add Muse Code to the
     introductory provider list and its embedded provider-CLI architecture brief.

4. Bring the pending provider diagram brief up to date without generating a new image.
   - Rename `docs/images/blog/one_prompt_five_clis.prompt.md` to a count-neutral
     provider-diagram filename, and update its title, target asset path, alt text,
     composition, lane count, provider list, deterministic labels, and Muse-specific
     visual cue for six supported CLIs.
   - Update the corresponding placeholder reference and wording in
     `docs/blog/posts/structured-agentic-software-engineering.md`.
   - Do not generate or add a raster asset; the brief explicitly records image
     generation as follow-up work.

5. Re-audit the finished prose for completeness and precision.
   - Search tracked Markdown for exhaustive five-provider lists,
     `five bundled agent CLIs`, and required/authenticated CLI inventories that omit
     `muse` or Muse Code.
   - Review every remaining hit and retain only intentional historical,
     scenario-specific, non-exhaustive, or generated-history cases.
   - Confirm all new high-level Muse mentions use the product name consistently and do
     not imply auto-detection.

## Validation

1. Run `just install` so the workspace dependencies and editable package are current.
2. Run `just fmt-md-check` to verify Markdown formatting.
3. Run `just docs-check` to build the MkDocs site in strict warnings-as-errors mode and
   catch broken links or renamed-brief references.
4. Run `just check`, as required for file changes in this repository, and investigate
   any scoped-test escalation or unusual selection result according to the project
   instructions.
5. Re-run the tracked-Markdown audit after formatting/building and manually inspect the
   rendered README tables, Muse-qualified command examples, and all intentional
   exclusions.

## Completion criteria

- README clearly lists Meta's Muse Code and gives Muse-only users a working explicit
  first-run path.
- Installation and onboarding docs list all six real built-in agent CLIs without
  claiming Muse is auto-detected.
- Current and draft docs contain no stale exhaustive five-provider claim about SASE;
  intentional demo and historical statements remain accurate.
- The provider diagram brief represents six CLIs under a count-neutral filename, with no
  raster generation included in this change.
- Markdown formatting, strict docs build, and `just check` all pass.
