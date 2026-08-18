---
tier: tale
title:
  Make `@<path>` work on bead free-text CLI options and repair the four beads that
  silently stored the literal path
goal:
  "`sase bead create -d @file`, `sase bead update -d/-n @file`, and `sase bead note
  @file` read the file, a missing or unreadable path is a loud exit-1 error instead of a
  stored literal, and the four beads whose descriptions are currently the string
  `@/tmp/...` carry their real prose again."
size: medium
proposed_by: bbugyi200.athena.06d
create_time: 2026-08-18 13:07:15
status: wip
---

# Plan: `@<path>` for bead free-text values

## Problem

`sase bead show sase-pu` renders a one-line description:

```
DESCRIPTION
  @/tmp/sase-visual-flake-phase-context/description-clean.md
```

The stored `description` really is that 57-character literal. The 945-byte flake
diagnosis the authoring agent wrote into that file was never stored on the bead, and
`/tmp` is cleared on reboot.

This is not a one-off. Scanning the whole bead store (`sase/repos/beads/issues.jsonl`)
for a description, note, or title whose first non-space character is `@` returns exactly
four hits, all descriptions, all from 2026-08-18, all pointing into `/tmp`:

| Bead      | Status  | Created by                      | Stored description                                           |
| --------- | ------- | ------------------------------- | ------------------------------------------------------------ |
| `sase-pn` | closed  | `bbugyi200.athena.sase-p5.land` | `@/tmp/p5_ci_desc.md`                                        |
| `sase-po` | snoozed | `bbugyi200.athena.sase-p5.land` | `@/tmp/p5_resume_desc.md`                                    |
| `sase-pp` | closed  | `bbugyi200.athena.sase-p5.land` | `@/tmp/p5_flag_desc.md`                                      |
| `sase-pu` | ready   | `05z.f0.f0--1`                  | `@/tmp/sase-visual-flake-phase-context/description-clean.md` |

Two independent agents, ~20 minutes apart, made the same mistake. All four `/tmp` files
still exist as of this writing, so all four are recoverable **now**; the appendix embeds
their contents verbatim so the repair survives a reboot.

## Diagnosis

### Root cause: `@<path>` is a SASE-wide CLI convention that `-d` does not honor

`@<path>` means "read this value from a file" in at least four places already:

- `sase bead create -f k=@path` — `_resolve_field_value`,
  `src/sase/task_types/fields.py:97-104`
- `sase var ... @path` / `-` — `_read_output_variable_value`,
  `src/sase/main/var_handler.py:470-495`
- the gate CLI's JSON readers — `src/sase/notification_gates/cli_support.py:100-118`
- `sase run --payload @path` — `_launch_request_payload_from_args`,
  `src/sase/main/launch_handler.py:117-119`

`sase bead create -h` documents it right on the adjacent option:

```
-f, --field K=V   Task-type field value (repeatable). A value of @<path> is read from that file
```

But the description is stored raw:

```python
# src/sase/bead/cli_crud_create.py:211
description=args.description or "",
```

```python
# src/sase/bead/cli_crud_update.py:70-71
if args.description is not None:
    fields["description"] = args.description
