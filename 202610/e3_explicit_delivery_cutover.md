---
tier: epic
title: 'E3 (instruction migration): Explicit instruction delivery cutover'
goal: 'Every SASE root agent on Claude, Codex, Grok, and Muse (unless its trust spike
  fails) gets its memory-built instruction bundle exactly once through an explicit
  adapter channel, with SASE-owned native instruction files suppressed. Claude helpers
  get the helper bundle. Each provider switch has a per-provider sunset kill switch
  and an atomic per-invocation legacy fallback. A trustworthy scoreboard, `sase instructions
  show`, and a handoff-aware readout prove the result and support the soak.

  '
decisions:
  foreign_memory:
    ask: Turn off Claude auto-memory, Codex memories, and Grok memory v2 in SASE runs?
    default: true
    why: They vary by workspace and provider, and memory review never sees them
    answer: true
  home_layer:
    ask: Deliver the home memory layer to Grok and Muse too, not only to Claude and
      Codex?
    default: true
    why: Parity is the point of E3; the home layer is about 246 B of machine and vault
      pointers
    answer: true
  flag_soak:
    ask: How long must a provider's explicit delivery soak before its sunset flag
      may be removed?
    choices:
      seven_days: At least 7 days of clean readout and conformance; E4 can start about
        a week after E3
      fourteen_days: At least 14 days; more Grok and Muse samples, and E4 starts about
        two weeks later
    default: seven_days
    why: Matches the epics report; the readout shows each provider's explicit runs
      separately
    answer: seven_days
  decision_record:
    ask: Add the decisions:instructions-per-invocation record for this delivery architecture?
    memory:
    - decisions:instructions-per-invocation
    requested: I would like to implement the E3 epic described by the memory_built_instruction_migration_epics.md
      file in the research sidecar repo.
    default: true
    answer: true
phases:
- id: spikes
  title: Provider channel and suppression spikes, with outcomes recorded on the epic
  depends_on: []
  size: small
  description: 'spikes: run cheap direct CLI probes for Claude native exclusion, auto-memory,
    and the system-prompt files; Codex project-doc suppression, fresh shadow state,
    and fork children; Muse without workspace trust; and Grok memory v2. Record each
    outcome and CLI version as a structured note on the epic bead.'
- id: delivery-wire
  title: Instruction manifest wire v2 in sase-core (delivery slots and fallback),
    binding, adapter, pin move
  depends_on: []
  size: medium
  description: 'delivery-wire: bump the instruction manifest wire to v2. Add the subagent
    delivery slot, the delivery fallback block, and the channel and slot invariants.
    Keep v1 manifests readable. Update the binding, the Python adapter constants,
    the golden fixtures in both repos, and the sase-core pin.'
- id: run-discovery
  title: Scoreboard sees dismissed agents, and the new sase instructions show command
  depends_on: []
  size: medium
  description: 'run-discovery: make verify and render enumerate and resolve hidden
    (dismissed) agents. Fail loudly when an agent name resolves to nothing. Teach
    the manifest reader about helper manifests, and fix the per-purpose coverage denominator.
    Add `sase instructions show NAME`, which prints an agent''s recorded bundle and
    manifest and checks the stored bytes against the manifest digest.'
- id: readout
  title: Handoff-aware declaration and memory-read readout, with the pre-cutover baseline
    attached
  depends_on:
  - run-discovery
  size: medium
  description: 'readout: add `sase instructions readout`. Per provider and delivery
    mode, it classifies finished root runs as accepted, plan handoff, monitor handoff,
    not required, or miss, and joins memory reads per run. Reproduce the planning-time
    baseline, and attach the pre-cutover baseline JSON to the epic.'
- id: delivery-plan
  title: Delivery-plan hook, active delivery, atomic legacy fallback, and helper manifests
    at the boundary
  depends_on:
  - delivery-wire
  size: medium
  description: 'delivery-plan: add the `llm_instruction_delivery` hookspec and the
    DeliveryPlan model, with declared-legacy built-ins. The boundary resolves the
    plan, renders the root bundle and (when declared) the helper bundle, validates
    transport, writes v2 manifests, and exposes the active delivery to adapters. It
    records session ids and falls back atomically to legacy delivery with a deduplicated
    notification. Bundle files become 0644.'
- id: channel-counting
  title: Channel-aware, occurrence-exact scoreboard counting and the conformance check
  depends_on:
  - run-discovery
  - delivery-plan
  size: medium
  description: 'channel-counting: count layers across native and explicit channels
    by occurrence, not containment. Use what partial transcripts show. Add the bundle-marker
    column and helper marker checks, and associate sessions by marker and session
    id. Fix the Claude auto-memory and Muse double-attribution misreads. Add the `instructions.conformance`
    doctor check.'
- id: claude
  title: Claude root bundle via system-prompt file, helper bundle, exact-path native
    exclusion
  depends_on:
  - spikes
  - delivery-plan
  - channel-counting
  size: medium
  description: 'claude: create the `claude_explicit_instructions` sunset flag. When
    it is on, deliver the root bundle with `--append-system-prompt-file` and the helper
    bundle with `--append-subagent-system-prompt-file`. Exclude SASE-owned root and
    home instruction files by exact path, and turn auto-memory off per decision. Flag
    off restores E1''s after-state exactly.'
- id: codex
  title: Codex developer-instructions delivery, project-doc suppression, and shadow-home
    hygiene (folds sase-11s)
  depends_on:
  - spikes
  - delivery-plan
  - channel-counting
  size: medium
  description: 'codex: create the `codex_explicit_instructions` sunset flag. When
    it is on, deliver the whole root bundle through `developer_instructions`, add
    `project_doc_max_bytes=0` in SASE-managed projects only, and leave the shadow
    home without any SASE AGENTS.md. Rebuild the shadow home from an allowlist so
    shared sqlite state is never linked (closes sase-11s), and declare the subagent
    slot inherits_root.'
- id: grok-muse
  title: Grok full-bundle rules and Muse prompt-prefix delivery
  depends_on:
  - spikes
  - delivery-plan
  - channel-counting
  size: medium
  description: 'grok-muse: create the `grok_explicit_instructions` sunset flag, plus
    `muse_explicit_instructions` if the Muse spike passed. Grok''s `--rules` carries
    the full root bundle, and memory v2 is set per decision. Muse gets the bundle
    as its prompt prefix without `--trust-workspace`, or is declared legacy with no
    flag. Interrupt and wait reconstruction never nests a second bundle.'
- id: record
  title: Live acceptance probes, decision record, docs, and acceptance record
  depends_on:
  - readout
  - claude
  - codex
  - grok-muse
  size: medium
  description: 'record: prove every exit criterion with harness probes in both flag
    states (plus launch probes once the installed code has E3), the Claude helper
    probe, readout JSON, and parity in all four projects. Write the decision record,
    the docs, and the inventory update. Attach the acceptance record, with its notes
    for E4, to the epic.'
proposed_by: bbugyi200.apollo.research.13.linker.w0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-13 18:10:29
status: wip
bead_id: sase-1ko
---

- **PROMPT:** [prompts/202610/e3_explicit_delivery_cutover.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/e3_explicit_delivery_cutover.md)
- **BEAD:** [sase-1ko](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ko/README.md)

# Plan: E3 — Explicit instruction delivery cutover

## Context

### Where this comes from

- **Epic definition (primary input):**
  `research:202610/memory_built_instruction_migration_epics/memory_built_instruction_migration_epics.md`.
  Read these sections: "E3 Explicit delivery cutover", "The yardstick and what it should
  show after each epic", "Issues in the three prior reports" (issues 2, 4, 6, 7, 12,
  15), "Rust and Python boundary", and "Checklist for every epic plan". Read research
  with `sase artifact read`, never by opening files. **This plan is E3 only.**
- **Readiness review (amendments folded in here):**
  `research:202610/e3_delivery_cutover_readiness_and_bundle_inspection/e3_delivery_cutover_readiness_and_bundle_inspection.md`.
  It finds that the entry gates are met, and it requires six amendments, all adopted
  below:
  - make the scoreboard trustworthy before the provider phases;
  - replace the "re-render reproduces the hash" criterion;
  - account for Claude 2.1.296 `instructionFiles` and Codex multi-agent v2;
  - fold or sequence `sase-11s`;
  - create new sunset flags and an atomic fallback;
  - record the decisions.
- **R-delivery**
  (`research:202610/sase_md_instruction_delivery/sase_md_instruction_delivery.md`)
  defines the principles R1, R4, R6, R9, R10, and R14 that the decision record cites.
- **E1** (epic `sase-1gu`, closed) built:
  - `sase instructions verify` (the scoreboard);
  - the Grok `--rules` stopgap;
  - the Claude helper template plus the PreToolUse guard;
  - `docs/instruction_inventory.md`.

  Its after-scoreboard JSON is attached to `sase-1gu`.

