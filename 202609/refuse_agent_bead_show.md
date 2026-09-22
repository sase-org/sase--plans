---
tier: tale
title: Refuse agent-run sase bead show so every agent bead consultation records a reason
goal:
  Agents consult beads only through sase bead read, so the Agents-tab Beads rows show
  why each bead was read instead of a reasonless viewed row.
size: medium
proposed_by: bbugyi200.athena.0p5
create_time: 2026-09-22 08:19:09
status: wip
---

# Plan: Make agents use `sase bead read` by refusing agent-run `sase bead show`

## Diagnosis

**Suspicion confirmed.** In the screenshot, `sase-13i.2 · viewed` has no `↳` reason line
because agent `0p4` ran `sase bead show sase-13i.2 2>&1 | head -120`. It found the bead
ID in a `SASE_BEAD=[sase-13i.2]` commit trailer while running `git show 45a7895b6`. That
call wrote a `bead_views.jsonl` row, which the Agents-tab `Beads:` block renders as the
weaker `viewed` verb. A `viewed` row never has a reason. The synthetic view touch also
has an empty title, so the `↳` line is omitted.

**The problem is wider than one agent.** `sase bead read` landed on 2026-09-21. After
the skills were redeployed (about 19:42Z that day), `bead_views.jsonl` recorded about
120 agent `show` views across 21 agents. These include research swarm members and
finals, land agents, phase agents, and ad hoc `0o*`/`0p*` agents across Claude, Gemini,
and Muse. Over the same window, agents made about 113 audited `bead:` reads, and nearly
all of them came from agents whose xprompt spelled out the exact `sase bead read ... -r`
command.

**Root cause.** The rule "agents must use `sase bead read`" exists only as
documentation. Nothing in the command path enforces it or even reports it:

