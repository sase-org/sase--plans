---
tier: tale
title: sase artifact create publishes an artifacts output variable
goal:
  Every successful sase artifact create run by an agent records the new artifact in a
  SASE-managed `artifacts` output variable that shows up in the agent's Main deck panel
  and is readable by later agents.
size: medium
proposed_by: bbugyi200.apollo.5p
create_time: 2026-10-07 18:31:57
status: wip
---

# Plan: `sase artifact create` publishes an `artifacts` output variable

## Goal

Every successful `sase artifact create` (and its `sase artifact-file create` alias) run
by a SASE agent records the new artifact in a SASE-managed output variable named
`artifacts` on the calling agent. This value shows up with no extra agent work in:

- the agent's Main deck panel (Context card → `OUTPUT VARIABLES` section), plus the
  clan/tribe summary panels that aggregate member variables;
- `sase var get` (the agent's own snapshot), selectors such as
  `sase var get 'research.0k.*.artifacts'`, and `sase var list` history;
- later agents' Jinja context as `{{ agents["<producer>"].artifacts }}`;
- Telegram completion messages and agents-sidecar pages (both already render every
  non-`STOP` output variable).

Today a research-swarm researcher (`#research_swarm` → `#research(suffix=…)`) registers
its report with `sase artifact create -p … -l "research:<repo-relative-path>"`, but the
only places to see the result are the chat transcript and the `wait.artifacts` context
that only `%wait` consumers get. A human looking at `research.<N>.cld` in the Agents tab
cannot see which report it produced or its `file:` ref, and an agent outside the wait
graph cannot discover it without searching the artifact index.

## Variable design

The variable is a **list of maps, one per registered artifact, in registration order**.
Each map uses the same field names as the existing `wait.artifacts` Jinja entries, so a
consumer can use one mental model (and often the same template code) for both:

| Field         | Present                                     | Meaning                                                                                                                            |
| ------------- | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `ref`         | always                                      | Canonical `file:explicit:<hash>` identity; works with `sase artifact read`, `show`, `open`, and `path`.                            |
| `label`       | always                                      | The `-l/--label` value (default: the source file name). Research labels such as `research:202610/x/x__cld.md` are themselves refs. |
| `kind`        | always                                      | `markdown`, `image`, `pdf`, `plan`, `chat`, or `file` (lets consumers filter, e.g. `a.kind == "image"`).                           |
| `path`        | always                                      | Absolute path of the immutable stored snapshot under `~/.sase/artifacts/`.                                                         |
| `source_path` | only when the source was kept (no `--move`) | Absolute path of the living source file. Omitted for `--move`, where it would be a dangling path.                                  |
| `bead`        | only when `--bead` attached successfully    | The bead id the artifact was attached to.                                                                                          |

Rendered in the Main deck panel (keys sort alphabetically, list order is preserved):

```text
OUTPUT VARIABLES
artifacts:
  - kind: markdown
    label: research:202610/topic/topic__cld.md
    path: /home/<user>/.sase/artifacts/agents/<project>/<timestamp>/topic__cld-0123456789ab.md
    ref: file:explicit:0123456789abcdef01234567
    source_path: /…/research/202610/topic/topic__cld.md
```

Rationale for the choices that matter:

- **Name `artifacts`.** It is the natural name, parallels `wait.artifacts`, and reads
  well in Jinja (`agents["research.0k.final"].artifacts[0].ref`) and selectors
  (`sase var get 'research.0k.*.artifacts'`). `sase var list --key 'artifact*' --hidden`
  shows no historical use of the key, so collisions are unlikely; the collision rule
  below keeps the rare case safe.
- **List, not a map keyed by label or ref.** Order matters to humans ("what did this
  agent produce, in order"), labels are not unique (the default label is the bare file
  name), and a list of uniform maps is what Jinja loops and `selectattr` want.
- **Latest snapshot per logical artifact.** Re-registering the same file with the same
  label after editing it mints a new `ref` (the id hashes stored content). Showing both
  snapshots would be noise in the deck panel and make "the report" ambiguous for agents.
  The older snapshot remains in the artifact index (`sase artifact list`), so nothing is
  lost.
- **Both `path` and `source_path`.** `path` is the immutable snapshot that is guaranteed
  to exist; `source_path` is where a human finds the living file (for example in the
  research repo). Both are already what `wait.artifacts` exposes.
- **No opt-out flag.** The request is "always"; the variable is small, visible metadata
  like any other output variable.

### Merge rules

Recording is a single locked read-modify-write of `agent_meta.json` so concurrent
`sase artifact create` calls from parallel tool calls cannot lose entries.

1. Absent `artifacts` → start a new list.
2. Existing value must be the SASE-managed shape: a list whose items are all maps with a
   non-empty string `ref`. Anything else (an agent stored its own `artifacts` value with
   `sase var set`) is left untouched; print a one-line stderr warning naming the ref
   that was not recorded and continue with exit status unaffected.
3. Replace the first existing entry whose `ref` equals the new ref, or whose `label` and
   `source_path` both equal the new entry's (both present), **in place**; drop any
   further matching entries. Otherwise append.
4. Keep at most `MAX_CREATED_ARTIFACT_ENTRIES = 100` entries, evicting the oldest. If
   `normalize_var_value` still rejects the list (node or 64 KiB encoded-size limit),
   keep evicting the oldest entries until it is accepted. If the new entry alone is
   rejected (for example an oversized label), skip recording with a stderr warning. Any
   eviction prints a stderr note that older entries were dropped and that
   `sase artifact list` still has every artifact.
5. Variable recording is best-effort: any failure (missing/locked artifacts dir, I/O
   error, validation error, the 256-variable cap) prints
   `warning: … artifacts output variable was not updated: <reason>` to stderr and never
   turns a successful artifact creation into a failure. This mirrors the existing
   best-effort `_derive_links_for_created_artifact` call.

### CLI output

After the existing `id:`, `source:`, `path:`, `ref:` lines, print
`var: artifacts[<index>]` (the zero-based index of the recorded entry after eviction)
when the variable was updated, then the existing `bead: <id>` line when applicable. The
`var:` line tells the agent and a transcript reader exactly where the entry lives in the
agent's variable map (`sase var get --format json` shows the whole snapshot; another
agent can select `sase var get '<agent>.artifacts[0]["ref"]' --format raw` — selector
JSON paths use `[INDEX]` and `["KEY"]`, never dotted traversal, and an unscoped
`sase var get artifacts` selects the newest occurrence from history, not the current
agent). No `var:` line is printed when recording was skipped.