- **E2** (epic `sase-1h3`, closed; `plan:202610/e2_instruction_bundles_shadow_mode.md`)
  built:
  - the manifest wire in sase-core;
  - the Python compiler and cache;
  - the shadow render at `invoke_with_instructions`;
  - `sase instructions render [-p]`;
  - the coverage and section diff.

  Its decisions 4–10 (facts, layers, section ids, layout, overlays, manifest v1,
  artifacts layout, cache) still hold. This plan only extends them.

- **Flags:** `sase-1jc.10` retired every E1/E2 flag (`grok_rules_delivery`,
  `claude_helper_channel`, `instruction_shadow_render`) on 2026-10-10. The stopgaps and
  the shadow render are now unconditional. **Do not revive those keys.**
- **Name collision:** closed epic `sase-18j` (failure triage) is also called "E3". This
  epic's title says "instruction migration" to keep them apart.

### Entry gates (verified by the readiness review, 2026-10-13)

| Gate                                    | Status                                                                         |
| --------------------------------------- | ------------------------------------------------------------------------------ |
| E1 landed, ≥3 days of runs, readout OK  | Met: 7 days. Grok misses fell from 14% to 5%; Claude helpers: 0 attempts       |
| E2 parity clean and shadow render works | Met: `render -p` exits 0; 804 manifests, 0 `*.error.json`, all three purposes  |
| Warm render within budget               | Met: warm p95 99–171 ms. Muse declaration-recovery p95 is 291 ms, but from n=5 |

### Planning-time baseline (handoff-aware; measured 2026-10-13 by the readiness lead)

The classes are: accepted submission, plan handoff, monitor handoff (with or without a
prepared declaration), declaration not required, and **miss** (neither a declaration nor
a recognized handoff). The table covers finished, non-research root runs. The post-E1
column is an upper bound. `readout` must reproduce this table and attach it as JSON.

| Provider | Sep 10 – Oct 5 misses | Oct 6 – Oct 13 misses |
| -------- | --------------------: | --------------------: |
| Claude   |            6/233 (3%) |            ≤2/72 (3%) |
| Codex    |            8/236 (3%) |           ≤3/123 (2%) |
| Grok     |          22/158 (14%) |            ≤4/80 (5%) |
| Muse     |           40/525 (8%) |          ≤22/245 (9%) |

Memory reads per run, Oct 7–13 (`memory_reads.jsonl` joined to manifest runs):

| Provider | Mean audited reads per run | Share of runs with ≥1 read |
| -------- | -------------------------: | -------------------------: |
| claude   |                       1.71 |                        70% |
| codex    |                       1.65 |                        89% |
| grok     |                       1.87 |                        93% |
| muse     |                       0.57 |                        35% |
| agy      |                       1.12 |                        69% |

The scoreboard today (`sase instructions verify --since 7d`) shows E1's after-state:

```
provider manifest contract home project directive native-full foreign          helpers
claude   7/7      2×       ✓    ✓       ✓         2×          auto-memory      0 attempts, 0 accepted
codex    25/25    2×       ✓    ✓       ✓         2×          —                0 spawns
muse     12/12    1×       ✗    ✓       ✓         1×          —                —
grok     22/22    1×       ✗    ✓       ✓         0           memory-v2½:False —
agy      2/2      ◌        ✗    ◌       ◌         ◌           —                —
```

### Facts verified while planning (2026-10-13; sase `bda93bab64`; sase-core `d30e1337` = pin)

- **CLI versions:** Claude Code 2.1.296, codex-cli 0.162.1, grok 1.0.52, Muse Code
  1.4.7.
- **Boundary.**
  `src/sase/llm_provider/_instruction_boundary.py::invoke_with_instructions` is the only
  root `provider.invoke` caller. The three root sites route through it:
  - `_invoke.py::invoke_agent`, purpose `ordinary`, which passes `provider_name` and
    `agent_type`;
  - `finalizers/declaration_recovery.py`, purpose `declaration_recovery`;
  - `finalizers/commit_repair_conflict.py`, purpose `conflict_repair`.

  The architecture test is `tests/instructions/test_invoke_boundary.py`.
  - The delivery dict is hard-coded: `status: "shadow"`, `channel: None`,
    `session_ids: []`, `provider_cli_version: None`, `parent_invocation_id: None`.
  - With no `artifacts_dir`, the boundary calls `provider.invoke` directly.
  - `_reserve_bundle_path` opens the `.md` with `os.open(path, O_CREAT|O_EXCL|O_WRONLY)`
    and no mode. Bundles come out **0775**; confirmed on disk.

- **Adapters today:**
  - **Claude** (`llm_provider/claude.py::_invoke_loop`):
    - It passes `--append-system-prompt _SINGLE_TURN_DIRECTIVE` and
      `--settings <helper_channel_settings_json()>` (the PreToolUse guard).
    - When `subagent_prompt_supported()` (a capability probe) passes, it adds
      `--append-subagent-system-prompt-file <packaged template>`.
    - A wait continuation uses `--resume` with the same session id. Claude's
      `--system-prompt-snapshot` (default `on`) reuses the first request's system prompt
      on resume.
    - An interrupt starts a **new** session (`active_session_uuid = None`) with the
      user's message.
  - **Codex** (`llm_provider/codex.py`):
    - It passes `-c developer_instructions=<json directive>`.
    - `_create_shadow_codex_home` copies `config.toml` and **symlinks every other
      entry** of the real `~/.codex`, including the sqlite state, `sessions/`,
      `memories/`, `memories_1.sqlite`, and `history.jsonl`. This is `sase-11s`.
    - `_link_home_agents_fallback` symlinks `~/AGENTS.md` into the shadow home. On
      apollo, `~/.codex` has no `AGENTS.md`.
    - Interrupts and stranded-handoff continuations rebuild the prompt from the original
      `prompt`.
  - **Grok** (`llm_provider/grok.py`):
    - `--rules _grok_rules_text(cwd)` is the directive plus the project root
      `AGENTS.md`, in SASE-managed projects only.
    - The 120 KiB argv guard is `_GROK_RULES_ARGV_BYTE_LIMIT`.
    - It never passes `--trust`; `agents_md_files` is always `[]`.
  - **Muse** (`llm_provider/muse_provider.py`):
    - `prompt = wrap_muse_prompt(prompt)` runs once at the top. Interrupt and wait
      reconstruction reuse that wrapped `prompt`.
    - Its argv includes `--trust-workspace` ("Load this workspace's skills and rules")
      and `--no-foreign-personal-context`.
    - Skills deploy globally to `~/.config/muse/skills`.
  - **agy** (prompt-prefix directive), **qwen**, **opencode** (no directive), and
    **fakey**.
- **Hooks.** `_hookspec.py` holds `firstresult` `sase_llm` hookspecs. There is no
  delivery hook and no `DeliveryPlan` anywhere. Built-ins and third-party providers both
  register through the `sase_llm` entry points, and `LLMPluginManager` delegates to
  `self._pm.hook.llm_*`.
- **Wire** (sase-core `crates/sase_core/src/instruction_manifest/wire.rs`, v1):
  - `DeliveryStatusWire` already has `explicit`, `inherits_native`, `inherits_root`, and
    `none`.
  - `DeliveryWire` has optional `channel: Option<String>`, `session_ids`,
    `parent_invocation_id`, and `provider_cli_version`.
  - Every struct is `deny_unknown_fields`.
  - `normalize.rs` validates nothing about delivery.
  - The Python mirror is `src/sase/core/instruction_manifest.py`.
- **Flags.** The registry (`src/sase/feature_flags/registry.py`) holds one flag
  (`artifacts_agent_pane_retired`, sunset).
  - Code checks a flag with `current_flags().enabled(FeatureFlag.<key>)`. Tests use
    `with override_flags(<key>=True|False):`.
  - `sase -F <key>` disables a flag for one process, and the `SASE_FEATURE_FLAGS` JSON
    env is the process transport. `sase flag disable` persists a choice per machine.
  - `project_is_sase_managed(cwd)` is in `src/sase/feature_flags/managed.py`.
- **Scoreboard gaps** (all confirmed in code):
  - **Dismissed runs are skipped.** `run_index.enumerate_runs` uses
    `listing_snapshot(...)`, which queries the artifact index with
    `include_hidden=False` and at most 200 recent completed runs. `verify -a` on a
    dismissed agent prints `runs=0` and exits 0; `render -a` says "no runs found".
    `sase agent show` resolves the same agent through
    `agent/names/_lookup_named.py::find_named_agent`, including dismissed bundles.
  - **Large transcripts blank the diff.** With `ROOT_TRANSCRIPT_BYTE_CAP = 512 KiB`, any
    larger transcript is `partial`. `coverage_diff.py` then zeroes every row.
  - **Containment, not occurrences.** `coverage_diff.py` counts with
    `sum(1 for text in flat_native if flat_body in text)`.
  - **Headline columns read native sources only.** Claude's are computed from
    instruction-file attachments, and `foreign = "auto-memory"` fires on
    `"Memory" in snapshot` (`instructions/claude.py`). Codex's come from the user-role
    AGENTS.md block. Muse's `_payload_texts` counts `rules_file` text in both the native
    and the explicit channel. Grok's already include `<human_rules>`.
  - **Time-only session matching.** Sessions are matched to manifests by provider and
    render time, with 5 s of skew.
  - **Per-purpose denominator.** `coverage_summary.py` uses
    `sessions=len(provider_sessions)` for every purpose.
  - There is no `sase instructions show`.
