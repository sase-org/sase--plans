---
tier: tale
title:
  Give the builtin `bug` task type a `+1` bar of 1 and clear the notifications it
  retires
goal:
  "The builtin `bug` task-type spec declares `triage.min_plus_ones: 1`, every committed
  artifact generated from that spec and every doc sentence that asserts today's `0` is
  updated to match, and each currently-pending `TaskTriage` notification for a `bug`
  bead with zero `+1` reports is dismissed without any gate being re-raised."
size: small
proposed_by: bbugyi200.athena.0fm
---

- **AGENTS:**
  - [bbugyi200.athena.0fm](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0fm.md)
- **COMMITS:**
  - [70e2d52](https://github.com/sase-org/sase/commit/70e2d5250765cbae1fdb892ee6bc00d7aed20199)
    — fix(task-types): require one plus-one for bug triage

# Plan

The builtin `bug` task type ships no `triage` block, so `TaskTypeRecord.min_plus_ones`
falls through to its spec default of `0` and every ready `bug` bead earns a `TaskTriage`
gate on the next five-minute `bead_task_triage` tick, uncorroborated. This plan raises
that bar to `1`, so a `bug` bead needs one independent `+1` before it interrupts anyone,
and then retires the notifications the old bar already produced.

## What the investigation established

Measured at `master` `f24aed1df`. Every claim below was read out of the tree or the live
host state, not assumed.

1. **Where the bar lives.** `_bug_spec()` in `src/sase/task_types/_builtin.py:46-92`
   returns no `"triage"` key. `TaskTypeRecord.min_plus_ones`
   (`src/sase/task_types/_models.py:60-68`) reads `spec["triage"]["min_plus_ones"]` and
   returns `0` when the block is absent. Adding `"triage": {"min_plus_ones": 1}` to the
   spec is the whole behavioral change — the shape is already proven by `_ci_spec()`
   (`{"min_plus_ones": 0}`) and `_flake_spec()` (`{"min_plus_ones": 3}`).

2. **Nothing else needs threading.** Every consumer already resolves the bar through the
   one predicate module: `effective_min_plus_ones` in
   `src/sase/bead/task_triage_policy.py:29-43`, called from
   `src/sase/scripts/sase_chop_bead_task_triage.py:387` and
   `src/sase/scripts/sase_chop_bead_stale_cleanup.py:284`. `sase bead task-type show`
   already prints the value (`src/sase/task_types/cli_show.py:85`, via
   `src/sase/task_types/detail.py:97`). No Rust wire change is needed: the spec field is
   already validated and digested by `task_type_spec_digest`
   (`src/sase/task_types/_validation.py:56-60`).

3. **Two committed artifacts are generated from the spec and will go stale.**
   - `sase/memory/task_types/bug.md` carries both
     `## Triage / - min_plus_ones: `0``(line 76) and a`Digest:` line (line 17) that is a
     hash of the spec, so both change.
   - `sase/task_types.json` carries the same `"min_plus_ones": 0` (line 46) and the
     matching digest. Both are written by `sase memory init`
     (`src/sase/main/init_memory/root_rendering_task_types.py:152-154, 319-325`). The
     flat descriptor `sase/memory/task_types.md` renders only labels and summaries, so
     it does **not** change; neither does `AGENTS.md`.

4. **One test pins today's value.** `tests/task_types/test_builtin.py:89-95`
   (`test_flake_requires_corroboration_and_ci_does_not`) asserts
   `"triage" not in specs["bug"]`. It must be updated, not deleted.

5. **Four doc passages assert the current default in prose.** `docs/beads.md:211-223`,
   `docs/beads.md:412-414`, `docs/axe.md:259-262`, and the
   `bead.task_triage.min_plus_ones` table row at `docs/configuration.md:3577`. Three of
   them say some variant of "`0` for most builtins"; `docs/beads.md` uses a `bug` bead
   as its worked example of a type that "needs none". `docs/configuration.md` is
   hand-written here, not generated, and the schema description at
   `src/sase/config/sase.schema.json:707` describes the _spec_ default (still `0`)
   rather than any builtin's declared value, so it stays as is.
   `src/sase/default_config.yml` documents only the global fallback (`1`), which is
   unchanged.

6. **The live notification inventory.** `sase notify list -j -l 500` returns 58 pending
   (non-dismissed) `TaskTriage` notifications; joining each one's
   `action_data.request_id` to
   `~/.sase/interaction_requests/task_triage/<request_id>/request.json` shows **29**
   whose `payload.task_type` is `bug`. Of those, 27 carry `payload.plus_one_count == 0`
   and two do not — `bob-cli-1g` and `sase-u9`, both at `1`. Those payload counts are
   snapshots taken when each gate was created, so the implementer must re-derive the
   count from live bead state, not from the payload (step 5 below).

7. **HAZARD — cancel the gates and you will re-raise them.** `sase gate cancel` /
   `cancel_gate` would settle the notification, but the `bead_task_triage` chop runs
   from the _installed_ `~/.local/bin/sase` (0.16.0), which still resolves `bug` at `0`.
   On the next tick that chop reads the gate as non-pending, deletes its state entry
   (`sase_chop_bead_task_triage.py:485-490`), finds the bead still gateable, and creates
   a fresh generation-`N+1` gate with a **new** notification
   (`sase_chop_bead_task_triage.py:492-528`). Dismissing the notification instead leaves
   the gate pending, so the chop takes the unchanged-fingerprint `skipped` branch
   (`sase_chop_bead_task_triage.py:260-270`) and stays quiet. The bar change does not
   perturb that fingerprint either: `presentation_fingerprint`
   (`src/sase/scripts/_bead_task_triage_gates.py:49-78`) hashes the frozen
   glyph/name/accent/facts display block, which `triage` is not part of. Once the new
   release is installed, the chop cancels these gates itself with reason
   `task_bead_below_plus_one_threshold` and the already-dismissed notifications settle
   cleanly.

8. **Second-order consequence, deliberately accepted.** `bead_stale_cleanup` uses the
   same effective bar (`sase_chop_bead_stale_cleanup.py:284`), so once the new release
   is installed those ~27 uncorroborated `bug` beads become stale-eligible after
   `bead.task_triage.stale_after_days` (7). With `stale_cleanup_min_beads` at `10` they
   will be offered in a `BeadStaleCleanup` gate proposing closure. That gate is a
   proposal the project owner answers — nothing closes automatically — but it is the
   expected follow-on and should not be treated as a regression.

## Work to do

### 1. Declare the bar on the builtin spec

`src/sase/task_types/_builtin.py` — add `"triage": {"min_plus_ones": 1},` to the mapping
returned by `_bug_spec()`, as the last key after `"body_template"`, matching the
placement `_ci_spec()` and `_flake_spec()` already use.

Extend `_bug_spec()`'s one-line docstring, or the `when_to_use` prose if it reads better
there, so the reason is recorded: a `bug` bead is a defect one agent noticed while doing
something else, and one independent reproduction is what separates it from a misreading.
Do not change any other field of the spec — glyph, accent color, fields, and body
template must stay byte-identical so only the triage bar and the digest move.

### 2. Update the test that pins today's value

`tests/task_types/test_builtin.py` — `test_flake_requires_corroboration_and_ci_does_not`
currently asserts `"triage" not in specs["bug"]`. Replace that with
`assert specs["bug"]["triage"]["min_plus_ones"] == 1`, keep the `feature` and `memory`
assertions as they are, and rename the test to describe the new grouping (for example
`test_declared_corroboration_bars_match_the_shipped_defaults`). Keep it a single test
over `_by_slug()`; do not split it.

### 3. Regenerate the committed artifacts

```bash
just install                       # numbered workspaces go stale; the Rust extension
                                   # is not built in this one right now
.venv/bin/sase memory init --no-commit
```

`--no-commit` is required: `sase memory init` otherwise runs a git commit/push sequence,
and an agent never creates commits.

Then confirm the diff is exactly what it should be:

```bash
git diff --stat
git diff sase/task_types.json sase/memory/task_types/bug.md
.venv/bin/sase memory init --check   # must report no drift
```

Expect two files to move, and within them only: `bug`'s `min_plus_ones` `0` → `1` and
`bug`'s digest. If `sase/task_types.json` loses its `flag` or `github` entries, a
required plugin is missing from the workspace venv — stop, fix the install, and
regenerate; do not commit a truncated catalog. If `AGENTS.md`, the provider shims,
`sase/memory/README.md`, or any other memory note also moves, that is unrelated
pre-existing drift: report it and leave it out of this change rather than folding it in.

### 4. Correct the four doc passages

Each of these states the old fact directly. Fix the fact; do not restructure the
surrounding sections.

- `docs/beads.md:211-215` — the first bullet under `<a id="per-type-triage-bar"></a>`.
  The spec default is still `0`; what changes is which builtins declare their own. Say
  that `flake` ships `3`, `bug` ships `1`, and `ci`, `feature`, and `memory` ship `0`,
  and give each declared bar its one-clause reason (`flake` because a single failure is
  the case most often misread as a real defect; `bug` because a defect found while doing
  something else deserves one independent reproduction before it interrupts anyone).
- `docs/beads.md:220-223` — the "Those two defaults differ on purpose" paragraph uses a
  `bug` bead as its example of a typed bead that "needs none". That example is now
  wrong; reach for `ci` or `memory` instead, and keep the point intact: the spec default
  of `0` is not the same thing as "the default bar is zero".
- `docs/beads.md:412-414` and `docs/axe.md:259-262` — both parenthesize the bar as "`0`
  for most builtins". Replace with the accurate enumeration (`0` for `ci`, `feature`,
  and `memory`; `1` for `bug`; `3` for `flake`).
