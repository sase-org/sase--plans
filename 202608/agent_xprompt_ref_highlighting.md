---
tier: tale
title: Artifact reference highlighting in agent xprompts
goal:
  Make artifact refs in the Agents metadata panel immediately legible, theme-aware, and
  robust in every AGENT XPROMPT render path.
size: medium
proposed_by: bbugyi200.athena.097
create_time: 2026-08-21 08:49:06
status: wip
---

# Plan: Beautiful artifact refs in AGENT XPROMPT

## Outcome and visual contract

The `AGENT XPROMPT` section will give every typed artifact reference a consistent,
high-information visual structure. For a ref such as `@plan:202608/design.md#L12-L18`,
the `@` sigil and `:` separator will provide subdued structural punctuation, `plan` will
be a bold kind label, the payload will use a readable companion color, and the optional
fragment will use a distinct italic accent. Quoted payload delimiters will read as
punctuation rather than as part of the target. The palette will be derived from the
active Textual theme and will share the same roles as the live prompt editor, so dark
and light themes feel intentional rather than merely tolerable.

Artifact-ref styling will win over broad xprompt-argument styling, including for refs
nested inside invocations such as `#work(@plan:202608/design.md#L12)`, while Markdown
inline/fenced code will remain literal and keep its code treatment. Malformed ref-shaped
candidates will stay fully visible and receive a restrained error treatment;
syntactically well-formed candidates will be highlighted without claiming that their
target currently resolves. The read-only historical panel will not perform project
catalog I/O or resolution during rendering.

Hint mode will treat a typed `@kind:payload` ref as one semantic token. Generic
file-path hinting must not splice a `[N]` marker into its kind, payload, or fragment, or
resolve a document-ref payload as though it were an ordinary workspace path. Bare file
mentions such as `@src/module.py` will retain their existing numbered open-file hints.
Ref highlighting is limited to `AGENT XPROMPT`; expanded `AGENT PROMPT` and agent
replies retain their current Markdown and semantic roles.

## Implementation

1. Add a shared ACE artifact-ref syntax utility under `src/sase/ace/tui/util/` and make
   the existing Rust-backed `scan_artifact_refs` result its grammar source of truth. The
   utility will:
   - convert the scanner's UTF-8 byte ranges into exact Python/Rich character ranges,
     including non-ASCII text;
   - expose the candidate range and the sigil, kind, separator, payload, quoted
     delimiter, and fragment roles needed by both editable and read-only surfaces;
   - exclude candidates overlapping fenced or inline literal zones;
   - distinguish well-formed, malformed, cold-catalog, and unknown-kind presentation
     without resolving targets;
   - enforce the existing prompt highlight byte/line caps and fail open on scanner or
     range-conversion errors so source text is never lost; and
   - centralize theme-derived Rich styles and a stable style signature for cache keys.
     Preserve the live editor's current visual language: bold secondary sigil, bold
     success-colored kind, dim secondary separator/delimiters, success-derived payload,
     accent-derived italic fragment, muted unknown/cold states, and a restrained
     theme-error treatment for malformed syntax.

2. Refactor `src/sase/ace/tui/widgets/_artifact_ref_highlight.py` to consume the shared
   candidate spans and palette instead of maintaining its own byte conversion and style
   definitions. Preserve its existing project-aware warm catalog lifecycle, cold neutral
   state, unknown-kind state, worker refresh behavior, overlay caps, and current
   TextArea role names. This keeps the prompt editor and metadata panel visually aligned
   without coupling the read-only panel to editor workers or filesystem reads.

3. Compose the shared ref overlay into `src/sase/ace/tui/util/xprompt_syntax.py` and the
   authored-prompt overlay helper in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_xprompt_highlighting.py`. Establish one
   explicit paint order: Markdown base, glossary/repository semantic roles,
   xprompt/alternative structure, artifact-ref parts, then UI affordances such as
   numbered hints. Apply the ref layer only when rendering an authored xprompt. Extend
   `AgentPromptHighlightContext` with the derived ref palette and include its full theme
   signature in the xprompt cache fingerprint so changing any relevant theme color
   cannot reuse stale spans.

4. Route every `AGENT XPROMPT` variant through the same overlay contract:
   - ordinary terminal-agent rendering in
     `src/sase/ace/tui/widgets/prompt_panel/_agent_display_render.py`;
   - terminal-agent file-hint rendering in
     `src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_render.py`; and
   - family/container rendering and its hint variant in
     `src/sase/ace/tui/widgets/prompt_panel/_agent_display_family_render.py`.

   Add a ref-aware xprompt file-hint matcher in the prompt-panel hint utilities. It will
   reuse the scanner's complete candidate ranges to suppress generic path matches that
   overlap typed refs, and it will be passed only for xprompt bodies; other metadata,
   prompts, and replies keep their current hint behavior. Keep the same bounded-content
   behavior in both the matcher and appender so truncation cannot produce divergent hint
   maps.

5. Add focused regression coverage:
   - unit-test the shared span builder and theme palette for multi-ref lines, fragments,
     quoted payloads, malformed candidates, Unicode before and inside refs, literal
     zones, exact size boundaries, theme adaptation, and fail-open scanner errors;
   - retain and strengthen `tests/ace/tui/widgets/test_prompt_artifact_ref_highlight.py`
     so the editor's warm/cold/unknown behavior and theme-switch registration remain
     unchanged after the refactor;
   - extend `tests/ace/tui/util/test_xprompt_syntax.py` and
     `tests/ace/tui/widgets/test_agent_display_xprompt.py` to assert exact component
     styles, ref-over-xprompt precedence, literal exclusion, cache invalidation,
     terminal/family parity, and intact ref text in normal and hint modes;
   - prove that hint mode still numbers bare `@path` mentions while producing no
     misleading file mapping or inline marker for typed refs; and
   - add representative nested, document, entity, quoted, and fragmented refs to
     `tests/ace/tui/visual/test_ace_png_snapshots_agents_xprompt.py`, regenerate the
     dark and light PNG goldens, and inspect both images plus their diffs for contrast,
     hierarchy, wrapping, and visual collisions.

## Verification

Run `just install` before repository checks. Exercise the focused unit/widget tests for
the shared utility, prompt editor, xprompt syntax renderer, semantic precedence, and
agent display paths. Run the dedicated Agents xprompt visual test in dark and light
themes, accepting golden updates only after inspecting the generated
actual/expected/diff artifacts. Finish with `just check`; if scoped selection escalates
or reports an unusual selection, use `/sase_monitor` for `just check-full` as required
by the repository verification policy.
