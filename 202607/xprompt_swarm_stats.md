---
tier: epic
title: Count xprompt swarms in Statistics → XPrompts
goal: 'Every agent launched by an xprompt swarm records that swarm at its launch boundary,
  so swarms such as #research_swarm appear as first-class rows in the Admin Center
  Statistics XPrompts sub-tab, are focusable there, and are visible in the Agents-tab
  XPROMPTS panel.

  '
phases:
- id: swarm-provenance
  title: Carry the swarm chain on expansion records
  depends_on: []
  size: small
  description: 'swarm-provenance: add an ordered outer-to-inner swarm-name chain to
    the expanded-segment record in sase/agent/xprompt_swarm.py, populated in all four
    expansion branches and inherited unchanged by pass-through segments, without altering
    template-group behavior.

    '
- id: launch-env-plumbing
  title: Thread provenance to the spawn point
  depends_on:
  - swarm-provenance
  size: medium
  description: 'launch-env-plumbing: thread the per-segment swarm chain through the
    CLI and ACE launch paths alongside segment_template_groups, inject it into each
    spawned slot''s environment like SASE_MULTI_AGENT_PROMPT_FILE, cover the single-segment
    fall-throughs, and scrub it from nested launches.

    '
- id: core-swarm-kind
  title: Teach the Rust scanner the swarm kind
  depends_on: []
  size: small
  description: 'core-swarm-kind: in the sibling sase-core repo, normalize a "swarm"
    kind in the agent-scan xprompts.json loader so it survives into the statistics
    wire, extend the Rust tests, and commit with Conventional Commits so release-plz
    computes the version.

    '
- id: runner-capture
  title: Write the swarm into launch-boundary metadata
  depends_on:
  - launch-env-plumbing
  size: medium
  description: 'runner-capture: give the used-xprompts collector a swarm-names parameter
    that prepends derived records with kind "swarm", catalog-resolved tags and no
    arguments, upgrade rather than duplicate names the lexical scan already found,
    and read the env var in the agent runner''s launch-boundary capture.

    '
- id: tui-labels
  title: Render the swarm kind everywhere kinds are rendered
  depends_on:
  - core-swarm-kind
  - runner-capture
  size: small
  description: 'tui-labels: label the new kind in the Statistics XPrompts table and
    focus header, give it its own glyph and summary counting in the Agents-tab XPROMPTS
    panel, and document the attribution contract in the statistics help modal.

    '
- id: docs-and-goldens
  title: Floor bump, docs, snapshots, full check
  depends_on:
  - tui-labels
  size: small
  description: 'docs-and-goldens: bump the sase-core-rs floor to the published version
    containing the swarm kind, add the CHANGELOG entry and documentation updates,
    refresh the affected Statistics XPrompts PNG goldens, and run the full check plus
    visual suites.

    '
create_time: 2026-07-29 21:09:40
status: done
bead_id: sase-b1
---

- **PROMPT:** [prompts/202607/xprompt_swarm_stats.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/xprompt_swarm_stats.md)
- **BEAD:** [sase-b1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b1/README.md)

# Record xprompt-swarm origin so swarms appear in Statistics → XPrompts

## Problem

`#research_swarm` never appears in the Admin Center **Statistics → XPrompts** sub-tab, and it never can. The same is
true for every xprompt swarm. Verified against live data (90-day window, `sase.stats.query_run_stats`):

```
runs_with_xprompts 1801 · distinct 26 · rows 26
research_swarm in rows? False
sase/reads     in rows? False
research-ish rows: [('research', 13), ('research/image', 8)]
```

So the xprompts _inside_ the swarm body are counted, but the swarm that launched them is invisible. In the current
catalog three xprompts are swarms (bodies with top-level `---` separators): `research_swarm`, `old_research_swarm`, and
`sase/reads`. None of them can ever produce a row, so none can be focused either — the focus picker
(`src/sase/ace/tui/modals/statistics_xprompt_picker_modal.py`) is built from the same rows.

Impact is not marginal: `#research_swarm` was referenced ~20 times in one month of prompt history, each launch fanning
out to ~4 agents. Attributed correctly it would rank near the top of the table, behind `#gh`.

