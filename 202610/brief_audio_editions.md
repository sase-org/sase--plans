---
tier: tale
title: Default research audio to brief and expose the swarm edition
goal: Newly authored research podcasts default to brief, and audio_edition on research_swarm
  controls the narration edition end to end.
size: small
proposed_by: bbugyi200.athena.0vo
status: done
---

# Default research audio to brief and expose the swarm edition

## Outcome

Make `brief` the default for agent-authored narration through `sase-listen` and the
research xprompts. Let callers request a full podcast directly from the swarm:

```text
#research_swarm(prompt="A research topic", audio=true)
#research_swarm(prompt="A research topic", audio=true, audio_edition=full)
```

The first invocation should author a brief narration (about 600 words / four minutes);
the second should author a full narration (up to 2,400 words / about 16 minutes).
`audio` remains opt-in. One coding agent can implement this focused change across two
linked repositories, so this is a small tale.

## Repositories and evidence

From the implementing agent's SASE workspace, open both repositories through
`/sase_repo` and work only in the returned paths:

```bash
sase repo open sase-listen -r "Implement brief narration defaults and guide consistency"
sase repo open sase-research-artifacts -r "Expose research swarm audio_edition and align narration defaults"
```

Read each repository's `AGENTS.md`. All paths below are relative to the named
repository. No primary `sase` source changes are expected.

In **sase-listen**:

- `src/sase_listen/cli/guide.py` defaults both argparse's `--edition` and
  `render_guide()` to `full`. Its CLI accepts `full` and `brief`.
- `src/sase_listen/data/guide.md` has edition-specific shape and budget sections, but
  its common frontmatter example hard-codes `edition: full` and `target_minutes: 15`. A
  read-only rendering of both guides confirmed that the brief guide still emits this
  full-edition metadata.
- `tests/test_script_cli.py::test_guide_prints_and_editions_differ` assumes the omitted
  CLI edition means full. Merely checking for `600` is insufficient: both guides contain
  the common edition-budget table.
- `ScriptMeta`, the parser, lint fallback, and deterministic Markdown normalization use
  `verbatim`. Rendering consumes the script's edition; it does not have a separate
  full-edition default.

In **sase-research-artifacts**:

- `src/sase_research_artifacts/xprompts/research_audio.md` independently defaults its
  `edition` input to `full` and explicitly runs
  `sase-listen guide --edition {{ edition }}`. Changing only the CLI default would
  therefore leave research podcasts full by default.
- `src/sase_research_artifacts/xprompts/research_swarm.md` ends its optional audio
  segment with a bare `#research/audio`. It has `audio` and `audio_model` inputs but no
  edition input.
- `tests/test_xprompt_loading.py` exercises the actual packaged xprompt loader, typed
  inputs, rendered swarm segments, and audio wait/fork/queue behavior. Its direct-audio
  input test also expects `full` today.

The standalone tool explicitly owns its narration behavior without SASE or Rust imports;
the plugin owns these prompt templates. This work needs no Rust API, core revision pin,
or new package dependency.

## Implementation

### 1. Make the guide default and rendered examples agree

In `sase-listen`:

1. Change the argparse default and the `render_guide()` default in
   `src/sase_listen/cli/guide.py` to `brief`. Make `--help` state that brief is the
   default and show a matching example. Preserve explicit `--edition full`.
2. Make the example in `src/sase_listen/data/guide.md` reflect the selected edition. Use
   two explicit template tokens for the edition and optional target minutes, replaced by
   `render_guide()` alongside its existing edition-section filtering; no general
   template engine is necessary. Render `edition: brief` / `target_minutes: 4` for brief
   and `edition: full` / `target_minutes: 16` for full. Leave no tokens or edition
   delimiter comments in the emitted guide.
3. Keep the existing brief and full chapter shapes and word budgets. A raw Markdown
   input must still normalize to `edition: verbatim`, and explicit editions on existing
   narration scripts must retain their meaning.

### 2. Carry the swarm setting through to the audio author

In `sase-research-artifacts`:

1. Change `research_audio.md`'s `edition` default to `brief`, so direct
   `#research/audio` calls share the new default. In the script-authoring instructions
   explicitly require `edition: {{ edition }}` in narration frontmatter, in addition to
   the existing source and research metadata.
2. Append an optional `audio_edition` input to `research_swarm.md` after the existing
   inputs, with `type: word` and `default: "brief"`. Appending keeps every existing
   positional argument at its current position. Describe it as the narration edition
   passed to `#research/audio` when `audio=true`, with the supported authoring choices
   `brief` and `full`.
3. Replace the bare audio call with `#research/audio(edition={{ audio_edition }})`. Keep
   it inside the existing audio segment. Supplying `audio_edition` alone must not launch
   an audio agent or imply the linker.
