---
tier: tale
title: Fix Antigravity usage collection broken by agy's epoch-zero reset sentinel
goal:
  agy usage probes record fresh windows again, and one drifted bucket no longer blanks
  every agy window.
size: small
decisions:
  bucket_isolation:
    ask:
      Should one unreadable agy bucket be dropped with a Partial diagnostic instead of
      failing the whole observation?
    choices:
      isolate:
        Drops only the bad bucket; the other windows land as a Partial ok observation
        naming it
      strict: Any unreadable bucket still blanks every agy window, as in this outage
    default: isolate
    why:
      One drifted bucket blanked the in-use Gemini windows for 40h; the store merges
      Partial safely
    answer: isolate
proposed_by: bbugyi200.athena.1g0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 10:37:56
status: wip
---

# Fix Antigravity usage collection: recognize agy's epoch-zero reset sentinel

## Diagnosis (confirmed by live reproduction)

The suspicion is correct: Antigravity (`agy`) usage windows have not been collected
since 2026-11-23 18:31 EST.

Evidence:

- `sase usage list -p agy --json` reports `collection_status: error`,
  `collection_reason: malformed_payload`, and `diagnostic: agy_usage_malformed`. The
  collector has failed 81 consecutive times. The last success was five minutes before
  the first failure. The four stored windows are about 40h old and still read "100%
  left". The live Gemini 5-hour bucket is actually at about 39% left.
- The probe transport is healthy. `agy 1.5.17` passes the 1.1.11 version floor. Running
  `agy -p /usage --output-format json --mode plan --sandbox --print-timeout 15s` with
  `AGY_CLI_DISABLE_AUTO_UPDATE=1` exits 0 with `status: SUCCESS`, `num_turns: 0`, an
  empty `conversation_id`, and valid JSON. No auth, timeout, rate-limit, or JSON-parse
  path is involved; the failure is produced inside the Rust normalizer.
- The vendor payload shape changed for _unstarted_ buckets. The "Claude and GPT models"
  buckets (`3p-weekly`, `3p-5h`) carry `remaining_fraction: 1` and no `description`.
  They now report `"reset_time": "1970-01-01T00:00:00Z"`. The matching `response` TSV
  rows show the same timestamp. Earlier agy builds reported a rolling
  request-time-plus-window reset for unstarted buckets; the existing
  `unstarted_bucket_rolling_reset_passes_through` test pins that older shape. The stored
  windows from the last success (`period_start == observed_at`) match it too.
- Replaying the captured payload through the `provider_usage_normalize_agy_usage`
  binding:
  - as captured: `error` / `malformed_payload` / `agy_usage_malformed`, no windows
  - with the sentinel replaced by a real timestamp, removed, or set to `null`: `ok`,
    four windows
  - with the `3p` group removed: `ok`, two Gemini windows
  - with the sentinel replaced by `""`: still malformed, as expected

Mechanism, in the linked `sase-core` repo's
`crates/sase_core/src/provider_usage/agy.rs`:

1. `bucket_window` calls `parse_reset_time`, which parses the sentinel to `0.0`. The
   function then builds `resets_at = Some(0.0)` and
   `period_start = Some(0.0 - duration)`.
2. `normalize_agy_usage` runs `validate_usage_observation`. Its private
   `validate_window` (in `provider_usage/mod.rs`) calls `validate_epoch_seconds`, which
   rejects any timestamp `<= 0`.
3. `normalize_agy_usage` handles _any_ validation failure by replacing the whole
   observation with the `agy_usage_malformed` error. The two valid Gemini windows are
   discarded with the bad 3p buckets, so a single unstarted bucket blanks all agy usage.

The root cause has two parts:

- The normalizer does not recognize the vendor's epoch-zero "no reset clock yet"
  sentinel. This is the trigger, and it is required to fix.
- Bucket failures are all-or-nothing, so one drifted bucket wipes out every agy window.
  This sets the blast radius and is the `bucket_isolation` decision.

