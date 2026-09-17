---
tier: tale
title: Add the record-before-admit decision record
goal:
  The decisions memory web durably records the sase tool ordering invariant (record
  first, admit last) so all eight upcoming epic plans can cite it.
size: small
proposed_by: bbugyi200.athena.0mk
create_time: 2026-09-17 15:43:49
status: wip
---

# Add the `record-before-admit` decision record

## Objective

Complete step #2 of the "First moves" section of the consolidated `sase tool` epic
roadmap (`research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md`): add one
new strand to the `decisions` memory web recording the accepted ordering invariant of
the upcoming `sase tool` epics — _expensive commands are recorded before they are
admitted_ — so that all eight epic plans can cite it instead of re-litigating it.

Context already settled (do not redo):

- Step #1 is done: epic `sase-zm` was closed with resolution `superseded` on 2026-09-17,
  reason pointing at the roadmap report.
- The strand content below was authored against the consolidated roadmap report and the
  existing `decisions` strands (`corpus-before-mechanism`, `two-speed-verification`,
  `host-owned-completion`) to match the web's established **Claim / Why / Cost / Reopens
  when** shape. Use it verbatim.
- Strand frontmatter accepts `keyword`, `aliases`, `summary`, and optional `metadata`;
  it must NOT declare `type` or `parent` (`parse_memory_strand` in
  `src/sase/memory/web/frontmatter.py` rejects both). The generated roster in
  `AGENTS.md` / provider shims is derived from strand frontmatter by `sase memory init`,
  so no descriptor edit is needed.

## Steps

1. Invoke the `/sase_memory_write` skill and record its use with
   `sase skill use sase_memory_write --reason "..."`. Authorization: this approved plan
   names the change (plan approval is user approval).

2. Create `sase/memory/decisions/record-before-admit.md` with exactly this content:

   ```markdown
   ---
   keyword: Expensive Commands Are Recorded Before They Are Admitted
   aliases:
     - record before admit
     - record first, admit last
     - ToolRun ledger first
   summary:
     The sase tool control plane lands its ToolRun record first; prediction and
     admission only ever consume a corpus that already exists.
   ---

   **Claim.** An expensive command becomes a named tool whose every run is durably
   recorded — fingerprint, per-stage timings, host load samples — before anything
   predicts its duration or prices its admission. The ToolRun ledger lands first, in a
   new per-machine Rust-owned store with its own retention; prediction consumes the
   corpus only after weeks of samples and stays advisory until backtested; admission
   consumes calibrated prediction, never guesses. A ToolRun is a semantic record that
   delegates execution to an existing executor (inline child, monitor, durable proc):
   recording adds no new supervisor, and recording stays fail-open even where admission
   later becomes fail-closed.

   **Why.** Load samples cannot be backfilled — the tree records none today — so every
   unrecorded week delays calibration by a week, while a record pays for itself even if
   no consumer ever ships. This applies [[decisions/corpus-before-mechanism]] to
   execution machinery. Rejected alternatives: **admission-first** (epic `sase-zm`,
   closed superseded 2026-09-17) priced and queued work before any evidence of what work
   costs — 14 phases, zero closed, nothing landed — and the measured waste points the
   other way: on apollo over 14 days, 37.2 h of repeated verification commands and 16.24
   h of `check-full` runs re-discovering one month-old known-red master (waste that
   records — triage, receipts — attack and a queue does not), with zero
   concurrent-duplicate request fingerprints for admission to prevent.
   **Recognition-by-profile** — inferring cost by matching argv patterns instead of
   naming tools — leaves no stable identity for fingerprints, receipts, or quantiles,
   and opt-in recognition measurably goes unused (`verify` profile on 1 of 69 monitors,
   prepared host completion on 0 of 69). **Reusing the proc store** fails on retention:
   `procs.history_limit: 100` would destroy the corpus in days, and the telemetry store
   keeps only aggregates. Evidence and epic decomposition:
   `research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md`.

   **Cost.** A new per-machine durable store and a new disk-retention class registered
   with `sase disk`; recording ships well before anything user-visible consumes it; and
   adoption is instruction-led (guidance plus the wrapper — provider hooks are gone), so
   bypass is measured, not enforced. The commands being recorded are the same expensive
   verification runs that [[decisions/two-speed-verification]] rations.

   **Reopens when.** The ledger itself measures a concurrent-duplicate or
   queue-starvation rate that recording, triage, and receipts do not mitigate — evidence
   that admission or single-flight joining must move earlier than calibrated prediction
   allows.
   ```

   (Strip the outer four-backtick fence; the file starts at byte 0 with the `---`
   frontmatter line. Keep the `[[...]]` links exactly as written — the web renders them
   as authored references per the memory-links convention.)

3. Run `sase memory init` to regenerate `AGENTS.md`, the provider instruction shims, and
   the memory README. Never hand-edit the generated files.

4. Verify:
   - `sase memory web show decisions` lists the new strand with keyword, aliases, and
     summary.
   - `sase memory show decisions:record-before-admit` renders the body and resolves both
     `[[...]]` links.
   - The regenerated agent-instruction roster includes the new record's one-line
     summary.

5. Read `sase/memory/lint_and_test.md` with the `/sase_memory_read` skill (tracked files
   changed) and run the checks it requires (`just check` is the agent default per the
   two-speed-verification decision).

## Non-goals

- Do not start roadmap first-moves steps #3 (LLM Calls rename) or #4 (the E1 plan).
- Do not edit the `decisions` web descriptor, any existing strand, or any generated file
  by hand.
- Do not create any new task beads for this work.
