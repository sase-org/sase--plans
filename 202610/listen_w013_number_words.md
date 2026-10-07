---
tier: tale
title: Fix sase-listen W013 false positives on spelled-out source numbers
goal: sase-listen render of the-dot-and-the-swarm (and any article that spells multi-digit
  numbers in words) no longer fails the writer's W013 number-fidelity lint; already-saved
  scripts that now lint clean are reused, and a writer failure no longer marks the
  fetch stage failed.
size: medium
proposed_by: bbugyi200.athena.0xn
status: done
---

# Plan: Fix sase-listen W013 false positives on spelled-out source numbers

All work happens in the linked `sase-listen` repo. Open it with
`sase repo open sase-listen -r "<reason>"` and work only in the printed path. Read that
repo's `AGENTS.md` first.

## Problem

`bob ref create https://www.oneusefulthing.org/p/the-dot-and-the-swarm -i -L` runs
`sase-listen render <url> -e full`. The Write script stage fails after 3 Gemini attempts
(~4 minutes) with
`Generated article script still has required lint findings after 3 attempt(s).` (exit 6,
`ExitCode.SCRIPT_STRUCTURAL`). The progress board also shows `✗ Fetch article` with the
writer's error and the same 3m 58s duration, even though fetching succeeded.

## Root cause (verified)