- `docs/configuration.md:3577` — the `bead.task_triage.min_plus_ones` table row says
  "`flake` ships as `3`, and every other builtin ships as `0`". Correct it to name `bug`
  as `1`. Keep the row on one line; the table is not wrapped.

Run `just fmt-md-check` after editing — the docs are prettier-formatted at 88 columns.

### 5. Dismiss the retired notifications

Do this last, after the code change is in the tree, and do it against **live** bead
state.

Build the candidate set: every notification from `sase notify list -j -l 500` with
`action == "TaskTriage"` and `dismissed == false`, whose gate request at
`~/.sase/interaction_requests/task_triage/<action_data.request_id>/request.json` has
`payload.task_type == "bug"`. For each candidate, read the current count from the bead
itself rather than the gate payload:

```bash
sase bead show <payload.bead_id> -f json -N   # -> .plus_one_count
```

`sase bead show` resolves a full ID against the enabled project named by its prefix, so
`bob-cli-*` and `sase-*` ids both work without `--project`. Use the host `sase` here
(the workspace venv is not a second source of truth for live host state).

Dismiss exactly those whose live `plus_one_count` is `0`:

```bash
sase notify apply-state <notification_id> dismiss
```

Print the id, bead id, and live count for every candidate — including the ones you skip
— so the decision is auditable, and report the final counts (expected: 29 candidates, 2
skipped for `bob-cli-1g` and `sase-u9`, 27 dismissed — but let the live counts decide,
not these numbers). A bead that has gained a `+1` since this plan was written must be
skipped; a bead that has lost one must be dismissed.