## Implementation steps

### 1. Atomic single-key update helper (`src/sase/core/agent_output_variables.py`)

Add `update_agent_output_variable(artifacts_dir, key, update)` where
`update: Callable[[VarValue | None], VarValue]` receives the current value (`None` when
the key is absent or stored as `null`; this caller treats both the same) and returns the
new value. It runs inside `update_agent_meta_locked`, validates the key with
`_validate_output_variable_key`, normalizes the result with `normalize_var_value`,
enforces `MAX_OUTPUT_VARIABLES` exactly like `set_agent_output_variables`, writes the
merged map back to `meta["output_variables"]`, and returns the stored normalized value.
Export it in `__all__`. Let exceptions raised by `update` propagate (the lock is
released by `update_agent_meta_locked`'s `finally`).

This stays in Python: explicit artifact storage (`sase.core.artifact_file_explicit`) and
output-variable storage (`sase.core.agent_output_variables`) are both Python-owned today
and no frontend other than this CLI command produces the value, so no `sase-core`
Rust/binding change is required.

### 2. Created-artifact variable module (`src/sase/core/created_artifacts_variable.py`, new)

Keep the shape rules in one small, pure, testable module:

- `CREATED_ARTIFACTS_OUTPUT_VARIABLE = "artifacts"` and
  `MAX_CREATED_ARTIFACT_ENTRIES = 100`.
- `created_artifact_entry(artifact_file: ArtifactFile, *, source_retained: bool, bead_id: str | None) -> dict[str, VarValue]`
  building the map described above (`ref` is `f"file:{artifact_file.id}"`; `source_path`
  only when `source_retained` and `artifact_file.source_path` is set; `bead` only when
  `bead_id`).
- `merge_created_artifact_entry(current: VarValue | None, entry) -> CreatedArtifactMerge`
  implementing merge rules 1–4 and returning the new list, the entry index, and the
  number of evicted entries; raise a dedicated `ValueError` subclass for the
  incompatible-shape case (rule 2) and for "entry alone rejected" so the caller can word
  the warnings.
- `record_created_artifact(artifacts_dir, artifact_file, *, source_retained, bead_id) -> CreatedArtifactMerge`
  that calls `update_agent_output_variable` with a closure over
  `merge_created_artifact_entry`.

Use a frozen dataclass for `CreatedArtifactMerge`. Keep everything that tests need
public (symvision flags private-name imports from tests).

### 3. Wire it into `handle_create` (`src/sase/artifact_cli/create.py`)

After `_derive_links_for_created_artifact(...)`:

1. If `bead_id` is set, run `_attach_reference_to_bead` and remember whether it
   succeeded (do not return yet).
2. Call a new best-effort
   `_record_artifacts_output_variable(agent_artifacts_dir, artifact_file, source_retained=not args.move, bead_id=bead_id if attached else None)`
   that wraps `record_created_artifact` in `try/except Exception`, prints the
   `var: artifacts[<index>]` line on success, and prints the stderr warnings/notes from
   the merge rules otherwise.
3. Then keep the current bead behavior: on attach failure return the existing
   `failed to attach … to bead …` error (exit 1); on success print `bead: <id>`.

This order means a failed bead attachment still records the artifact (without `bead`),
and an agent's retry with the same file and label replaces that entry in place (same
`ref`) and adds `bead`.

