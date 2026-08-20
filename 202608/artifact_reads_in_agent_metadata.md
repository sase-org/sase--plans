---
tier: tale
title: Audited artifact reads in agent metadata
goal: "Make every explicit `sase artifact read` by a selected SASE agent or family
  visible, attributable, and safely navigable in a polished `Reads:` block inside `SASE
  CONTEXT / ARTIFACTS`, without adding hot-path I/O or conflating reads with prompt
  citations.

  "
size: medium
proposed_by: bbugyi200.athena.08m
create_time: 2026-08-20 11:58:46
status: wip
---

# Plan: Audited artifact reads in agent metadata

## Outcome and user experience

The Agents metadata panel will treat explicitly read artifacts as inputs to the agent's
work and place them first in the existing output-oriented `ARTIFACTS` lane:

```text
▸ ARTIFACTS · 2 reads · 1 commit · 3 files · 1 artifact file
  Reads:
    14:22:08  coder  ← [3] plan:202608/design.md
                       ↳ compare the approved constraints with this implementation
    14:18:31  plan   ← [4] research:202608/prior-art.md
                       ↳ reuse the established interaction language
  Commits:
    ...
  Deltas:
    ...
  Files:
    ...
```

`Reads` comes before `Commits`, `Deltas`, and `Files`, establishing a natural
inputs-to-outputs narrative. Rows use the ARTIFACTS blue palette, a one-cell incoming
arrow, local time, the canonical artifact reference, and the audited reason on a
losslessly wrapped continuation line. Family rows retain the existing compact producer
column (`plan`, `coder`, and so on); single-agent rows do not gain an empty column.

Show the newest five read events and then a dim `+ N more · HH:MM earliest` footer.
Repeated reads of the same reference remain separate because their timestamps and
reasons are meaningful. The lane summary counts every retained read event, while the
existing commit/delta/artifact-file counts retain their current meanings and wording.
When hint mode is active, a read with a recorded resolved path gets the next yellow
`[N]` marker and opens that exact path. A legacy or non-filesystem read without a path
still renders its canonical reference and reason, consumes no hint number, and never
attempts live reference resolution.

## Source of truth and compatibility

Use the project-scoped `artifact_reads.jsonl` audit log as the sole source of this UI.
It is written for every successful `sase artifact read`, including when the
`artifact_links` feature flag is disabled. Do not derive the section from the artifact
consumption ledger, because that ledger also contains prompt-reference expansion, and do
not derive it from `read` graph edges, because those edges are flag-gated.
`sase artifact show`, `path`, and `open` remain silent and must never appear in `Reads`.

Extend the version-1 `ArtifactReadEvent` wire shape with an optional `resolved_path`.
Keep the schema version at 1 and make the reader accept an absent or null value so all
existing rows remain displayable. Have `artifact read` pass the actual path returned by
`_prepare_body` into the audit event; this is important for materialized VCS-backed
files, where the usable path can differ from or be absent on the original resolution
object. Persist the canonical, fragment-free reference as the stable display identity
and use the resolved path only for navigation. Continue to refuse to print an artifact
when the required audit append fails.

## Cached agent and family projection

Add an ACE artifact-read loader alongside `memory_reads.py`, with a small immutable
display wrapper carrying the event and optional family-role label. Follow the proven
audited-context rules rather than introducing a new generic cache abstraction:

- Determine the project from the agent workspace, with the existing current-project
  fallback, and read `artifact_read_log_path(project)`.
- Match by realpath-normalized `artifacts_dir` first. Only events with no artifacts
  directory may fall back to exact `agent_name`; an event owned by another artifacts
  directory must never leak onto the selected row.
- For an ordinary row, preserve the single-agent shape. For a family, use
  `build_context_members`, `match_event_label`, and `context_cache_key` to aggregate the
  selected agent plus its known follow-ups, de-duplicate by event ID, attach the compact
  producer label, order newest first, and cap the retained snapshot at 50.
- Cache the parsed project log by `(mtime_ns, size)` with the existing short reread
  throttle and a bounded project snapshot cache. Keep filtered single-agent and family
  results separately cached. Malformed, wrong-schema, missing, or unreadable log rows
  degrade to an empty/partial result rather than breaking metadata enrichment.

