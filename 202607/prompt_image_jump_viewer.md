---
tier: tale
title: Display prompt jump image targets in the terminal viewer
goal: 'Ctrl+] on an image path in the ACE prompt input displays the image with the
  existing terminal artifact viewer instead of opening the binary in an editor.

  '
create_time: 2026-07-18 06:52:06
status: done
prompt: 202607/prompts/prompt_image_jump_viewer.md
---

# Plan: Display prompt jump image targets in the terminal viewer

## Context

The prompt input's normal-mode `Ctrl+]` command resolves xprompt, skill, and file references into a `JumpTarget`. It
then offers editor-oriented actions: opening the target in the current pane or, in tmux, opening an editor in a split.
That behavior is appropriate for source and text definitions, but it sends resolved image files such as PNGs to Neovim
and exposes their binary contents.

ACE already has a public terminal artifact-viewer path for supported images. Other TUI surfaces classify media with the
shared graphics helpers, suspend Textual while the external viewer owns the terminal, call `view_artifact_file`, and
report any structured viewer warning to the user. Prompt jump should reuse that behavior rather than introduce another
image command or renderer.

## Implementation

Update the prompt jump presentation flow in `src/sase/ace/tui/widgets/_prompt_jump.py` so a resolved file target whose
path matches the shared supported-image predicate is handled before editor/tmux/load choices are built. Route that
target directly to the public terminal artifact viewer while the ACE app is suspended, then surface any returned warning
with warning severity and restore prompt focus as needed after terminal control returns.

Keep the media decision at this post-resolution boundary: the existing resolver already validates and canonicalizes the
path, while the presentation layer owns the choice of external tool. Use the shared `is_supported_image_path`
classification and `view_artifact_file` launch API from the graphics package so supported extensions and viewer behavior
remain consistent with notification attachments, file hints, and other ACE image surfaces.

Image targets must bypass both existing editor-backed options, including the tmux split action, so `Ctrl+]` never
launches an editor for a recognized image. Do not change token detection, path/base-directory resolution, xprompt
loading, `gd` in-place authoring, non-image file jumps, the jump chooser, or keymap/help text; the binding and its
documented meaning remain the same, only the resolved image presentation becomes type-aware.

## Verification

Extend the prompt normal-mode jump tests to exercise the command through resolution and presentation:

- A supported image file target invokes the terminal artifact viewer inside the app suspension context and does not push
  the jump chooser or call either editor-backed jump path, including when tmux is available.
- A viewer result containing a warning is shown through the prompt notification channel with warning severity.
- Existing non-image file behavior remains editor-oriented, and existing xprompt/skill chooser and load behavior remains
  unchanged.

Run the focused prompt jump test module during development, then run `just install` followed by the repository-required
`just check` for the completed implementation.

## Acceptance criteria

- Pressing `Ctrl+]` with the cursor on a resolvable supported image path displays the image via ACE's terminal artifact
  viewer.
- No editor process or editor-in-tmux split is launched for that image target.
- Viewer dependency/rendering warnings are visible to the user, and ACE returns to a usable, focused prompt after the
  viewer exits.
- Text files, xprompts, skills, and workflows retain their current jump-to-definition behavior.