```

No expansion, no validation, no warning. `-n/--notes` on `update` and the `text`
positional on `sase bead note` have the same gap.

### Why agents specifically reach for it here

`/sase_new_task` (`src/sase/xprompts/skills/sase_new_task.md:117-124`) puts the two
forms one line apart, and tells the agent exactly when to use a file:

> supply a repeatable `-f/--field name=value` for each of its required fields
> (`-f name=@<path>` reads a value from a file, **for prose too long to survive shell
> quoting**):
>
> ```bash
> sase bead create -T "task(<slug>)" -t "<title>" -d "<reproduction, impact, and scope>" ...
> ```

An agent holding a 3 KB diagnosis with backticks, `$`, and newlines in it has just been
told the sanctioned way to pass long prose. Generalizing it to `-d` is the obvious
inference, and nothing contradicts it. `sase-pu`'s author even split its prose into two
files (`description-clean.md` for `-d`, `evidence.md` for `-f evidence=`) — the `-f` one
expanded, the `-d` one did not, and the CLI reported success either way.

### Why the failure is silent and near-permanent

- `sase bead create` prints `Created task: sase-pu — <title>` and exits 0.
- The store commits and pushes; the published page at
  `github.com/sase-org/sase--beads/blob/main/pages/sase-pu/README.md` renders the
  literal too.
- The only surviving copy of the prose is a `/tmp` file on one machine, which any reboot
  removes. The bead page, the JSONL projection, and the event log all carry the literal,
  so there is no recovery path once `/tmp` is gone.

### Denied: "just make `-f` cover it"

`task_type_fields` is create-only — `sase bead update` has no `--field`, and
`handle_bead_update` hard-rejects `--task-type` as immutable
(`src/sase/bead/cli_crud_update.py:53-61`). Descriptions must be editable after
creation, so the fix belongs on `-d`/`-n` themselves.

### Layering (`rust_core_backend_boundary`)

This is argv sugar: a Python CLI reading a local file before handing a plain string to
the existing store API. A web frontend creating a bead would never pass `@path`. No
`../sase-core` change, no wire change.

## Design

One shared resolver, used by every SASE CLI text value that accepts `@<path>`, with
three properties:

1. **Expand** `@<path>` to the file's contents, verbatim (no stripping), `~` expanded.
2. **Never fall back to the literal.** A missing, unreadable, or non-UTF-8 path is
   `Error: ...` on stderr and exit 1. Silence is what caused this bug.
3. **Give a documented escape hatch.** `@@` collapses to a single literal leading `@`,
   so a description that genuinely starts with `@` (an `@agent-name`, an `@ref`) is
   still expressible. A bare `@` with nothing after it stays literal.

Property 3 matters because `@` is already load-bearing in SASE prose (`@ref` artifact
references, `@agent` names, `@small`/`@large` model aliases). The scan above found zero
existing beads with a legitimate leading `@`, so this is insurance, not a migration.

Expansion happens in the **handlers**, not in argparse `type=`, matching how `-f` is
already handled: `Error: <message>` + `sys.exit(1)`, not argparse's exit-2 usage dump.

## Implementation

### Step 1 — Shared resolver

Add `src/sase/cli_file_values.py`:

```python
AT_PATH_PREFIX = "@"

class CliFileValueError(ValueError):
    """A user-facing problem reading an ``@<path>`` CLI value."""

def read_at_path_value(raw: str, *, target: str) -> str:
    """Resolve one CLI text value that may name a file with ``@<path>``."""
```

Behavior, in order:

- `raw` does not start with `@` → return `raw` unchanged (covers `""`).
- `raw` starts with `@@` → return `raw[1:]` (one `@` unescaped, no file read).
- `raw == "@"` → return `"@"`.
- otherwise → `path = Path(raw[1:]).expanduser()`, then
  `path.read_text(encoding="utf-8")`.

Errors, each prefixed with `target` (e.g. `--description`, `--notes`,
`--field evidence`, `note text`) so the message names the option the user actually
typed:

- `FileNotFoundError` / `IsADirectoryError` → `f"{target}: file not found: {path}"`.
  Keep the exact substring `file not found` —
  `test_parse_field_args_rejects_missing_file` matches on it.
- `UnicodeDecodeError` → `f"{target}: file is not valid UTF-8: {path}"`.
- other `OSError` → `f"{target}: cannot read {path}: {exc}"`.

Append to every one of those messages: `" (use @@ for a literal leading @)"`.

Export `AT_PATH_PREFIX`, `CliFileValueError`, and `read_at_path_value` in `__all__` so
symvision sees them used (`sase/memory/symvision.md`).

**No stripping.** `sase var` and `-f` both store file bytes verbatim, and
`test_parse_field_args_reads_at_path` asserts the trailing `\n` survives. Consistency
wins over a cosmetic trailing newline.

### Step 2 — Delegate `-f/--field` to the shared resolver

In `src/sase/task_types/fields.py`, replace the body of `_resolve_field_value` with a
call to `read_at_path_value(value, target=f"--field {key}")`, catching
`CliFileValueError` and re-raising as `TaskTypeCreateError(str(exc))`.

This preserves the existing exception type and message substring — the three tests in
`tests/test_bead/test_task_type_create.py:82-102` must pass unchanged — while giving
`-f` the `@@` escape and the better UTF-8/OSError messages for free. Drop the now-unused
`_FILE_VALUE_PREFIX` constant in favor of the shared `AT_PATH_PREFIX`.

### Step 3 — `sase bead create -d` and `sase bead update -d/-n`

`src/sase/bead/cli_crud_create.py`, `handle_bead_create`: resolve `args.description`
inside the existing `try:`/`except TaskTypeCreateError` block that already wraps
`parse_field_args` (extend it to catch `CliFileValueError` too), before the store
mutation opens. Do not read files inside `bead_store_mutation` — a mid-mutation failure
would abort a partially prepared commit.

`src/sase/bead/cli_crud_update.py`, `handle_bead_update`: resolve `args.description` and
`args.notes` at the top of the function, **before** `bead_store_mutation` opens and
before the `is not None` guards, so `-d ""` still clears a description and a bad path
never opens a store mutation.

Do **not** expand `-D/--design`: that field stores a plan path, where a leading `@` has
no meaning and expansion would be actively wrong. Do not expand `-t/--title`.

### Step 4 — `sase bead note`

`src/sase/bead/cli_crud_evidence.py`, `handle_bead_note`. The `text` positional is
`nargs="+"`, so expand only when it is a single token:

```python
text = args.text
if isinstance(text, list):
    text = read_at_path_value(text[0], target="note text") if len(text) == 1 else " ".join(text)
