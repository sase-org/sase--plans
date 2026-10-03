---
tier: tale
title: Start research swarm audio after the lead completes
goal:
  Allow audio narration to proceed alongside image generation and linker publication
  using the completed lead report
size: small
proposed_by: bbugyi200.athena.0vy
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0vy](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.0vy.md)
- **COMMITS:**
  - [e26ee6a](https://github.com/sase-org/sase-research-artifacts/commit/e26ee6afda1c6c0dc09944e6fd6d46d1b5b9978e)
    — fix(research-audio): start narration after lead completion

# Start research swarm audio after the lead completes

## Why the dependency can be removed

The authoritative macros are packaged in the linked `sase-research-artifacts`
repository. Its `src/sase_research_artifacts/xprompts/research_swarm.md` currently makes
audio wait for both `research.{@1}.final` and, when enabled, `research.{@1}.linker`.
Audio already forks the lead with `#fork:research.{@1}.final`.

When the linker runs (`linker=true` or `image=true`), the lead writes
`<name>/<name>__final.md`. The linker explicitly preserves that file and publishes
`<name>/<name>.md`, changing organization and links while preserving the lead's meaning,
findings, and recommendations. Without a linker, the lead writes `<name>/<name>.md`
itself. Audio can therefore narrate the completed lead report without waiting for
publication.

There is one prompt inconsistency to fix: `research_audio.md` currently tells a forked
audio agent to prefer the published report and fall back to `__final.md`. Removing only
the wait would make source selection depend on whether publication happened to finish
first. Choose the lead's own report consistently instead.

The infographic is optional cover art. `sase-listen` already falls back to a generated
title card when no image is available (`pipeline.py:resolve_cover_bytes`), so no audio
CLI changes are required. Earlier audio may complete without the swarm infographic.

## Repository and scope

From the implementation agent's own host checkout, run:

```sh
sase repo open sase-research-artifacts -r "Implement research swarm audio immediately after the lead"
```

Use the printed checkout path and read its `AGENTS.md`. All paths below are relative to
that repository. This is a macro, documentation, and contract-test change; it requires
no scheduler, Rust binding, dependency-version, or `sase-listen` changes.

## Implementation

1. In `src/sase_research_artifacts/xprompts/research_swarm.md`, remove the conditional
   `%wait:research.{@1}.linker` from the audio segment. Retain its explicit wait for
   `research.{@1}.final`, lead fork, audio model, edition forwarding, and weighted queue
   options. Audio becomes eligible after the lead completes, subject to normal queue
   admission. The image still waits for the lead; the linker still waits for the lead
   and, when requested, the image. Segment order need not change.

2. In `src/sase_research_artifacts/xprompts/research_audio.md`, make the forked-lead
   source instruction explicit: narrate the report the lead wrote. When the lead
   produced `<name>__final.md`, use that file even if `<name>.md` has since appeared;
   otherwise use the lead's `<name>.md`. Do not poll or wait for publication. An
   explicit `@research:` input continues to select exactly that report, including a
   published report. Keep the current macro inputs; a new source-selection option is
   unnecessary. Use the same selected report for narration provenance (`source` and
   `source_blob`) and `lint --source`.

3. Clarify optional cover behavior in the audio prompt: use the corresponding
   `<stem>_infographic.png` when it is already available beside the report; otherwise
   omit `cover` and let the renderer generate its title card. Do not wait for the image
   or linker, and do not rerender automatically when an image later arrives. Preserve
   `__final` stem stripping, script reuse and explicit `rewrite=true`, edition behavior,
   rendering, and MP3 artifact registration.

4. Update the swarm's top-level description and `audio` input description, plus the
   audio paragraph in `README.md`. Describe narration of the lead's consolidated report
   in parallel with optional image/linker work and explain that infographic cover art is
   available only when already present. Remove claims that swarm audio waits for or
   necessarily narrates the published report.

## Verification

Update existing contracts in `tests/test_macro_loading.py`, especially
`test_research_swarm_dependency_graph_preserved`,
`test_research_swarm_audio_opt_in_waits_for_linker`, and
`test_research_swarm_audio_edition_with_linker_and_image`. Rename the obsolete test to
describe audio waiting only for the lead. Exercise the packaged macro expansion and
typed launch planner to verify actual wait edges, rather than relying only on template
substring checks.

Cover these source and dependency cases with the existing test helpers:

| Swarm options                         | Audio waits for | Lead report used for audio |
| ------------------------------------- | --------------- | -------------------------- |
| `audio=true`                          | lead only       | `<name>.md`                |
| `audio=true, linker=true`             | lead only       | `<name>__final.md`         |
| `audio=true, image=true`              | lead only       | `<name>__final.md`         |
| `audio=true, linker=true, image=true` | lead only       | `<name>__final.md`         |

For these combinations, verify the linker's own dependencies remain intact and audio has
neither an image nor linker wait. Retain coverage of `audio=false`, `brief`/`full`
editions, model and queue overrides, and audio not implying a linker. Check the expanded
audio prompt contract for the deterministic forked-lead source, explicit-reference
behavior, and optional cover instructions. Do not launch real research agents or perform
paid speech/image generation for verification.

Run the plugin's required check from its opened checkout:

```sh
sase tool run check
```

Use `/sase_monitor` if a check needs a handoff. Respect the repository's guarded recipe
and coordinated local dependency setup. If setup requires another repository checkout,
obtain it through `/sase_repo` rather than locating or cloning it manually.

## Acceptance criteria

With `audio=true`, the audio agent can run as soon as the lead finishes even while the
image or linker remains unfinished. Its source is always the lead-authored report,
independent of linker timing. Image and linker publication retain their existing
dependencies. Audio without an available infographic still renders with the CLI's
default cover, and the updated macro contracts pass the plugin check.
