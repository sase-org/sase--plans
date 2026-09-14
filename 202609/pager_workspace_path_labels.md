---
tier: tale
title: Collapse workspace parent directories into a quiet ~ws root in pager labels
goal:
  Pager labels for files inside managed SASE workspaces render as ~ws/<workspace>/<rest>
  with a muted root token, so the workspace and file stay visible while every identity,
  copy, edit, and link path stays exact.
size: medium
proposed_by: bbugyi200.apollo.y
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.y](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.apollo.y.md)
- **COMMITS:**
  - [e2d9d64](https://github.com/sase-org/sase/commit/e2d9d64f1a70cb1770d36421f25681c001517fcb)
    — feat(pager): mute workspace root in trail chrome and titles

# Plan: Collapse Workspace Parent Directories Into A Quiet `~ws` Root

## Problem

The pager's breadcrumb trail, subject line, and section rules print file-section titles
as raw absolute paths. For files inside an ephemeral SASE workspace clone that means
every label opens with the same long, uninformative prefix — the workspace's parent
directory — for example:

```
/home/<user>/.local/state/sase/workspaces/acme-org/acme/acme_3/sase/repos/research/202609/decision.md
```

The prefix eats the width budget, so the trail's `fit_label` middle-ellipsis kicks in
and the second crumb degrades to `/home/<user>/.local/state/sase/wor…_confidence.md` —
exactly the part the user does not care about survives, and the part that matters (which
workspace, which repo, which file) is cut. The goal is for the eye to land on the
important part of the path.

## Design

### The label: `~ws/<workspace>/<rest>`

Replace a managed workspace checkout's **parent directory** with the root token `~ws`,
keeping the workspace directory name and everything below it:

| Path                                                                       | Label                                               |
| -------------------------------------------------------------------------- | --------------------------------------------------- |
| `…/workspaces/acme-org/acme/acme_3/sase/repos/research/202609/decision.md` | `~ws/acme_3/sase/repos/research/202609/decision.md` |
| `…/workspaces/acme-org/acme/acme_3` (the checkout itself)                  | `~ws/acme_3`                                        |
| `/home/<user>/projects/acme/src/app.py` (primary checkout, no marker)      | `~/projects/acme/src/app.py`                        |
| `/opt/data/report.md` (outside home, no marker)                            | `/opt/data/report.md` (unchanged)                   |
| `notes.md`, `../acme_3/x.md` (relative, as typed)                          | unchanged                                           |
| `bead:acme-12`, `stdin` (not a path)                                       | unchanged                                           |

Why this shape:

- **Intuitive.** `~ws/` reads as a path root the same way `~/` does, and SASE already
  renders home paths as `~/…` (`sase.content_layout.display_path`, `shorten_path`). It
  is also the zsh named-directory idiom (`hash -d ws=…` shows `~ws/…` in prompts). The
  workspace directory name (`acme_3`) stays verbatim because it is the real directory
  the user sees in agent cwds, `sase repo list`, and shell prompts — the plan removes
  noise, not identity.
- **Path-shaped, so truncation keeps working.** `fit_label` preserves a prefix and the
  basename suffix; a short `~ws/acme_3/` prefix now survives truncation where the old
  absolute prefix never did. The token is 3 ASCII cells, so cell-width math is trivial.
- **Consistent across layouts.** Defining `~ws` as "the parent directory of a managed
  workspace checkout" is true under every `workspace.root` policy (`xdg-state`,
  absolute, `SASE_WORKSPACE_ROOT`, legacy `adjacent`) without the display code knowing
  which policy produced the checkout.

### Detection: the checkout marker is the authority (no guessing)

Every managed (non-primary) workspace has `<checkout>/.sase/checkout.json`, written by
`sase.workspace_provider.marker.write_marker` (never for the primary checkout), and
nested linked/sidecar clones under `<checkout>/sase/repos/…` do not carry their own
marker. So the nearest ancestor marker found by
`sase.workspace_provider.marker.find_marker_from_cwd` is exactly the workspace checkout.
No regex on `_<N>` directory names, no dependency on the configured root policy.

- Walk from the lexical absolute path (`os.path.abspath` of the expanded input, **not**
  `resolve()`), so the displayed remainder is the same path the section really
  represents. Starting the walk at a file path is fine: `read_marker` treats the
  `NotADirectoryError` as "no marker" and the walk continues upward.
- Only collapse when the marker's `workspace_num` is greater than
  `PRIMARY_WORKSPACE_NUM` (defensive; markers are never written for the primary).
- Any failure (malformed marker, `OSError`, `ValueError`, path not under the returned
  checkout) falls back to the `~/` rendering — never raise from a display helper.