- **Claude native surfaces.**
  - `~/.claude/CLAUDE.md` does not exist. The home layer arrives through the
    **ancestor** `~/CLAUDE.md`, and the project layer through the workspace `CLAUDE.md`.
  - Nested `tools/`, `src/sase/ace/`, and `demos/tapes/` `AGENTS.md` files are
    hand-written (repo-owned). Their shims are projections. Claude loads them lazily,
    and they stay.
  - Linked repos under `sase/repos/**` carry repo-owned guides.
  - Claude 2.1.296 has `instructionFiles`, whose default `claude-md-or-agents-md` falls
    back to `AGENTS.md` when a directory has no `CLAUDE.md`. It also has
    `claudeMdExcludes`, `autoMemoryEnabled`, and `CLAUDE_CODE_DISABLE_AUTO_MEMORY`.
  - `--append-subagent-system-prompt-file` is still hidden from `--help`.
- **Remote dispatch and notifications.**
  - `%dispatch` runs `invoke_agent` on the target host, so the target renders with its
    own code. No instruction bytes cross the wire. `sase machine list` shows no remote
    machine, so this can only be tested, not probed.
  - Notifications use `sase.notifications.store.upsert_notification` with a dedup key
    and a `+1` note. See `src/sase/service/notifications.py` for the guarded pattern.
- **Run evidence.**
  - `memory_reads.jsonl` lives per project at
    `~/.sase/projects/<project>/memory_reads.jsonl`.
  - Run artifacts carry `agent_meta.json`, `done.json`, `final_context.json`,
    `final_submission.json`, `final_submission_attempts.jsonl`, `tool_calls.jsonl`, and
    `instructions/`.
- **Inventory.** The `job` project (an enabled, generated `AGENTS.md` plus 4 shims) is
  missing from `docs/instruction_inventory.md`.
- **Install semantics.** The installed `sase` is an **editable install of the primary
  checkout**, which auto-syncs from origin. A phase goes live for new agents on this
  host once it lands and the primary checkout syncs. Global installs are human-only.

## Design decisions (owned by this plan; phase workers do not re-litigate them)

