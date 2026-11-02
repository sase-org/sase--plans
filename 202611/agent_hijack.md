---
tier: epic
title: "Agent Hijack: stop a stalled agent, keep its work, recover in one keypress"
goal: "A user can hijack a stalled sase agent (,k, the kill dialog's h, or `sase agent
  hijack NAME`). Its process tree stops, but its workspace claim, Agents-tab node, and
  evidence stay. The node shows a ⏏ HIJACKED status, and <enter> opens a
  single-keypress recovery panel: escalate to a smarter model, continue, or instruct in
  the same workspace; restart or edit from a fresh workspace after a verified snapshot;
  view the diff, open a shell, or dismiss. Every path is durable, exactly-once, and
  fails closed so work is never silently lost.

  "
decisions:
  enter_default:
    ask: Which keep-work action should a second Enter (⏎⏎) run in the recovery panel?
    choices:
      escalate:
        Escalate (or Switch at the top rung); Continue only when no other model exists
      continue: Continue on the same model; climbing to a bigger model always needs u
    default: escalate
    why:
      Escalate is the feature's headline, and like Continue it never releases the
      workspace
    answer: escalate
  successor_context:
    ask:
      How much of the hijacked transcript should keep-work successors receive by
      default?
    choices:
      fork:
        "Inline the transcript via #fork; fall back to the brief when over the
        continuation budget"
      brief:
        Send only the bounded evidence brief plus the transcript path; cheaper, more
        re-exploration
    default: fork
    why: The successor re-explores least, and the budget fallback caps the cost
    answer: fork
  glossary_term:
    ask: Publish an Agent Hijack glossary strand when the feature lands?
    memory:
      - glossary:agent-hijack
    requested:
      An "agent hijack" (aka "hijack"--we should add a good new glossary term for this)
      is meant to describe the process of killing a sase agent without releasing its
      workspace or dismissing it.
    default: true
    answer: true
  decision_records:
    ask:
      Record the two hijack design decisions (never downgrade; custody held until
      resolved) as decision records?
    memory:
      - decisions:hijack-escalation-never-downgrades
      - decisions:hijack-holds-custody-until-resolved
    requested:
      Review the agent_hijack_held_workspace_recovery.md file in the research sidecar
      repo for context and inspiration before planning. I agree with all of the
      requirements recommended in that research file
    default: true
    answer: false
phases:
  - id: core_ledger
    title: "sase-core: hijack statuses, ledger wire, and eligibility"
    depends_on: []
    size: medium
    description:
      "core_ledger: in sase-core, add the HIJACKED/ESCALATED/CONTINUED status rows and
      the done-marker-to-status mapping, the AgentHijackRecordWire v1 state machine, the
      eligibility function, and the auto-restart exclusion. Then expose PyO3 bindings,
      bump sase's pin, and add the thin Python wrappers."
  - id: core_escalation
    title: "sase-core: escalation policy, continuation prompt, and config"
    depends_on: []
    size: medium
    description:
      "core_escalation: add the never-downgrade escalation_target function and the
      continuation-prompt builder in sase-core with bindings and a pin bump, plus the
      agent_hijack config section (escalation_ladder, successor_context), its schema,
      and its accessor in sase."
  - id: settle
    title: Runner settlement that keeps custody
    depends_on:
      - core_ledger
    size: medium
    description:
      "settle: add the Python ledger store, the hijack-mode user-kill intent, and the
      runner capability stamp. The runner learns to settle `outcome: hijacked` while
      holding its workspace claim: the SIGTERM handler, the killed iteration, the
      shutdown hold, done.json, and the finalize boundary."
  - id: stop
    title: The stop protocol and `sase agent hijack`
    depends_on:
      - settle
    size: medium
    description:
      "stop: create the agent_hijack beta flag. Implement the ordered stop orchestrator:
      eligibility, record and intent, pin, durable signal, fallback settlement,
      survivors, lost race, then evidence and a strict snapshot before `held`. Expose it
      through `sase ops agent persist-hijack` and `sase agent hijack`."
  - id: custody
    title: Every release path respects a hijack
    depends_on:
      - stop
    size: medium
    description:
      "custody: teach stale cleanup (with a TTL measured from hijacked_at and a snapshot
      before expiry release), kill persistence, auto-restart, late runner callbacks,
      waits, the stranded detector, and `sase agent list/show` about hijacked turns."
  - id: recover_keep
    title: Keep-the-work recovery (Escalate, Switch, Continue, Instruct)
    depends_on:
      - custody
      - core_escalation
    size: large
    description:
      "recover_keep: add the exactly-once resolution transaction, strict same-workspace
      successor launches with required inheritance, the hijack #fork frame, predecessor
      ESCALATED/CONTINUED settlement, crash reconciliation, the recovery preview API,
      and `sase agent recover` for escalate and continue."
  - id: recover_release
    title: Start-over and leave recovery (Restart, Edit backend, Dismiss)
    depends_on:
      - recover_keep
    size: medium
    description:
      "recover_release: add snapshot-verified restart and dismiss resolutions, restart
      with a replacement prompt for Edit, `sase agent restart` and `sase agent kill`
      awareness of HIJACKED targets, and preview availability reasons for those actions."
  - id: tui_render
    title: Agents-tab rendering and the ,k way in
    depends_on:
      - stop
    size: medium
    description:
      "tui_render: add the ⏏ HIJACKED row styling, the HIJACKING…/RECOVERING… overlays,
      the stop-failed badge, the workspace-held detail line, and footer chips. Add the
      flag-gated ,k leader key and the kill dialog's h choice, the hijack_agent keymap
      config, help entries, and visual snapshots."
  - id: tui_panel
    title: The recovery panel
    depends_on:
      - tui_render
      - recover_release
    size: medium
    description:
      "tui_panel: extend AgentActionChooserModal with a header, single-line sectioned
      rows, disabled rows with reasons, a pending preview state, and an Enter arming
      delay. Route Enter on HIJACKED to the panel and wire u/c/r/v/t/d plus x-to-dismiss
      through durable procs, with 80x24 visual snapshots."
  - id: tui_prompt
    title: Instruct and Edit through the prompt bar
    depends_on:
      - tui_panel
    size: medium
    description:
      "tui_prompt: wire panel keys i and e to the prompt bar. Instruct pre-fills the
      continuation prompt and submits through recover. Edit pre-fills the original
      prompt and releases nothing until submit, so cancelling either leaves the agent
      HIJACKED."
  - id: land
    title: Remove the flag, document, and prove it end to end
    depends_on:
      - custody
      - tui_prompt
    size: medium
    description:
      "land: run an end-to-end hijack-and-escalate test across primary and linked repos,
      delete the agent_hijack flag's Off branch and close its bead, update the reference
      docs, and publish the glossary strand and decision records per the memory
      decisions."
proposed_by: bbugyi200.apollo.research.33.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 17:42:06
status: wip
---

- **PROMPT:**
  [prompts/202611/agent_hijack.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/agent_hijack.md)

# Plan: Agent Hijack

## Context and source

This plan implements the recommendation in the consolidated research report. Every phase
worker should skim it with
`sase artifact read research:202611/agent_hijack_held_workspace_recovery/agent_hijack_held_workspace_recovery.md "<why>"`.
Read it for the reasoning; this plan is authoritative where the two differ. The user
accepted all twelve of the report's adjusted requirements (A1–A12). This plan adopts
them, settles the report's open questions, and corrects a few details that it got wrong
or that I verified differently against the code (listed under
[Deviations from the research](#deviations-from-the-research)).

**The one idea:** a hijack separates _stopping execution_ from _giving up the work_.
Today every stop path (`x`, `,x`, `sase agent restart`, auto-restart) releases the
workspace claim. A hijack is a durable settlement, `outcome: hijacked`. It keeps the
claim, the node, and the evidence until the user resolves it **exactly once**.

**Rust/Python boundary** (`rust_core_backend_boundary`): sase-core holds the shared
semantics: statuses and buckets, the ledger state machine, eligibility, escalation, and
the continuation prompt text. The CLI, TUI, web, and fleet must all agree on these.
Python holds effects: file I/O, locks, signals, the runner, launches, the CLI, and
Textual. Follow the existing auto-restart split. sase-core's `agent_auto_restart` module
is a pure state machine, and `src/sase/agent/auto_restart/ledger.py` does the I/O.

## User experience (the contract every phase serves)

### Way in

- **`,k`**: a new leader key, `hijack_agent`, unbound today ("kill, keep"). It hijacks
  the selected agent when it is eligible. Otherwise it toasts the refusal reason.
- **`x` on a running agent** keeps its meaning. `ConfirmKillModal` gains a third choice,
  **`h` Hijack instead · stop it but keep workspace #N**. The choice shows only when the
  agent is eligible and the flag is on.
- **`sase agent hijack NAME`**: the CLI.
- The row immediately shows **`HIJACKING…`**. It shows **`⏏ HIJACKED`** only after the
  process tree is verified dead and custody is confirmed. If any process survives, the
  row stays `RUNNING` with a red `⏏ stop failed` badge and a toast that lists the
  survivor pids. If the agent finished first, its real outcome is kept and a toast says
  "finished before the hijack".

### v1 eligibility (A11)

Hijack is available for local LLM agent turns that have a numbered workspace (>0) in
RUNNING, STARTING, or RETRYING, and whose runner stamped the hijack capability at
launch. It is refused, with a one-line reason, for these cases:

- FINALIZING (lifecycle `declaring`/`executing`): "finalizing; try again in a moment";
- monitor turns, gate turns, named procs, home mode, remote fleet rows (with a copyable
  `sase agent hijack NAME` hint for the owning host), clan containers, and bulk marks;
- runners launched before this feature (no capability stamp): "launched before hijack
  support; use x".

### Rendering on the Agents tab

- **`⏏ HIJACKED`** in `bold #FF5FAF`.
  - Fonts: ⏏ U+23CF is present in the pinned Fira Code Regular and Bold (verified).
  - Colour: `#FF5FAF` is not used by any agent status. It is the `fakey` test provider's
    name colour and one of eight session hues, which is acceptable.
  - Bucket: the Rust **Stopped** bucket. The row counts in the header as needing you,
    `,J` jumps to it, and it outranks Running in clan aggregation.