## Root cause

The swarm reference is consumed in the **dispatching** process, before any agent's launch boundary exists:

1. `src/sase/agent/xprompt_swarm.py:211` (`_expand_xprompt_swarms_with_metadata`) renders the swarm body and replaces
   the `#<swarm>` reference with one prompt per body segment. The returned `_ExpandedXpromptSwarmSegment`
   (`src/sase/agent/xprompt_swarm.py:47`) keeps only a `template_group` (`"xprompt:<name>:<n>"`, used for agent-name
   allocation) — the swarm identity is not retained as provenance, and for nested swarms only the outermost group
   survives (`group = template_group or _next_template_group(...)`).
2. Each expanded segment is spawned as an independent agent (`src/sase/agent/multi_prompt_launch_execution.py`), so each
   child's prompt is a rendered body segment with no `#<swarm>` reference left in it.
3. In the child, `src/sase/axe/run_agent_runner_setup.py:254` calls `write_used_xprompts(artifacts_dir, prompt)`, and
   `collect_used_xprompts` (`src/sase/xprompt/used_xprompts.py:29`) is a purely **lexical** scan of that prompt. It
   records what it finds (`gh`, `research`, `fork`, `research/image`) and nothing else.
4. The Rust scanner aggregates exactly what `xprompts.json` contains (`crates/sase_core/src/agent_scan/scanner.rs`,
   `load_used_xprompts`), so the statistics payload can never mention the swarm.

Confirmed on disk for one real launch (`#research_swarm::` at `260729_193349`, four spawned agents):

| run                            | `xprompts.json` names          |
| ------------------------------ | ------------------------------ |
| `…193349` (`research.r.cdx`)   | `gh`, `research`               |
| `…193350` (`research.r.cld`)   | `gh`, `research`               |
| `…193351` (`research.r.final`) | `gh`                           |
| `…193352` (`research.r.image`) | `gh`, `fork`, `research/image` |

The original text (`#gh:… #research_swarm:: I recently produced some research …`) is persisted at
`~/.sase/multi_prompts/202607/gh_sase_org__sase-multiprompt-260729_193349.md`, so the information exists at launch — it
is simply never handed to the child.

## Approach

Attribute the swarm to **each spawned child run**, the same unit every other xprompt row uses. There is no parent agent
run to attach it to (the dispatcher is the ACE app or the CLI, not an agent), and per-child attribution gives runs,
success rate, wall-clock, model/project breakdowns and pairings for free — a `#research_swarm` row will pair with `#gh`,
`#research`, `#fork`, `#research/image`, which is exactly the useful reading.

Mechanism, mirroring the existing `SASE_MULTI_AGENT_PROMPT_FILE` precedent (`src/sase/history/multi_agent_prompt.py:7` →
injected in `multi_prompt_launch_execution.py:323` → read in `src/sase/axe/run_agent_runner_launch.py:177` → scrubbed in
`src/sase/agent/launch_spawn.py:53`):

1. Expansion records carry the swarm chain (outer → inner) as provenance.
2. The launch path threads that chain per segment and injects it into the child's environment.
3. The child's launch-boundary writer merges it into `xprompts.json` as a first-class record with `kind: "swarm"`.
4. The Rust normalizer learns the `swarm` kind so it survives into the statistics wire; the TUI labels it.

### Decisions

- **`kind: "swarm"`**, not `"part"`. Kind classification is core domain behavior, and a distinct kind lets the row read
  as a swarm in the table, the focus header, and the Agents-tab XPROMPTS panel. Until a core build with the new kind is
  installed, `crates/sase_core/src/agent_scan/scanner.rs` normalizes it to `"unknown"` and the row renders as
  `#research_swarm  unknown` — degraded label, correct counts. Phase `core-swarm-kind` closes that window.
- **No arguments recorded** on the derived record. `#research_swarm::` passes the user's whole research prose as a
  positional argument; storing it would bloat every child's `xprompts.json` and add nothing (the Rust scanner ignores
  `positional`/`named`, and the Agents-tab panel truncates args to 40 chars anyway). `Refs` will equal `Runs` for
  swarms, which is honest.
