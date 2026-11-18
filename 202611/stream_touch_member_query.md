---
tier: tale
size: small
title: Select touch rows while decoding, then close out sase-1t3.11.1
goal:
  The Rust touch member query keeps only claimed rows while it decodes the touch index,
  so its transient memory no longer grows with all touches in the project. Then epic
  sase-1t3.11.1 and its parent phase sase-1t3.11 are closed with verified evidence.
proposed_by: bbugyi200.athena.sase-1t3.11.1.land
proposed_with: claude/opus
bead: sase-1t3.11.1
status: done
---

- **PARENT:**
  [202611/bounded_log_queries.md](https://github.com/sase-org/sase--plans/blob/main/202611/bounded_log_queries.md)
- **BEAD:**
  [sase-1t3.11.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/sase-1t3.11.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1t3.11.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1t3.11.1.land.md)
- **COMMITS:**
  - [958e382](https://github.com/sase-org/sase--plans/commit/958e382253f6acb3de32531f3568d7ebe5134964)
    — chore(plan): mark bounded log query epic complete

# Select touch rows while decoding, then close out sase-1t3.11.1

This is the closeout tale for epic `sase-1t3.11.1` (plan
`plan:202611/bounded_log_queries.md`). The landing agent verified all six phases and
integrated later commits. It found one unmet requirement in the epic's own contract. The
shared implementation contract says "a query may scan the source on a result-cache miss,
but it must release unrelated decoded rows as it scans". The `bead_touches` phase says
"read only the touch-query fields of the JSON index and select during decoding".
Everything else in the epic is done. Read the landing note with
`sase bead read sase-1t3.11.1 -r "<why>"`; it is the latest note, which starts with
`LANDING TRIAGE`. Follow-up triage is finished: do not create task beads.

## Evidence

In linked `sase-core` (open it with `/sase_repo`: `sase repo open sase-core`; read its
`AGENTS.md`), `crates/sase_core/src/bead/touch_member_query.rs`
`query_bead_touches_for_members()` calls `std::fs::read(index_path)`. Then
`serde_json::from_slice` decodes `TouchIndexQueryDoc { touches: Vec<BeadTouchWire> }`
for every touch in the project, and only afterwards runs `match_member_index` to filter.
The decoded `signatures` map is already skipped. Every unclaimed touch is still fully
allocated, with its title, verbs `BTreeMap`, creation reason, note preview, and close
wire.

Probe on the live index (`~/.sase/projects/gh_sase-org__sase/agent_bead_touches.json`,
13,132,118 bytes, 16,257 touches) in a fresh process. The query returned 0 rows, but
VmHWM rose 41,000 kB over baseline in 57 ms, and RSS stayed 21.7 MB higher after the
call. By comparison, the streamed memory audit query peaks at about +3.6 MB on 100k
rows. Reproduce with the workspace venv, using the locally built core:

```python
# .venv/bin/python -I probe.py <index_path>
import sys, time, sase_core_rs
def status():
    d = {}
    for line in open("/proc/self/status"):
        if line.startswith(("VmRSS", "VmHWM")):
            k, v = line.split(":"); d[k] = int(v.split()[0])
    return d
open("/proc/self/clear_refs", "w").write("5")
before = status(); t = time.perf_counter()
out = sase_core_rs.bead_touch_member_query(sys.argv[1], {"members": [
    {"globalized_name": "nobody.nowhere.none", "local_name": "none"}]})
after = status()
print(len(out["rows"]), round(time.perf_counter() - t, 3),
      after["VmHWM"] - before["VmRSS"], after["VmRSS"] - before["VmRSS"])
```

## Implementation (sase-core)

1. In `touch_member_query.rs`, make the document decode select claimed rows while it
   parses. Do this with a custom `Deserialize` for the query document, or a
   `DeserializeSeed` for its `touches` sequence. The sequence visitor must test each
   element against the ordered members, with the resolved owner and `known_owner_roots`,
   and drop an unclaimed element before it decodes the next one. Store only
   `BeadTouchMatchWire` rows for claimed elements. Read `schema_version` and
   `generation`. Skip every other top-level key (including `signatures`) with
   `serde::de::IgnoredAny`. Do not use `macro_rules!`.
2. Avoid holding a second full copy of the source where practical. Two options are
   acceptable; pick whichever the measurement favors and say which in the bead note:
   - decode with `serde_json::from_reader` over a `BufReader<File>`; or
   - keep the byte buffer and borrow each element as `&serde_json::value::RawValue`,
     decoding only the actor until it matches. The workspace `serde_json` does not
     enable `raw_value` today; adding that feature is allowed if you choose this route,
     and it may need `cargo hakari generate && cargo hakari manage-deps` for the
     features check.

   Either way, the transient peak must no longer scale with every decoded touch.

3. Preserve all observable behavior exactly:
   - The schema check happens after the full parse, and a mismatch is a miss. Keys may
     appear in any order.
   - A missing, truncated, unparseable, or wrong-schema index returns no rows with an
     empty generation: never partial rows, never an error.
   - Rows are sorted newest `last_at` first, then by bead id, then by actor.
   - Member precedence, legacy bare-local recovery, whitespace trimming, and the result
     wire shape stay the same.
   - The source identity is taken from the same file the query read.

   Keep `TOUCH_MEMBER_QUERY_WIRE_SCHEMA_VERSION` unchanged; the request and response
   shapes do not change. Leave the unfiltered `bead_touch_index_query` reader alone.

4. Tests, next to the existing `mod tests` in `touch_member_query.rs`:
   - All existing tests must keep passing unchanged.
   - Add a case where `schema_version` appears after `touches`.
   - Add a case where a claimed row appears before a truncation point; it must still
     produce a miss with no rows.
   - Add an index with a large `signatures` map and many unclaimed touches. Assert that
     the claimed rows and order match a naive full-decode-then-filter reference built in
     the test.
   - Keep the binding round-trip tests in `crates/sase_core_py/src/beads/tests.rs`
     passing.
5. Run `just fmt`, then `sase tool run check` inside the sase-core checkout (about 5+
   minutes; use an explicit timeout of at least 15 minutes, or `/sase_monitor`).

## Verification in sase

1. Build the local core into this workspace's venv the way `docs/rust_backend.md`
   describes for local core changes (e.g. `just rust-install`, through `/sase_monitor`
   if long). Then re-run the probe above on the same live index and on a synthetic index
   with at least 20,000 unrelated touches plus a fixed matching set. Record before/after
   VmHWM delta, RSS after call, latency, and returned rows.
2. Run the sase touch suites against the locally built core:
   `tests/core/test_bead_touch_index_facade.py`,
   `tests/ace/tui/widgets/test_agent_bead_touches_loader.py`,
   `tests/ace/tui/widgets/test_agent_bead_touches_session_alias.py`,
   `tests/ace/tui/widgets/test_agent_bead_touches.py`, and
   `tests/test_bead/test_bead_views.py`. Use `sase tool run test -- <paths>`.
3. No sase source change is expected. Do not hand-edit `sase-core-revision.txt`: no new
   binding or wire change needs the pin. The host or the scheduled ratchet moves it
   after the core commit exists, and this tale's closeout must not wait for that. If you
   do change sase files, read `lint_and_test.md` through `/sase_memory_read` and run
   `sase tool run check` before finishing.

## Closeout (final step, do not skip or reorder)

Do these in the same turn as the final code. None of them waits for this work's commit
SHA, push, or CI.

1. `sase bead note sase-1t3.11.1 "<touch-query before/after: VmHWM delta, RSS after call, latency, rows, index sizes, sase-core check run id, chosen decode route>"`.
2. `sase bead epic-symbols sase-1t3.11.1`. For each listed `--epic-symbol` entry,
   resolve the symbol (wire it up, privatize it, add a non-test pragma, or delete it per
   the Symvision epic-whitelist policy in `symvision.md`). Re-key the Justfile line only
   if a still-open later bead needs the exemption. At landing time this listed no
   entries.
3. Close the epic: `sase bead close sase-1t3.11.1 --note "<verification>"`. The note
   must summarize:
   - all six phases verified against source and commits (8a6451068e, 1c626f39e5,
     cd0b9c5273, 4894d41991, 5e3931c592, 88431cab71, 81e124eca8; core 19b89ac9,
     c5e60d78, 9f9341e5, 3992fce8, 71c59dca on origin/master, pin 71c59dca);
   - the integration finding: dd56fe2075 deferred the memory-query facade off TUI first
     paint, and the import-budget test passes;
   - the follow-up outcomes from the `LANDING TRIAGE` note: sase-1th created, sase-xo
     corroborated, the others declined as resolved;
   - this tale's touch-query fix and measurements.

   Never use `--force` to make the close succeed. If the close refuses because of
   leftover epic-symbol entries, finish that cleanup and close again.

4. Run `just symvision` in the sase checkout and confirm it is clean.
5. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   `sase bead read sase-1t3.11.1` shows for `plan:202611/bounded_log_queries.md`,
   currently `status: wip`.
6. Parent handling. `sase-1t3.11.1`'s `parent_bead` is phase bead `sase-1t3.11`
   ("Replace whole-file log caches with targeted queries"), a phase of epic `sase-1t3`.
   - Confirm the child plan completed that phase: artifacts index, artifact_reads,
     memory_reads, bead touches, and stranded parses all use targeted indexed or
     streamed queries. The per-agent query shape is kept, and render paths stay stat-
     and glob-free.
   - Run `sase bead epic-symbols sase-1t3.11` and resolve any entries the same way.
   - Append the touch-query result to the existing `EVIDENCE SUMMARY` note on
     `sase-1t3.11` with `sase bead note sase-1t3.11 "..."`. Then close only that phase:
     `sase bead close sase-1t3.11 --note "<what you verified>"`.
   - Do not close `sase-1t3` or any higher ancestor; its land agent is already waiting.
7. Finish through `/sase_final`, declaring the sase-core change, any sase change, and
   the plans-repo frontmatter edit.