- **Expired hold:** `⏏ HIJACKED` followed by a dim `· workspace released`.
- **Detail header:** reuses the existing "Workspace held" line:
  `Workspace held: hijacked · ws #4 · 5 files +212 −31 · ⏎ recover`.
- **After a keep-work resolution**, the predecessor member settles to a grey
  **`ESCALATED`** if the successor's model differs, or **`CONTINUED`** if it does not.
  These are handoff-settled labels exactly like `ANSWERED`, and the session tree tells
  the story: `--1 ESCALATED claude/opus@high` → `--2 RUNNING claude/opus@xhigh`.
- **`RECOVERING…`** overlays the row while a resolution is in flight.
- **Footer** conditional chips, per the `ace` note:
  - `⏎ recover` and `x dismiss` on HIJACKED rows;
  - `,k hijack` in the leader footer when an eligible running agent is selected and the
    flag is on.
- Selection, scroll, and folds never jump. Rows repaint in place.

### The recovery panel

`<enter>` on a HIJACKED row opens this panel directly; hijack recovery outranks every
other Enter target. If the agent also has a Patch, the Patch goes last as `p`. All
glyphs below were verified present in the pinned Fira Code. The report's `⇄` and `▸` are
**not** in Fira Code, so this plan uses `↔` and the chooser's existing `›`.

```text
╭─ ⏏ Recover sase-7f.2 ──────────────────────────────── HIJACKED · ws #4 ─╮
│ claude/opus@high via @large · ran 47m · hijacked 3m ago                 │
│ 5 files +212 −31 · 1 linked repo · last tool: Bash just check (22m)     │
│ ⏎ again → Escalate · or press a key                                     │
│                                                                         │
│ KEEP THE WORK · next turn runs in workspace #4                          │
│ › [u] ⇧ Escalate   @large → @xlarge · claude/opus@xhigh                 │
│   [c] ▶ Continue   same model · claude/opus@high                        │
│   [i] ↪ Instruct   add guidance, then continue                          │
│ START OVER · snapshot first, then workspace #4 is released              │
│   [r] ⟳ Restart    same prompt, same name, fresh workspace              │
│   [e] ⌨ Edit       edit the prompt; nothing is released until submit    │
│ INSPECT OR LEAVE                                                        │
│   [v] ± View diff  changes in the pager · stays hijacked                │
│   [t] $ Shell      tmux in workspace #4 · stays hijacked                │
│   [d] ⌫ Dismiss    snapshot first, then workspace #4 is released        │
│                                                                         │
│ j/k move · ⏎ select · Esc decide later                                  │
╰─────────────────────────────────────────────────────────────────────────╯
```

At the top of the ladder, or when a refusal applies, rows change **in place** rather
than disappearing:

```text
│ › [u] ↔ Switch     already @xlarge → codex/gpt-6.1-sol@xhigh            │
│   [r] ⟳ Restart    unavailable: fan-out prompt; use Edit                │
```

**Design rules.**

- **Group by fate.** The rows are grouped by what happens to the work, which answers the
  only question that matters at that moment: does my diff survive? No row needs a
  warning label.
- **Show the target.** The concrete target model is always printed. An escalation
  candidate identical to the stalled model is skipped, so `u` never claims to be smarter
  while relaunching the same model.
- **Palette: one accent.**
  - `#FF5FAF` for the title glyph, the HIJACKED badge, and the keep-work glyphs;
  - `#8787AF` (the existing `STOPPED_COLOR`) for the start-over and leave glyphs;
  - the existing `#87D7FF` keycaps;
  - disabled rows entirely `dim`, with the reason in place of the detail.

  Red is reserved for the stop-failed error. No spinner remains after death is verified.

- **Single-line rows.** Label and detail share one line, unlike the existing two-line
  chooser rows, so the panel fits 80×24, scrolls in shorter terminals, and ellipsizes
  long names.
- **Keys.**
  - `u` means "up a size". `x` is deliberately unused inside the panel, because on the
    Agents tab `x` means destroy.
  - `j`, `k`, and `q` stay the chooser's. `p` is the Patch row.
  - A disabled row's key toasts its reason; the cursor can rest on it so the reason is
    readable.