- **Tags come from the swarm's own `XPrompt.tags`**, resolved from the catalog in the child, matching how every other
  record's tags are produced.
- **Nested swarms record every link in the chain.** A swarm composing another swarm produces a row for each, so
  `template_group` (which collapses to the outermost group) cannot be reused as the provenance carrier.
- **Environment, not a pre-written file.** The parent computes the child's artifacts dir before spawn, but that
  directory does not exist yet, and `write_used_xprompts` overwrites the shared `xprompts.json` unconditionally. The env
  channel keeps the child's writer as the single owner of that file.
- **Env hygiene is mandatory.** An agent spawned _by_ a swarm child must not inherit the attribution, or every
  descendant would over-count the swarm. Scrub it in `launch_spawn` exactly like the multi-agent prompt file env.
- **Deduplicate against the lexical scan.** If a prompt genuinely still contains `#<swarm>` (embedded-reference
  expansion keeps surrounding prose), the record must appear once, not twice.

### Non-goals

- **No backfill.** Historical child runs cannot recover their swarm from their own artifacts (their
  `submitted_xprompt.md` is the rendered segment). A join through `~/.sase/multi_prompts/*` and
  `agent_meta.json:agent_clan_generation` would only work for clan-based swarms and would rewrite historical artifacts
  for a cosmetic gain. Attribution is forward-only, and phase `docs-and-goldens` states that in the help modal.
- No change to how non-swarm xprompts, workflow step templates (`xprompts_<step>.json`), or the runtime/projects views
  are counted.
- No change to swarm expansion semantics: segment splitting, directive handling, VCS-ref inheritance, and agent-name
  allocation all keep their current behavior.

## Phases

### Phase `swarm-provenance` — carry the swarm chain on expansion records

**Depends on:** nothing.

Add swarm provenance to `src/sase/agent/xprompt_swarm.py`:

- Extend `_ExpandedXpromptSwarmSegment` with `swarm_xprompts: tuple[str, ...] = ()`, ordered outermost → innermost.
- Populate it in all four expansion paths: the sole-top-level-reference branch (`_expand_xprompt_swarms_with_metadata`),
  the single embedded reference branch (`_expand_embedded_xprompt_swarm_reference`), the multiple-embedded-references
  branch (`_expand_multiple_embedded_xprompt_swarm_references`), and the pass-through/fast-path branches (which must
  inherit the parent chain unchanged, not reset it).
- Recursion appends the current swarm name to the inherited chain, so a swarm composing a swarm yields both names in
  order. Duplicate names in one chain (a swarm reached twice along one path) collapse to a single entry.
- Non-swarm segments keep an empty chain.

Tests in `tests/test_xprompt_swarm_expansion.py`: sole reference, embedded reference with surrounding prose, multiple
embedded references, two-level nesting, and a plain segment (empty chain). Assert `template_group` behavior is unchanged
by these edits.

**Done when:** provenance is available for every expanded segment and the existing swarm-expansion suite still passes
unchanged.

### Phase `launch-env-plumbing` — thread provenance to the spawn point

**Depends on:** `swarm-provenance`.

Define the env constant next to the collector it feeds (e.g. `SASE_LAUNCH_SWARM_XPROMPTS` in
`src/sase/xprompt/used_xprompts.py` or a small module beside it), encoded as a stable, parse-safe list (JSON array, or
comma-separated names — xprompt names contain `/` but never `,`).

Thread the per-segment chain alongside the existing `segment_template_groups` parameter, which already runs the full
length of this path:

- `src/sase/agent/launch_cwd_agents.py` — both the `segment_extra_env` loop and the single-call branch collect
  `record.swarm_xprompts`; pass them to `launch_multi_prompt_agents`. **Do not miss the single-agent path**: a swarm
  that renders exactly one segment falls through to `query = normalize_default_vcs_workflow(expanded_segments[0])`, so
  the chain must be merged into `extra_env` there too.
