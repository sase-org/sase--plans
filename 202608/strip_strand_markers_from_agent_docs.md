---
tier: tale
title: Strip strand-roster markers from inlined agent instruction files
goal:
  AGENTS.md and the provider instruction shims no longer contain `<!-- sase:strands -->`
  marker comments, while the canonical memory-web descriptor files under sase/memory/
  keep them.
size: small
proposed_by: bbugyi200.athena.0dr
---

# Strip strand-roster markers from inlined agent instruction files

## Problem

Memory webs mark their managed strand roster inside the web descriptor note with a pair
of HTML comment lines:

```
<!-- sase:strands -->

- **Bug** (`bug`) - A defect an agent found while doing unrelated work, ...

<!-- /sase:strands -->
```

Those markers are load-bearing _in the canonical descriptor file_ under `sase/memory/`:
`sase memory init` uses them to find and replace the managed region on every run, and
`roster_region_error` validates that they are balanced and unique.

They are pure noise everywhere else. When a core-tier web descriptor's body is inlined
into `AGENTS.md` (and the byte-for-byte provider shims `CLAUDE.md`, `GEMINI.md`,
`QWEN.md`, `OPENCODE.md`), the marker comments are copied verbatim. In this repo that is
3 marker pairs (`decisions`, `glossary`, `task_types`) x 5 files = 30 wasted lines of
context that mean nothing to a reading agent, plus the same again in the home/chezmoi
memory root.

The markers must stay in the canonical descriptor files; only the _rendered agent
instruction_ copy should drop them.

## Desired output shape

Given this descriptor body region:

```
...intro paragraph.
<blank>
<!-- sase:strands -->
<blank>
- **Bug** (`bug`) - ...
<blank>
<!-- /sase:strands -->
<blank>
## Next Heading
```

the inlined `AGENTS.md` copy should read:

```
...intro paragraph.
<blank>
- **Bug** (`bug`) - ...
<blank>
## Next Heading
```

That is: drop each marker line **and any blank lines immediately preceding it**. The
blank line that _follows_ each marker is kept, which is what preserves exactly one blank
line of separation on both sides of the roster payload. No other content moves.

## Implementation

### 1. Add a pure strip helper next to the markers

In `src/sase/memory/web/roster.py` (which already owns `START_MARKER` / `END_MARKER`),
add:

```python
def strip_managed_roster_markers(body: str) -> str:
    """Return *body* with roster marker lines and their preceding blanks removed."""
```

Behavior:

- Split on lines. A line is a marker when its stripped text equals `START_MARKER` or
  `END_MARKER` exactly.
- On a marker line: discard it, and pop any trailing blank lines already accumulated in
  the output.
- Preserve the original trailing-newline shape of _body_ (the callers pass bodies that
  end in `\n`; do not add or drop a final newline).
- A body with no markers must be returned unchanged.

Do **not** add code-fence tracking. Existing `roster_region_error` already rejects a
descriptor that contains more than one marker pair anywhere in its body — fenced or not
— so "a fenced example marker plus a real roster region" is not a representable state,
and a lone fenced marker pair is already treated as the managed region by
`render_web_body_with_roster`. Exact-line matching therefore matches the semantics the
rest of the module already enforces. Note this reasoning in the docstring so a future
reader does not "fix" it.

Export the new name from `roster.py`'s `__all__` and from
`src/sase/memory/web/__init__.py` (both the import block near line 52 and the `__all__`
list), matching the existing alphabetical placement convention in that file.

### 2. Apply it at the single AGENTS.md choke point

In `src/sase/main/init_memory/root_planning.py`, `_memory_web_root_plan` is the only
place a web descriptor body ever becomes a Tier 1 inlined section. At lines 251-255:

```python
        if web.rendering_type == "core":
            core_note_bodies[web.relative_path] = GeneratedShortMemoryNote(
                body=strip_managed_roster_markers(body),
                priority=web.priority,
            )
```

Add `strip_managed_roster_markers` to the existing `from sase.memory.web import (...)`
block at the top of the file.

Leave `note_overlay[web.path] = content` and the `MemoryExpectedFile` for `web.path`
untouched — those are what write the canonical `sase/memory/*.md` descriptor, which
keeps its markers.