1. **Guide and lint contradict each other.** The packaged writer guide
   (`src/sase_listen/data/guide.md`, "Keep every argument-carrying number exactly, in
   digits ... do not spell numbers out") tells the writer to turn number words into
   digits. The article says "When I sketched three teams ... I got thirteen." The writer
   correctly wrote "they got 13 agents".
2. **W013 is a literal substring match.** In `src/sase_listen/script/lint.py`
   (`lint_text`, the "Number fidelity against the source report" block), each script
   number with two or more digits, a decimal point, or a percent sign passes only if
   `norm in normalized_source`. `normalized_source` only removes commas and underscores.
   "13" does not appear in a source that spells it "thirteen", so the lint reports
   `W013 Number '13' not found in the source.` Single-digit numbers ("three" → "3") are
   skipped by the rule, so only two-or-more-digit words fail.
3. **W013 blocks the render.** `src/sase_listen/writer/author.py` treats
   `_FINDING_REPAIR_RULES = {"W011", "W013"}` as required. The repair prompt says "Keep
   every argument-carrying number exactly", and the guide still demands digits, so every
   attempt reproduces "13". After `writer.max_attempts` (3) the writer saves the script
   and raises. The `W013` row is the only finding in the saved record
   (`full_writer.json`: `required_findings` = one W013 at 56:63).

The diagnosis has been checked with the installed CLI. Running
`sase-listen lint ~/.local/share/sase-listen/sources/the-dot-and-the-swarm-249d66/full_narration.md --source <same dir>/source.md`
reports exactly that one W013. Running it against a copy of `source.md` with "thirteen"
changed to "13" reports `✓ Script is clean.` The saved script is otherwise faithful.

The same false-positive class covers any number word with two or more digits:
"twenty-five", "ten thousand" (this article escaped only because it also says "10,000"),
"a million", ordinals such as "thirteenth", and "fifty percent" vs a `50%` script
number.

Secondary defect (the misleading `✗ Fetch article` row): in
`src/sase_listen/pipeline.py` `_load_acquired`, the SOURCE stage is not marked done
until after the WRITE stage finishes. `events.on_stage_done(Stage.SOURCE.value, ...)`
comes after the write block. So both rows are active while the writer runs. When
`author_script` raises, `LiveProgress.__exit__` calls
`ProgressState.mark_active(STATUS_FAILED, ...)` (`src/sase_listen/cli/progress.py`),
which fails every active row. The `RenderEvents` contract in `src/sase_listen/events.py`
and the existing `tests/test_progress.py` sequence both expect SOURCE to finish before
WRITE starts.

## Changes

### 1. Make W013 accept spelled-out source numbers (primary fix)

- Add a small pure module, `src/sase_listen/script/numwords.py` (the script phase owns
  `script/`). It exposes one function, for example
  `source_number_forms(text: str) -> set[str]`, which scans English number phrases in
  the source and returns their canonical digit strings (no commas). Write it in-house.
  Do not add a dependency, and do not edit `pyproject.toml` or `uv.lock`. It must
  handle, case-insensitively:
  - cardinals zero–nineteen, tens twenty–ninety, hyphen or space compounds
    ("twenty-five", "twenty five"), and scale words hundred, thousand, million, billion,
    trillion, with an optional "and" ("one hundred and twelve" → `112`, "Ten thousand" →
    `10000`);
  - a leading "a" or "an" before a scale word ("a hundred" → `100`, "a million" →
    `1000000`);
  - ordinals ("thirteenth" → `13`, "twenty-first" → `21`, "hundredth" → `100`), because
    script ordinals like `13th` lint as `13`;
  - a digit followed by a scale word ("1 million" → `1000000`, "2.7 million" →
    `2700000`, "10 thousand" → `10000`), only when the product is an integer;
  - "<number> percent" or "<number> per cent", in words or digits ("fifty percent" →
    `50%`, "42 percent" → `42%`), because the guide's example form is `42%`.

  Do not map approximate idioms ("a dozen", "a couple", "a few", "half"). W013 has to
  stay strict for numbers the writer invents.

- In `lint_text`, compute the forms once per call when `source_text` is given. A
  candidate passes if `norm in normalized_source` (keep the current substring semantics,
  so nothing that passes today starts failing) **or** `norm in source_forms`. Leave the
  rule ID, severity, message, and hint as they are.
- Keep `mypy --strict`, ruff, and codespell clean. Number words in tests and source must
  not trip codespell.

### 2. Reuse a cached script whose recorded findings no longer apply

Today `_load_cached` in `src/sase_listen/writer/author.py` rejects any cache record with
non-empty `required_findings`. Once the lint fix lands, rerunning the user's command
would then spend another ~4 minutes of Gemini tokens to rewrite a script that already
lints clean. It also means the exhausted-attempts hint ("Edit the saved script ...")
does not work through a rerun of the same command, which is what `bob ref` tells the
user to do.

- Extract the required-findings computation from `author_script` into a shared helper:
  the `budget`/`repair_under_budget` logic plus `_repair_findings` over
  `lint_text(script, source_text=...)`.
- In the cache path (`load_cached_script` / `_load_cached`), when the record passes
  every existing match check (source sha, prompt version, edition, model, file present)
  but lists `required_findings`, re-lint the cached script against
  `article.markdown_path` with the shared helper. If nothing required remains, reuse the
  script. Best-effort, rewrite the writer record's `final_findings` /
  `required_findings` atomically with `_atomic_write` so the record stays honest, and
  ignore `OSError`. If findings remain, return `None` as today. Records with empty
  `required_findings` keep their current behavior, with no re-lint.
- Change the exhausted-attempts `SaseListenError` hint to name both recovery paths, for
  example: "Edit the saved script and rerun the same command, or run
  `sase-listen render <script_path>`."

### 3. Close the SOURCE stage before the WRITE stage starts

- In `pipeline._load_acquired`, for the `brief`/`full` branch, emit
  `events.on_stage_done(Stage.SOURCE.value, <summary>)` before
  `events.on_stage(Stage.WRITE.value)`. Do not emit it again after the write. The
  summary must be the same text as today (`host · N words[ · cached]` for URLs and
  `name · N words` for PDFs). Compute it from the acquired metadata, `source_url`, and
  `source_label`, for example by splitting the metadata-based part out of
  `summarize_source` so it doesn't need a parsed `LoadedSource`. Leave the `verbatim`
  branch and the non-URL `load_source` paths unchanged.
- Leave `ProgressState.mark_active` as it is. Once the stages are properly sequenced,
  only one row is active when the writer fails.

## Tests

- `tests/test_lint.py`: a script containing `13` lints clean against a source saying
  "thirteen". Also cover "twenty-five" → `25`, "Ten thousand" → `10000` and `10,000`,
  "one hundred and twelve" → `112`, "thirteenth" → `13`, "a million" → `1000000`, "2.7
  million" → `2700000`, "fifty percent" → `50%`. Negative cases: "thirteen" does not
  validate `14`, and "a dozen" does not validate `12`. The existing
  `test_number_fidelity` must still pass. Add focused unit tests for
  `numwords.source_number_forms` too.
- `tests/test_writer.py`:
  - An article source containing a spelled-out number, with a writer reply using its
    digits, succeeds in one attempt.
  - Edit-and-rerun flow: exhaust attempts with a reply containing an invented number
    (like the existing `test_exhausted_lint_repairs_save_script_and_fail`), remove the
    number from the saved `*_narration.md`, then call `author_script` with a writer that
    raises if called. The edited script is reused, and the writer record now has empty
    `required_findings`.
  - A cached record whose findings still apply is regenerated (the writer is called).
  - The new exhausted-attempts hint text.
- `tests/test_events.py`: for a URL `brief` load (pattern:
  `test_url_brief_write_steps_and_cached_rerun`), the recorded events show
  `on_stage_done("source", ...)` before `on_stage("write")`. Add a variant where the
  fake writer always produces a required finding, or raises. It asserts that SOURCE was
  already done when the failure propagated, so only the write row can end up failed. If
  practical, drive a `ProgressState` and assert only the `write` row has
  `STATUS_FAILED`.

## Docs

- `docs/cli.md` (`lint` section) and `docs/narration-scripts.md` (number fidelity
  sentence): say that a source number may appear as digits or spelled out in words, for
  example "thirteen" matches `13`, and "fifty percent" matches `50%`.
- `docs/web-articles.md` (writer caching paragraph): say that a cached script saved with
  lint findings is re-checked on the next run and reused once it lints clean, whether
  from rule fixes or hand edits.
- Do not hand-edit `CHANGELOG.md`; release-please manages it. Use a conventional commit
  subject such as `fix(lint): match spelled-out source numbers in W013 number fidelity`.

## Verification

1. `sase tool run check` in the sase-listen checkout (lint + mypy + codespell + tests)
   passes.
2. From the sase-listen checkout, run `uv run sase-listen lint` (not the globally
   installed tool, which points at the primary checkout) on
   `~/.local/share/sase-listen/sources/the-dot-and-the-swarm-249d66/full_narration.md`
   with `--source` set to the `source.md` in the same directory. It must report no W013
   finding.
3. From the sase-listen checkout, run
   `uv run sase-listen render https://www.oneusefulthing.org/p/the-dot-and-the-swarm -e full --dry-run`.
   The Write script row must report the cached script (`· cached`) with no Gemini call,
   and the source row must finish as done, not failed. Do not run a non-dry-run render
   or publish.
4. In the final summary, tell the user that the global `sase-listen` is an editable
   `uv tool` install of the primary sase-listen checkout. The fix is live once that
   checkout syncs, after which
   `bob ref create https://www.oneusefulthing.org/p/the-dot-and-the-swarm -i -L` can be
   rerun and should reuse the cached script.