4. Preserve the audio model, fork target, waits for the lead and optional linker, queue
   parameters, report selection, and output paths. Preserve `rewrite=false` reuse of
   existing narration files: these defaults govern newly authored scripts. Document that
   an existing script is reused unless direct `#research/audio(..., rewrite=true)` is
   requested.

### 3. Document the new defaults and the override

In `sase-listen`, update `docs/cli.md`, `docs/getting-started.md`, and
`docs/narration-scripts.md` to present bare `sase-listen guide` as brief by default,
with an explicit full example. Align the narration-script sample with brief and its
four-minute target. Update `docs/sase-integration.md` to describe `audio_edition` and
the direct-audio default; make the concise authoring explanation in `README.md` mention
the default as appropriate.

In the plugin, update `docs/xprompts.md`'s direct-audio default and swarm input table,
and its audio-stage explanation. Update the audio examples in `README.md`. Show the two
swarm invocations above and the direct override `#research/audio(edition=full)`. Clarify
that edition selection affects newly authored narration, not whether audio is enabled.

The guide currently supports only `full` and `brief`, although script metadata also
supports `digest` and `verbatim` and older plugin prose lists all four. Describe `full`
and `brief` as the supported guide-backed authoring choices; do not expand the CLI's
accepted editions or change script-format support as part of this task. Leave historical
background, field notes, and changelog entries intact.

## Regression coverage

Use the existing test suites; no synthesis, network TTS, or podcast publication is
needed to prove these contracts.

In `sase-listen/tests/test_script_cli.py`, update and strengthen guide tests:

- `main(["guide"])`, explicit `--edition brief`, and `render_guide()` produce the brief
  guide; default and explicit brief outputs are equal.
- Brief emits the brief frontmatter and four-minute target, the 2–3 chapter instructions
  and brief budget paragraph, without the full-specific shape and authoring paragraph.
- Explicit full still emits full metadata and its matching duration, the 4–8 chapter
  instructions and full budget paragraph, without brief-specific instructions. Parse the
  frontmatter of the fenced example to check its values, rather than matching words in
  the common budget table.
- Both modes retain lint instructions and emit no unresolved template tokens or
  edition-selection markers. CLI help identifies the brief default.
- Strengthen the existing deterministic `script --json` test to assert that the output
  script remains `edition: verbatim`.

In `sase-research-artifacts/tests/test_xprompt_loading.py`:

- Update the direct-audio default assertion and the swarm input contract for the
  appended `audio_edition: word = brief` input.
- Expand direct `research/audio` with omitted edition and explicit full, asserting the
  resulting guide command and narration-edition instruction.
- Expand the real swarm with `audio=true`: omitted edition must produce
  `#research/audio(edition=brief)`; explicit brief and full must propagate their values.
  Assert the selected call occurs only in the audio member and no unresolved
  `{{ audio_edition }}` remains.
- Test the next expansion step using the real loaded `research/audio` prompt and SASE's
  macro processor with an isolated catalog, so the swarm's nested named argument
  demonstrably becomes `sase-listen guide --edition brief` or `--edition full`. Avoid
  executing shell substitution, launching agents, or relying on user-installed xprompt
  overrides in this test.
- Verify `audio_edition=full` without `audio=true` leaves the default three agents and
  no audio member. Exercise the override alongside linker/image opt-ins using the
  existing dependency tests; retain wait, fork, queue, model, and segment-count
  expectations.

## Verification and completion

Use each repo's local development environment, not bare system `python` or a globally
installed plugin. The listen checkout needs its documented `just install` if
dependencies are absent. The plugin's documented setup uses the coordinated local SASE
and sase-core checkouts; if it needs sase-core, open it with `/sase_repo` before setup
and use the printed path. Do not alter dependency versions or pins just for this change.

Run `sase tool run check` from each changed repository root. These are guarded recipes;
do not run bare `just check`. They cover lint and the offline tests, including the
changed tests. During iteration, the relevant focused test files are
`tests/test_script_cli.py` and `tests/test_xprompt_loading.py` in their respective
repos. Use `/sase_monitor` if a build or check requires a long-command handoff. No
exhaustive wheel lane or live audio run is required for this template/default change
unless a new failure justifies it.

Review the resulting diffs against the documented examples and confirm both repos'
checks pass. Only if implementation unexpectedly changes tracked files in the primary
SASE repo, read its `lint_and_test.md` memory and follow its required verification as
well. Finish with `/sase_final`, declaring both changed repositories; host finalizers
own commits.

Acceptance: a fresh swarm podcast defaults to brief, `audio_edition=full` reaches the
narration author intact, the guide's sample metadata agrees with the selected edition,
and existing audio opt-in/dependencies and deterministic verbatim conversion continue to
work.