- **The primary (`⏎⏎`)** is chosen by the `enter_default` decision below, and is never a
  release action.
- **No accidental actions.** Opening the panel never triggers an action: Enter is
  ignored for 300 ms after mount, which swallows key-repeat from the opening ⏎.
- **Pending preview.** Rows whose availability needs the preview (u, c, r) render
  `computing…` with inert keys until it arrives. The preview is computed off the event
  loop and cached per `(hijack_id, ledger revision)`. v, t, and d are live immediately.
- **`Esc`** leaves the agent HIJACKED ("decide later").

> [!decision] enter_default = escalate The first row, and therefore `⏎⏎`, is Escalate,
> or Switch at the top rung. When neither has a different model available, the first
> enabled keep-work row (Continue) becomes primary.

> [!decision] enter_default = continue Row order stays u, c, i, but the cursor and `⏎⏎`
> start on Continue (`initial_index` points at it). Escalate always needs its own `u`
> keypress.

### What each option does

- **`u` Escalate / Switch** (workspace kept)
  - Claims the resolution and reserves the previewed concrete target in the ledger. It
    then launches the next session turn in the **same** workspace through the strict
    follow-up path.
  - The prompt is the continuation prompt with `%model:<concrete target>`.
  - On failure, the agent stays HIJACKED with an error toast. Nothing ever degrades to a
    pool workspace or `#0`.
- **`c` Continue** (workspace kept)
  - Same as `u`, but pinned to the stalled run's own concrete `provider/model@effort`,
    never its alias, because a `|` pool could rotate.
- **`i` Instruct** (workspace kept)
  - Opens the prompt bar pre-filled with the continuation prompt, including its editable
    `%model:` line. The cursor lands on an empty guidance line at the top.
  - Submit routes through the same path as `c`. The model it lands on decides ESCALATED
    vs. CONTINUED.
  - Cancel leaves the agent HIJACKED.
- **`r` Restart** (workspace released)
  - Claims the resolution, verifies (or refreshes) the recovery snapshot, then runs the
    existing restart planner and executor: same raw prompt, same name, fresh workspace.
  - Planner refusals render as disabled-row reasons.
- **`e` Edit** (released on submit)
  - Opens the prompt bar with the original raw prompt, reusing `,x`'s prompt
    reconstruction and stash. **Nothing is dismissed or released until submit.**
  - On submit, it runs the Dismiss transaction, then launches the edited prompt through
    the `,x` relaunch barrier.
  - Cancel leaves the agent HIJACKED. If the snapshot fails, the edited prompt is
    stashed and the agent stays held.
- **`v` View diff** (kept): opens the snapshot diff (live diff if newer) in the pager.
- **`t` Shell** (kept): the existing tmux flow, rooted in the held workspace. The next
  release re-snapshots if the workspace fingerprint changed.
- **`d` Dismiss** (released)
  - Verifies (or refreshes) the snapshot, then runs the existing ownership-checked
    dismiss. Waiters now see `killed`.
  - If the snapshot fails, nothing is released. In that error state only, a separately
    labelled "Discard without snapshot" is offered behind a deliberate confirmation.
- **`x` on a HIJACKED row** runs the `d` transaction with no extra dialog: the snapshot
  makes the release safe.

### Escalation policy (A1)

1. **Rung lookup, in order:**
   - the successor lineage `hijack_lineage.to_rung` in `agent_meta.json`, so a second
     hijack keeps climbing;
   - the highest configured ladder rung in `model_alias` / `model_alias_trail` (which
     store aliases without `@`);
   - the highest rung whose candidate set contains the run's concrete
     `provider/model@effort`. This covers the ~59% of runs that record no alias.

   Never infer the rung from prompt text.

2. **Target:**
   - below the first rung, or unknown → the first rung (`@large`);
   - on a rung → the next rung (`@large` → `@xlarge`);
   - on the top rung → **Switch** to the first candidate of that alias whose
     provider/model differs from the stalled run;
   - no differing candidate → disabled: "no different model available; use Continue or
     Instruct".
3. **Resolve once.** The preview resolves candidates read-only, honouring provider
   disables, and skips candidates identical to the stalled run. The keypress persists
   that exact concrete target. A target that became unavailable since the preview fails
   closed, and the agent stays held.

### CLI parity (cli_rules: alphabetized options, short aliases, excellent `-h`)

- **`sase agent hijack [-j] [-r REASON] NAME`**
  - Synchronous. Exit 0 = HIJACKED, 2 = refused with nothing changed, 1 = stop failed or
    error.
  - Prints a coloured result line:
    `⏏ hijacked sase-7f.2 · ws #4 held · 5 files +212 −31 · next: sase agent recover sase-7f.2`.
- **`sase agent recover [-a ACTION] [-j] [-m MODEL] [-n] [-p PROMPT_FILE] NAME`**
  - `ACTION` ∈ `continue | dismiss | escalate | restart`.
  - Bare (no `-a`), it prints the recovery menu (the same preview as the panel) and
    exits 0. `-n/--dry-run` previews one action.
  - `-p` replaces the prompt: the full continuation prompt for `continue` (Instruct), or
    the edited prompt for `restart` (Edit).
  - `-m` pins the concrete target for `escalate`/`continue`.
- **Durable internal ops**, which the TUI submits as procs so that closing the TUI never
  strands a half-done action: `sase ops agent persist-hijack` and
  `sase ops agent persist-recover`. Both follow the `persist-cleanup` request-sidecar
  pattern in `src/sase/ops/commands/agent.py`.
- `sase agent restart NAME` on a HIJACKED target claims the `restart` resolution.
  `sase agent kill -n NAME` on a HIJACKED target refuses with "already stopped
  (hijacked); use sase agent recover NAME -a dismiss".
- `sase agent list` and `sase agent show` render HIJACKED without `-a`.

### Config (kept deliberately small)

```yaml
# Recovery for hijacked agents (`,k` / `sase agent hijack`). A hijack stops a
# stalled agent but keeps its workspace claim until it is recovered.
agent_hijack:
  # Ordered size rungs for the recovery panel's Escalate action. A hijacked run
  # moves to the first rung above the one it ran on; runs below the first rung
  # (or with unknown provenance) move to it. On the top rung, Escalate becomes
  # Switch: the first candidate of that alias whose provider/model differs.
  escalation_ladder: ["@large", "@xlarge"]
  # What Escalate/Continue/Instruct successors learn about the hijacked turn:
  # "fork" inlines its transcript (falling back to "brief" when it would exceed
  # llm_provider.continuation_budget); "brief" sends only the evidence brief.
  successor_context: fork
```

Plus `ace.keymaps.leader_mode.keys.hijack_agent: "k"`. Remember the `gotchas` rule:
keymaps live in `src/sase/default_config.yml`.

- **Reused, not added:** `workspace.held_claim_ttl_days` (14) bounds the hold, measured
  from `hijacked_at`, and the existing 6 s terminate grace stops the tree.
- **Not added:** an `enabled` switch (the flag covers that during the epic),
  `max_concurrent`, `stall_detection_seconds`, a configurable default action, and a
  preamble template (the text is core-owned).

> [!decision] successor_context = fork The default is `successor_context: fork`.

> [!decision] successor_context = brief The default is `successor_context: brief`.
> `fork` stays available as an opt-in.

