---
tier: tale
title:
  Restore the configured timezone and close the remaining hardcoded-UTC display sites
goal:
  SASE surfaces show wall-clock times in the owner's configured timezone again, the
  config layer that silently forced UTC is removed and made detectable by `sase doctor`,
  and every remaining user-facing timestamp that hardcodes UTC renders through
  `sase.core.time`.
size: medium
proposed_by: bbugyi200.athena.0bf
---

- **AGENTS:**
  - [bbugyi200.athena.0bf](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0bf.md)
- **COMMITS:**
  - [e21d050](https://github.com/sase-org/sase-core/commit/e21d05004b3b342eaddf2ab2ec68bb276de22129)
    — feat(completion): thread configured UTC offset into commit-age display

# Plan: Restore the configured timezone and close the remaining hardcoded-UTC display sites

## Outcome

The reported symptom — the ACE agent-shell `Timestamps:` block showing `11:15:31` while
the host clock reads `07:15:31 EDT` — is gone, the failure mode that caused it is
reported by `sase doctor` instead of being invisible, the handful of surfaces that still
format a human-readable wall clock in UTC are converted to the configured timezone, and
`sase/memory/gotchas.md` carries a bullet that keeps the convention in front of future
agents.

## Root cause (verified, do not re-derive)

The display code was never wrong for the reported sighting. **The configured timezone
itself was UTC**, and every surface honored it faithfully.

Evidence gathered on the reporting host (`athena`, system zone `America/New_York`):

- `~/.config/sase/sase.yml` sets `timezone: "America/New_York"`.
- `~/.config/sase/sase_extra.yml` exists and contains exactly `timezone: UTC\n`.
  Overlays matching `~/.config/sase/sase_*.yml` merge on top of the base file, so this
  14-byte overlay wins.
- `sase config layers` lists it as `overlay:sase_extra.yml [loaded] … keys: timezone`.
- `load_merged_config()["timezone"] == "UTC"`, so `sase.core.time.get_timezone()`
  returns UTC and `local_now()` is four hours ahead of `datetime.now()`.
- The file is 14 bytes, sha256
  `be81d3abf8e7aaf610ddf70685a51e8ea0d5223eba199204957cc7504a09462e`, birth time equals
  its mtime (`2026-08-21 16:09:15 -0400`, i.e. written once and never edited), and
  `chezmoi source-path` reports it is `not in source state`.

Those bytes are the fixture payload written by
`tests/feature_flags/test_cli_journeys.py::_seed_portable_config`, which at the time
resolved `CONFIG_DIR` at module-import scope and therefore escaped its pytest sandbox
onto the real host config directory. That is the already-diagnosed leak recorded in
`sase/repos/plans/202608/prevent_host_config_test_leak.md` (still `status: wip`).

**The code half of that plan has landed.** `_seed_portable_config` now routes through
`_resolve_config_dir_inside_pytest_sandbox`,
`tests/feature_flags/test_host_config_safety.py` rejects module-scope `CONFIG_DIR`
snapshots, and the autouse `_forbid_real_host_sase_config_layer_writes` fixture in
`tests/_conftest_environment.py` fails closed on any write to a real host config layer.
Do not re-implement any of that. Only the host-recovery half (step 5 of that plan) was
never performed, so the poisoned overlay is still live.

### Two consequences worth stating plainly

1. Removing the overlay fixes display immediately, but **artifact directory names minted
   during the poisoned window keep their UTC stamp forever** — e.g.
   `…/artifacts/ace-run/202608/23/20260823111531/` for an agent that actually started at
   `07:15:31` local. Those 14-digit names are identity, not display, and
   `_running_loaders.py` reads them back as naive configured-tz wall time. Rows for
   agents launched between 2026-08-21 16:09 and the fix will therefore still show a
   START four hours off. That is expected and must not be "fixed" by renaming artifact
   directories.
2. Nothing warned the user. A configured timezone that disagrees with the host system
   timezone by hours is legitimate on some machines, but on a workstation it is almost
   always a mistake, and today no diagnostic surfaces it. Step 2 closes that gap.

## Triage rule for the audit

UTC is **correct** for: persisted instants, wire payloads, sort keys, dedupe keys,
duration arithmetic, and `Z`-suffixed identifiers/filenames. Do not touch those; the
established repo pattern is _store UTC, display configured-tz_, and `sase.core.time`'s
module docstring already states it.

UTC is **wrong** whenever the resulting string is a wall clock, a calendar date, or a
date-derived filename that a human reads. Those must resolve through
`sase.core.time.format_local` / `parse_local` (for stored values) or `local_now` /
`to_local` (for the naive-local model).

Most of the codebase is already correct — `bead_time_presentation.py`,
`artifact_cli/link_ops.py`, `logs_pane_toasts.py`, `memory/review_tui/_render.py`,
`plans_rendering.py`, and `_meta_enrichment_common.parse_utc_to_local` were all checked
and are fine. Prior plans (`timezone_runtime_consistency` and friends) did most of this
work. Expect a short list, not a sweep.

## Implementation

### 1. Restore the configured timezone on the host

This is the fix for the reported symptom; do it first so the rest can be verified
against a correct clock.

- Re-verify before touching anything: `~/.config/sase/sase_extra.yml` must still be 14
  bytes with sha256 `be81d3ab…09462e` and contain only `timezone: UTC`. If the bytes,
  size, or hash differ, **stop and report** — the user may have since made the file
  their own; do not guess.
- Move it (never delete) to a timestamped recovery path outside `~/.config/sase`, for
  example `~/.sase/recovered/sase_extra.yml.<YYYYmmddHHMMSS>`, so restoring it is one
  `mv`.
- Do not modify the chezmoi source; the file has no chezmoi source and `sase.yml` is
  already clean.
- Audit the rest of `~/.config/sase` for other overlays with the same provenance
  (unmanaged by chezmoi, birth time inside the 2026-08-21 test window, contents matching
  a test fixture). Recover only files with equally conclusive evidence; leave anything
  ambiguous in place and report it.
- Confirm: `sase config layers` no longer lists `overlay:sase_extra.yml`, and
  `load_merged_config()["timezone"]` is `America/New_York`.

### 2. Make the failure mode detectable: `config.timezone` doctor check

Add a check so this never again requires a screenshot and a code audit to diagnose.

- New `src/sase/doctor/checks_config_timezone.py` exposing `check_config_timezone()`,
  following the `DiagnosticCheck` shape used by
  `src/sase/doctor/checks_config_layers.py`.
- Register it in `src/sase/doctor/checks_config.py::config_check_specs` as
  `CheckSpec(id="config.timezone", group="config", title="Configured timezone", …)`.
- The check reports the resolved zone, **which config layer set it** (reuse
  `load_config_layers()` and pick the last loaded layer whose `keys` contain
  `timezone`), and the host system zone.
- Status `OK` when the configured zone matches the system zone or no `timezone` key is
  set. Status `WARNING` — never `ERROR` — when they diverge, since remote and
  intentionally-UTC hosts are legitimate. The `next_steps` must name the offending layer
  path so the user can act on it in one step.
- Include the resolved zone, source layer path, and system zone in the check's `data` so
  `sase doctor -j` carries it.

### 3. Fix the remaining hardcoded-UTC display sites

Each of these was individually confirmed to reach a human-readable surface.

- `src/sase/agents/cli_sync.py` (`_timestamp`, ~line 170): the `sase agents sync` table
  renders `datetime.fromtimestamp(value, tz=UTC).strftime("%Y-%m-%d %H:%M UTC")`.
  Replace with `format_local(value, "%Y-%m-%d %H:%M %Z", default="-")` so the zone label
  is real rather than a hardcoded literal.
- `src/sase/llm_provider/_registry_routing.py` (`format_provider_disable_expiry`, ~line
  130): renders `"%Y-%m-%d %H:%M:%S UTC"`. This string reaches the user through
  `src/sase/agent/launch_guard.py` when a launch is blocked on a disabled provider. Its
  TUI twin at `src/sase/ace/tui/modals/models_panel_provider_rendering.py:143` already
  uses `get_timezone()`; make the non-TUI path agree (keep its own format string — the
  TUI uses a compact `%b %-d %-I:%M%p` and this one is a full instant).
- `src/sase/xprompt/_catalog_render.py`: `generated_at=datetime.now(UTC)` (~line 113) is
  formatted as `%Y-%m-%d` both into the rendered catalog page (~line 169) and into the
  `xprompts_catalog_<date>.pdf` filename (~line 59). Between 20:00 and midnight Eastern
  the catalog is stamped with tomorrow's date. Mint it with `local_now()`.

### 4. Fix the one Rust-core display site

`crates/sase_core/src/editor/completion.rs::commit_age_label` formats ages under a week
as relative (`3h`, `2d`, tz-independent and correct) but falls back to
`DateTime::<Utc>::from_timestamp(timestamp, 0).date_naive().format("%Y-%m-%d")` for
anything older. That date is the `age` column of `@commit` completion rows, so it can be
a day off.

Open the core repo with `/sase_repo` (`sase repo open sase-core -r "<reason>"`) and work
only in the path it prints.

- Add an additive, defaulted `utc_offset_seconds: Option<i32>` to
  `ArtifactRefContextWire` in `crates/sase_core/src/artifact_ref/wire.rs` (every
  existing field is `#[serde(default)]`, so this stays backward compatible), thread it
  to `commit_age_label`, and apply it before `date_naive()`. Default `None`/`0` must
  preserve today's behavior so existing Rust tests and callers are unaffected.
- Populate it on the Python side wherever the artifact-ref context is built, from
  `get_timezone().utcoffset(local_now())`.
- Update the Rust unit test `commit_age_labels_match_prompt_bar_thresholds` with a case
  that pins the offset-applied date.

If threading the offset turns out to ripple beyond the completion path, **stop, leave
the Rust side untouched, and file a `bug` task bead via `/sase_new_task`** describing
the site and why it was deferred. Do not let this step expand the tale.

### 5. Confirm the sweep is complete

Run these and triage every hit against the rule above; the expected result is that
nothing new appears beyond steps 3 and 4.

```bash
grep -rn -E "(tz=UTC|, *UTC|astimezone\(UTC\)|now\(UTC\)|now\(tz=UTC\))\)?\.strftime\(" --include="*.py" src/
grep -rn "datetime\.now()\|utcnow\|\.astimezone()\|date\.today()" --include="*.py" src/
grep -rn "fromtimestamp(" --include="*.py" src/ | grep -v "get_timezone\|_timezone()\|tz=tz\|, timezone)"
grep -rn '%Y\|%H:%M\|%b ' --include="*.rs" <sase-core checkout>/crates/
```

Record in the final report any site deliberately left in UTC and the one-line reason, so
the next audit does not re-litigate it.

### 6. Regression coverage

- Extend `tests/test_timezone_runtime_consistency.py` (Part C, "display conversions")
  with one test per site fixed in step 3, each using the existing `tz_divergence`
  fixture (configured `America/New_York` vs system `UTC`) so a regression to the system
  or hardcoded clock fails.
- Add coverage for `check_config_timezone`: `OK` when zones agree, `WARNING` naming the
  overlay path when they diverge.
- Do not add tests that write to `~/.config/sase`; the autouse
  `_forbid_real_host_sase_config_layer_writes` guard will fail them, which is correct.

### 7. Memory note

Append exactly one bullet to `sase/memory/gotchas.md`, then run `sase memory init` to
regenerate `AGENTS.md` and the provider shims. The user granted permission for this edit
in the prompt that produced this plan; no further approval is needed for the edit or the
regeneration.

```markdown
**Store Timestamps in UTC, Display Them in the Configured Timezone**  
Persist, compare, and sort instants in UTC or epoch; render every wall clock a human
reads through `sase.core.time` (`format_local`/`parse_local` for stored values,
`local_now`/`to_local` for the naive-local model). Never `strftime` a UTC-aware value or
a bare system clock onto a user-facing surface. When times still look wrong, suspect the
config before the code: `sase doctor -C config.timezone` names the resolved zone and the
layer that set it.
```

Keep it to that bullet. Do not restate the storage/display split anywhere else in the
memory tree; `sase/core/time.py`'s module docstring already owns the long form.

## Verification

1. `sase config layers` shows no `sase_extra.yml` overlay, and a fresh Python process
   resolves `get_timezone()` to `America/New_York` with `local_now()` matching
   `datetime.now()`.
2. `sase doctor -C config.timezone` reports `OK` on the restored host. Temporarily
   pointing the check at a divergent config in a test proves the `WARNING` path and its
   `next_steps` text.
3. `sase agents sync` and a blocked-provider launch message both render a local instant
   with a real zone abbreviation (`EDT`), not `UTC`.
4. Open `sase ace`, select a _newly launched_ agent, and confirm the `Timestamps:` block
   START/RUN/PLAN rows match the host clock. Rows for agents launched during the
   poisoned window will still read UTC — see consequence 1 above; confirm this rather
   than chasing it.
5. `just install` (the workspace may be stale), then `just check`. If the scoped test
   lane escalates or the diff touches the broadening set, run `just check-full` through
   `/sase_monitor` with `--start-status TESTING --stop-status TESTED`.
6. If step 4's Rust change landed, run the core crate's tests in the checkout that
   `sase repo open` printed, and rebuild the binding so the Python side links the new
   wire field.
7. Report the before/after fingerprints of the recovered overlay file and the final
   `sase config layers` output.

## Safety and rollback

- The only host-state mutation is moving one conclusively-identified 14-byte fixture out
  of `~/.config/sase`. It is moved, never deleted; restoring it is a single `mv`.
- Do not touch the chezmoi source repository. `~/.config/sase/sase.yml` is already clean
  and is the recovery source, not the cause.
- Do not weaken or remove the landed host-config write guards in
  `tests/_conftest_environment.py` or `tests/feature_flags/test_host_config_safety.py`.
- Do not rename or rewrite existing agent artifact directories to "correct" their UTC
  stamps; those names are identity and are referenced by the artifact index, RUNNING
  claims, and prompt lookups.
- Leave `sase/repos/plans/202608/prevent_host_config_test_leak.md` alone; note in the
  final report that its host-recovery step is now satisfied so the owner can retire it.