### Why this call site and not `sase.amd.inline_memory`

`inline_memory_section` in `src/sase/amd/inline_memory.py` is the other candidate: it is
the literal "core note body -> `### Title (file)` section" renderer, and it is used by
both `render_project_agents_content` and `plan_minimal_agents_sync`. Rejected, because:

- Tier 1 sections there are built exclusively from the `short_memory_bodies` mapping,
  and `root_planning.py` is the only producer of web-descriptor entries in that mapping.
  Stripping upstream covers every path that reaches an agent instruction file, including
  every provider shim (they are byte-for-byte copies of `AGENTS.md`, see
  `sase/amd/constants.py:11` and `sase/amd/_shared.py:143`).
- `sase.amd` does not currently import `sase.memory.web`. Adding that edge would pull
  the whole web package (and its Rust glossary facade import chain) into `sase.amd` for
  no behavioral gain, or force the marker constants to be duplicated and kept in sync.
- Roster markers only ever exist in web descriptor bodies, so a generic strip in `amd`
  would guard a case that cannot occur.

## Tests

1. **Unit** — `tests/memory/test_memory_web.py` (which already imports `START_MARKER` /
   `END_MARKER`): cover `strip_managed_roster_markers` directly.
   - Round-trip the real shape: feed the output of `render_web_body_with_roster` for a
     descriptor with an intro paragraph, a roster region, and a following `##` heading;
     assert the exact expected string (one blank line before the payload, one after, no
     marker text).
   - Marker at the very start of the body (no preceding blank).
   - Body with no markers is returned byte-identical.
   - Trailing-newline shape preserved.

2. **Integration** — `tests/main/test_init_memory_memory_webs.py`, extending
   `test_memory_web_updates_roster_without_inlining_strand_bodies` or adding a sibling:
   assert both halves of the contract from one `plan_memory()` run —
   `START_MARKER not in agents` and `END_MARKER not in agents`, while
   `START_MARKER in updated_descriptor` and `END_MARKER in updated_descriptor`. Also
   assert the roster payload itself (`**TERMS:** Alpha Term (alpha)`) still appears in
   `agents`, so the strip cannot silently eat the region it was meant to expose.

Search for any other test asserting marker presence in generated agent docs before
finishing (`grep -rn "sase:strands" tests/` is currently empty, so no existing goldens
should break — confirm this still holds after the change).

## Regenerating the committed artifacts

This changes generated output, so the checked-in copies go stale. After the code change:

```bash
sase memory init
```

Expect `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `QWEN.md`, and `OPENCODE.md` in the repo
root to lose 12 lines each (3 marker pairs x [marker line + preceding blank]). Verify
with `grep -c "sase:strands" AGENTS.md` returning 0, and confirm the four shims are
still byte-identical to `AGENTS.md` (`diff AGENTS.md CLAUDE.md` etc.). Confirm the
canonical notes still have their markers:
`grep -n "sase:strands" sase/memory/decisions.md sase/memory/glossary.md sase/memory/task_types.md`
must still show three pairs.

Do **not** hand-edit any file under `sase/memory/` — the canonical notes are unchanged
by this work, and regenerating the instruction files is the whole point.

If `sase memory init` also reports writes to the home / chezmoi memory root (its
`AGENTS.md` and provider shims carry the same markers), those live in the chezmoi repo:
open it with the `/sase_repo` skill **before** any files are written there, since
`sase repo open` cleans the checkout. If the run does not touch that root, skip this and
say so.

## Verification

- `just check` (run `just install` first if the workspace is stale).
- Confirm `sase memory init --check` exits 0 on the regenerated tree.

## Out of scope

- `sase memory read` output for a web descriptor note (it prints `note.content.body`
  verbatim and would still show markers). The ask was specifically about agent
  instruction files; if that read path is worth cleaning too, it is a separate follow-up
  and should reuse the same helper.
- Any change to how the markers work in the canonical descriptor files, to
  `roster_region_error`, or to the Tier 2 (reference) rendering path, which only emits a
  one-line description and never had markers.
