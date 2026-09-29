---
tier: tale
title: Bead attachment previews and full-fidelity viewing
goal:
  Bead attachments have safe inline previews, navigable pager links, an open command,
  and artifact-reference access across supported terminals.
size: medium
proposed_by: bbugyi200.athena.sase-1ck.7
bead: sase-1ck.7
create_time: 2026-09-29 17:14:19
status: wip
---

- **PARENT:**
  [202609/bead_note_attachments.md](https://github.com/sase-org/sase--plans/blob/main/202609/bead_note_attachments.md)
- **BEAD:**
  [sase-1ck.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ck/sase-1ck.7.md)

# Bead attachment viewing (`sase-1ck.7`)

Implement phase `show_images` from `plan:202609/bead_note_attachments.md`. The bead is
already in progress. Current code provides attachment descriptors, cached view paths,
plain chips, and `sase bead attachment list|path`; shared-store fetching may land
concurrently. Complete this phase as one cohesive CLI and pager integration. Do not
close the parent epic or any ancestor.

## Implementation

1. Trace the current `show`/`read` pipeline (`src/sase/bead/cli_query.py`,
   `cli_show_batch.py`, `cli_detail_sections.py`), the attachment roster and view-path
   helpers, CLI parser wiring, and the shared viewer. Add
   `-i/--images auto|cells|kitty|never` to `sase bead show` only; keep `read`, JSON, and
   piped `show` free of image bytes, file contents, and graphics escapes. Add
   `bead.show.images: auto` to `src/sase/default_config.yml`,
   `src/sase/config/sase.schema.json`, a fail-open config accessor, and
   `docs/configuration.md`. Follow `cli_rules.md`: a short alias, sorted options, and
   useful help.
2. Resolve the image mode from CLI override, config, stdout TTY, `NO_COLOR`,
   `SASE_AGENT`, tmux, and kitty capability. `auto` chooses `never` for a pipe,
   `NO_COLOR`, or an agent; otherwise direct kitty when supported outside tmux, else
   cells. Explicit kitty falls back to cells with one dim hint when unsupported or
   inside tmux. Reuse the doctor's terminal capability logic. Render cached raster image
   thumbnails through `CellImageRenderable` and a Rich `Console` at terminal width,
   preserving aspect ratio, limited to 10 rows and 4 thumbnails per note. Bound decode
   work and show a useful failure card or chip on any missing/corrupt/unsupported image.
   Use the kitty graphics protocol for direct pixel output outside the pager; keep all
   other modes compatible with the existing pager. Preview text attachments only when
   previews are on: at most five dim, sanitized lines for files no larger than 1 MiB.
   Never decode SVG/EPS for automatic previews. Keep beads without attachments on the
   existing cheap path.
3. Upgrade attachment chips and descriptor rows in the full bead pager document to
   labeled `AttachedTarget`s pointing to `attachment:<bead-id>/<name>`. Preserve
   readable plain output for `read` and `show -i never`. Extend
   `src/sase/pager/resolve.py` through the existing resolver helpers to materialize an
   attachment view and delegate file classification to `link_target_for_existing_path`:
   image/video/PDF open in the viewer, text in a sanitized pager section, other binaries
   in the existing binary card. For media, build the viewer's ordered list from all
   viewable attachments on the bead so n/p navigation works. Pass attached target
   handlers through `cli_pager` if necessary and add attachment icons to the pager label
   markers. Show a graceful dead end for unavailable attachments.
4. Add `sase bead attachment open <id> [<name>]`, reusing the same roster,
   materialization, and viewer logic. If the name is omitted, present/choose from
   viewable attachments according to existing CLI conventions. Add
   `attachment:<bead-id>/<name>` resolution for `sase artifact path|open|read`. Because
   reference parsing and kind registration are shared core behavior, use `/sase_repo` to
   open `sase-core` if the Rust catalog/resolver must change; update binding tests and
   move `sase-core-revision.txt` past the required core commit. Do not implement a
   divergent Python-only parser for a shared artifact-ref rule. Ensure `read` has
   meaningful behavior for text versus binary/image content and that `path` returns an
   absolute extension-preserving view path.
5. Add focused tests for mode-selection matrix (TTY, pipe, `NO_COLOR`, agent, tmux,
   kitty markers), cell rendering on a recorded Console, dimensions and limits, decode
   failure, kitty escape framing/chunking and placement, text sanitization/size cap,
   pager target spans and per-file-class resolution, n/p media order, CLI open and
   artifact ref commands, and no escapes/content in `read`/JSON/piped output. Cover both
   beta-flag states for rendering. Update pager visual snapshots with
   `just fix-tui-screenshots` and inspect the report and golden diff.

## Verification and completion

Read `tui_screenshot.md` before touching pager snapshots and `lint_and_test.md` after
changing tracked sase files. Run `just fix`, then `sase tool run check` as required by
the parent plan; use `/sase_monitor` for commands too long for this single turn. Resolve
any phase-specific `--epic-symbol` entries or re-key them to an open bead. Run
`sase bead epic-symbols sase-1ck.7` immediately before close. If a check failure
reproduces identically on the clean base tree, record it as `PROPOSED FOLLOW-UP:` on
this phase and close anyway. Record other discovered out-of-scope work the same way; do
not create beads. Close only `sase-1ck.7` with
`sase bead close sase-1ck.7 --note "<verified behavior and checks>"`; never close
`sase-1ck` or another ancestor.
