---
tier: tale
title: Hint bare bead IDs from every project in the pager
goal:
  Every bead pager document hints its parent, child, and related bead IDs whatever the
  project prefix, so the user can always jump parent to child and child to parent, with
  no noticeable pager slowdown.
size: medium
proposed_by: bbugyi200.athena.0yw
create_time: 2026-10-09 09:13:16
status: wip
---

# Pager: hint bare bead IDs from every project, not just `sase-*`

## Problem

In a bead pager document the user cannot always jump between parent and child beads with
a hint. Example: `bob-cli-5s` (an epic in the `bob-cli` project). Its CHILDREN rows
(`bob-cli-5s.1` … `bob-cli-5s.10`) get no `[x]` capsule, while `agent:bob-cli-5s.2` and
`plan:…` refs in the same document do. The same bead opened from a `sase-*` epic hints
every child, PARENT, BLOCKS, and NOTES ID. So hints seem to come and go, depending only
on which project owns the bead.

## Root cause (diagnosed and reproduced)

1. **The bare bead-ID recognizer is hard-coded to the `sase-` prefix.**
   `src/sase/pager/link_scan.py` defines

   ```python
   _BARE_BEAD_ID_RE = re.compile(r"(?<![\w-])sase-[0-9a-z]{1,4}(?:\.[A-Za-z0-9]+)*(?![\w-])")
   ```

   and maps it to `PagerOrigin.BEAD` and `PagerOrigin.AGENT`. The comment above it calls
   generalizing to other stores' keys a deferred follow-up. Bead detail bodies print
   relations (CHILDREN/PHASES/CHILD EPICS, PARENT, DEPENDS ON, BLOCKS, assignee, notes)
   as **bare IDs**, never as `bead:` refs. So the Rust document-link scanner never sees
   them, and only this regex can turn them into hint targets. For any project whose bead
   prefix is not `sase` (`bob-cli`, `actstat`, …), the regex matches nothing and the
   document has no bead hints at all. Reproduced: building the real `bob-cli-5s`
   document through `sase.pager.beads._bead_id_link_resolution` and running
   `section_target_spans` produced only `url`/`artifact_ref`/`file_path` spans. The same
   path for `sase-1h8.1` produced `bare_token` spans for `sase-1h8`, `sase-1h8.8`, and
   others.

2. **A latent mis-styling tied to the same assumption.**
   `src/sase/pager/_labels.py::_target_artifact_tab` classifies a BARE_TOKEN as
   `"beads" if target.text.startswith("sase-") else "patches"`. Once (1) is fixed, every
   non-`sase` bead ID would be painted with the Patches icon and accent.

Following a hint is **not** broken. `target_resolution_ref` turns a BEAD/AGENT bare
token into `bead:<id>`. `_bead_id_link_resolution` → `resolve_show_batch` falls back
from the local store to `ShowStoreRouter.foreign_target_for_bead_id`, which routes by
prefix over enabled projects (verified: `bob-cli-5s.10` resolves from a `sase`
checkout). Only recognition is missing.

## Design

Keep bare-token recognition I/O-free and owned by `link_scan.py`, as its module
docstring requires. The prefixes it accepts become an explicit, per-section input that
each document producer supplies, following the existing `known_kinds` precedent. That
input is computed once per document build, never in render or keypress paths.

This stays in Python on purpose. `link_scan.py` already documents that Rust owns
document-link grammar and Python owns origin-scoped bare-token recognition. This fix
does not move that boundary.

### Prefix sources (all cheap)

For a bead document section, the prefix set is the union of:

- **the bead's own prefix**, derived from `issue.id`. Pure string work. This alone
  guarantees parent↔child hints, because children are always `<parent-id>.<n>` in the
  same store.
- **prefixes of every structured relation ID** on the `IssueDetail` (`ancestors`,
  `phases`, `child_epics`, `depends_on`, `blocks` → `IssueRef.issue_id`). Pure, and it
  also covers a cross-store dependency.
- **enabled projects' refs**: `{project_name, effective_project_name(record), *aliases}`
  for each enabled project record. These are exactly the values the routing layer
  already accepts as prefixes (`cross_project._prefix_matches_store`), and the default
  issue prefix is derived from them (`prefix_policy.default_issue_prefix`). This keeps
  prose mentions of another enabled project's beads hinted, for example a `sase-…` ID
  inside a `bob-cli` bead that `sase-` previously matched by accident, so nothing
  regresses. Measured cost is about 0.7 ms warm and about 7 ms for the first
  `list_project_records` call in a process. Do **not** read each project's stored
  `issue_prefix` from `config.json`. That needs `canonical_beads_dir_for_project`, about
  11 ms per project. The own-prefix source already covers custom prefixes for the
  document's own bead.