### 4. Help text (`src/sase/main/parser_artifact.py`)

Give the `create` subparser a `description=` (keep the short `help=`) using
`argparse.RawDescriptionHelpFormatter` like `doctor`, explaining: what is stored; the
printed `id:`/`source:`/`path:`/`ref:`/`var:`/`bead:` lines; that the artifact is also
recorded in the agent's `artifacts` output variable (shown in the Agents tab
`OUTPUT VARIABLES` section, readable with `sase var get` from inside the agent and by
later agents as `{{ agents["<name>"].artifacts }}`); and that re-registering the same
label and source replaces that entry with the newer snapshot. No new options are added,
so option ordering and short-alias rules are unaffected.

### 5. Agent-facing skill source (`src/sase/macros/skills/sase_var.md`)

In "Cross-agent facts":

- Add a bullet: `sase artifact create` automatically maintains the SASE-managed
  `artifacts` list (fields `ref`, `label`, `kind`, `path`, optional `source_path` and
  `bead`); do not `sase var set artifacts` yourself; read your own with `sase var get`,
  another agent's with `sase var get 'research.0k.*.artifacts'` or
  `sase var get 'research.0k.cld.artifacts[0]["ref"]' --format raw`, or in a later
  prompt with `{% raw %}{{ agents["research.0k.cld"].artifacts[0].ref }}{% endraw %}`.
- Reword "store a report as an artifact file and publish its path instead" to say that
  `sase artifact create` already publishes it under `artifacts`.

Wrap every `{{ … }}` example in `{% raw %}…{% endraw %}` like the existing bullets. Only
edit the source template; do **not** run `sase skill init --force` or deploy to chezmoi
from this unlanded tree (generated skills deploy only from a landed revision).
`sase skill init --diff` is fine for previewing.

### 6. Docs

- `docs/agent_images.md` → "Explicit Artifact Contract": update the "prints four lines"
  example to include `var: artifacts[0]` (and mention the conditional `bead:` line), and
  add a short "`artifacts` output variable" subsection with the field table, the merge
  rules (latest snapshot per label+source, 100-entry cap, agent-owned shape left alone,
  best-effort), and one Jinja and one selector example.
- `docs/macros.md` → cross-agent output variables (next to the `plan_file` paragraph): a
  paragraph on the SASE-managed `artifacts` variable with a `research_swarm`-style
  example such as `{{ agents["research.final"].artifacts[0].ref }}`, noting that its
  entries share field names with `wait.artifacts`.
- `docs/configuration.md` → `sase var` section (next to the `STOP` paragraph) and the
  `sase artifact create` row/section around the artifact command table: one or two
  sentences each.