Not involved: the Python collector (`src/sase/llm_provider/usage/agy.py`), the version
gate, the capability cache, the refresh scheduler, and the store merge logic. The agy
model catalog drift is real but separate; see Out of scope.

## Changes

All Rust work happens in the linked `sase-core` repo. Open it with
`sase repo open sase-core -r "<why>"` and read its `AGENTS.md` before editing.

### 1. sase-core: treat a non-positive reset time as "no reset reported" (required)

In `crates/sase_core/src/provider_usage/agy.rs`, `bucket_window`:

- When the parsed `reset_time` is `<= 0.0` epoch seconds, treat it as absent, the same
  as `null`: `resets_at = None`, so `period_start = None`. This covers the Unix-epoch
  sentinel agy now emits. It also covers Go's zero time `0001-01-01T00:00:00Z`, which
  parses to a negative value.
- Keep `duration_seconds` from the `window` token, and keep `used_percent` from
  `remaining_fraction` exactly as today.
- Do not widen the rule. A strictly positive reset whose derived period starts before
  the epoch (the existing `1970-01-01T00:01:40Z` case) is still bad vendor data. So are
  unparseable strings (`"tomorrow"`, `""`) and non-string values.
- Add an "Unstarted sentinel" bullet to the module doc comment's hardening-guard list.
  It should explain that an at-or-before-epoch `reset_time` means the vendor has no
  reset clock for that bucket.

Tests go in the inline `#[cfg(test)] mod tests` of `agy.rs`:

- Add a second fixture, for example `live_payload_1_5_17()`, mirroring the current
  vendor shape. The Gemini buckets keep real fractions and future RFC 3339 resets. The
  3p buckets have `remaining_fraction: 1`, no `description`,
  `"reset_time": "1970-01-01T00:00:00Z"`, and matching sentinel TSV rows. Assert:
  - `outcome == Ok`, `completeness == Complete`, and `diagnostic == None`
  - four windows
  - both 3p windows have `resets_at == None`, `period_start == None`,
    `used_percent == 0.0`, and durations of `604_800.0` / `18_000.0`
  - the Gemini windows are unaffected
- Add a Go-zero-time variant (`"0001-01-01T00:00:00Z"`) asserting `resets_at == None`.
- Keep `unstarted_bucket_rolling_reset_passes_through`. Older agy builds still emit that
  shape.

### 2. sase-core: isolate per-bucket failures

> [!decision] bucket_isolation = isolate
>
> - Validate each built window on its own inside `bucket_window`. `agy` is a child
>   module of `provider_usage`, so it can call the private
>   `super::validate_window(&request.provider, window, request.now)`. Treat a failure as
>   a bucket-level error.
> - In `buckets_observation`, an error from a bucket whose `id` is a readable, non-empty
>   string no longer returns the whole-observation error. Push that id onto a new
>   `malformed` list and continue. Bad `reset_time`, bad `remaining_fraction`, a missing
>   or empty `name`, and per-window validation failures are all isolated this way.
> - These still fail the whole observation, because a bucket without an id cannot be
>   attributed:
>   - a non-object bucket
>   - a bucket with no readable `id`
>   - group-level structural errors: a non-object group, or a missing group `name` or
>     `buckets`
> - Add `const MALFORMED_BUCKET_DIAGNOSTIC: &str = "agy_usage_bucket_malformed";`.
>   Report the dropped ids as `agy_usage_bucket_malformed:<id>,<id>` in the existing
>   `; `-joined notes, next to the disabled and unmatched notes. Mark the observation
>   `Partial` whenever the malformed or unmatched list is non-empty.
> - If no windows survive and any bucket was malformed, return the whole-observation
>   `agy_usage_malformed` error. An unreadable inventory must never look like an empty
>   one.
> - Keep the whole-observation validation fallback in `normalize_agy_usage` as the last
>   resort.
> - Rework the tests:
>   - `out_of_range_fractions_are_malformed` and `bad_bucket_members_are_malformed`
>     corrupt bucket `(0, 0)` of the four-bucket live payload. They now assert `Ok`,
>     `Partial`, three windows, and a diagnostic naming `gemini-weekly`.
>   - The `1970-01-01T00:01:40Z` case now drops only that bucket.
>   - Add single-bucket payload controls showing that a lone bad bucket still yields the
>     whole-observation `agy_usage_malformed` error.
>   - A bucket missing `id`, and the `[{"name": "x"}]` groups case, still fail the whole
>     observation.
>   - Add one test where the vendor's sentinel bucket coexists with a genuinely
>     malformed bucket. Both the sentinel fix and the isolation must hold in it.