1. Where a template names the command (the phase, land, and task xprompts in
   `src/sase/default_config.yml` and the `/sase_new_task` skill), agents comply. Every
   post-fix `show` I traced was **ad hoc**. The bead ID came from a user prompt ("see
   the sase-11y epic bead for context", or a research prompt that literally said "Use
   `sase bead show sase-14s`"), a commit trailer, a plan file, or a status-sweep loop
   like `sase bead show $b -f json | python3 ...`. In those cases agents fall back to
   the conventional verb (`show`, as in `bd show`, `git show`, and `gh ... view`).
2. The rule lives in reference memory (`sase_beads.md`, which agents read only when they
   are about to work with beads) and in `sase bead show --help`. An agent that thinks it
   knows the command reads neither.
3. `sase bead show` gives an agent no feedback. It prints the same output as `read`,
   exits 0, and quietly records a `viewed` row. So an agent never learns, even later in
   the same session. Phase agents `sase-15b.6`, `.7`, `.9`, and `.10` each ran `read`
   once as told and then `show` on the same bead.
4. Some agent-facing text still teaches `show`. The sase-core plan schema field text
   (`PHASE_DESCRIPTION_DESCRIPTION` in `crates/sase_core/src/plan/validate.rs`) tells
   every planning agent that "`sase bead show` already displays it". The Python
   `src/sase/main/plan_explain.py` prose was already updated to say `read`.

Instructions alone cannot fix this: they are probabilistic, and users' own prompts
sometimes name `show`. The only change that would have put a reason under `sase-13i.2`
is a deterministic guard at the command.

## Approach

When `sase bead show` runs with an agent identity, it refuses before printing anything
and names the exact `sase bead read` command to run instead. The agent re-runs with a
reason on its very next call, so every agent bead consultation reaches the audited log
with a `why`. The guard fires under exactly the condition that writes a `viewed` row
today:

- `sase.bead.attribution.acting_agent_name()` returns a name, **and**
- `SASE_BEAD_SKIP_VIEW_LOG` is not `"1"`.

Automation that runs inside agents is therefore unaffected, because it already sets
`SASE_BEAD_SKIP_VIEW_LOG=1`: `tools/sase_bead` (used by symvision and
`tools/check_feature_flags`) and `_resolve_bead_issue` in
`src/sase/workflows/commit/bead_hooks.py`. Interactive human use and the sase-telegram
`/bead` command run with no agent identity and are also unaffected. A census of every
`~/.sase/projects/*/bead_views.jsonl` row shows only real agents running `show` from
their own shells. Those rows are precisely the calls the guard will refuse, so no
automation breaks.

No feature flag is needed. The only callers affected are agents, which switch commands
on the spot from the error, and there is no old branch that any caller has to keep
reaching while it migrates.

## Changes

### 1. Guard `sase bead show` for agents (`src/sase/bead/cli_query.py`)

- At the very top of `handle_bead_show`, call a new helper, for example
  `_refuse_agent_bead_show(args, argv=None)`. The call must come before
  `_run_bead_view`, store routing, the pager, and any output. Do not put the guard in
  `_run_bead_view`, because `handle_bead_read` shares it.
- The helper returns without doing anything when `SASE_BEAD_SKIP_VIEW_LOG == "1"`
  (import the constant from `sase.bead.bead_views`) or when `acting_agent_name()` is
  falsy. Import it from the module so tests can monkeypatch
  `sase.bead.attribution.acting_agent_name`, as the existing tests do.
- Otherwise it prints to stderr and exits with code `2` (refused, nothing printed,
  nothing recorded, matching the `2` = refused convention in `docs/cli.md`). The message
  is actionable and ready to paste, for example:

  ```text
  Error: agents must read beads with `sase bead read`, which records why you read them:
    sase bead read sase-13i.2 -r "<why you need this bead>"
  `sase bead read` accepts the same IDs and options as `sase bead show`; `show` is the human command.
  ```

- Build the suggested command from the caller's own tokens after the `show` subcommand,
  so options such as `-f json`, `--no-links`, `-P bob-cli`, and `sase-tt..` survive.
  Take the tokens from an injectable `argv` that defaults to `sys.argv[1:]`: find the
  first `show` token that follows `bead`, and quote with `shlex.join`. If the tokens
  cannot be found (for example, a handler invoked in-process), fall back to
  `shlex.join(_show_ids(args))`. Append `-r "<why you need this bead>"`. Never mention
  `SASE_BEAD_SKIP_VIEW_LOG` in the message, because it would teach agents how to bypass
  the guard.

### 2. Stop writing the view log; keep reading legacy rows

- Delete `record_bead_show_views` from `src/sase/bead/bead_views.py`, and remove its
  `__all__` entry, its import in `cli_query.py`, and the call at the end of
  `_run_bead_view`. After step 1 it could only ever write zero rows.
- Keep `BEAD_VIEWS_FILENAME`, `BeadViewEvent`, `bead_views_log_path`,
  `read_bead_view_events`, `views_to_touches`, and `view_touches_for_agent`, plus their
  consumers in `src/sase/ace/tui/bead_touches.py` and `src/sase/bead/cli_touched.py`.
  Existing `viewed` rows from 2026-09-21/22 still render. Remove any imports or helpers
  (for example `uuid4`, `_event_timestamp`, `fcntl.LOCK_EX` usage, `os`) that only the
  writer used and that lint or symvision flags.
- Keep `SASE_BEAD_SKIP_VIEW_LOG` under its current name, because renaming would churn
  `tools/sase_bead`, the commit hook, and tests for no behavior gain. Rewrite its
  docstring to say what it now means: it marks automation that runs inside an agent,
  whose `show` probes neither count as agent consultations nor trip the agent guard.
- Update the module docstrings to match the new model: agents are refused at `show`, and
  `bead_views.jsonl` is a legacy, no-longer-written log that is still read for display.
  The docstrings to update are in `bead_views.py`, `bead_reads.py`, `cli_touched.py`,
  and `ace/tui/bead_touches.py`. Also update the comment in `tools/sase_bead`.

### 3. Help text (`src/sase/main/parser_bead_queries.py`)

- In the `sase bead show` description, replace the last sentence with one saying that
  this is the human command, and that inside a SASE agent run it refuses and names the
  matching `sase bead read ... -r "<why>"` command. Keep the `sase bead read`
  description consistent.
- If help output is snapshotted, refresh it (`tests/completion/snapshots/cli_spec.json`
  or the equivalent) through the repo's normal snapshot-update path.

### 4. Docs (`docs/beads.md`)

- In the `sase bead show` section, replace the paragraph that begins "Run inside a SASE
  agent, `show` also appends one machine-local row ..." with the refusal behavior: agent
  identity plus no `SASE_BEAD_SKIP_VIEW_LOG=1` means exit `2`, no output, no rows, and a
  stderr message naming the `read` command. Note that `viewed` rows written before this
  change still appear in `sase bead touched` and the Agents-tab `Beads:` rows.
- In the `sase bead read` section, drop or reword the "`read` writes no `viewed` row, so
  it never double-counts as `read · viewed`" sentence.
- Grep `docs/` for other statements that agent `show` records `viewed` rows (for
  example, the `sase bead touched` section) and align them.

### 5. Stop sase-core teaching `show` to planners

- Open the linked repo with `/sase_repo` (`sase repo open sase-core -r "<why>"`) and, in
  `crates/sase_core/src/plan/validate.rs`, change `PHASE_DESCRIPTION_DESCRIPTION`'s
  "`sase bead show` already displays it" to "`sase bead read` already displays it". This
  matches the Python prose in `src/sase/main/plan_explain.py`. The existing Rust test
  compares against the constant, so it needs no edit. Grep sase-core for any other
  agent-facing `sase bead show` guidance and fix it the same way. Run that repo's own
  checks, and commit the change there as part of this turn's declaration.

### 6. Tests

- New tests (for example in `tests/test_bead/test_bead_read.py` or a focused
  `tests/test_bead/test_bead_show_agent_guard.py`):
  - With an agent identity (monkeypatch `sase.bead.attribution.acting_agent_name`),
    calling `handle_bead_show` raises `SystemExit(2)`. Stdout is empty, stderr contains
    `sase bead read <id>` and `-r`, and neither `bead_views.jsonl` nor
    `artifact_reads.jsonl` gets a row.
  - The suggestion preserves options when given an injected argv such as
    `["bead", "show", "sase-64", "-f", "json", "--no-links"]`, and falls back to the IDs
    alone when no `show` token is present.
  - With an agent identity **and** `SASE_BEAD_SKIP_VIEW_LOG=1`, `show` prints the bead
    normally and exits 0.
  - With no agent identity, `show` prints normally and exits 0.
  - `sase bead read` with an agent identity still works and is never refused.
- Update existing tests:
  - `test_read_writes_no_viewed_row_but_show_does`
    (`tests/test_bead/test_bead_read.py`): `show` is now refused, and no view row exists
    after either command.
  - `test_view_log_opt_out_writes_nothing`: rewrite it as the opt-out-bypasses-guard
    test above, or delete it.
  - `test_sase_bead_wrapper_exports_view_log_opt_out`: keep it.
  - `tests/test_bead/test_bead_views.py`: delete the `record_bead_show_views` tests
    (`test_record_writes_nothing_without_agent_identity`,
    `test_record_appends_one_row_per_bead`, and
    `test_record_never_raises_on_write_failure`). Keep the reader, fold, merge, and
    loader tests. Where they need fixture rows, have them write the JSONL directly.

## Non-goals

- **No memory changes.** `sase_beads.md` already calls `read` the agent command and
  `show` the human one, and that stays accurate. No always-loaded core-memory rule is
  added either: the guard teaches each agent on its first attempt at the cost of one
  tool call, which is cheaper than paying tokens on every turn for every agent.
- The legacy `viewed` reader, and its TUI and `sase bead touched` rendering, stay.
  Retiring them is a separate cleanup after the old rows have aged out.
- Human-facing hints such as `cli_work_from_plan_render.py`'s "or run: sase bead show
  <epic>" and the sase-telegram `/bead` command keep using `show`.

## Verification

- Run `just check` in this repo (read `sase/memory/lint_and_test.md` first) and the
  equivalent check in sase-core.
- Manual smoke test: the implementing agent runs `sase bead show <some-bead>` from its
  own shell, which already has an agent identity. It must print only the stderr guidance
  and exit 2. Then run `sase bead read <some-bead> -r "Smoke-test the agent show guard"`
  and confirm that it prints and records the reason.
- `env -u SASE_AGENT_NAME -u SASE_ARTIFACTS_DIR sase bead show <some-bead>` prints
  normally.