Do **not** use `sase gate cancel`, `sase gate answer`, or any bulk gate sweep here — see
investigation item 7. Dismissing the notification is the entire operation; the pending
gate is left alone on purpose.

## Verification

- `just install` first.
- `.venv/bin/python -m pytest tests/task_types/ tests/test_bead/test_task_triage_policy.py tests/test_axe_chop_bead_task_triage_plus_ones.py tests/test_axe_chop_bead_stale_cleanup.py tests/main/test_init_memory_task_types_note.py tests/main/test_init_memory_task_types_snapshot.py -q`
- `.venv/bin/sase bead task-type show bug` — `TRIAGE / min_plus_ones` must print `1`.
- `.venv/bin/sase memory init --check` — no drift.
- `just check-full` through `/sase_monitor` with the `TESTING` / `TESTED` status pair,
  never inline. `check-full` rather than `check` because this change regenerates
  committed artifacts that the scoped import-graph closure does not model.

## Non-goals

- Do not change the global `bead.task_triage.min_plus_ones` fallback. It ships as `1` in
  `src/sase/default_config.yml:1303` and governs untyped and unregistered beads only.
- Do not change any other builtin's bar, and do not touch the `flag` type's
  `triage.min_plus_ones: 0` in `sase/sase.yml:138-139` (that is a project-config type,
  and flag beads are exempt from the bar anyway).
- Do not edit `sase/memory/task_types/bug.md` or `sase/task_types.json` by hand. They
  are generated; regenerate them.
- Do not hand-edit `sase/memory/task_types.md`, `AGENTS.md`, or the provider instruction
  shims. `sase memory init` owns them and should leave them untouched here.
- Do not cancel, answer, or re-fingerprint any `TaskTriage` gate.
- Do not `+1`, close, snooze, or otherwise mutate any of the 29 `bug` beads.
- `CHANGELOG.md` is release-please generated; put the rationale in the commit message.