> [!decision] bucket_isolation = strict
>
> Skip this section. Bucket failures stay all-or-nothing, the existing
> `bad_bucket_members_are_malformed` and `out_of_range_fractions_are_malformed`
> assertions stay unchanged, and only change 1 ships.

### 3. sase: collector-level regression test

No Python source changes are needed. `src/sase/llm_provider/usage/agy.py` already hands
the whole envelope to the binding.

- In `tests/llm_provider/fixtures/usage_probe/agy_usage_cli.py`, add a
  `SASE_FAKE_AGY_MODE` value such as `unstarted_epoch` that emits the agy 1.5.x shape
  from change 1.
- In `tests/llm_provider/test_agy_usage_probe.py`, add
  `test_agy_usage_probe_unstarted_epoch_reset_collects_windows`. It asserts:
  - `outcome == "ok"` and `completeness == "complete"`
  - four windows, with 3p `resets_at` and `period_start` both `None`
  - the expected durations and Gemini values
- Note the open flake bead `sase-1il`: `test_agy_usage_probe_collects_four_windows`
  fails under a loaded check lane and passes in isolation. Do not weaken assertions to
  get a green run.

## Landing

- This turn modifies both repos. The final declaration commits both, and the host
  commits the pinned `sase-core` sibling first. It then writes the pushed SHA into
  sase's `sase-core-revision.txt` (`repos.linked[].revision_pin`), so the new sase test
  runs against the fixed core. Do not hand-edit the pin.
- The change is not breaking. `resets_at` and `period_start` are already optional on the
  wire, and the binding name and request shape are unchanged. Use a Conventional Commit
  subject such as `fix(provider-usage): treat agy epoch reset sentinel as no reset`.
- Verify `sase-core` with `sase tool run check`. It takes about 5 minutes, so give it a
  10-minute or longer tool timeout.
- Verify `sase` through its guarded `just check` via `sase tool run`. Read the
  `lint_and_test` reference memory first.

## Live verification

1. Capture a live probe into a fresh temp directory:
   `AGY_CLI_DISABLE_AUTO_UPDATE=1 agy -p /usage --output-format json --mode plan --sandbox --print-timeout 15s --log-file <tmp>/agy-usage.log`.
   It is a zero-token command.
2. Replay that stdout through `provider_usage_normalize_agy_usage` using the workspace
   venv's interpreter, which builds `sase_core_rs` from the linked `sase-core` checkout.
   Confirm that `outcome == "ok"` and that four windows come back.
3. The globally installed `sase`, which the scheduler and the TUI use, keeps the old
   core until it is reinstalled. Global installs are human-only, so after landing,
   propose `just install` through `/sase_gate`.
4. After that reinstall, `sase usage refresh -p agy` followed by
   `sase usage list -p agy` should show `status=ok` and fresh windows. Collector health
   recovers on the first success.

## Out of scope

- **agy model catalog drift.** Live `agy models` now lists `gemini-4-carbon-*`,
  `gemini-pro-orchestrator*`, `claude-opus-5-5-*`, and `claude-sonnet-5-5-*`, but
  `src/sase/llm_provider/models.yml` does not. As a result, window applicability
  (`model_ids` scoping) reports "does not apply" for those models. This is a catalog
  problem, not a collection problem, and the open bead `sase-1l4` already tracks it.
- How stale windows are presented while a collector is failing. They currently show
  "100% left" with their age and an error status row. That is existing design and is
  left as is.