```

Resolve before `bead_store_mutation` opens, same as Step 3. Multi-token text joins as it
does today — an unquoted `@path` cannot contain spaces anyway.

This step is separable: if it complicates the positional handling more than it is worth,
drop it and file a follow-up. Steps 1–3 are the fix for the reported bug.

### Step 5 — Help text

Per `sase/memory/cli_rules.md` (help output must be complete; options stay alphabetical
by long name — no new options here, so ordering is untouched).

In `src/sase/main/parser_bead_lifecycle.py`:

- Line 171, `create -d/--description`:
  `"Issue description; @<path> reads it from that file, @@ escapes a literal leading @"`
- Line 476, `update -d/--description`: same help string (it currently has none).
- Line 497, `update -n/--notes`: same shape, for notes.
- `note`'s `text` positional (line 236): mention the single-token `@<path>` form if Step
  4 lands.
- The `create` parser description (line ~144) already says fields "take repeatable
  -f/--field values"; add one sentence that free-text values accept `@<path>`.

Add an example to the `create` epilog:

```
sase bead create -T 'task(bug)' -t "Fix retry race" -z medium -d @/tmp/diagnosis.md -f location=src/retry.py -f repro='fails on retry'
```

### Step 6 — Fix the skill example that trained the mistake

In `src/sase/xprompts/skills/sase_new_task.md`, the step-7 block: change the
parenthetical so it covers both options, e.g. "`-d @<path>` and `-f name=@<path>` read a
value from a file, for prose too long to survive shell quoting", and show `-d @<path>`
in the example command.

This is a skill _source_ under `src/`, not a `sase/memory/` note, so it is a normal code
change — but it is a generated skill. Read `sase/memory/generated_skills.md` with
`/sase_memory_read` **before** touching it and follow that note's commit-then-deploy
order. If the deploy workflow blocks (dirty tree, chezmoi integrity guard), leave the
source edit in place, do not force it, and say so in the completion report.

### Step 7 — Repair the four damaged beads

Do this **after** Steps 1–5 land, so it doubles as the end-to-end acceptance test.

For each bead, recreate its description file from the appendix below (or reuse the
`/tmp` original if it still exists and is byte-identical), then:

```bash
sase bead update sase-pn -d @/tmp/p5_ci_desc.md
sase bead update sase-po -d @/tmp/p5_resume_desc.md
sase bead update sase-pp -d @/tmp/p5_flag_desc.md
sase bead update sase-pu -d @/tmp/sase-visual-flake-phase-context/description-clean.md
```

Four separate invocations: `sase bead update` applies the _same_ field values to every
listed ID, so these cannot be batched. Each is its own store commit and push.

Notes on the targets:

- `sase-pn` and `sase-pp` are **closed**. A `-d`-only update sets no status, so it
  should not reopen them or their ancestors. Confirm from the command output that it
  prints only `✓ Updated issue:` and no `○ Reopened ancestor:` line. If the store
  refuses to edit a closed bead, stop and report it rather than reopening them.
- `sase-po` is **snoozed**. `-d` does not touch status, so the
  `update --status`-vs-`snooze` rule in `sase/memory/sase_beads.md` does not apply.
- Verify each afterwards with `sase bead show <id>` and confirm the published page
  (`sase/repos/beads/pages/<id>/README.md`) no longer contains the literal `@/tmp`.

Re-run the detection scan and confirm it now returns zero hits:

```bash
python3 - <<'EOF'
import json
for line in open('sase/repos/beads/issues.jsonl'):
    if not line.strip(): continue
    o = json.loads(line)
    for f in ('description', 'notes', 'title'):
        v = o.get(f) or ''
        if isinstance(v, str) and v.lstrip().startswith('@'):
            print(o.get('id'), f, v[:80])