### Feature flag

`agent_hijack` is a **beta** flag (default off) and is epic scaffolding per
`sase_flags`.

- **Gated (ingress only):** `,k`, the kill dialog's `h`, the `,k hijack` footer chip,
  and `sase agent hijack`. Off, the CLI refuses with how to enable it.
- **Never gated:** rendering, the recovery panel, `sase agent recover`, and every
  custody exemption. An agent hijacked while the flag was on must stay safe and
  recoverable after it is turned off.

The `land` phase deletes the Off branch.

## Durable records (shared by all phases)

- **Intent marker**, in the artifacts directory. The existing user-kill intent
  (`.sase_user_kill_pending`, written by `write_user_kill_intent`) gains
  `"mode": "hijack"` and `"hijack_id"`. Every existing "user kill drops pending
  handoffs" branch therefore applies unchanged.
- **Ledger record**, `~/.sase/agent_hijacks/<hijack_id>.json`.
  - Active records live there; terminal records move to
    `~/.sase/agent_hijacks/settled/`. Hot reads stay O(unresolved), like the goal
    ledger.
  - The schema is a Rust wire type, `AgentHijackRecordWire` v1:
    - **identity:** `hijack_id`, agent name, artifacts dir, session, and host;
    - **target:** pid and process-identity token;
    - **custody:** project file, workspace number and dir, the expected claim owner pid,
      and the prior pin state;
    - **model facts at hijack time:** alias, trail, origin, provider, model, and effort;
    - **evidence:** run duration, the last tool and its age, diff stats per repo,
      snapshot refs and fingerprints, stale lock files, and survivor pids;
    - **lifecycle:** `state`, `revision`, `hijacked_at`, and `updated_at`;
    - **resolution:** action, reserved successor name, concrete target model,
      `model_changed`, error, and acknowledgement.
- **States:** `requested → stopping → held → resolving → resolved | dismissed`, plus
  `held → expired` (hold TTL), `stopping → stop_failed`, and
  `requested|stopping → finished_first`.
  - `expired` can still go to `resolving` for Restart, Edit, View, and Dismiss only.
  - `resolving → held` is allowed: a failed or reconciled resolution returns there.
  - Every write is a compare-and-set on `revision` under a file lock.
- **done.json** of the hijacked turn: `outcome: "hijacked"` and
  `recovery: {state: "pending", kind: "hijack", hijack_id}`.
  - Resolution later rewrites only `recovery.state` (`resolved` / `dismissed` /
    `expired`) plus `recovery.action`, `recovery.successor`, and
    `recovery.model_changed`.
  - A real (non-hijack) outcome is never rewritten.

## Phase: core_ledger

Work in the linked sase-core repo: open it with `sase repo open sase-core` and read its
`AGENTS.md`. Then bump sase's `sase-core-revision.txt` per `docs/rust_backend.md`.

1. **Status table** (`crates/sase_core/src/agent_archive/status.rs`):
   - `HIJACKED` → `"Stopped"`, with an explicit regression test that it is _not_
     `"Running"`; unknown labels fall through to Running today.
   - `ESCALATED` and `CONTINUED` go everywhere `ANSWERED` goes: the bucket, the archive
     outcome, `fleet_owner_facts.rs`, and the agent-scan wire. Grep `"ANSWERED"` across
     the crate.
   - A dismissed or expired hijack archives as `interrupted`.
2. **Done-marker → status derivation:**
   - `outcome: hijacked` with `recovery.state` in {pending, expired} → `HIJACKED`;
   - `resolved` → `ESCALATED` if `model_changed`, else `CONTINUED`;
   - `dismissed` → `interrupted` (archive).

   Add this wherever Rust derives status from done facts (agent scan / fleet). Run state
   and fleet projection map `hijacked` to a parked/needs-input state, never `failed`.

3. **New module `agent_hijack`** (pure, no file I/O, mirroring `agent_auto_restart`):
   - **`wire.rs`:** `AgentHijackRecordWire` v1 with the schema above.
   - **`ledger.rs`:** the state constants plus
     `advance_agent_hijack_record(record, event) -> Result<record>`, which validates the
     transitions above and bumps `revision`.
   - **`eligibility.rs`:**
     `agent_hijack_eligibility(facts) -> Eligible | Refused{code, message}` over the
     facts listed in v1 eligibility.
4. **Auto-restart exclusion:** `agent_auto_restart/classify.rs` explicitly excludes any
   run whose facts carry a hijack intent or `outcome: hijacked`. This covers the window
   before done.json too.
5. **Bindings and tests:**
   - PyO3 bindings in `crates/sase_core_py/src/agent_hijack/`, registered in `lib.rs`;
   - Rust unit tests for every transition (legal and illegal), every refusal code, and
     the status and archive rows.
6. **In sase:**
   - bump `sase-core-revision.txt`;
   - add thin wrappers in `src/sase/agent/hijack/core.py` (new package) that call
     `require_rust_binding`;
   - add the Python mirror rows if `src/sase/agent/status_buckets.py` keeps any
     hard-coded tables, with a binding test that `HIJACKED` buckets as Stopped.

## Phase: core_escalation

Work in sase-core first, as in `core_ledger`, then in sase.