- `find_marker_from_cwd` already honors the pytest sandbox guard, so tests must build
  marker trees under `tmp_path` (the sandbox is pytest's basetemp).

### Display-only, deliberately

Only human-facing titles change. Everything that locates or acts on the file keeps the
exact absolute path: `PagerSection.identity`, `subject_ref` (`file:<absolute>`),
`owner`/`link_anchors`, `LinkTarget.edit_path`, copy (`y`) and edit (`E`) behavior,
trail identity/dedupe, and link resolution. The `?` help sheet's secondary identity row
keeps printing the exact `file:<absolute>` identity — that is the one intentional place
the full location stays visible, so nothing is ever unrecoverable.

### Beauty: mute the root token, let the workspace and file carry the weight

When a rendered label begins with `~ws/`, draw the `~ws/` token in the trail chrome's
existing `MUTED_STYLE` (`#8A8A8A`, from `sase.pager._trail_chrome_model`) and the rest
in the label's normal style. Apply this on surfaces whose label style is not reverse
video:

- back/forward crumbs in `_append_crumb` (`src/sase/pager/_trail_chrome_path.py`);
- `subject_line` document and section titles (`src/sase/pager/_chrome.py`), keeping the
  remainder bold where it is bold today;
- `section_rule` labels (`src/sase/pager/_chrome.py`), remainder in the accent color.

Keep the reverse-video current crumb and the already-dim `goto_command_line` context
uniform (a dim span inside reverse video looks broken). If truncation removed any part
of the `~ws/` prefix, render the label plainly — only style an intact prefix. Styling
must never change `Text.plain`, so every existing width invariant still holds.

### Where the label is computed (TUI perf)

`tui_perf` rule 8 forbids stat/glob in render paths. Compute the label once, when a file
section is built — those builders already do file I/O (`read_text`) and an owner marker
walk (`document_owner_from_path`), and the ACE hint path already runs them off the event
loop. Renderers only do a string `startswith` check on the stored title. Do not add
caching: the walk is a handful of failed opens, the same cost owner provenance already
pays per section, and a cache would add staleness and test-isolation hazards for no
measured benefit.

### Rust core boundary

Workspace store and checkout-marker semantics live in Python (`sase.workspace_provider`)
today, and this helper is presentation glue over them, so it stays in this repo. It is
written as one small, dependency-light seam so the CLI, ACE, and a future core port can
share a single definition.

## Implementation

### 1. Shared helper: `src/sase/workspace_provider/display.py` (new)

```python
WORKSPACE_ROOT_TOKEN = "~ws"

def workspace_display_path(
    path: str | Path,
    *,
    home_root: Path | str | None = None,
) -> str:
    """Render *path* for humans: ``~ws/<workspace>/…`` inside a managed workspace,
    ``~/…`` under home, otherwise unchanged. Non-path and relative strings pass through."""
```

Behavior, in order:

1. `text = os.fspath(path)`. If `text` does not start with `/` or `~`, return it
   unchanged (relative paths as typed, refs like `bead:…`, `stdin`).
2. `absolute = os.path.abspath(os.path.expanduser(text))`.
3. `found = find_marker_from_cwd(absolute)` inside `try/except (OSError, ValueError)`.
   If found with `marker.workspace_num > PRIMARY_WORKSPACE_NUM`, compute
   `PurePath(absolute).relative_to(checkout_dir)` (lexical; `ValueError` → fall back)
   and return `f"{WORKSPACE_ROOT_TOKEN}/{Path(checkout_dir).name}"` plus `/<relative>`
   when the relative part is not `.`.
4. Otherwise return `sase.content_layout.display_path(text, home_root=home_root)`.

Also export a tiny pure predicate used by renderers, e.g.
`split_workspace_root(label: str) -> tuple[str, str]` returning `("~ws/", rest)` when
the label starts with `~ws/`, else `("", label)`. Keep both functions public in the
module `__all__` and used (symvision).

### 2. Apply the label where file sections are titled

- `src/sase/pager/adapters.py` `path_section`: `title=workspace_display_path(path)`
  instead of `str(path)`. Keep `identity`/`subject_ref` as `file:<absolute_path>`.
- `src/sase/pager/_resolve_common.py` `directory_link_target`: section
  `title=workspace_display_path(path)`; keep `identity`/`subject_ref` exact. The
  document title (`"{n} entries · {path.name}"`) is already concise — leave it.
- `src/sase/pager/landings.py` `binary_card_document`: it currently uses `title` for
  both `identity` and the displayed titles. Add a keyword-only
  `display_title: str | None = None`; when given, use it for section and document
  `title` while `identity` and the `reference:` body line keep the exact `title`. Pass
  `display_title=workspace_display_path(path)` from the binary branch of
  `link_target_for_existing_path` in `_resolve_common.py`. Other callers are unchanged.
- `src/sase/main/pager_handler.py` `_input_document_title`: for a single input, return
  `workspace_display_path(values[0])` (non-path inputs pass through unchanged by
  construction). This fixes the bold subject title for `sase pager /abs/path`, and the
  plain (non-TTY) multi-section `-- i/n: <title> --` headers follow automatically from
  step 1's section titles.
- Leave `landings.ambiguous_source_resolution` alone: its title is the literal link text
  from the document, not a landed path.
- `src/sase/ace/tui/actions/hints/_files.py` needs no change; it goes through
  `document_from_paths` → `path_section`.

### 3. Quiet the root token in the pager chrome

Add one rendering helper next to the trail text helpers (e.g. in
`src/sase/pager/_trail_chrome_text.py` or a sibling pager module), roughly
`append_path_label(text: Text, label: str, *, style: str, root_style: str) -> None`,
built on `split_workspace_root`. Use it in:

- `_append_crumb` for `back`/`forward` crumbs (not `current`);
- `render_current_only` stays uniform (it is reverse video);
- `subject_line` for `document.title` (bold remainder) and `current_section.title`;
- `section_rule` for the section title (accent remainder; keep the glyph as today).

Do not touch the `?` help sheet rows beyond what the new titles naturally produce.

## Tests

Place helper tests alongside the existing workspace-provider/marker tests; pager tests
in `tests/pager/`; CLI tests with the existing pager CLI tests.

1. **Helper** (build a real tree under `tmp_path` with a JSON marker written through
   `CheckoutMarker(...).to_dict()`, using a nested project-key directory like
   `workspaces/acme-org/acme/acme_3`):
   - file deep inside the workspace → `~ws/acme_3/docs/notes.md`;
   - file inside a nested linked-repo directory (`acme_3/sase/repos/linked/lib/x.py`, no
     marker of its own) → anchored at `acme_3`;
   - the checkout directory itself, with and without a trailing slash → `~ws/acme_3`;
   - a marker with `workspace_num == 0` and an unmarked directory under `home_root` →
     `~/…`;
   - an unmarked path outside `home_root` → unchanged;
   - malformed marker JSON → falls back, does not raise;
   - `~`-prefixed input expands before lookup;
   - relative (`notes.md`, `../acme_3/x.md`) and non-path (`bead:acme-12`, `stdin`)
     strings → unchanged;
   - `split_workspace_root` for `~ws/acme_3/x`, `~wsx/y`, `~/x`, and `""`.
2. **Section builders:** `path_section` for a workspace file has
   `title == "~ws/acme_3/…"` while `identity`/`subject_ref` stay `file:<absolute>`;
   `directory_link_target` likewise; the binary card keeps `identity` exact and shows
   the compact title.
3. **CLI:** `_input_document_title` for one absolute workspace path returns the `~ws`
   label; for a ref and for multiple inputs it is unchanged.
4. **Chrome styling** (`tests/pager/test_trail_chrome.py`,
   `tests/pager/test_chrome.py`):
   - a back crumb labeled `~ws/acme_3/a.md` renders `~ws/` with the muted style and the
     remainder with the state style; `Text.plain` is identical to the unstyled label;
   - the current crumb has no muted span;
   - a label whose `~ws/` prefix was truncated renders without a muted span;
   - `subject_line` and `section_rule` mute only the intact root token;
   - extend `test_rows_never_exceed_requested_cell_width` with a `~ws/…` label so the
     width invariant is exercised across 0–160 cells.

## Verification

- `just install` if the workspace venv is stale, then `just check` (lint gates plus the
  scoped test lane). Fix any symvision findings for the new public helpers by using
  them, not by deleting them.
- Manual: from any managed workspace, run
  `sase pager <absolute path to a markdown file in that workspace that links to another file>`;
  confirm the subject line reads `~ws/<workspace>/…` with a muted `~ws/`, follow a link,
  and confirm both crumbs in the trail band now keep the workspace and file visible.
  Press `?` and confirm the identity row still shows the exact `file:` path. Press
  `y`/`E` on a link and confirm the copied or edited path is the real absolute path.

## Non-goals And Follow-ups

- Out of scope: the ACE link rail, Artifacts tab path columns, and `sase repo list` /
  `sase workspace list` PATH columns. If the implementer judges any of these would
  clearly benefit from `workspace_display_path`, capture it as a follow-up task bead via
  `/sase_new_task` rather than widening this change.
- No new config option: `~ws` is a fixed, documented rendering, like `~`.