EOF
```

## Tests

Each must fail before its corresponding change.

**`tests/test_cli_file_values.py`** (new)

1. A plain value round-trips unchanged; `""` round-trips as `""`.
2. `@<file>` returns the file's exact bytes including the trailing newline.
3. `@~/...` expands the home directory.
4. `@@name` returns `@name`; `@@@name` returns `@@name`; bare `@` returns `@`.
5. A missing path raises `CliFileValueError` whose message contains the target label,
   `file not found`, and `use @@`.
6. A directory path raises rather than returning the literal.
7. A non-UTF-8 file raises with `not valid UTF-8`.

**`tests/test_bead/test_task_type_create.py`** (extend; existing three keep passing)

8. `parse_field_args(["evidence=@@literal"])` stores `@literal`.
9. The missing-file error still raises `TaskTypeCreateError` and still says
   `file not found` (regression guard on Step 2's delegation).

**`tests/test_bead/test_task_type_create.py` or a new
`tests/test_bead/test_cli_at_path_values.py`**

10. **The direct regression test for `sase-pu`:**
    `sase bead create -T 'task(flake)' -d @<file>` stores the file's contents, and
    `show`'s description does not equal the `@...` string.
11. `sase bead create -d @<missing>` exits 1, prints `file not found` on stderr, and
    creates **no** bead (assert the store is unchanged — this is the "never silently
    store the literal" invariant).
12. `sase bead create -d @@literal` stores `@literal`.

**`tests/test_bead/test_cli_update_bulk.py`** (extend)

13. `sase bead update <id> -d @<file>` and `-n @<file>` store file contents.
14. `sase bead update <id> -d ""` still clears the description.
15. `sase bead update <id> -d @<missing>` exits 1 and leaves the bead unmodified —
    proving the read happens before the store mutation opens.
16. `-D/--design @<path>` is stored literally (guard against someone "helpfully"
    extending expansion to the plan-path field).

**`tests/test_bead/test_cli_note.py`** (extend, if Step 4 lands)

17. Single-token `@<file>` note text expands; multi-token text still joins with spaces.

**`tests/main/test_parser_command_help.py`** (extend)

18. `sase bead create --help` and `sase bead update --help` both document `@<path>` on
    `-d/--description`, and `update --help` documents it on `-n/--notes`.

## Verification

```bash
just install     # ephemeral workspace — required first
just check       # inline
```

Then, before landing:

```bash
sase monitor start --command 'just check-full' --next '<follow-up action>'
```

`check-full` rather than `check` alone: this touches CLI parser surface and a shared
`src/sase/` module used by the task-type path, which is inside the broadening set.
`check-full` outruns a turn, so it goes through `/sase_monitor`, never inline.

Manual acceptance, after Step 7:

```bash
sase bead show sase-pu   # full flake diagnosis, not a /tmp path
sase bead show sase-po   # full resume-identity diagnosis
```

## Out of scope

Note these; do not fix them here.

- **`sase-pu`'s duplicated `## Flake report` heading.** The `flake` body template
  (`sase bead task-type show flake`) already emits `## Flake report` plus the `node_id`
  and `repro_cmd` bullets, and the author's `evidence` value repeats all three, so the
  rendered bead shows the header twice. It is unfixable through the CLI anyway:
  `sase bead update` has no `--field`. Worth a separate look at whether the type's field
  description should warn that `evidence` renders _under_ the template header.
- **Stdin (`-`) support** for `-d`/`-n`, the way `sase var` and the gate CLI accept it.
  Genuinely useful for heredocs, but it is new surface, not the reported defect.
- **`sase bead close -r/--reason`** and other short-text options. Only extend `@<path>`
  to an option once someone actually needs a file there.