1. **`agent_hijack/escalation.rs`:**
   `agent_hijack_escalation_target(model_facts, ladder, alias_candidates, lineage) -> Target{kind: escalate|switch, from_rung, to_rung, concrete} | Disabled{reason}`,
   exactly per [Escalation policy](#escalation-policy-a1).
   - Inputs are plain data: `alias_candidates` maps each rung to an ordered,
     already-filtered (enabled) candidate list that Python resolves from `models.yml`.
   - Normalize `large` ↔ `@large`.
   - Parametrized tests: `@xsmall`, `@medium`, `@large`, `@xlarge`, an explicit
     `claude/opus@xhigh`, a custom alias whose trail contains `large`, no alias at all,
     `default_model` origin, lineage present, a candidate identical to the stalled
     model, and no differing candidate. The current `models.yml` puts
     `codex/gpt-6.1-sol@xhigh` in both `@large` and `@xlarge`.
2. **`agent_hijack/continuation.rs`:**
   `agent_hijack_continuation_prompt(inputs) -> String`. Inputs: the original raw
   prompt, the hijacked name, the model directive, evidence, the context mode
   (`fork|brief`), an optional transcript path, an optional guidance slot, and stale
   lock files.
   - **Output:** a `%model:` line, a `#fork:<name>` line in fork mode, a blank guidance
     line when requested, the hijack frame, then the original prompt.
   - **Frame text** (adapt wording, keep the facts): "**Hijack continuation.** The user
     stopped `<name>` (`<model>`) because it stopped making progress (ran 47m; last
     activity: Bash `just check`, 22m before the hijack). You are its successor in the
     same workspace. Its uncommitted changes are still here (5 files, +212 −31; recovery
     snapshot: `<ref>`). Inspect `git status` and `git diff` before acting. Keep correct
     work, repair or revert wrong work, and do not repeat whatever stalled the previous
     run. The original task follows, unchanged."
   - Mention stale lock files; never delete them.
   - **Strip launch-only directives** from the original prompt: workspace references
     (`+proj`, `#gh:`, `#git:`), `%id`, `%clan`, `%tab`, `%wait`, `%model`, `%effort`,
     and fan-out syntax. Export the strip list, and test every entry. A quoted `%model:`
     inside a code block is not a directive.
3. **Bindings, tests, and pin bump.** Python wrappers go in
   `src/sase/agent/hijack/core.py`. If `core_ledger` already created it, extend it;
   resolve the trivial conflict if both phases run concurrently.
4. **Config in sase:**
   - add the `agent_hijack:` section, with the comments above, to
     `src/sase/default_config.yml`, next to `agent_auto_restart:`;
   - add schema entries in `src/sase/config/sase.schema.json`: the ladder is a non-empty
     list of unique `@alias` strings, and `successor_context` is the enum `fork|brief`;
   - add a typed accessor `agent_hijack_config()` in `src/sase/agent/hijack/config.py`
     that validates rung names against the loaded alias catalog and falls back to the
     defaults with a warning;
   - add a Python helper that builds `alias_candidates` from the alias catalog,
     honouring provider disables;
   - run `just fix` so the generated config docs refresh.

> [!decision] successor_context = brief The schema and YAML default become `brief`.

## Phase: settle

The runner must be able to die _without_ releasing its claim. Today
`install_workspace_release_sigterm_handler` (`src/sase/axe/run_agent_runner_signals.py`)
releases on SIGTERM, and `_should_hold_workspace`
(`src/sase/axe/run_agent_runner_lifecycle.py`) returns `False` whenever `was_killed`.

1. **Ledger store** in `src/sase/agent/hijack/ledger.py`, modelled on
   `src/sase/agent/auto_restart/ledger.py`:
   - an exclusive create (`O_EXCL`);
   - read, and a CAS transition through the Rust `advance_agent_hijack_record` under an
     `fcntl` lock;
   - an atomic replace, and the move to `settled/` on terminal states;
   - list active records, and look them up by artifacts dir, by agent name, and by
     `(project_file, workspace_num)`.

   Tests cover concurrent CAS (exactly one winner), torn-write safety, and the settled
   move.

2. **Intent:** extend `write_user_kill_intent` / `_read_user_kill_intent` in
   `src/sase/agent/_user_kill_intent.py` with an optional `mode` and `hijack_id`, plus a
   `has_hijack_intent(artifacts_dir)` helper. Old markers (no mode) still read as plain
   kills.
3. **Capability stamp:** new runners write `"runner_capabilities": ["hijack"]` into
   `agent_meta.json` at setup (`run_agent_runner_setup_meta.py`). Eligibility refuses
   runs without it, because an old runner's SIGTERM handler releases the claim
   regardless.
4. **SIGTERM handler:** `_release_workspace_claim` returns early when
   `has_hijack_intent(artifacts_dir)`, the same pattern as `_has_staged_handoff`.
5. **Killed iteration** (`_handle_killed_iteration` in
   `src/sase/axe/run_agent_exec.py`): inside the existing user-kill branch, a
   hijack-mode intent performs the same discards (plan, questions, monitor, gate, pipe,
   and staged markers) and records kill provenance, but returns `"hijacked"`.
   - Thread the new outcome through the exec result and runner shutdown state.
   - `_should_hold_workspace` returns `True` for `hijacked`, and that check runs
     **before** the `was_killed` short-circuit.
   - The run sends no completion notification and does not auto-dismiss.
   - It keeps the bead claim, mirroring how failure holds keep it.
6. **done.json:** the runner writes `outcome: "hijacked"` and
   `recovery: {state: "pending", kind: "hijack", hijack_id}`.
   - The runner pins the claim (`hold_workspace_claim(..., expected_pid=own pid)`) so
     the custody story does not depend on the orchestrator alone.
   - It moves the ledger record to `stopping` if it is still `requested`. It never
     writes `held`; evidence is the orchestrator's job.
7. **Finalize boundary:** before host finalizers run, the runner checks for a hijack
   intent. If one is present, it skips finalization, as it does for a user kill, and
   settles hijacked. A late declaration from the dead turn is therefore fenced.
8. **Python TUI status mirrors:** the loaders that derive row status from done markers
   and meta (`src/sase/ace/tui/models/_loaders/_meta_enrichment_status.py`,
   `_agent_status_apply.py`) map `hijacked` per the Rust derivation, so the row reads
   HIJACKED once `held`.
9. **Tests:**
   - SIGTERM with hijack intent keeps the claim; without it the claim is released
     (unchanged);
   - the killed iteration returns `hijacked` and discards the markers;
   - done.json carries the shape above;
   - no notification is sent;
   - the hold survives runner exit;
   - an old-format intent still kills normally.

## Phase: stop

1. **Flag:** run `sase flag new agent_hijack -k beta` with the three sentences:
   - `--when-enabled`: "`,k`, the kill dialog's Hijack choice, and `sase agent hijack`
     can hijack eligible running agents.";
   - `--when-disabled`: "Hijack ingress is hidden and `sase agent hijack` refuses;
     already-hijacked agents still render and stay recoverable.";
   - `--remove-when`: "The agent_hijack epic's land phase passes its end-to-end
     hijack-and-escalate test."

   Paste the printed registry entry. Do not hand-edit it.

2. **Orchestrator** `src/sase/agent/hijack/stop.py`. **The order is the design:**
   1. **Eligibility.** Gather facts (the agent row and meta, pid identity, owning host,
      claim, lifecycle phase, and capability), then call the Rust eligibility function.
      A refusal returns exit 2 and changes nothing.
   2. **Record.** Exclusively create the ledger record (`requested`), then write the
      hijack-mode intent. Both exist before any signal.
   3. **Pin.** Call
      `hold_workspace_claim(project_file, workspace_num, ..., expected_pid=pid)` under
      the claim lock (`src/sase/running_field/_hold.py`), recording the prior pin state.
      A mismatched or replaced claim aborts.
      - If done.json already holds a real outcome, the record becomes `finished_first`,
        the intent is removed, and the pin is restored only if this hijack set it.
   4. **Stop.** The record moves to `stopping`. The stop runs the existing verified
      terminator (`_user_kill_tree.py` / `user_kill.py`): process-identity checks,
      descendants and registered supervisors, the 6 s grace, then SIGKILL. Do not add a
      shortcut `killpg`.
   5. **Survivors.** If any process survives escalation, the record becomes
      `stop_failed` with the survivor pids, and the intent is removed so the runner
      cannot later settle it as hijacked. The pin is restored to its prior state, only
      if this hijack set it, so the still-running runner's normal completion releases as
      usual. The row stays RUNNING. Exit 1.
   6. **Fallback.** If the runner died before settling (wedged or SIGKILLed), write the
      same done.json and re-assert the pin. This is idempotent with `settle`.
   7. **Evidence**, off the TUI and best-effort:
      - diff stats per repo (primary plus linked and sidecar repos the turn opened, with
        untracked counts), via the existing worktree diff helpers;
      - the last tool call and its age, from the tool-call ledger;
      - the run duration;
      - stale `index.lock` files.
   8. **Strict snapshot.** Snapshot every repo with changes through `snapshot_git_repo`
      (`src/sase/workspace_provider/_snapshot_capture_source.py`), recording refs and
      fingerprints. A snapshot failure does **not** block `held`; it is recorded, and
      every later release must succeed at snapshotting first.
   9. **Publish.** The record moves to `held`, and HIJACKED is published.
3. **`sase ops agent persist-hijack`:** a durable op reading a private request sidecar
   (the `persist-cleanup` pattern). It runs the orchestrator so that the TUI can submit
   it as a proc.
4. **`sase agent hijack [-j] [-r REASON] NAME`:** register it in
   `src/sase/main/parser_agent_lifecycle.py` and the agent handler.
   - It is flag-gated.
   - It uses the exit codes and coloured output above. `-j` emits one stable JSON
     envelope.
   - `-h` explains what is kept, what is stopped, and the next command.
5. **Tests:**
   - the happy path reaches `held` with the claim pinned and done.json written;
   - a refusal changes nothing;
   - the lost race keeps the real outcome;
   - survivors leave no HIJACKED status and no lingering intent;
   - a reused or foreign-host pid is never signalled;
   - an old runner (no capability) is refused;
   - the fallback path works with a SIGKILLed runner;
   - a snapshot failure still reaches `held` but is recorded;
   - both flag states are covered.

## Phase: custody

Every path that releases a dead process's claim must learn about hijacks.

1. **Stale cleanup** (`src/sase/ace/scheduler/stale_running_cleanup.py`):
   - Add `_hijack_retains_claim(project_file, claim)`, modelled on
     `_staged_origin_retains_claim`. The presence of an active record (`requested`,
     `stopping`, `held`, or `resolving`) matching the project, workspace, and expected
     pid retains the claim even before done.json exists.
   - Measure the TTL from `hijacked_at`, not the claim's launch stamp; today
     `_held_claim_exceeds_ttl` uses launch.
   - **On expiry:** verify the snapshot, re-taking it if the workspace fingerprint
     changed. A failure keeps the hold and logs. Then release the claim, move the record
     to `expired`, and set done.json `recovery.state: "expired"`. The node stays
     visible.
2. **Kill persistence** (the `persist-cleanup` op): skip claim release for hijacked
   turns that have an active record.
3. **Auto-restart:** wire the hijack facts into the healer's classify inputs
   (`src/sase/agent/auto_restart/healer_claim.py` and related), so the Rust exclusion
   from `core_ledger` applies.
4. **Late callbacks:** any old-runner release path (shutdown and retention callbacks)
   checks that the claim's current owner pid is still the old pid before releasing. A
   late callback can then never release a successor's claim.
5. **Waits:**
   - `outcome: hijacked` with recovery `pending` or `expired` maps to parked or
     needs-input in `%wait` dependency classification and clan waves, never failure;
   - `dismissed` reads as `killed`.

   Successor-following is added in `recover_keep`.

6. **Stranded detector:** a HIJACKED bead owner is "parked by the user", so Enter offers
   no confusing "resolve stranded" row.
7. **Listing:** `sase agent list` shows HIJACKED rows without `-a`, with a dim
   `ws #N held` hint. `sase agent show` adds a "Hijack" block with the state, evidence,
   snapshot refs, and next command.
8. **Tests:** each release path leaves a hijacked claim held:
   - stale cleanup with a dead pid and no done.json yet;
   - the TTL measured from `hijacked_at` (a 13-day-old run hijacked today is not
     expired);
   - expiry snapshots before release;
   - a snapshot failure keeps the hold;
   - kill persistence skips;
   - auto-restart never selects the run;
   - a late callback does not release a transferred claim;
   - waits stay parked.

## Phase: recover_keep

This is the path that makes the feature worth building, so it takes most of the test
budget.

1. **Strict follow-up:** add `strict_workspace: bool = False` to `launch_turn_followup`
   (`src/sase/turns/followup.py`).
   - When strict, only step 1 (transfer from the held pid) and step 2 (a fresh claim on
     the same number) are allowed. The pool-workspace and `#0` fallbacks become
     `record_not_launchable`.
   - Test that strict never degrades.
2. **Resolution transaction** `src/sase/agent/hijack/recover.py`:
   - A CAS moves `held` → `resolving`, recording the action, reserved successor name,
     concrete target, and resolver pid, **before** any effect. A second resolver gets
     "already being resolved".
   - **Compose the prompt.** It is the original `raw_prompt.md` through the Rust
     continuation builder. `successor_context: fork` uses `#fork:` unless the existing
     continuation-budget preflight says the transcript will not fit, in which case it
     falls back to the brief plus the transcript path.
   - **Launch.** Use `spawn_agent_session_successor`
     (`src/sase/agent/detached_child.py`) through strict `launch_turn_followup` with
     `transfer_from_pid=<held pid>`. Continuing a solo agent promotes it to a session,
     which is existing behaviour.
   - **Required inheritance.** These are not best-effort; failure aborts the launch and
     returns to `held`:
     - the goal binding (`goals-host-binds`);
     - the bead claim;
     - the finalizer baselines of the original turn, so the successor's finalization
       owns the predecessor's diff;
     - opened-repo obligations;
     - the permission scope.

     Verification receipts and old declarations are **not** inherited. Reuse the
     pipe-successor inheritance helpers (`src/sase/axe/run_agent_successor*.py`,
     `_run_agent_successor_shared.py`) and make their failures fatal for this path.

   - **Lineage.** The successor's `agent_meta.json` gets
     `hijack_lineage: {hijack_id, from_rung, to_rung, predecessor}`.
   - **Settle.** On launch, the record moves to `resolved` and the predecessor's
     done.json `recovery` gets `{state: resolved, action, successor, model_changed}`, so
     the row reads ESCALATED or CONTINUED. On failure it returns to `held` with the
     error.
   - **Admission.** The successor goes through the normal admission path, as pipe and
     gate successors do. If it is queued, the claim stays pinned until the transfer.

3. **Crash reconciliation:** a `resolving` record whose resolver pid is dead and that is
   older than 60 s is adopted if the reserved successor exists; otherwise it returns to
   `held` with an error. Run it at the start of every recover call and from the
   stale-cleanup pass.
4. **`#fork` frame:** next to `format_failed_agent_body`
   (`src/sase/history/chat_fork/failure.py`), add a hijack frame. A fork of a hijacked
   turn says "the user stopped it because it was not making progress; it did not
   finish", not the generic failure text.
5. **Waits follow the successor:** session-target and name-target waits on the
   predecessor follow `recovery.successor`, reusing the existing pipe-successor
   wait-follow semantics.
6. **Preview API:** `hijack_recovery_preview(agent_or_name) -> RecoveryPreview`, cheap
   and side-effect-free, gives per-action `{available, reason, label, detail, target}`
   plus the header facts.
   - This phase fills in `escalate` and `continue` (and therefore Instruct). The
     `restart` and `dismiss` entries return "not yet available" until `recover_release`.
   - It powers both `sase agent recover NAME` (bare menu) and the TUI panel.
7. **CLI:** `sase agent recover` with `-a escalate|continue`, `-m`, `-p`, `-n`, and
   `-j`, plus `sase ops agent persist-recover`. Bare `recover` prints the menu, rendered
   with the same grouping and glyphs as the panel, using Rich.
8. **Tests:**
   - Escalate, Continue, and Instruct (`-p`) each launch in the same workspace and own
     the predecessor's changes through finalization; use the fake provider;
   - two concurrent resolvers yield exactly one winner;
   - a crash mid-`resolving` is adopted, or returns to `held`;
   - a transfer failure stays HIJACKED and never uses a pool workspace or `#0`;
   - an inheritance failure aborts cleanly;
   - an unavailable target fails closed;
   - waits follow the successor;
   - ESCALATED vs. CONTINUED labelling.

## Phase: recover_release

1. **Snapshot gate:** `ensure_recovery_snapshot(record)` verifies that the recorded
   snapshot still matches the workspace fingerprint, re-snapshotting otherwise, across
   the primary and every linked or sidecar repo with changes. It must succeed before any
   release. The recovery details note that the snapshot depends on the remote keeping
   its base commits.
2. **Restart:** claim `restart`, run `ensure_recovery_snapshot`, then call the existing
   restart planner and executor (`src/sase/agent/_restart_planning.py`,
   `_restart_execute.py`): same raw prompt, same name, fresh workspace. The ledger keeps
   the prompt and snapshot location, so a failed relaunch after release is recoverable.
   - Planner refusals (fan-out, multi-segment, container, disabled provider) and "wipe
     reaches related agents" become preview reasons. The TUI shows the last one as
     unavailable with a hint to use the CLI.
   - **`-p PROMPT_FILE` with `restart`** relaunches the given prompt instead. This is
     the Edit backend: run Dismiss, then a normal launch, so a changed `%id` works.
3. **Dismiss:** claim `dismiss`, run `ensure_recovery_snapshot`, then the existing
   ownership-checked dismiss. It also releases the bead claim, as a kill does. The
   record moves to `dismissed`, and done.json gets `recovery.state: dismissed`.
   - A snapshot failure keeps everything held and returns an error naming the explicit
     escape hatch: `--discard-without-snapshot`, flagged as dangerous in `-h` and given
     a short alias.
4. **Expired holds:** the preview omits the keep-work actions; Restart, Edit, View (from
   the snapshot), and Dismiss remain.
5. **Existing commands:** `sase agent restart NAME` on HIJACKED goes through the
   `restart` resolution. `sase agent kill` on HIJACKED refuses with the hint above.
6. **Tests:**
   - Restart, Edit (restart `-p`), and Dismiss with staged, untracked, and binary
     changes and unpushed commits: the snapshot is verified before release;
   - a snapshot failure keeps the hold;
   - restart refusals surface as reasons;
   - an expired hold offers only release actions;
   - waits read `killed` after Dismiss and follow the name after Restart.

## Phase: tui_render

Read the `tui.md` children (`tui_perf.md`, `tui_screenshot.md`) and `ace.md` first. No
blocking I/O on the event loop: ledger reads join the existing off-loop loader refresh.

1. **Status presentation**
   (`src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`,
   `src/sase/ace/tui/models/agent_status.py`):
   - `HIJACKED_STATUS`, `HIJACKED_COLOR = "#FF5FAF"`, and `HIJACKED_GLYPH = "⏏"`;
   - the expired `· workspace released` hint;
   - the `stop failed` badge;
   - `ESCALATED`/`CONTINUED` styled like `ANSWERED`.

   Do **not** add HIJACKED to `DISMISSABLE_STATUSES`, which also drives unread-done.
   Give `x` its own HIJACKED branch instead.

2. **Overlays**, following the FINALIZING render-time overlay pattern
   (`src/sase/ace/tui/models/finalizer_row_state.py`):
   - `HIJACKING…` from active records in `requested`/`stopping`;
   - `RECOVERING…` from `resolving`;
   - static text, no spinner, for snapshot stability.
3. **Detail header:** the "Workspace held: hijacked · ws #N · <diff summary> · ⏎
   recover" line, built from ledger evidence.
4. **Leader key `,k`:** `hijack_agent: "k"` in `default_config.yml`, the keymap
   dataclass and metadata (`src/sase/ace/tui/keymaps/`), the action, the help modal
   entry (keeping the 57-column box rules), and the conditional leader-footer chip.
   - The action submits `sase ops agent persist-hijack` as a durable proc.
   - It toasts the outcome: hijacked, refused (with the reason), finished first, or stop
     failed (with the pids).
5. **Kill dialog:** `ConfirmKillModal` (`src/sase/ace/tui/modals/confirm_kill_modal.py`)
   gains an optional third choice, `h` "Hijack instead · stop it but keep workspace #N".
   The result widens from `bool` to a small literal; update `_kill_action_flow.py`
   accordingly. Show it only when the agent is eligible and the flag is on.
6. **Footer:** `⏎ recover` and `x dismiss` chips on HIJACKED rows
   (`src/sase/ace/tui/widgets/_keybinding_bindings_agents.py`). While `tui_panel` is
   pending, `⏎` on HIJACKED may fall through to the existing Enter behaviour; the flag
   hides ingress anyway.
7. **Remote rows:** `,k` toasts the owning host and copies `sase agent hijack NAME` to
   the clipboard.
8. **Tests:**
   - unit tests for presentation, overlays, the footer chips, the eligibility gating of
     `,k` and `h`, and both flag states;
   - PNG goldens for a HIJACKED row, the HIJACKING overlay, an expired hold, and the
     stop-failed badge, captured with targeted `just fix-tui-screenshots -- <selectors>`
     through `/sase_monitor`.

## Phase: tui_panel

1. **Chooser extension** (`src/sase/ace/tui/modals/agent_action_chooser_modal.py`), all
   opt-in so existing callers render byte-identically:
   - **Sections:** add `hijack_keep`, `hijack_fresh`, and `hijack_other` to
     `AgentActionSection`, with headings that take a dim subtitle ("KEEP THE WORK · next
     turn runs in workspace #4"). Ordering: keep → fresh → other → patch.
   - **Header:** an optional renderable slot under the title, updatable when the preview
     arrives.
   - **Title:** an optional right-aligned title badge (`HIJACKED · ws #4`).
   - **Rows:** `AgentActionChoice.disabled_reason` (dim row; its key toasts the reason),
     a `pending` flag, and an `inline_detail` layout for single-line rows.
   - **Arming:** an `arm_delay_seconds` parameter (300 ms for recovery).
   - **Primary:** `guidance_text` names the first _enabled_ row (or the `initial_index`
     row, per `enter_default`).
2. **Enter routing:** in `_agent_enter_targets.py` / `_agent_enter_action.py`, a
   HIJACKED agent (local) resolves to a `hijack` target that opens the recovery panel
   directly. A Patch, if any, is appended as the trailing `p` row. A remote HIJACKED row
   toasts `sase agent recover NAME` for the owning host.
3. **Panel builder** `src/sase/ace/tui/actions/agents/_hijack_recovery_panel.py`:
   - It builds choices from cached ledger evidence immediately.
   - It fetches `hijack_recovery_preview` in a worker, and updates the header and rows
     in place.
   - It caches the preview per `(hijack_id, revision)`.
   - It renders the mocks above exactly: glyphs `⇧ ↔ ▶ ↪ ⟳ ⌨ ± $ ⌫` and the
     one-accent palette.
4. **Actions:**
   - **u, c, r, d** submit `sase ops agent persist-recover` procs with the previewed
     target pinned. The row flips to RECOVERING… immediately, and completion and failure
     toast.
   - **v** opens the pager on the snapshot or live diff.
   - **t** reuses the existing tmux-in-workspace flow.
   - **x** on a HIJACKED row runs `d`.
   - **i and e** show "coming soon" toasts until `tui_prompt`; this is unreachable with
     the flag off.
5. **Tests:**
   - every hotkey;
   - disabled-row reasons;
   - the pending state;
   - `⏎⏎` follows `enter_default`;
   - the arming delay swallows a repeat Enter;
   - existing chooser callers are unchanged.

   PNG goldens:
   - the full panel at 80×24;
   - the Switch variant with a disabled Restart;
   - the pending preview;
   - an expired hold;
   - a long agent name.

> [!decision] enter_default = continue Pass `initial_index` at the Continue row, and
> make `guidance_text` name Continue.

## Phase: tui_prompt

Read `_entry_relaunch.py` (`_finish_kill_and_edit_agent`), `_relaunch_barrier.py`, and
the `_prompt_bar_*` modules first. `,x` today dismisses optimistically before the bar
mounts, and that timing must **not** be copied.

1. **Instruct (`i`):**
   - Mount the prompt bar pre-filled with the Rust continuation prompt (guidance slot
     requested, `%model:` set to the stalled concrete model), with the cursor on the
     guidance line.
   - Submit is intercepted: route it through a narrow submit hook (reuse an existing
     prompt-bar submit seam if one exists; otherwise add one) to
     `persist-recover -a continue -p <file>`.
   - Cancel or Esc leaves the agent HIJACKED; the draft goes to the prompt stash.
2. **Edit (`e`):**
   - Mount the prompt bar with the original raw prompt using `,x`'s reconstruction.
     Nothing is dismissed at mount.
   - On submit, run `persist-recover -a restart -p <file>`, and hold the launch behind
     the relaunch barrier until dismissal succeeds.
   - On a snapshot failure, stash the edited prompt, keep HIJACKED, and toast the error.
   - Cancel leaves HIJACKED.
3. **Tests:**
   - Instruct submit runs `continue` with the edited prompt;
   - a changed `%model:` yields ESCALATED;
   - Edit releases nothing before submit;
   - cancel leaves HIJACKED;
   - a snapshot failure stashes the prompt;
   - no optimistic dismissal.

   Add a PNG golden of the pre-filled Instruct bar.

## Phase: land

1. **End-to-end:** a fake-provider integration test.
   - **Setup:** modify the primary repo and a linked repo, then hijack.
   - **Reload:** simulate a TUI restart by reloading state from disk.
   - **Escalate:** assert that the successor sees **and** owns both repos' changes
     through finalization.
   - **Releases:** run Restart and Dismiss variants and assert the snapshots exist.
2. **Remove the flag** per `sase_flags`:
   - delete every Off branch and make the On branch unconditional;
   - remove the registry entry;
   - delete the Off-state tests;
   - close the flag bead in the same change.
3. **Docs**, routed per `docs_writing`:
   - Reference updates in `docs/reference/tui/agent-statuses.md` (HIJACKED, ESCALATED,
     CONTINUED, and the overlays), `docs/reference/tui/agents-tab.md` (the way in and
     the recovery panel), and `docs/reference/agents/workspaces.md` (held-claim reasons
     and the TTL from `hijacked_at`);
   - regenerate the keymap, config, and CLI references with `just fix`;
   - in `docs/guides/recover-interrupted-agent.md`, fix only steps that became wrong. If
     hijack deserves its own narrative rung, record a "guide candidate"
     `PROPOSED FOLLOW-UP:` instead of appending it.
4. **Memory**, through `/sase_memory_write`, then `sase memory init`:
   - **Glossary strand** `sase/memory/glossary/agent-hijack.md`: keyword `Agent Hijack`,
     aliases `hijack` and `hijacked agent`. The definition is adapted from the
     research's proposed strand: a user-initiated stop of a running agent turn that
     keeps its workspace claim, node, and recovery evidence; it settles as `hijacked`,
     shows HIJACKED in the Stopped bucket, and is resolved exactly once by continuing in
     the same workspace as the next session turn, starting over after a recovery
     snapshot, or dismissing. Waiters stay parked. It never resumes the killed process,
     and it is distinct from a kill, auto-restart, `sase agent hold`, `sase pipe`, and
     the `STALLED` wait label.
   - **Decision records** in `sase/memory/decisions/`:
     - `hijack-escalation-never-downgrades`: the rung comes from provenance, it switches
       laterally at the top, and it skips identical models;
     - `hijack-holds-custody-until-resolved`: intent and pin precede the signal, every
       release path defers to the record, releases are snapshot-gated, and resolution is
       exactly-once.

     Each states the claim, the alternatives, the cost, and the reopen condition.

5. **Final verification:** run `sase tool run check`. Record the deferred follow-ups as
   `PROPOSED FOLLOW-UP:` notes on the land bead:
   - Telegram/mobile answering of unresolved hijacks;
   - a read-only "quiet for N min" stall hint;
   - an `x`-as-fate-chooser revisit;
   - an expiry-warning notification;
   - Salvage & Commit.

> [!decision] glossary_term = no Skip step 4's glossary strand. Record it as a
> `PROPOSED FOLLOW-UP:` on the land bead.

> [!decision] decision_records = no Skip step 4's decision records. Record them as a
> `PROPOSED FOLLOW-UP:` on the land bead.

## Deviations from the research

- **Glyphs:** `⇄` (Switch) and `▸` (cursor) are missing from the pinned Fira Code, so
  this plan uses `↔` and the chooser's existing `›`. The other panel glyphs were
  checked present.
- **Snapshot helper location:** the strict snapshot is in
  `src/sase/workspace_provider/_snapshot_capture_source.py`, not `snapshot.py`.
- **Escalation provenance:** escalation targets are persisted as **concrete** models,
  because Switch is inherently concrete and a `|` pool could rotate between preview and
  launch. Rung provenance survives through `hijack_lineage`, so a second hijack keeps
  climbing.
- **Settled labels:** `ESCALATED` vs. `CONTINUED` is decided by whether the model
  changed, which also covers Instruct and Switch honestly. There is no separate SWITCHED
  label.
- **`x` and dismissal:** HIJACKED is kept out of `DISMISSABLE_STATUSES` (that set also
  drives unread-done), and `x` gets an explicit branch.
- **Settled records:** terminal ledger records move to `settled/` so TUI hot reads stay
  O(unresolved).
- **Phase split:** the work is split into eleven phases, not five, so each phase is
  bounded and the Rust phases and `tui_render` run in parallel with the custody and
  recovery work.

## Risks and mitigations

- **Custody leak through an unknown release path.**
  - _Mitigation:_ `settle` and `custody` each own an explicit list.
  - _Backstop:_ `land`'s end-to-end test asserts the claim owner at every step.
- **Status falls through to Running.** A Rust regression test, plus a Python binding
  test.
- **TUI stalls.** All ledger reads, previews, and diffs run off the loop, and the
  preview is cached per revision (`tui_perf.md`).
- **Half-shipped UI.** Ingress is beta-flagged until `land`, while custody code is never
  flagged.