1. **Exactly-once invariants** (they replace the absolutes in the research):
   - one full render per provider attempt, for the right actor;
   - no second full projection (no SASE-owned native full file alongside the bundle);
   - a resumed session inherits its own snapshot;
   - helpers are judged against the helper contract.

   An adapter whose slot is `explicit` must do three things together:
   - deliver the slot's delivered text once through its channel;
   - suppress SASE-owned native instruction files;
   - drop its legacy directive, because the bundle carries `pkg.provider.<p>`.

   **Without an active explicit delivery, an adapter behaves exactly as it does today**
   (E1's after-state). That is the only legacy path.

2. **Rust/Python split.**
   - sase-core owns the manifest wire: the v2 delivery slots, the fallback block, and
     the invariants. `delivery-wire` moves the pin once.
   - Python owns the delivery-plan model, adapter argv, channel id strings, and Markdown
     composition. This is unchanged from E2's split.
   - The decision record states the split and its reopen condition: a non-Python front
     end needs to preview a render.

3. **Delivery-plan hook.** `llm_instruction_delivery(self) -> dict | None`
   (`firstresult`). It returns `{"root": SLOT, "subagent": SLOT}`, where SLOT is
   `{"status", "channel"?, "max_bytes"?, "reason"?}`.
   - `status` is one of `explicit | inherits_native | inherits_root | none`. The root
     slot may only be `explicit` or `inherits_native`.
   - A provider with no hookimpl (unknown or plugin), a hook error, or an invalid return
     resolves to legacy: root `inherits_native`, subagent `inherits_native`, reason
     `provider_legacy`.
   - Each adapter's hookimpl checks its own sunset flag. With the flag off, it returns
     legacy with reason `flag_disabled`.

4. **Delivered text and the bundle marker.** The delivered text is the bundle bytes
   followed by one trailing marker line:
   `<!-- sase-instructions bundle=<sha256[:12]> invocation=<invocation_id> -->`.
   - The bundle `.md` and its sha256 stay exactly the compiler's bytes.
   - The marker is trailing so that every byte before it stays cacheable across
     sessions.
   - Helper delivered text carries the **helper** bundle's sha and the **root**
     invocation id.
   - The scoreboard counts markers to prove "bundle 1×" and associates a session with
     its exact invocation.

5. **Atomic fallback** (never an empty instruction context).
   - Each of these keeps the root slot on legacy delivery, with native discovery left
     intact, for that one invocation:
     - render failure (`.error.json` gains `"delivery_fallback": "legacy"`);
     - root bundle validation failure (not exactly one `SASE Final Declaration`, or the
       provider directive section missing): `invalid_bundle`;
     - delivered text over the slot's `max_bytes`: `oversize`;
     - adapter materialization failure: `delivery_failed`, reported by the adapter.
   - A failed helper render or validation sends helpers back to the E1 static helper
     template. **The root bundle is never substituted across actors.**
   - `render_failed`, `invalid_bundle`, `oversize`, and `delivery_failed` raise one
     notification, deduplicated per host, provider, reason, and day (`+1` on repeats).
     `provider_legacy` and `flag_disabled` are declared states and never notify.
   - No `artifacts_dir` means no plan and no active delivery: legacy, exactly as today.
     This covers the TUI fix-hook path.

6. **Per-provider channels and slots** (`channel` ids are shared literals):

   | Provider            | Root slot (flag on)                                                                                        | Subagent slot                                                                                             | Native suppression (flag on)                                                                                           |
   | ------------------- | ---------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
   | claude              | `explicit`, `claude.append_system_prompt_file`                                                             | `explicit`, `claude.append_subagent_system_prompt_file` (`none` + `unsupported_cli` when the probe fails) | exact-path `claudeMdExcludes` of SASE-owned root and home files (plus `instructionFiles` only if `spikes` requires it) |
   | codex               | `explicit`, `codex.developer_instructions`, `max_bytes` 32 KiB                                             | `inherits_root` (v1 children copy developer instructions; v2 forks inherit history)                       | `-c project_doc_max_bytes=0` in SASE-managed projects only; no SASE `AGENTS.md` in the shadow home                     |
   | grok                | `explicit`, `grok.rules`, `max_bytes` 120 KiB                                                              | `none` (child channel deferred)                                                                           | none needed (no `--trust`, so `agents_md_files: []`)                                                                   |
   | muse                | `explicit`, `muse.prompt_prefix` if `spikes` says trust can go; else `inherits_native` (`provider_legacy`) | `none`                                                                                                    | drop `--trust-workspace`                                                                                               |
   | agy, qwen, opencode | `inherits_native` (`provider_legacy`), declared until E4                                                   | `inherits_native`                                                                                         | none                                                                                                                   |
   - **Codex gets the whole root bundle through one channel**
     (`developer_instructions`), not the research's split across a shadow `AGENTS.md`
     and developer instructions. The split buys nothing for helpers: v1 children copy
     developer instructions and v2 forks inherit full history. One channel makes
     "exactly once" a single marker count and keeps the contract at developer priority.
   - **Claude moves from a user-message `CLAUDE.md` to the system prompt**
     (`--append-system-prompt-file`). This is an accepted change, recorded in the
     decision record:
     - the system-prompt snapshot survives `--resume`;
     - interrupts restart with the same argv;
     - compaction cannot summarize it away;
     - helpers get a parallel channel.

     The readout watches its effect.

   - **SASE-owned instruction files** (`sase_owned_instruction_paths(cwd)`, shared by
     Claude and Codex):
     - the project root's `AGENTS.md` and `CLAUDE.md`, when the project is SASE-managed;
     - `~/AGENTS.md` and `~/CLAUDE.md`, when they are SASE renders (they contain the E1
       contract fingerprint `SASE Final Declaration`).

     The set has exact paths only and never a glob. Nested repo-owned guides, linked
     repos under `sase/repos/**`, and hand-written home files are never in it.

7. **Flags.** Each adapter phase creates its own `sunset` flag with `sase flag new`:
   `claude_explicit_instructions`, `codex_explicit_instructions`,
   `grok_explicit_instructions`, and `muse_explicit_instructions`. The Muse flag exists
   only if Muse switches.
   - **On:** that provider's explicit slots and suppression. If `foreign_memory`
     applies, its foreign-memory switch is tied to this flag too.
   - **Off:** exactly E1's after-state for that provider, including the Grok `--rules`
     stopgap and the Claude helper template plus guard.
   - **`--remove-when`:**

     > [!decision] flag_soak = seven_days
     >
     > Use: "After at least 7 days of this provider's explicit runs on every execution
     > host:
     >
     > - `sase instructions readout` shows explicit-run misses no more than 2 points
     >   above the attached pre-cutover baseline;
     > - there are zero non-declared fallbacks;
     > - `instructions.conformance` is OK.
     >
     > Then delete the Off branch."

     > [!decision] flag_soak = fourteen_days
     >
     > Use the same text with "at least 14 days".

   - Keep the default `remove_by` thresholds.
   - Rollback is `sase flag disable <key>` on each execution host.
   - No `beta` flags.

8. **Foreign memory.**

   > [!decision] foreign_memory
   >
   > With the provider's flag on, turn off each provider's own memory channel in SASE
   > runs:
   >
   > - **Claude:** auto-memory off, using whichever switch `spikes` shows removes the
   >   auto-memory section from `prompt_snapshot` (`autoMemoryEnabled: false` in the
   >   per-run `--settings`, or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` in the subprocess
   >   env).
   > - **Codex:** the shadow home omits `memories/`.
   > - **Grok:** memory v2 off through the per-run switch `spikes` finds. If none
   >   exists, leave it on, report it in the foreign column, and add a
   >   `PROPOSED FOLLOW-UP`.
   >
   > `claude` records a `PROPOSED FOLLOW-UP` to harvest the two remaining non-empty
   > auto-memory directories (5 files, none newer than Sep 25) as one `memory` task
   > bead.

   > [!decision] foreign_memory = no
   >
   > Leave every foreign memory channel as it is today. The scoreboard keeps reporting
   > them in the foreign column. Codex's shadow home still never links
   > `memories_1.sqlite` (`sase-11s`), but `memories/` stays linked.

9. **Home layer.**

   > [!decision] home_layer
   >
   > Every explicit provider gets the same bundle, home layer included. Codex and Grok
   > renders differ only in `pkg.provider.*` and share `common_digest`.

   > [!decision] home_layer = no
   >
   > - `delivery-wire` adds section reason `privacy`.
   > - `delivery-plan` makes the compiler exclude `home.*` sections with reason
   >   `privacy` when `facts.provider` is not `claude` or `codex`.
   > - Grok and Muse then show home `✗` by declaration.
   > - Cross-provider diff checks compare claude with codex.

10. **Manifests after E3.**
    - Root manifests record:
      - `delivery.status` (`explicit`, or `inherits_native` plus `fallback`);
      - `channel`;
      - `provider_cli_version` (from a cached probe, one subprocess per executable
        fingerprint; `null` on failure);
      - `session_ids` (registered by the adapter, written back after the call);
      - the `subagent` slot.
    - `shadow` is no longer written. Readers keep accepting it for E2-era runs.
    - A Claude helper bundle gets its own manifest, `NN-<provider>.helper.json` and
      `.helper.md`. It has `facts.actor = native_helper`, status `explicit`, and
      `parent_invocation_id` set to the root invocation id.
    - The manifest is written before the call. It is atomically rewritten after the call
      (re-normalized) to add session ids and any adapter-reported fallback. A killed
      process keeps the pre-call manifest.

11. **The historical re-render criterion is replaced.** `render -a` compiles today's
    sources and is not a history replay. E3 proves two things instead:
    - **archived bytes:** `sase instructions show NAME` checks that the stored `.md` (or
      the content-store copy) matches the manifest's `bundle.sha256`;
    - **determinism:** the existing identical-inputs test.

12. **CLI shape** (per `cli_rules`; options sorted, every long option has a short
    alias).
    - New `sase instructions show NAME` and `sase instructions readout`.
    - `verify` gains a `bundle` column and a `delivery` column.
    - New deep doctor check `instructions.conformance`; `instructions.delivery` becomes
      manifest-aware.
    - No `sase doctor <subcommand>`.

13. **Shared literals** (parallel phases use these exact strings):

    | Literal                   | Value                                                                                                                                               | Owner              |
    | ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
    | Hook                      | `llm_instruction_delivery`                                                                                                                          | `delivery-plan`    |
    | Active-delivery accessor  | `sase.llm_provider.instruction_delivery.current_instruction_delivery()`                                                                             | `delivery-plan`    |
    | Marker line               | `<!-- sase-instructions bundle=<12 hex> invocation=<uuid> -->`                                                                                      | `delivery-plan`    |
    | Helper manifest / bundle  | `NN-<provider>.helper.json` / `NN-<provider>.helper.md` (same `NN` as its root)                                                                     | `delivery-plan`    |
    | Channel ids               | `claude.append_system_prompt_file`, `claude.append_subagent_system_prompt_file`, `codex.developer_instructions`, `grok.rules`, `muse.prompt_prefix` | `delivery-plan`    |
    | Fallback reasons          | `provider_legacy` `flag_disabled` `render_failed` `invalid_bundle` `oversize` `unsupported_cli` `delivery_failed`                                   | `delivery-wire`    |
    | Flag keys                 | `claude_explicit_instructions` `codex_explicit_instructions` `grok_explicit_instructions` `muse_explicit_instructions`                              | provider phases    |
    | Doctor check              | `instructions.conformance`                                                                                                                          | `channel-counting` |
    | Spike results note prefix | `E3 SPIKE RESULTS:` (on the epic bead)                                                                                                              | `spikes`           |
    | Existing                  | `SASE Final Declaration` (contract), `# SASE Helper Instructions` (helper), `SASE_INSTRUCTIONS_FILE`                                                | E1/E2              |

14. **Memory edits:** only the decision record (decision `decision_record`), authored in
    `record`.
    - No other phase creates, edits, or deletes memory. The contract template is not
      edited.
    - Phase workers record discovered work as `PROPOSED FOLLOW-UP:` notes and create no
      beads. The exceptions are the flag beads that `sase flag new` creates. `codex`
      also closes `sase-11s`.
15. **Out of bounds.** No phase edits `~/AGENTS.md`, `~/CLAUDE.md`, the chezmoi source,
    or another SASE project's files. The other projects (bob-cli, actstat, and job) are
    opened read-only with `sase repo open`. The only other repo that changes is the
    linked `sase-core`, in `delivery-wire`.

## Phase details

### spikes (small; xsmall-cost probes)

Run cheap, direct CLI probes (no SASE agent launch) on the cheapest model per provider.
Build fixture trees under a fresh directory beneath `~/.cache/sase/e3-spikes/`, so the
real `~/CLAUDE.md` and `~/AGENTS.md` stay ancestors. Do not change any user config file.

- **Claude**:
  - (a) Pass exact-path `claudeMdExcludes` in `--settings` JSON, covering the fixture
    workspace's `CLAUDE.md` and `AGENTS.md` plus `~/CLAUDE.md` and `~/AGENTS.md`.
    Confirm the transcript's `instructions` attachments show none of them.
  - (b) Excluding a `CLAUDE.md` must not trigger the default `instructionFiles`
    `AGENTS.md` fallback. If it does, find the smallest setting that stops it
    (`"claude-md"`) and confirm it keeps (c).
  - (c) A nested `sase/repos/linked/<x>/CLAUDE.md` (with `@AGENTS.md` import) still
    loads when the probe reads a file there. A nested repo-owned `tools/AGENTS.md` still
    loads where it does today.
  - (d) `--append-system-prompt-file` plus `--append-subagent-system-prompt-file` with
    20 KB files: both texts appear in `prompt_snapshot` (root) and in the helper
    transcript's system prompt, and a `--resume` reuses the snapshot.
  - (e) `autoMemoryEnabled: false` versus `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`: which one
    removes the auto-memory section. Record its exact fingerprint text for
    `channel-counting`.
- **Codex**:
  - (a) Fixture with `codex debug prompt-input`: in a SASE-managed tmp repo,
    `-c project_doc_max_bytes=0` drops the project doc, while `developer_instructions`
    (a 20 KB bundle) still appears. In a non-managed repo with its own `AGENTS.md`, the
    doc loads when the option is absent. Commit nothing; save the outputs for `codex`'s
    regression test.
  - (b) A real `codex exec` with an allowlist shadow home (no sqlite state,
    `history.jsonl`, `memories*`, or `AGENTS.md`; `sessions/` still linked) succeeds,
    writes its rollout under the real `~/.codex/sessions`, and logs no "stale rollout
    path" errors.
  - (c) If cheap, check what a `fork_turns: "none"` multi-agent v2 child receives.
    Otherwise record it as unprobed.
- **Muse:** `muse exec` **without** `--trust-workspace` (keeping `--disable-approval`,
  `--disable-sandbox`, and `--no-foreign-personal-context`) in a fixture workspace with
  an `AGENTS.md`:
  - project rules are **not** loaded (`session.jsonl` has no `rules_file`);
  - globally deployed skills are still listed and usable;
  - shell and file edits still work.

  Any failure means Muse stays declared legacy.

- **Grok:** find a per-run switch for memory v2 (env var, CLI option, or per-run config
  override; never an edit to `~/.grok/config.toml`) that makes `prompt_context.json`
  report it off. Do not re-probe `--rules`; E1 already proved it lands in
  `system_prompt.txt`.
- **Output.** Append one note to the **epic bead**, starting `E3 SPIKE RESULTS:`. For
  each question, give the CLI version, the outcome (`pass`, `fail`, or `unprobed`), and
  the exact setting to use. Also attach the raw evidence with
  `sase bead attach <epic-id> …`. Downstream phases read it with
  `sase bead read <epic-id>`. Honest `fail`/`unprobed` outcomes are acceptable; the
  provider phases have fallbacks for them.
- **Done when:** the note exists, and every question has an outcome. No repo change is
  required. If you add a fixture for later phases, put it under
  `tests/instructions/fixtures/`.

### delivery-wire (medium; crosses sase-core)

- **Open sase-core** with `sase repo open sase-core -r "<why>"`, read its `AGENTS.md`,
  and work only in the printed path. This phase has commit obligations in both repos.
- **Wire v2** (`crates/sase_core/src/instruction_manifest/`):
  - `INSTRUCTION_MANIFEST_WIRE_SCHEMA_VERSION = 2`. Readers accept `1..=2`. Writers
    emit 2.
  - Add `DeliveryFallbackWire { reason: FallbackReasonWire, detail: Option<String> }`.
    The closed reasons are in decision 13.
  - Add
    `DeliverySubagentWire { status: DeliveryStatusWire, channel: Option<String>, bundle_sha256: Option<String>, fallback: Option<DeliveryFallbackWire> }`.
  - Add `DeliveryWire.subagent: Option<DeliverySubagentWire>` and
    `DeliveryWire.fallback: Option<DeliveryFallbackWire>`. Both are
    `#[serde(default, skip_serializing_if = "Option::is_none")]`, and every new struct
    is `deny_unknown_fields`.
  - If `home_layer = no`, add section reason `privacy`.
- **New invariants in `normalize`:**
  - `channel`, when present, matches `^[a-z0-9_]+\.[a-z0-9_]+$`.
  - Status `explicit` requires a `channel` and forbids `fallback`.
  - `fallback` requires status `inherits_native`.
  - `subagent` is allowed only when `facts.actor = sase_root` and the status is
    `explicit` or `inherits_native`. Its status is never `preview` or `shadow`.
  - A subagent with status `explicit` requires a `channel` and a 64-hex `bundle_sha256`.
    Any other subagent status forbids `bundle_sha256`.
  - `facts.actor = native_helper` with status `explicit` requires
    `parent_invocation_id`.
  - A v1 document carrying a v2-only field is rejected. v1 documents otherwise normalize
    unchanged.
- **Fixtures:**
  - valid v2 manifests: root explicit with an explicit subagent, Codex root with an
    `inherits_root` subagent, helper explicit, and legacy with fallback;
  - one invalid fixture per new invariant;
  - the v1 golden still normalizes byte-identically.
- **Cross-language golden.** Store `instruction_manifest_v2.json` byte-identically in
  both repos, and extend the sase parity test. Keep the v1 golden.
- **Python adapter** (`src/sase/core/instruction_manifest.py`): mirror the schema
  version, `INSTRUCTION_MANIFEST_FALLBACK_REASONS`, and the subagent status tuple. Test:
  - the version equals the binding's;
  - both goldens normalize unchanged;
  - each new invariant is rejected with its message;
  - the E2 shadow writer still produces valid (v2, `shadow`) manifests.
- **Docs:** `docs/rust_backend.md` rows.
- **Verify:** `sase tool run check` in both repos. The sase-core change is a breaking
  `feat!:`. Commit both repos in this phase's declaration, so the host moves
  `sase-core-revision.txt`. Then confirm that `tools/check_sase_core_rs_bindings`
  passes; if the pin did not move, run `just ratchet-core-revision`.

### run-discovery (medium)

- **Archived runs.**
  - `run_index.enumerate_runs` includes hidden (dismissed) runs. Query the artifact
    index with `include_hidden=True`, and with full history when `--since` reaches past
    the recent window. Extend `listing_snapshot` with a parameter, or query the index
    directly.
  - Stay bounded: there must be no unbounded filesystem walk on default paths (doctor,
    `verify` without `-a`).
- **Agent resolution.** `-a NAME` (`verify` and `render`) and the new `show` resolve
  through `find_named_agent`, the same lookup `sase agent show` uses, so dismissed
  bundles and Agents-tab member names work.
  - An explicitly named agent is not dropped by the `--since` window.
  - A name that resolves to nothing prints `unknown agent NAME` and exits 2.
- **Reader** (this phase owns it). `instructions/manifests.py::read_run_manifests` also
  returns helper manifests (`NN-<provider>.helper.json`, decision 13), flagged by
  `facts.actor` and linked to their root by `parent_invocation_id`. It accepts v1 and v2
  documents. Helper fixtures can be v1-valid (`native_helper`, `explicit`), so this does
  not wait for `delivery-wire`.
- **Per-purpose coverage.** In `coverage_summary.py`, each purpose's denominator counts
  only the sessions whose matched manifest has that purpose. Ordinary sessions must no
  longer show as uncovered for `declaration_recovery` or `conflict_repair`.
- **`sase instructions show NAME`** (`parser_instructions.py`,
  `instructions_handler.py`, `RawDescriptionHelpFormatter` with examples). Options,
  alphabetical:
  - `-H/--helper`: show the helper bundle (`NN-<provider>.helper.*`) of the chosen
    invocation.
  - `-j/--json`: print the manifest JSON.
  - `-l/--list`: one row per recorded invocation, root and helper:
    - `NN`, provider, purpose, attempt, actor;
    - `delivery.status` and channel;
    - sha prefix, `common_digest` prefix, bytes, `tokens_est`;
    - integrity.
  - `-n/--seq NN`: choose an invocation. The default is
    `agent_meta.json["instructions"]["latest"]`.
  - `-s/--sections`: reuse `render -s`'s section table.
  - **Default output:** the bundle bytes verbatim on stdout, plus one stderr summary
    line.
  - **Integrity:** sha256 the run copy. If the run copy is missing, use
    `bundle.store_path`. Compare with `bundle.sha256` and report `bytes match` or
    `MISMATCH` (exit 1). Runs with no `instructions/` print a clear note and exit 2.
- **Docs and plumbing:**
  - `docs/instruction_bundles.md` "Inspecting your instructions": `show` replaces the
    manual `jq` recipe.
  - Rows in `docs/cli.md` and `docs/configuration.md`.
  - Completion for agent names; refresh with `just sync-completion-spec`.
  - Update `tests/main/test_parser_command_help.py`.
- **Tests:**
  - a dismissed fixture agent is found by `verify -a`, `render -a`, and `show`;
  - an unknown name exits 2;
  - the purpose denominators;
  - `show`'s integrity pass, mismatch, and store fallback;
  - `-l` with root and helper manifests, written by hand from decision 13's names;
  - every existing scoreboard fixture test stays green.
- **Done when:** `sase tool run check` passes, and
  `sase instructions show <a dismissed agent>` works against real data.

### readout (medium; depends on run-discovery)

- **`sase instructions readout`.** Options, alphabetical:
  - `-j/--json`;
  - `-p/--provider` (repeatable);
  - `-s/--since` (default `7d`);
  - `-u/--until`.

  It is explicit-only: no doctor or TUI path calls it.

- **Population:** finished, provider-backed root runs in the window, from
  run-discovery's enumeration with hidden runs included.
  - Monitor and gate turns are excluded.
  - Research runs are reported in their own rows.
  - Still-running runs are excluded.
  - Dismissed runs with no `done.json` count as `unknown`.
- **Delivery mode per run:**
  - `explicit` when its ordinary root manifest has `delivery.status: explicit`;
  - `legacy` for `inherits_native` or `shadow`;
  - `none` for pre-E2 runs.

  It also prints each provider's first explicit `rendered_at`; the watch starts there.

- **Classes** (per run, first match wins):
  1. accepted submission;
  2. plan handoff;
  3. monitor handoff with a prepared declaration (`sase final prepare` followed by
     `sase monitor start`);
  4. plain monitor handoff;
  5. other recognized handoff (pipe, questions, handoff);
  6. declaration not required (from `final_context.json`);
  7. **miss**.

  Read only small files, plus a bounded tail scan of `tool_calls.jsonl`. Misses are an
  upper bound and are labelled `≤`.

- **Memory reads:** join `~/.sase/projects/<project>/memory_reads.jsonl` rows to runs by
  agent identity and time. Report the mean audited reads per run and the share of runs
  with at least 1 read.
- **Output:** one row per provider × mode, with counts and percentages per class, plus
  the memory columns. The JSON has `schema_version: 1`.
- **Validation:** run it for 2026-09-10..2026-10-05 and 2026-10-06..2026-10-13.
  - Misses must match the planning-time baseline within ±2 points per provider.
  - Explain any larger gap in the phase note.
  - Attach both JSON files to the epic bead as `e3_readout_baseline_*.json`.
- **Tests:** synthetic run fixtures for every class (including the prepared-plus-monitor
  case that a naive check misses), the mode split, the memory join, and the window
  bounds.
- **Docs:** a "Soak readout" section in `docs/agent_providers.md` "Verifying instruction
  delivery", plus the `cli.md` and `configuration.md` rows and completion.
- **Done when:** `sase tool run check` passes, and the baseline JSON is attached.

### delivery-plan (medium; depends on delivery-wire)

- **Hook and model.**
  - Add the `llm_instruction_delivery` hookspec (decision 3) to `_hookspec.py`.
  - Add `LLMProvider.instruction_delivery()` and its `LLMPluginManager` delegation.
  - Add the public module `src/sase/llm_provider/instruction_delivery.py`:
    - frozen `SlotPlan` and `DeliveryPlan`;
    - `resolve_delivery_plan(provider)`, which validates, falls back to legacy, and
      never raises;
    - `delivered_text(bundle_text, sha256, invocation_id)` and the marker regex;
    - `sase_owned_instruction_paths(cwd)` (decision 6);
    - `ActiveInstructionDelivery`, with:
      - `root` and `subagent` slots, each with its delivered text and sha;
      - `invocation_id`;
      - `record_session(session_id)`;
      - `report_fallback(slot, reason, detail)`;
      - `materialize(slot) -> Path`, a context manager that writes a private 0600 file
        under SASE's managed tmpdir, removes it after the call, and on failure reports
        `delivery_failed` and returns `None`;
    - the contextvar accessor from decision 13.
  - Any object without `instruction_delivery` resolves to legacy, so test fakes keep
    working.
- **Declared-legacy hookimpls** for agy, qwen, and opencode (root and subagent
  `inherits_native`, reason `provider_legacy`). Also add legacy hookimpls for claude,
  codex, grok, and muse. Their phases flip them, so this phase changes no agent
  behavior.
- **Boundary rewrite** (`_instruction_boundary.py`), in this order:
  1. No `artifacts_dir`: invoke directly, with no plan and no active delivery.
  2. Resolve the plan.
  3. Render the root bundle (as today). On failure: write `.error.json` with
     `delivery_fallback`, notify, and invoke with no active delivery.
  4. If the subagent slot is `explicit`, render the helper bundle (facts
     `actor=native_helper`). Validate that it contains the helper heading and not the
     contract. On failure, use the subagent fallback (decision 5).
  5. Validate the root bundle for explicit delivery (decision 5) and check `max_bytes`
     against the delivered text. Degrade to legacy with a reason when needed.
  6. Write `NN-<provider>.md` (mode **0644**; also fix the 0775 bug for every path) and
     the v2 root manifest. If the helper is explicit, also write the helper `.md` and
     manifest. Update the `agent_meta.json["instructions"]` summary, which gains
     `delivery_status` and `helper_manifest`.
  7. Set the contextvar, then `provider.invoke`.
  8. In `finally`: reset the contextvar, atomically rewrite the manifest with
     `session_ids` and any adapter fallback, and restore `SASE_INSTRUCTIONS_FILE`.

  Every exception from steps 2–6 and 8 is caught, recorded, and the call proceeds
  through legacy delivery. `KeyboardInterrupt` and `SystemExit` propagate.

- **Supporting pieces:**
  - `provider_cli_version`: cached probe (decision 10). Reuse
    `sase.agent_clis.detect.probe_version` with the provider's install metadata.
  - **Notifications:** dedup key
    `instructions:fallback:<host>:<provider>:<reason>:<YYYY-MM-DD>`, with the
    state-write guard used by `service/notifications.py`.
  - **Reader:** `run-discovery` owns `read_run_manifests`. Write helper manifests with
    exactly the decision 13 names, so its reader picks them up unchanged.
  - **`home_layer = no`:** the compiler excludes `home.*` with reason `privacy` for
    providers other than claude and codex. Bump `COMPILER_VERSION`.
  - **Dispatch:** check that `%dispatch` launch requests do not forward
    `SASE_INSTRUCTIONS_FILE`. If they do, strip it. Add a test. Document that the target
    renders with its own code and that no instruction bytes cross the wire.
- **Route tests** (in-test fake providers whose `invoke` captures
  `current_instruction_delivery()`):
  - explicit root and helper: files, manifests, marker, contextvar, session-id
    writeback;
  - legacy hookimpl (`provider_legacy`);
  - no hookimpl or a raising hook;
  - render failure: `.error.json`, no active delivery, and one notification, deduped on
    repeat;
  - `invalid_bundle`;
  - `oversize`;
  - adapter-reported `delivery_failed` written back;
  - helper render failure: no cross-actor substitution;
  - no `artifacts_dir`;
  - declaration recovery, conflict repair, and fallback to another provider, each with
    the right `NN` and purpose;
  - file modes 0644;
  - the existing architecture test stays green.
- **Docs:** a "Delivery" section in `docs/instruction_bundles.md` covering:
  - the plan hook and slots;
  - the marker;
  - the helper manifests;
  - the fallback and notifications;
  - the manifest v2 fields;
  - the adapter contract (decision 1), written for plugin authors;
  - a `docs/plugins.md` pointer.
- **Done when:** `sase tool run check` passes. One local smoke invocation through a fake
  explicit provider writes a v2 manifest with a channel and a session id; record its
  path on the phase bead.

### channel-counting (medium; depends on run-discovery, delivery-plan)

- **Explicit channel sources per provider** (in
  `instructions/{claude,codex,grok,muse}.py`):
  - Claude: `prompt_snapshot.systemPrompt`.
  - Codex: developer messages.
  - Grok: `<human_rules>`.
  - Muse: the prompt prefix before `--- User Prompt ---`.

  The headline `contract`, `home`, and `project` columns count over native and explicit
  sources combined. `native-full` stays native-only. JSON adds per-column
  `channels: {native, explicit}`.
  - Fix Muse's double attribution: `rules_file` text is native only.
  - Grok's `home` is no longer hard-coded false; it is detected like the other layers.
  - A native Codex global `AGENTS.md` block that is not a SASE render shows as
    `foreign: codex-agents`.

- **Occurrences, not containment:** count non-overlapping occurrences of the flattened
  section body (and of the contract heading) in each source, then sum.
- **Partial transcripts:** count what the prefix shows. A row is `◌` only when it is not
  found **and** the read was partial.
- **Bundle marker:**
  - New `bundle` column: the number of markers carrying the matched manifest's
    invocation id in the session.
  - It shows `—` for legacy manifests and `◌` when unobservable.
  - A new `delivery` column shows `explicit`, `legacy:<reason>`, or `shadow`.
- **Session association**, in order:
  1. the marker's invocation id;
  2. `delivery.session_ids`;
  3. today's provider-and-time match.
- **Helpers** (Claude `subagents/agent-*.jsonl`): per helper, `helper_bundle` (the
  helper manifest's marker is present) and `root_bundle_in_helper` (the count of root
  markers, which must be 0). The helpers label shows both.
- **Claude foreign detection:** replace `"Memory" in snapshot` with the exact
  auto-memory fingerprint recorded by `spikes`, so a bundle in the system prompt never
  reads as auto-memory.
- **Doctor:**
  - `instructions.delivery` judges explicit sessions against explicit expectations:
    - bundle 1×, contract 1×, project ✓, directive ✓, native-full 0;
    - home ✓ unless the manifest excludes `home.*`.

    Legacy sessions keep E1's rules, and agy is still skipped.

  - New deep check `instructions.conformance`: newest 10 runs per provider over 24 h
    that have explicit manifests. It WARNs, naming the runs, when:
    - a root session's marker count for its invocation is not 1;
    - a helper lacks its helper marker or carries a root marker;
    - any manifest has a non-declared fallback (`render_failed`, `invalid_bundle`,
      `oversize`, or `delivery_failed`).

    It SKIPs when there are no explicit manifests.

  - Add the id to the deep set in `tests/main/test_doctor_command.py`.

- **Fixtures** (written with the real `delivered_text` and manifest writers):
  - an explicit session per provider: Claude system prompt, Codex developer message,
    Grok rules, Muse prefix;
  - a doubled bundle (2×);
  - a transcript over 512 KiB with instructions in the prefix;
  - Claude helpers with a helper marker, and one with a leaked root marker;
  - a legacy session that still shows E1's numbers.

  E1's baseline and after-state fixture tests must still pass. Additive JSON keeps
  `schema_version: 1`.

- **Docs:** `docs/agent_providers.md` "Verifying instruction delivery" (the new columns,
  conformance, `◌` semantics).
- **Done when:** `sase tool run check` passes.

### claude (medium; depends on spikes, delivery-plan, channel-counting)

Read `E3 SPIKE RESULTS:` on the epic first.

- **Flag:** create `claude_explicit_instructions` with
  `sase flag new -k sunset --when-enabled … --when-disabled … --remove-when …` (decision
  7), and paste the registry entry. The hookimpl returns:
  - **On:** root `explicit` with `claude.append_system_prompt_file`; subagent `explicit`
    with `claude.append_subagent_system_prompt_file`, or `none` with `unsupported_cli`
    when `subagent_prompt_supported()` is false.
  - **Off:** legacy, with `flag_disabled`.
- **With an active explicit delivery:**
  - `materialize` the root text and pass `--append-system-prompt-file <path>` instead of
    `--append-system-prompt _SINGLE_TURN_DIRECTIVE`.
  - `materialize` the helper text and pass `--append-subagent-system-prompt-file <path>`
    instead of the packaged template.
  - Keep the PreToolUse guard unconditionally.
  - Build the per-run `--settings` JSON as the guard hooks, plus `claudeMdExcludes` from
    `sase_owned_instruction_paths(cwd)`, plus whatever `spikes` required
    (`instructionFiles`). With `foreign_memory`, also add the auto-memory switch.
  - `record_session` for every session id started, both the initial one and each
    interrupt restart. A `--resume` reuses its id.
  - If either file cannot be materialized, fall back to legacy for that slot
    (`delivery_failed`). A failed root slot also drops the excludes, so natives load.
- **Without an active delivery:** today's argv byte-for-byte. Prove it with a golden
  argv test.
- **Tests:**
  - argv in both flag states;
  - the exclude set: only exact SASE-owned paths; never `sase/repos/**`, a nested
    repo-owned guide, or a hand-written home file; no globs;
  - the no-plan path;
  - the unsupported subagent probe;
  - a materialization failure;
  - an interrupt restart re-sends the same delivered files;
  - temp files are removed after the call.
- **Docs:** the `docs/agent_providers.md` Claude section and "Native helpers" (helper
  bundle replaces the template; the guard stays).
- **Follow-up:** with `foreign_memory`, append
  `PROPOSED FOLLOW-UP: harvest the remaining Claude auto-memory notes — …` to this phase
  bead.
- **Done when:** `sase tool run check` passes. Run one local harness invocation (real
  `claude`, cheapest model, temp artifacts dir, through `invoke_with_instructions`) in
  each flag state, and record the observer rows on the phase bead: flag on shows bundle
  1× and native-full 0; flag off shows E1's 2×.

### codex (medium; depends on spikes, delivery-plan, channel-counting)

Read `E3 SPIKE RESULTS:` first. This phase folds `sase-11s`.

- **Flag:** create `codex_explicit_instructions`, a sunset flag. The hookimpl returns:
  - **On:** root `explicit` with `codex.developer_instructions` and `max_bytes` 32 KiB;
    subagent `inherits_root`.
  - **Off:** legacy, with `flag_disabled`.
- **Shadow home** (both flag states; this fixes `sase-11s`). Rebuild
  `_create_shadow_codex_home` from an allowlist:
  - **Copy** `config.toml`.
  - **Symlink** only read-mostly entries Codex needs: `auth.json`, `skills`, `plugins`,
    `cache`, `installation_id`, `version.json`, `hooks.json`, `models_cache.json`, plus
    any others `spikes` showed are required.
  - **Keep `sessions/` linked**, so rollouts stay under the real `~/.codex/sessions`
    that the scoreboard reads.
  - **Never link** `*.sqlite*`, `history.jsonl`, `log`, `tmp`, or `thread-writer-locks`.
  - Never write through a symlink.
  - `memories/` is linked only with the flag off or when `foreign_memory` is no.
  - `AGENTS.md` and `AGENTS.override.md` depend on the flag state (below).
- **Flag on:**
  - `-c developer_instructions=<json(delivered root text)>` replaces the directive-only
    value.
  - `-c project_doc_max_bytes=0` only when `project_is_sase_managed(cwd)`.
  - The shadow home contains no SASE `AGENTS.md`: skip `_link_home_agents_fallback`.
    Leave out a real `~/.codex/AGENTS.md` or `AGENTS.override.md` only when it is a
    symlink to, or a byte copy of, a path in `sase_owned_instruction_paths`. A
    user-owned file stays linked, and the scoreboard reports it as foreign.
  - Interrupt and stranded-handoff continuations keep the same argv.
  - `record_session` with the thread id parsed from the JSON stream's `thread.started`
    event, when present.
- **Flag off:** today's delivery. The directive goes in `developer_instructions`, a real
  `AGENTS.md` or `AGENTS.override.md` is linked as today, and otherwise the
  `~/AGENTS.md` link fallback applies. Project docs load. Only the state entries come
  from the new allowlist.
- **Tests:**
  - argv in both flag states;
  - managed vs non-managed projects (a foreign repo's `AGENTS.md` still loads);
  - the `codex debug prompt-input` fixture from `spikes` as a regression test, skipped
    when `codex` is absent;
  - shadow-home allowlist contents;
  - a fixture real home containing a regular `AGENTS.md` and a symlinked
    `AGENTS.override.md` is byte-identical after creation and teardown;
  - a stranded-handoff continuation keeps the bundle exactly once;
  - `oversize` falls back to legacy.
- **`sase-11s`:** close it with
  `sase bead close sase-11s --note "Fixed by E3 codex phase <phase bead id>: allowlist shadow CODEX_HOME"`.
- **Done when:** `sase tool run check` passes. Run one harness invocation per flag
  state, and record the observer rows: on shows bundle 1×, contract 1×, native-full 0;
  off shows E1's 2×. Confirm that no "stale rollout path" errors appear in the run's
  output.

### grok-muse (medium; depends on spikes, delivery-plan, channel-counting)

Read `E3 SPIKE RESULTS:` first.

- **Grok:**
  - Create the `grok_explicit_instructions` sunset flag.
  - **On:** root `explicit` with `grok.rules` and `max_bytes` 120 KiB; subagent `none`.
    `--rules <delivered root text>` replaces `_grok_rules_text`. With `foreign_memory`,
    apply the memory v2 switch from `spikes`; if none exists, add a
    `PROPOSED FOLLOW-UP`. `record_session` with the `--session-id` uuid of each cycle.
  - **Off:** E1's stopgap, unchanged.
  - Tests: argv in both states, interrupt reconstruction (exactly one bundle), and the
    `oversize` fallback.
- **Muse:**
  - **If `spikes` passed:** create the `muse_explicit_instructions` sunset flag.
    - **On:** root `explicit` with `muse.prompt_prefix`; subagent `none`.
      `prompt = <delivered root text> + "\n\n--- User Prompt ---\n" + prompt`, applied
      once at the top in place of `wrap_muse_prompt`. Drop `--trust-workspace`.
      `record_session` per cycle.
    - **Off:** today's directive prefix and trust.
  - **If `spikes` failed:** create no Muse flag, because a flag whose two states behave
    the same is not a flag. The hookimpl stays legacy (`provider_legacy`). Document Muse
    as declared legacy until E4 switches it atomically with its stub, and record the
    reason on the phase bead.
  - Tests: argv and prompt in both states, and that interrupt and wait-claim
    reconstruction contain exactly one marker and one `SASE Final Declaration`.
- **Docs:** the Grok "Instruction delivery" and Muse sections of
  `docs/agent_providers.md`.
- **Done when:** `sase tool run check` passes. Run one harness invocation per provider
  per flag state (Muse only if explicit), and record the observer rows on the phase
  bead.

### record (medium; depends on readout, claude, codex, grok-muse)

- **Precondition.** Find the primary checkout with `sase repo list`, and check with
  `git merge-base --is-ancestor` whether it contains every phase commit. Note the first
  explicit `rendered_at` per provider from live manifests, if any.
- **Harness probes** (always, from this workspace's venv; a throwaway script outside the
  repo, never committed):
  - For each of claude, codex, grok, and muse (if explicit), run one real cheapest-model
    invocation through `invoke_with_instructions` per flag state, toggled with
    `override_flags` in-process. Use a temp artifacts dir with a minimal
    `agent_meta.json`, and run from the workspace root so natives exist to suppress.
  - Locate each session through the manifest's `session_ids` or marker, and run the
    observers.
  - Flag on must show bundle 1×, contract 1×, home per `home_layer`, project ✓,
    directive ✓, and native-full 0.
  - Flag off must reproduce E1's after-state row for that provider.
- **Claude helper probe** (harness): the root spawns one general-purpose helper and one
  Explore helper, and each runs `sase final submit`.
  - Both attempts are denied by the guard.
  - Both helper transcripts carry the **helper** marker and no root marker.
  - The root manifest's subagent slot names the helper sha.
- **Launch probes**, only if the primary checkout has every commit: submit one
  LaunchApproval through `/sase_run` for xsmall `+sase` probes (claude, codex, grok,
  muse, plus a Claude helper probe). Then run
  `sase instructions verify --since <first explicit manifest>` and
  `verify -p claude -H`. If the commits are missing, record "launch probes pending
  install sync" as a declared gap. Never `sase flag disable` on the host.
- **Readout:** run `sase instructions readout -j` for the pre-cutover window (2026-10-06
  up to the first explicit manifest, or now) and, if available, the post-cutover window.
  Attach both JSON files.
- **Other checks:**
  - `sase instructions show <a dismissed agent>` reports `bytes match`.
  - `sase doctor -D -C instructions` runs and includes `instructions.conformance`.
  - `sase flag list` shows the four flags as sunset and on.
  - Muse declaration-recovery warm p95 from `verify -c` over a representative sample:
    accept it or report it.
- **Parity:** run `sase instructions render -p` in sase, and in read-only bob-cli,
  actstat, and **job** checkouts (`sase repo open <project> -r "<why>"`). `git status`
  must be identical before and after in each.
- **Decision record** (decision `decision_record`). Use `/sase_memory_write`, then add
  the strand `sase/memory/decisions/instructions-per-invocation.md`, titled
  "Instructions Are Rendered Per Invocation And Delivered By Adapters". Follow the
  existing strand format (Claim, Why with rejected alternatives, Cost, Reopens when). It
  records:
  - R1, R4, R6, R9, R10, and R14 (read R-delivery);
  - the Rust/Python split;
  - the home-layer and foreign-memory decisions as approved;
  - Claude's move to the system prompt;
  - the Codex single channel;
  - the sunset-flag removal condition.

  Link `[[decisions/helpers-return-roots-declare]]`,
  `[[decisions/adapters-normalize-harnesses]]`, `[[decisions/rust-core-required]]`, and
  `[[decisions/single-turn-agents]]`. Then run `sase memory init` (this regenerates the
  instruction files' decision roster) and confirm that `sase memory init --check` is
  clean.

- **Docs:**
  - `docs/agent_providers.md` "Instruction delivery": a per-provider channel, slot,
    suppression, and flag table; "Instruction double-load" is now resolved; rollback.
  - `docs/instruction_inventory.md`:
    - add the `job` project (generated memory-backed count 35 → 40);
    - mark the Codex shadow-home surface done in E3;
    - mark the root and home files as suppressed at runtime but still committed until
      E4/E5.
  - `docs/instruction_bundles.md`: link the decision record.
- **Acceptance record** (Markdown), attached to the epic with
  `sase bead attach <epic-id> <file> -N e3_acceptance_record.md -n "<summary>"`. It
  contains:
  - the sase and sase-core revisions and the pin;
  - CLI versions;
  - the commands run;
  - expected vs observed for every exit criterion;
  - the probe rows (both flag states);
  - the readout tables;
  - declared gaps;
  - **Notes for E4:**
    - Claude's default `instructionFiles` loads `AGENTS.md` once the `CLAUDE.md` shims
      are deleted;
    - the per-invocation legacy fallback depends on native files, so E4's
      last-known-good store must replace it before files go;
    - the no-`artifacts_dir` path is still legacy;
    - agy (and Muse, if legacy) still need a switch;
    - nested scopes still load natively.
- If a criterion fails because the scoreboard, `show`, or `readout` is wrong, fix it in
  this phase. Record any other failure as `DISCOVERED ISSUE:` on this phase bead, leave
  it open, and name the failed criterion.

## Exit criteria (checked at landing)

- [ ] `sase tool run check` passes in sase and in sase-core. `sase-core-revision.txt`
      includes the `delivery-wire` commit, and `tools/check_sase_core_rs_bindings`
      passes.
- [ ] The epic bead has an `E3 SPIKE RESULTS:` note, with a CLI version and an outcome
      for each spike question.
- [ ] Harness probes (and launch probes, if the installed code has E3), flag on: Claude,
      Codex, Grok, and Muse (unless declared legacy) each show bundle 1× (marker
      matching its manifest's invocation), contract 1×, home per `home_layer`, project
      ✓, directive ✓, and native-full 0. agy shows declared legacy.
- [ ] Claude helper probe: the general-purpose and Explore helper transcripts carry the
      helper bundle marker, never the root's. Their `sase final submit` attempts are
      denied, and the root manifest's subagent slot is `explicit` with the helper sha.
- [ ] Each flag off restores E1's after-state for that provider, with tests for both
      states plus one harness probe per provider. `sase flag list` shows one sunset flag
      per switched provider (four, or three if Muse stayed legacy), each with a flag
      bead and the `flag_soak` removal condition.
- [ ] Atomic-fallback tests pass for: render failure, invalid bundle, oversize, adapter
      delivery failure, unsupported Claude subagent flag, no artifacts dir, and an
      unknown or plugin provider. In each case, legacy delivery and native discovery are
      restored, the context is never empty, the notification is deduplicated, and the
      manifest or `.error.json` records the reason.
- [ ] Protected-surface tests pass: - only exact SASE-owned paths are excluded (no
      linked-repo, nested repo-owned, or hand-written home file); - a non-managed repo's
      `AGENTS.md` still loads for Codex; - a real `~/.codex/AGENTS.md` and
      `AGENTS.override.md` are byte-identical after a run; - the shadow home links no
      sqlite state (`sase-11s` closed).
- [ ] `sase instructions show <dismissed agent>` prints the bundle and reports
      `bytes match`. `verify -a` and `render -a` resolve dismissed agents, an unknown
      name exits 2, and transcripts over 512 KiB no longer blank the diff.
- [ ] `sase instructions readout` reproduces the planning-time baseline within ±2 points
      per provider, or explains the difference, and the baseline JSON is attached.
- [ ] `sase doctor -D -C instructions` includes `instructions.conformance`, and it is OK
      on the probe runs.
- [ ] Bundle files are 0644, and v2 manifests validate in Rust and Python. v1 E2
      manifests still read.
- [ ] Parity (`render -p`) passes live in sase, bob-cli, actstat, and job, with no
      tracked file changed in those projects.
- [ ] The `decisions:instructions-per-invocation` record exists, and
      `sase memory init --check` is clean.
- [ ] The acceptance record, including its notes for E4, is attached to the epic bead.

## Watch metrics (after landing; not landing criteria)

The watch starts at the first manifest with `delivery.status: explicit` on each
execution host, not at landing. It lasts at least the `flag_soak` window.

- `sase instructions readout --since <first explicit>`: per-provider explicit-run misses
  against the attached baseline. Muse (about 9%) is the provider to watch. Also compare
  memory reads per run, where Muse should gain most because it gains the home layer.
- Non-declared fallbacks and `.error.json` stay at 0.
- `instructions.conformance` stays OK.
- Claude helper `sase final` attempts and accepted helper declarations stay at 0.
- Codex spawn count (the `inherits_root` risk).
- Bundle bytes against Codex's 32 KiB slot cap.
- Claude behavior after the move to the system prompt (spot-check transcripts).

Then answer each flag bead in `FlagTriage`: **Remove**, or disable that provider with
`sase flag disable <key>` and fix its adapter.

## Confirm it yourself (about 10 minutes, after deployment)

```bash
sase instructions verify --since 24h            # bundle 1× · contract 1× · home ✓ · project ✓ · native-full 0
sase instructions verify -p claude -H           # helper bundle ids, no root ids, 0 accepted helper finals
sase instructions show <recent-agent> -l        # every invocation, delivery status, channel, bytes match
sase instructions readout --since 7d            # handoff-aware misses and memory reads, explicit vs legacy
sase flag list | grep explicit_instructions     # the four sunset kill switches
```

## Expected scoreboard after this epic

```
provider manifest bundle delivery  contract home project directive native-full foreign  helpers
claude   N/N      1×     explicit  1×       ✓    ✓       ✓         0           —        helper bundle ✓, root 0, 0 accepted
codex    N/N      1×     explicit  1×       ✓    ✓       ✓         0           —        inherits_root, 0 spawns
grok     N/N      1×     explicit  1×       ✓    ✓       ✓         0           off      —
muse     N/N      1×     explicit  1×       ✓    ✓       ✓         0           —        —            (or legacy:provider_legacy)
agy      N/N      —      legacy    ◌        ✗    ◌       ◌         ◌           —        —
```

Home shows `✗` for Grok and Muse if `home_layer = no`. Grok's foreign column shows
memory v2 if no switch exists or if `foreign_memory = no`.

## Rollback

- Run `sase flag disable <provider>_explicit_instructions` on each execution host
  (athena, apollo, mac). That provider returns to E1's after-state.
- A render or delivery failure already falls back per invocation.
- The `sase-11s` allowlist home applies in both flag states. Revert the `codex` commit
  to restore full linking.
- The wire v2 is additive for readers. The pin stays.

## Memory edits

- `decisions:instructions-per-invocation` (new strand), only if decision
  `decision_record` is yes. Authored in `record`, followed by `sase memory init`.
- Nothing else. The glossary rewrite and the new terms belong to E4.

## Out of scope

- Deleting or shrinking committed instruction files, `sase instructions export`/`sync`,
  the last-known-good store, retiring `capture_instruction_snapshot`, and switching agy
  (E4).
- Home files, other projects' files, and `sase tmux-agent` interactive delivery (E5).
- TUI surfaces: the MEMORY deck, chips, and ledger (E6). `show` is the only E6 piece
  pulled forward.
- `when:` audiences and budget gates (E7).
- The Grok, Muse, and agy child channels, and a Codex multi-agent v2 helper override
  (spike tasks after the watch).