- `docs/ace.md` → `OUTPUT VARIABLES` bullet: values come from `sase var set` and from
  `sase artifact create` (the `artifacts` list).
- `docs/notifications.md` → the output-variables paragraph: completion snapshots include
  `artifacts` like any other non-reserved variable.

Do not edit `CHANGELOG.md` by hand and do not edit `sase/memory/` notes (memory changes
were not requested).

### 7. Tests

- `tests/core/test_agent_output_variables.py`: `update_agent_output_variable` receives
  `None` for an absent key, sees the stored value otherwise, preserves other keys,
  normalizes the result, rejects an invalid key, and enforces `MAX_OUTPUT_VARIABLES`
  when adding a new key.
- `tests/core/test_created_artifacts_variable.py` (new): entry construction (field set,
  `source_path` omitted when not retained, `bead` only when given); append order;
  same-`ref` replace in place; same `label`+`source_path` with a new ref replaces in
  place; same label with a different source appends; incompatible existing shapes
  (string, list of strings, map entries without `ref`) raise the dedicated error; cap
  eviction keeps the newest 100 and reports the evicted count; an oversized-but-valid
  list evicts until `normalize_var_value` accepts it.
- `tests/main/test_artifact_handler.py`: update the existing create output assertions
  for the new `var: artifacts[0]` line; assert
  `agent_meta.json["output_variables"] ["artifacts"]` after a create; two different
  files produce two entries in order; re-registering an edited source with the same
  label leaves one entry pointing at the new ref; `--move` omits `source_path`; a
  pre-existing agent-authored `artifacts` string is left untouched with a stderr warning
  and exit 0; a forced recording failure (monkeypatch `record_created_artifact` to
  raise) still exits 0 with the artifact stored and a warning; other pre-existing output
  variables survive.
- `tests/test_artifact_create_bead_attachment.py`: a successful `--bead` attach records
  `bead` on the entry; a failing attach still exits 1 but records the entry without
  `bead`.
- `tests/test_agent_output_variable_context.py` (or its fixtures): a waiting consumer
  can render `{{ agents["producer"].artifacts[0].ref }}` and loop
  `{% for a in agents["producer"].artifacts %}{{ a.label }}{% endfor %}` from a producer
  whose `agent_meta.json` holds the new shape.
- A concurrency test: several threads (separate `open()` calls, so `flock` contends)
  each recording a distinct artifact into the same artifacts dir end with every entry
  present.

### 8. Verify

Run `just fix` (or at least `just fmt`), then `sase tool run check`. No TUI rendering
code changes, so PNG visual snapshots are not expected to move; do not run
`just check-full`.

## Acceptance criteria

- Inside an agent, `sase artifact create -p report.md -l "Report"` prints a
  `var: artifacts[0]` line, and `sase var get --format json` (inside that agent) shows
  an `artifacts` list with one entry holding `ref`, `label`, `kind`, `path`, and
  `source_path`.
- The agent's Main deck panel `OUTPUT VARIABLES` section shows the `artifacts` list
  without any `sase var set` call; a `#research_swarm` clan summary shows each member's
  `artifacts` preview.
- A later `%wait` consumer can render `{{ agents["<producer>"].artifacts[0].ref }}`.
- Re-registering an edited file with the same label yields a single entry with the new
  ref; registering 101 artifacts keeps the newest 100.
- No variable problem ever changes `sase artifact create`'s exit status; bead-attach
  failures still exit 1 as before.
- `sase tool run check` passes.

## Out of scope / notes for the implementer

- No changes to the `sase-research-artifacts` macros: `#research` already registers
  reports with `sase artifact create`, so researchers, the lead, the image agent, and
  the linker all gain the variable automatically, and the lead keeps using
  `wait.artifacts`.
- No TUI-specific rendering for the variable (for example clickable refs); the existing
  YAML-shaped `OUTPUT VARIABLES` rendering is used as-is.
- Observed while planning, not part of this change: the deployed Claude skill at
  `~/.claude/skills/sase_artifact/SKILL.md` has no source under
  `src/sase/macros/skills/` and is stale (it documents a nonexistent `-n` label flag and
  says the file is moved by default). Leave it alone here; it is worth a separate
  follow-up.