Sections that declare no prefixes keep today's behavior through a documented fallback
`DEFAULT_BARE_BEAD_ID_PREFIXES = ("sase",)`. That fallback covers tool-run log documents
and any other AGENT/BEAD producer this tale does not touch.

## Implementation steps

### 1. Prefix-aware recognizer — `src/sase/pager/link_scan.py`

- Replace the module-level `_BARE_BEAD_ID_RE` with:
  - `DEFAULT_BARE_BEAD_ID_PREFIXES: tuple[str, ...] = ("sase",)`.
  - `normalize_bead_id_prefixes(prefixes: Iterable[str]) -> tuple[str, ...]`. It strips
    each value and drops empty or unsafe values: whitespace, `.`, `/`, `\`, `--`, or a
    trailing `-`, matching `sase.bead.prefix_policy._is_safe_bead_prefix`. Re-implement
    the check locally so the cold path does not import the bead package. It then dedupes
    and returns a **sorted** tuple, which makes a stable cache key.
  - `_bare_bead_id_regex(prefixes: tuple[str, ...]) -> re.Pattern[str]`, wrapped in
    `functools.lru_cache(maxsize=64)`. It builds
    `(?<![\w-])(?:<alt>)-[0-9a-z]{1,4}(?:\.[A-Za-z0-9]+)*(?![\w-])`, where `<alt>` is
    the `re.escape`d prefixes ordered longest-first. The ID grammar after the prefix is
    unchanged, so `sase-*` matching is byte-for-byte identical when the prefix set is
    `("sase",)`.
- Replace the static `_BARE_TOKEN_RECOGNIZERS` mapping with
  `_bare_token_recognizer(origin, bead_id_prefixes) -> Callable | None`. BEAD/AGENT use
  the prefix regex: normalized prefixes, or `DEFAULT_BARE_BEAD_ID_PREFIXES` when the
  normalized tuple is empty. DIFF keeps `_BARE_SHORT_SHA_RE.finditer`. Every other
  origin returns `None`.
- Add a keyword `bead_id_prefixes: Iterable[str] = ()` to `scan_links` and
  `scan_bounded_links`, and thread it to the recognizer. Leave the first-wins precedence
  and the `_FrozenSpanIndex` overlap logic untouched.
- Add a public `is_bare_short_sha(text: str) -> bool`, a `fullmatch` against the SHA
  grammar, for step 3.
- Update the comment above the recognizer: the deferred follow-up is now done.

### 2. Section field — `src/sase/pager/document.py`

- Add `bead_id_prefixes: tuple[str, ...] = ()` to `PagerSection` as the **last** init
  field, after `version_pin`, so no positional construction shifts. In `__post_init__`,
  normalize it with `normalize_bead_id_prefixes`, mirroring how `known_kinds` is
  normalized.
- In `section_target_spans`, pass `bead_id_prefixes=section.bead_id_prefixes` to
  `scan_links`. The memo key can stay `("spans", effective_origin)`: the field is part
  of the immutable section, and `dataclasses.replace` produces a fresh section with an
  empty memo.
- Add `section_with_bead_id_prefixes(section, prefixes) -> PagerSection`. It returns
  `section` unchanged when the normalized prefixes are equal. Otherwise it returns a
  `copy.copy` with the field set and a **fresh empty `_memo`**, so a stale span memo is
  never shared. This avoids re-running `__post_init__`, which re-renders the body, when
  a producer stamps prefixes onto already-built sections (step 6). Export it in
  `__all__` next to the other `section_*` helpers.
- Confirm `sase.pager.document` still passes
  `tests/pager/test_cold_path_import_cost.py`. Steps 1 and 2 must add no new top-level
  imports beyond the stdlib.

### 3. Label tab classification — `src/sase/pager/_labels.py`

- In `_target_artifact_tab`, replace the `startswith("sase-")` test with
  `"patches" if is_bare_short_sha(target.text) else "beads"`. A bare short SHA (DIFF
  origin) is pure hex. A bead ID always contains `-`. Non-`sase` bead IDs then get the
  Beads icon and accent.

### 4. Prefix helpers

- `src/sase/bead/cross_project.py`: add public
  `enabled_project_ref_prefixes() -> tuple[str, ...]`. It returns
  `sorted(set().union(*(_project_refs(r) for r in _enabled_project_records())))`. Its
  docstring explains that project refs double as default bead-ID prefixes for routing,
  and that it reads no bead store. Add it to `__all__`.
- New module `src/sase/pager/bead_prefixes.py`. It must be Textual-free and must import
  `sase.bead.cross_project` lazily inside the function, so the pager cold path stays
  light. It holds:
  - `bead_id_prefix_of(bead_id: str) -> str | None`. Take the top-level segment before
    the first `.` and `rpartition("-")`. Return `None` when there is no separator or the
    prefix is empty. Pure; no bead-package import.
  - `pager_bead_id_prefixes(bead_ids: Iterable[str] = (), *, include_enabled_projects: bool = True) -> tuple[str, ...]`.
    This is the union of `bead_id_prefix_of` over `bead_ids` and, when requested,
    `enabled_project_ref_prefixes()`. The enabled-project part is **best-effort**: catch
    `Exception`, log at debug, and fall back to the ID-derived prefixes, because a
    document build must never fail over hints. Return `normalize_bead_id_prefixes(...)`
    of the union.

### 5. Bead documents

- `src/sase/bead/cli_show_batch.py::_show_batch_sections`: before the loop, compute the
  enabled-project part **once per batch**. Either call
  `pager_bead_id_prefixes((), include_enabled_projects=True)` once and union per entry,
  or pass a precomputed tuple; never call the project listing once per entry. For each
  entry, union in `bead_id_prefix_of(issue.id)` and the prefixes of every
  `IssueRef.issue_id` in `detail.ancestors`, `detail.phases`, `detail.child_epics`,
  `detail.depends_on`, and `detail.blocks`. Pass the result as `bead_id_prefixes=` to
  the `PagerSection(...)` constructor. The later `replace(section, targets=...)` for
  attachments keeps the field automatically. This one change covers `sase bead show`
  (CLI pager), pager `bead:` follows (`sase.pager.beads._bead_id_link_resolution`), and
  `cli_query.py`'s full render, since all of them go through
  `build_show_batch_document`.
- `src/sase/artifact_cli/read.py::_page_markdown`: when
  `result.parsed.kind_type == "bead"`, pass
  `bead_id_prefixes=pager_bead_id_prefixes((result.parsed.payload.id,))`. Guard
  `payload.id` being `None`. Leave non-bead artifacts unchanged.

### 6. Agent metadata documents — `src/sase/ace/tui/actions/agents/_metadata_pager_document.py`

- `build_agent_metadata_document` already runs off the event loop (its docstring
  requires `asyncio.to_thread`), and its refresh provider rebuilds through it. Compute
  the prefixes once from the agent's own bead IDs: `summary.bead_summary.id` when
  present, plus `agent.epic_bead_id` and `agent.phase_bead_id` when they are non-empty
  strings. Use `getattr` guards, as `models/agent.py` does. Combine those with the
  enabled-project refs via `pager_bead_id_prefixes(...)`. Then stamp the prefixes onto
  every assembled section, including `build_agent_conversation_sections` output and the
  "No metadata" fallback, using `section_with_bead_id_prefixes`. Do not thread a new
  kwarg through each section builder, and do not use `dataclasses.replace`, which would
  re-render bodies.
- Do **not** change the two tool-run log documents
  (`src/sase/ace/tui/tool_runs/hints.py`,
  `src/sase/ace/tui/modals/tool_runs_pane_tool_actions.py`). The modal builds its
  document in a UI-thread action handler, where project-record I/O is not allowed (TUI
  perf rule 1). Both keep the `("sase",)` fallback.

### 7. Docs

- `docs/pager.md` → "Document origins": say that `bead` and `agent` documents recognize
  bare bead IDs for the document's own bead prefix, its related beads' prefixes, and
  every enabled project's name or alias, for example `sase-uk.7` or `bob-cli-5s.1`. Say
  that documents without declared prefixes fall back to `sase`.

## Tests

Add or update these tests (pytest, existing style):

- `tests/pager/test_link_scan.py`
  - `bead_id_prefixes=("bob-cli",)` recognizes `bob-cli-5s`, `bob-cli-5s.1`,
    `bob-cli-5s.10`, and `bob-cli-5s.land` in BEAD and AGENT origins. It does not
    recognize them in FILE, RESEARCH, or DIFF origins. With the default (no prefixes), a
    `bob-cli-5s.1` token stays unrecognized and `sase-uk.1` stays recognized, which
    proves the fallback.
  - Multiple prefixes, including a prefix that is a dash-prefix of another (`sase` and
    `sase-github`): `sase-github-1a` matches as one token with the longer prefix, and
    `sase-uk.1` still matches.
  - Prefix normalization: an unsafe prefix (`"a.b"`, `" "`, `"x-"`, `"a--b"`) is
    dropped, and duplicates collapse. The regex cache key is order-independent: the same
    compiled object for `("a","b")` and `("b","a")`.
  - Update `_reference_scan_links` (the parity oracle) and
    `test_scan_links_matches_reference_on_random_inputs` to use the new recognizer
    selection, with a `bob-cli-5s.1` parity piece and a random prefix set chosen per
    trial from `{(), ("sase",), ("sase","bob-cli")}`. Extend
    `test_scan_links_stays_near_linear_on_link_dense_input` with a multi-prefix variant.
    Example lines: `... sase-ab.{i} bob-cli-cd.{i}`, scanned with
    `bead_id_prefixes=("sase","bob-cli")`. It should assert zero linear-overlap calls
    and one index query per bare token.
- `tests/pager/test_labels.py` (or the nearest existing marker test): a non-`sase` bare
  bead token gets the Beads icon and accent, and a DIFF bare short SHA still gets the
  Patches marker.
- `tests/pager/test_document.py`: `PagerSection(bead_id_prefixes=...)` normalizes, and
  `section_target_spans` honors it. `section_with_bead_id_prefixes` returns the same
  object for equal prefixes. For a changed set it returns a copy whose spans reflect the
  new prefixes, even after the original's spans were memoized.
- `tests/test_bead/test_bead_show_pager.py`: add a fixture epic with a non-`sase`
  prefix, such as `bob-cli-zz` with phases `bob-cli-zz.1` and `bob-cli-zz.2` and a child
  epic `bob-cli-zz.3`, plus a phase whose PARENT row names the epic. Use the existing
  fake-view pattern. Assert that the epic's document has BARE_TOKEN spans for every
  child ID in CHILDREN, and that the phase's document has a BARE_TOKEN span for the
  parent epic ID. Add a pilot test, mirroring
  `test_bead_show_bare_token_follows_through_the_real_resolver`, that follows a child
  hint from the parent and the parent hint from the child. The suite's isolated sase
  home has no enabled projects, so these tests prove the own-prefix and structured-ID
  sources without depending on the registry.
- `tests/test_bead/` (cross_project tests) or a new `tests/pager/test_bead_prefixes.py`:
  - `bead_id_prefix_of` on `bob-cli-5s.10`, `sase-1h8`, `nodash`, and `""`.
  - `pager_bead_id_prefixes` unions ID-derived and monkeypatched enabled-project
    prefixes, and degrades to ID-derived prefixes when the enabled-project lookup
    raises.
  - `enabled_project_ref_prefixes` against a seeded project record, following the
    existing cross_project test fixtures.
- Agent metadata document: assert that every section of a built document for an agent
  whose bead is `bob-cli-zz.1` carries a `bob-cli` prefix, and that its BEAD section
  scans a BARE_TOKEN for `bob-cli-zz.1`. Use the existing tests for
  `_metadata_pager_document` as the template.
- `tests/pager/test_cold_path_import_cost.py` must keep passing unchanged.

## Performance

Requirement: the fix must not noticeably slow the pager.

- Scanning: a cached compiled regex per normalized prefix tuple. Section spans are
  already memoized per section. Prototype measurement on a 296 KB bead-detail corpus
  (larger than the 128 KB hint budget), six prefixes vs the current single `sase`
  literal: 7.7 ms vs 6.6 ms per full scan, so at most about +0.5 ms at the budget cap,
  once per section.
- Prefix discovery: own and structured prefixes cost nothing. Enabled-project refs cost
  one `list_project_records` call per document build (about 0.7 ms warm), only on paths
  that already run off the UI thread or in the CLI: bead-link follow via
  `asyncio.to_thread`, the agent metadata build via `to_thread`, and the
  `sase bead show` CLI. No render, scroll, or keypress path does new I/O.
- Labels: dangling state is a set lookup, so extra targets add only linear
  label-allocation work, far below the 2,704 two-key label capacity.
- Verification during implementation: time
  `sase.pager.beads._bead_id_link_resolution("<a real multi-child epic>")` and
  `scan_links` on the rendered body before and after, and report both numbers. The
  document build should change by no more than a few milliseconds. Also confirm with
  `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` that the pager-adjacent j/k p95 is
  unchanged if that bench covers the pager. Otherwise, note that it does not.

## Out of scope

- Moving bare-token recognition into Rust `sase_core`. `link_scan.py` documents that
  Python owns it today.
- Tool-run log documents. They keep the `sase` fallback, as explained in step 6.
- Pre-existing false positives such as `sase-core` matching the ID grammar. With more
  prefixes, a project name followed by a short word, such as `bob-cli-docs`, can also
  match. Such a hint resolves to "could not be resolved" and is then painted as
  dangling, which is the same failure mode as today.

## Verification

- `just check` passes. Read the `lint_and_test` memory first, per the repo rules.
- Manual check: `sase bead show bob-cli-5s` (or any non-`sase` epic) in the pager shows
  `[x]` capsules on every CHILDREN row with the Beads icon. Following one opens the
  child. In the child, the PARENT row's epic ID has a hint that jumps back.
  `sase bead show sase-1h8` hints exactly as before.
