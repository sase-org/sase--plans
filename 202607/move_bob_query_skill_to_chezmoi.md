---
create_time: 2026-07-10 13:11:45
status: done
prompt: 202607/prompts/move_bob_query_skill_to_chezmoi.md
tier: tale
---
# Move and Rename the Bob Query XPrompt Skill

## Goal

Move Bryan's Bob-vault query skill out of SASE's package-owned xprompt templates and into the chezmoi-managed personal
xprompt catalog. Rename the skill from `/bob_dataview` to `/bob_query` at the same time, update it to invoke the
already-renamed `bob query` CLI command, regenerate every provider's `SKILL.md`, and remove all obsolete `bob_dataview`
generated skill directories.

This is a hard skill rename: no `/bob_dataview` alias or `bob dataview` command reference should remain. “Dataview” may
remain where it names the query language or Obsidian engine rather than the removed CLI command or skill.

## Current State and Constraints

- SASE currently owns the source template at `src/sase/xprompts/skills/bob_dataview.md` and tests it as a shipped skill.
- The SASE `reads` xprompt, its documentation, and its swarm test explicitly request `/bob_dataview`.
- Chezmoi currently contains seven generated `bob_dataview/SKILL.md` targets: the primary Claude, Codex, Gemini,
  OpenCode, and Qwen locations plus Gemini's Antigravity CLI and Jetski locations.
- Personal file xprompts are discovered from `~/.xprompts`; the corresponding chezmoi source location is
  `home/dot_xprompts/`. A file there takes precedence over package defaults and can use `skill: true` to participate in
  `sase skill init`.
- Generated provider `SKILL.md` files must not be edited by hand. They are rendered from xprompt sources with
  `sase skill init --force` and then deployed with chezmoi.
- Because `use_chezmoi` is enabled, an unqualified skill regeneration can commit, push, and apply automatically. Use
  `--no-commit` during implementation so both repositories remain reviewable and no implicit commit or push occurs.
- Regeneration discovers the live `~/.xprompts` catalog, not the encoded chezmoi source path. The new personal source
  therefore has to be applied to `~/.xprompts/bob_query.md` before regeneration.

## Implementation Plan

### 1. Transfer source ownership and rename the personal skill

- Delete `src/sase/xprompts/skills/bob_dataview.md` from the SASE package so new SASE installations no longer ship a
  Bryan-specific Bob-vault skill.
- Add `home/dot_xprompts/bob_query.md` to the chezmoi repo, preserving the existing read-only Dataview-query guidance
  while updating:
  - the frontmatter name to `bob_query`;
  - the description and instructions to invoke `bob query`;
  - the example command to use `bob query --format markdown --query-file ...`.
- Keep the safety and output-format rules intact: the skill remains read-only, defaults to Bryan's Bob vault, prefers
  query files for multiline DQL, and treats command errors as diagnostics rather than permission to scan the vault.

### 2. Update SASE-owned consumers and tests

- Change both skill references in `xprompts/reads.md` from `/bob_dataview` to `/bob_query`; leave Dataview query syntax
  and prose about the Dataview result table unchanged.
- Update `docs/development.md` to document `/bob_query` as the skill used by the reads research agents.
- Update `tests/test_xprompt_swarm_local_helpers.py` to require `/bob_query` in each expanded research segment.
- Remove the `bob_dataview` case from `tests/main/test_init_skills_sources.py`, because the skill is no longer a shipped
  SASE package source. Continue using the existing generic/config-overlay skill-generation tests to cover personal
  `skill: true` sources.

### 3. Deploy the personal source, regenerate skills, and remove stale outputs

Perform regeneration only after the source edits above are in place:

1. Run `just install` in the SASE repo so the workspace CLI sees the deleted package template and updated sources.
2. Apply the new chezmoi source narrowly to `~/.xprompts/bob_query.md`, making it discoverable without applying
   unrelated dotfile changes.
3. Run `sase skill init --force --no-commit`. This must generate `bob_query/SKILL.md` for all registered providers into
   the chezmoi source tree without committing, pushing, or applying automatically.
4. Remove the seven obsolete chezmoi source directories and verify the generator did not recreate any of them:
   - `home/dot_claude/skills/bob_dataview/`
   - `home/dot_codex/skills/bob_dataview/`
   - `home/dot_config/opencode/skills/bob_dataview/`
   - `home/dot_gemini/skills/bob_dataview/`
   - `home/dot_gemini/antigravity-cli/skills/bob_dataview/`
   - `home/dot_gemini/jetski/skills/bob_dataview/`
   - `home/dot_qwen/skills/bob_dataview/`
5. Confirm the corresponding seven `bob_query/SKILL.md` files were generated, including the generated
   `sase skill use bob_query` audit directive and `bob query` example.
6. Apply the new personal xprompt and generated `bob_query` targets with chezmoi. Explicitly remove the corresponding
   live `~/.*/skills/bob_dataview` files/directories as well: once their chezmoi sources are deleted, a normal apply
   does not guarantee that formerly managed targets are removed.

The generated files should be accepted exactly as rendered except for formatter-driven changes. Do not manually copy or
patch one provider and assume the others match.

### 4. Verify both repositories and the deployed catalog

- In the SASE repo:
  - run the focused shipped-skill and reads-swarm tests;
  - search `src/`, `xprompts/`, `docs/`, and `tests/` for `bob_dataview`, `/bob_dataview`, and `bob dataview` to ensure
    active package references are gone;
  - finish with the required `just check` after `just install`.
- In the chezmoi repo:
  - run `sase skill init --check` to detect generated-file drift;
  - run the Markdown formatting check for the source and generated skill files;
  - use path and content searches to prove no `bob_dataview` file, directory, frontmatter name, audit command, or
    `bob dataview` invocation remains;
  - inspect `git diff`/`git status` to ensure the diff contains the one personal source addition, seven generated
    `bob_query` additions, and seven generated `bob_dataview` deletions, with no unrelated dotfile changes.
- Against the deployed environment:
  - confirm `sase xprompt list` and `sase skill list` expose `/bob_query` from the personal source and do not expose
    `/bob_dataview`;
  - confirm all seven live provider locations contain `bob_query/SKILL.md` and no `bob_dataview` directory;
  - run `bob query --help` as a smoke test that the skill's documented command is available.

## Acceptance Criteria

- SASE no longer ships `src/sase/xprompts/skills/bob_dataview.md` or tests it as a package-owned skill.
- The personal source is chezmoi-managed as `home/dot_xprompts/bob_query.md`, declares `name: bob_query`, and documents
  `bob query` while retaining the original read-only safeguards.
- The reads xprompt and documentation consistently request `/bob_query`.
- All seven provider/runtime skill targets are generated under `bob_query`, and no `bob_dataview` skill source or
  generated directory remains in either the chezmoi repo or the live home targets.
- Skill generation reports no drift, the deployed catalogs expose only `/bob_query`, and the required SASE and Markdown
  checks pass.

## Out of Scope

- Changes to the Bob CLI, its Dataview engine, DQL semantics, environment variables, or output formats.
- A compatibility alias for `/bob_dataview` or `bob dataview`.
- Moving or redesigning the broader `reads` xprompt; only its skill references and associated documentation/tests are
  updated.
- Automatic commits or pushes from the regeneration command.