- `src/sase/agent/multi_prompt.py` — add a `swarm_xprompts` field to `MultiPrompt` beside `template_groups`.
- `src/sase/ace/tui/actions/agent_workflow/_launch_body_impl.py` — populate that field from `expanded_records`;
  `_launch_multi_prompt.py` forwards it to `launch_multi_prompt_agents`. Cover the ACE single-segment fall-through
  (`len(multi.segments) == 1`) the same way as the CLI path.
- `src/sase/agent/multi_prompt_launcher.py` → `src/sase/agent/multi_prompt_launch_execution.py` — accept
  `segment_swarm_xprompts`, validate its length like `segment_extra_env`, and inject into `slot_env` next to
  `MULTI_AGENT_PROMPT_FILE_ENV`, so every fan-out slot of a swarm segment inherits it.
- `src/sase/agent/launch_spawn.py` — add a `_remove_inherited_swarm_xprompts_env` scrub mirroring
  `_remove_inherited_multi_agent_prompt_env`, applied unless the launch explicitly supplies the variable.
- `src/sase/agent/launch_cwd_bead_work.py` passes `segment_extra_env` through the same launcher; confirm bead-work
  segments are unaffected (empty chains) rather than crashing on the new length check.

Tests: extend `tests/test_multi_prompt_launcher_xprompt_groups.py` (or a sibling) to assert the env var reaches each
spawned slot for a swarm launch, is absent for a plain multi-prompt launch, reaches the single-segment swarm path, and
is scrubbed from a nested launch spawned by an agent that already has it set.

**Done when:** every spawn path that can originate from a swarm carries the chain in the child environment, and nested
launches do not inherit it.

### Phase `core-swarm-kind` — teach the Rust scanner the `swarm` kind

**Depends on:** nothing (runs in parallel with the Python capture phases).

In the sibling Rust core repo (open it with `sase repo open sase-core -r "<reason>"`):

- `crates/sase_core/src/agent_scan/scanner.rs`, `load_used_xprompts`: accept `Some("swarm") => "swarm"` alongside
  `"workflow"` and `"part"`; unrecognized kinds keep falling back to `"unknown"`.
- Add a unit test for a `swarm` entry surviving normalization, and confirm `references` counting, name dedup, and the
  `unknown` fallback are unchanged.
- Check `crates/sase_core/tests/agent_scan_parity.rs` and the `agent_scan/index.rs` cached-record tests for fixtures
  that enumerate kinds; extend rather than rewrite. No index schema bump is needed — the record shape is unchanged, and
  `xprompts.json` is already signed so late writes refresh cached rows.
- Verify nothing in `crates/sase_core/src/agent_stats/run.rs` special-cases kind while aggregating rows, focus
  breakdowns, or partner pairings; if it does, extend it so swarm rows aggregate identically.
- Commit with Conventional Commits (`feat:`) so release-plz computes the version. Do **not** hand-edit
  `[workspace.package].version`.

**Done when:** the Rust suite passes with `swarm` normalized end-to-end, and the commit is on `master` awaiting a
release-plz publish.

### Phase `runner-capture` — write the swarm into the launch-boundary metadata

**Depends on:** `launch-env-plumbing`.

- `src/sase/xprompt/used_xprompts.py`: add `kind: Literal["workflow", "part", "swarm"]` to `_UsedXPromptRecord`, and
  give `collect_used_xprompts`/`write_used_xprompts` a `swarm_xprompts: Sequence[str] | None = None` parameter. Derived
  records are prepended (a swarm is the outermost provenance of the prompt), resolve `tags` from the catalog, and carry
  empty `positional`/`named`. A name already recorded by the lexical scan is upgraded in place to `kind: "swarm"` rather
  than duplicated. Unknown names (a swarm defined in config that has since been removed) are still recorded by name with
  empty tags, since provenance is a launch fact, not a catalog lookup.
- `src/sase/axe/run_agent_runner_setup.py` (~line 254): read the env var and pass the decoded names into
  `write_used_xprompts`. Keep the whole capture inside the existing best-effort `try/except` — metadata capture must
  never take down a detached launch.
- Leave the `step_only` / `xprompts_<step>.json` contract alone: provenance belongs to the launch boundary, so only the
  shared file gains the record. Verify `src/sase/xprompt/workflow_executor_steps_prompt.py` and
  `src/sase/main/query_handler/_embedded_workflows.py` still behave when the shared file already contains a swarm
  record.