- **`sase/memory/sase_beads.md`** documents `@<path>` for `-f` only (the "Types, Tiers,
  And Launching" section). After this lands that sentence is incomplete and should say
  the convention covers bead free-text values. **Memory-file edits require explicit user
  permission in the implementing conversation** — propose it, do not make it unprompted.
- **Auditing other SASE commands** for the same asymmetry (any option taking long prose
  that does not accept `@<path>`). The bead surface is where it demonstrably bit.

## Appendix — recovery payload for Step 7

Reboot-safe copies of the four descriptions. Use these if the `/tmp` originals are gone.
Each fenced block is the file's exact contents.

### `sase-pn` → `/tmp/p5_ci_desc.md`

```markdown
`tests/main/test_init_memory_glossary.py::test_memory_plan_renders_glossary_terms_block_in_tier2`
fails deterministically on master (HEAD af951d1f9). Discovered while landing epic
sase-p5 (`just test` on the epic's combined tree: 1 failed, 33079 passed, 12 skipped).

Failure:

    assert "Aliases follow in parentheses." in tier2
    AssertionError: ... 'definition plus every term the definition depends on. Aliases follow\nin parentheses.\n\n- Agent Clan (clan)\n- Workspace\n'

Root cause: the rendered Tier 2 glossary block is line-wrapped, so the sentence "Aliases
follow in parentheses." is split across a newline between "follow" and "in". The test
asserts the unwrapped literal substring, which no longer occurs in the rendered output.
Every other assertion in the test (the `### 2.1 Long-Term Memory Files` /
`### 2.2 Glossary Terms` headings, their ordering, the `sase glossary read` snippet)
passes; only this one substring assertion fails.

Introduced by 445afde7c ("feat: Split Tier 2 into 'Long-Term Memory Files' and 'Glossary
Terms' H3 sections"), the only commit touching both the assertion and the rendered
source string; the markdown reflow that wraps the sentence arrived with the same Tier 2
rendering work (see also 28613d6fb, "chore: Format with prettier"). Neither commit
carries a SASE_BEAD footer, so no bead owns this work.

Scope: decide whether the fix is the test (assert against wrap-insensitive text, e.g.
normalize whitespace before the containment check, matching how the sibling assertions
are written) or the renderer (emit the sentence unwrapped). Prefer making the assertion
wrap-insensitive unless the wrap is itself unintended, since the surrounding template is
prettier-formatted and will re-wrap on any width change.
```

### `sase-po` → `/tmp/p5_resume_desc.md`

```markdown
`sase stitch create --resume` decides "is HEAD the commit this run is entitled to
finalize?" by comparing HEAD's **subject line only** against the checkpointed payload
(`src/sase/workflows/commit/workflow_resume.py:63-78`). Any commit whose subject matches
passes that gate; the resume then amends and pushes it as this run's commit.

Proposed by epic sase-p5's plan (plan:202608/commit_finalizer_attribution.md, phase
`restamp` / bead sase-p5.1), which specified this alongside the footer restamp that
phase did implement:

> Also tighten the resume identity check itself. Subject-only matching is what let a
> substantially rewritten commit pass as "the expected commit"; it should additionally
> confirm that `HEAD` is a commit this run is entitled to finalize (for example, that
> `HEAD` is not already pushed to the upstream branch and that its tree still contains
> the checkpointed diff's paths). Keep the check advisory enough not to break legitimate
> conflict resolutions that change content — the goal is to stop silently finalizing an
> unrelated commit, not to force the body to be byte-identical.

sase-p5.1 shipped only the restamp (commit 22e5444bf touches `runtime_tags.py` +14,
`workflow_resume.py` +81 for `_restamp_missing_footer_tags`, and its test), left the
subject-only gate unchanged, and recorded no note descoping the identity work. Verified
unimplemented on master at af951d1f9: the resume path still has no upstream-containment
or changed-path check.

sase-p5's landing raised the stakes rather than lowering them: the restamp now _writes_
this run's `SASE_AGENT`/`SASE_TYPE`/`SASE_BEAD` footer onto whatever commit sits at HEAD
(`_restamp_missing_footer_tags`, called before `finalize_commit`). A HEAD that passes
the subject gate but is not this run's commit is therefore now stamped with this run's
provenance and pushed, instead of merely being pushed unattributed.

Proposed scope:

1. Reject a HEAD already reachable from the upstream tracking branch. Reaching the
   restamp/finalize block requires `"dispatch" not in cp.completed_steps`, so this run
   has not pushed anything yet; a HEAD already on upstream cannot be a commit this run
   created. The `git rebase --skip` resolution (agent drops their own commit because
   upstream already carries an equivalent one) lands exactly here with a matching
   subject.
2. Optionally confirm HEAD still touches the checkpointed diff's paths, kept advisory —
   the motivating sase-p5 incident had the agent legitimately drop a whole paragraph
   during conflict resolution, so this must not force byte or path equality.
3. Fail loudly with HEAD unpushed and recoverable, matching how
   `_restamp_missing_footer_tags` already reports its own failures.

Design note for the implementer: SASE has no VCS-agnostic upstream-containment primitive
today. `provider.revision_id()` exists (`src/sase/vcs_provider/_base.py:162`) but
nothing exposes "is revision X reachable from the tracking branch"; the only upstream
logic lives in the git plugin (`_git_query_ops.py:122-130`) and in the finalizer's
git-specific `git_upstream_ahead_count`. Expect to add a primitive across `_base.py`,
`_hookspec.py`, `_plugin_manager.py`, the git plugin, and `vcs_provider/testing.py`
fakes, and to check the Rust-core backend boundary (`../sase-core`) before implementing
it in Python. Fail open when the provider cannot answer, so an unknown upstream never
blocks a legitimate resume.
```

### `sase-pp` → `/tmp/p5_flag_desc.md`

```markdown
`sase flag new` makes a flag bead live in the **machine-wide, shared** bead store
immediately, but the flag's registry definition only exists on the **one workspace
tree** where the flag was authored, until that tree's commit lands and every other agent
rebases onto it. In that window, the `orphan_bead` rule
(`src/sase/feature_flags/integrity.py:120-136`, surfaced by
`tools/check_feature_flags:581`) compares the shared store against the local checkout
and fails every other agent:

    rule 8: live flag bead 'sase-pk' has no definition (key 'commit_finalizer_shared_clone_exempt')

`just check`'s `lint (feature flags)` gate is red repo-wide for the duration, for a
reason unrelated to any of those agents' diffs.

This is recurring, not a one-off — four separate epics have hit it and each recorded it
only as a note, so no task bead has ever been filed:

- sase-p5 (this epic): `DISCOVERED ISSUE` from agent 05l, 2026-08-18 — an unrelated
  glossary-inference tree at bf7e2bca2 failed `tools/check_feature_flags` on every run
  for flag bead sase-pk (`commit_finalizer_shared_clone_exempt`), created ~33m earlier
  by sase-p5.4. Two consecutive runs failed identically, so not a flake.
- sase-oo.3, 2026-08-17: repo-wide red for sase-om (`completion_refresh_on_update`) from
  the concurrent sase-oc epic; that phase had to verify via direct ruff/mypy/pytest runs
  instead of `just check`.
- sase-oo.2, 2026-08-17: same gate, "flag bead exists but this tree has no matching flag
  definition."
- sase-p3.10, 2026-08-18: same gate for sase-pa (`epic_resume` gate kind), blocking a
  fully green check.

Reproduction: on any workspace, run `sase flag new <key>`; on any _other_ workspace
whose tree predates that commit, run `just check` (or
`.venv/bin/python tools/check_feature_flags`). It fails on the orphan-bead rule until
the definition commit reaches that tree. Every instance above self-resolved once the
defining commit landed on master and the other agents rebased.

Scope: decide where the window should close. Candidates worth weighing:

- Do not mark the flag bead live until its definition commit lands (create it in a
  pre-live status that `LIVE_FLAG_STATUS_VALUES` excludes, promoting on the commit that
  adds the definition).
- Keep the bead live but make `orphan_bead` tolerant when the bead was created after the
  checkout's HEAD commit date — the drift is then provably "my tree is older than this
  bead", not a genuine orphan.
- Downgrade `orphan_bead` from `error` to a warning for beads younger than some
  threshold, keeping the error for genuinely abandoned definitions.

Whatever is chosen must still catch the real defect the rule exists for: a live flag
bead whose definition was deleted or never written.
```

### `sase-pu` → `/tmp/sase-visual-flake-phase-context/description-clean.md`

```markdown
`just test-visual` failed this existing golden under the full suite
(`agents_phase_bead_and_plan_context_120x40.png`, 8539/1520532 pixels, 8511 material).
The actual zoom-metadata frame is a mid-stream SASE CONTEXT paint: PLAN and BEAD are
complete, then `ARTIFACTS · resolving…` and `MEMORY · resolving…` appear and push
`AGENT PROMPT` off the snapshot. Isolated `just test-visual --` of the same node on the
same tree passed in 12.58s. Do not rebaseline the golden.

The test waits for `"Phase plan"` then `wait_for_visual_idle`, which can accept a stable
pending-lane frame before artifacts/memory finish. Fix the wait (require those lanes to
leave `resolving…`, or wait on a complete detail-header summary) rather than accepting
the transient.

Not caused by the in-flight monitor-phase rendering tale: that tree does not change
context-lane enrichment, the fixture has no monitor member, and isolation matches the
committed golden.
```