All stat, lock, JSON parsing, sorting, and filtering stays in the existing threaded
detail-header enrichment worker. Add `artifact_reads` to `DetailHeaderSummary` and to
the `artifacts` lane's merge fields, resolve it in the second (store-backed) batch, and
give it its own `tui_trace` child span. Do not add disk access to the immediate paint,
render functions, hint keystrokes, or the Textual event loop. Preserve the artifacts
lane's current ten-second freshness cadence and the thirty-second hint-session widening
so new events appear promptly without renumbering hints while the user is reading.

## Rendering and aggregation

Thread the precomputed display events through `build_header_text`,
`append_agent_context_section`, and `append_agent_artifacts_lane`. Extract the compact
row renderer into an artifact-read-specific helper so the ARTIFACTS lane remains the
single owner of sub-section ordering and count composition. Reuse the shared timestamp,
role-column, reason-wrap, truncation, and hint helpers; add only artifact-read palette
and glyph constants needed to make the row visually distinct. Omit `Reads:` entirely
when there are no matching events, leaving existing panels and snapshots unchanged. An
agent whose only artifact context is a read must still get `SASE CONTEXT` and an
`ARTIFACTS` lane.

Include artifact reads in clan disk aggregation as ARTIFACTS entries keyed by their
canonical reference, preserving member attribution and using `resolved_path` as the hint
target when present. Clan containers retain their established flattened context
presentation; the nested `Reads:` treatment applies to real SASE-agent and family
metadata rows, while the clan aggregate still makes the same read artifacts visible and
navigable.

## Documentation

Update the ACE and artifact-file documentation that describes the `SASE CONTEXT` lane:
document the `Reads`, `Commits`, `Deltas`, `Files` order; clarify that `Reads` means
audited `sase artifact read` invocations rather than prompt citations or silent artifact
inspection; and note newest-first ordering, reasons, family attribution, overflow, and
hint behavior. Correct any nearby stale ARTIFACTS sub-section terminology encountered
while editing the same paragraphs.

## Verification

Add focused regression coverage at each boundary:

1. Audit-log and CLI tests prove `resolved_path` round-trips, legacy rows without the
   field still parse, invalid field types are skipped, the prepared/materialized path is
   recorded, flag-off reads are represented, and audit-write failure still prevents
   output.
2. Loader tests cover exact artifacts-directory attribution, name fallback only for
   pathless ownership, symlink/trailing-slash normalization, cross-agent isolation,
   newest-first ordering, retained limits, family labels and ID de-duplication,
   malformed/missing logs, shared parse reuse, stat invalidation, and the reread
   throttle.
3. Renderer tests cover read-only ARTIFACTS lanes, summary grammar and sub-section
   order, timestamps, canonical refs, reasons, role-column alignment, 80-cell wrapping,
   five-row overflow, repeated refs, resolved and pathless hint behavior, and no-output
   compatibility.
4. Detail-summary lane, streaming/async, trace, and benchmark contracts include the new
   field/resolver and prove partial lane merges cannot erase reads and hot rendering
   performs no read-log I/O.
5. Clan aggregation and clan hint tests prove canonical-ref de-duplication, member
   attribution, and resolved-path navigation.
6. Add or adapt a deterministic Agents `SASE CONTEXT` PNG scenario containing multiple
   attributed reads plus existing artifact outputs. Inspect the result, accept only the
   intentional golden change, and rerun it at exact pixel equality.

Run `just install` before verification. Run the focused unit/integration tests while
iterating, then `just check`. Run the relevant visual test with
`--sase-update-visual-snapshots`, inspect the actual image and diff, and rerun the same
test without the update flag; finish with `just test-visual` if the focused visual lane
does not exercise all affected snapshots. If `just check` escalates or reports unusual
selection, use `/sase_monitor` for `just check-full` as required by the repository
instructions.

## Acceptance criteria

- A successful explicit `sase artifact read` appears under
  `SASE CONTEXT / ARTIFACTS / Reads` for the owning agent even when artifact links are
  disabled; prompt citations and silent artifact commands do not appear.
- Real agent and family rows render the specified input-first, newest-first visual
  hierarchy with reasons, correct producer attribution, bounded overflow, stable count
  grammar, and safe hints; empty panels remain unchanged.
- Existing audit rows remain readable. Missing paths and malformed/unreadable log data
  fail soft and never trigger live artifact resolution on a UI or keystroke path.
- Family and clan aggregation never leak another run's events and preserve navigable
  resolved paths where available.
- The new store read is off-thread, stat-keyed, throttled, bounded, traceable, and
  covered by cache/performance regressions.
- Documentation, unit/integration tests, and the inspected PNG golden agree on the final
  presentation, and repository verification passes.