Tests in `tests/test_used_xprompts.py`: derived record ordering and shape, tag resolution from the catalog, unknown
swarm name, upgrade-in-place when the prompt still contains the reference, nested chain producing two records, and no
`swarm` record when the env var is absent. Add an axe-level test that a swarm-launched child writes the record into
`xprompts.json`.

**Done when:** a `#research_swarm` launch produces four children whose `xprompts.json` each contain a `research_swarm`
record with `kind: "swarm"`, and `query_run_stats` returns a `research_swarm` row for the window.

### Phase `tui-labels` — render the swarm kind everywhere kinds are rendered

**Depends on:** `runner-capture`, `core-swarm-kind`.

- `src/sase/ace/tui/modals/statistics_pane_xprompts.py`: extend the kind maps in `_xprompt_cell` and
  `_xprompt_focus_header` with `"swarm": "swarm"`.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_xprompts.py`: give `kind == "swarm"` its own glyph and color instead of
  falling into the `part` branch, and count swarms in `_summary` (currently only `workflow` and `part` are counted, so a
  swarm-only agent would summarize as "1 xprompt"). This makes the Agents-tab XPROMPTS panel state which swarm launched
  the selected agent — a second user-visible payoff.
- `src/sase/ace/tui/modals/statistics_help_modal.py`, `_xprompt_methodology_text`: add a row explaining that swarm
  origins are attributed to every agent the swarm launched, and that attribution starts with runs launched after this
  change (no backfill).
- `src/sase/ace/tui/modals/statistics_pane_legends.py`: add the swarm kind if that legend enumerates kinds.
- Update the `?` help popup content per `src/sase/ace/CLAUDE.md` if any keybinding or option text changes (none is
  expected).

Tests: statistics-pane rendering tests for a swarm row and a swarm focus header; prompt-panel tests for the new glyph
and summary counting.

**Done when:** a swarm row reads `#research_swarm  swarm` in the table and focus header, and the Agents-tab panel shows
the originating swarm for a swarm-launched agent.

### Phase `docs-and-goldens` — floor bump, docs, snapshots, full check

**Depends on:** `tui-labels`.

- Bump the `sase-core-rs` floor in `pyproject.toml` to the release-plz-published version that contains the
  `core-swarm-kind` commit (follow the precedent of commit `e9b17a884`). If that version is not published yet, this
  phase waits on it — the floor bump is what guarantees users never see the `unknown` label.
- `CHANGELOG.md`: entry stating that xprompt swarms are now counted in Statistics → XPrompts and shown in the Agents-tab
  XPROMPTS panel, forward-only from this release.
- Refresh any documentation that describes what the XPrompts view counts (`docs/telemetry.md` and the statistics section
  of `docs/architecture.md` if they enumerate kinds or the counting contract).
- Regenerate the affected Statistics XPrompts PNG goldens with `--sase-update-visual-snapshots` after confirming each
  diff in `.pytest_cache/sase-visual/` is exactly the intended label/glyph change.
- Run `just install` then `just check` (this workspace is ephemeral, so the install comes first), plus
  `just test-visual`.

**Done when:** `just check` and `just test-visual` pass, the floor bump is in place, and docs plus CHANGELOG describe
the new counting contract.

## Verification

End-to-end, after the epic lands:

1. Launch a small swarm (e.g. `#sase/reads` or a throwaway two-segment swarm) and confirm every spawned child's
   `xprompts.json` contains the swarm record with `kind: "swarm"`.
2. `query_run_stats` over a window covering that launch returns a row for the swarm whose `runs` equals the number of
   spawned children, with partners listing the xprompts inside the body.
3. In `sase ace` → Admin Center → Statistics → XPrompts: the swarm row appears, is selectable in the focus picker, and
   its focus view shows models, projects, and partners.
4. Launch a nested swarm and confirm both names appear as separate rows.
5. Launch a plain `#gh …` prompt and confirm no `swarm` record is written.
6. From inside a swarm-launched agent, spawn a nested agent and confirm the nested agent's `xprompts.json` has no
   inherited swarm record.
