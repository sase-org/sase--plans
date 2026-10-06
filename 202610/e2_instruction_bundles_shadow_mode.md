---
tier: epic
title: 'E2: Instruction bundles in shadow mode (memory-built instruction migration)'
goal: 'Every root provider invocation renders the memory-built instruction bundle
  it would receive and records it with a sase-core-validated instruction manifest,
  without delivering it; a preview command, legacy parity checks, and scoreboard coverage
  prove the bundle matches today''s files with the contract exactly once, so E3 can
  switch delivery on a proven renderer.

  '
phases:
- id: memory-units
  title: Legacy instruction renderer exposes structured, cwd-free memory units
  size: medium
  depends_on: []
  description: 'memory-units: refactor the legacy AGENTS.md renderer onto a structured,
    side-effect-free per-root units API (title, contract inputs, core, reference,
    and web units with sources), threading the project root through hidden cwd reads;
    legacy output stays byte-identical.'
- id: manifest-wire
  title: Instruction manifest wire schema in sase-core, binding, adapter, and pin
    move
  size: medium
  depends_on: []
  description: 'manifest-wire: add the instruction manifest v1 wire types, closed
    vocabulary, invariants, and common_digest to sase-core with a dict binding, a
    golden cross-language fixture, a thin sase adapter, and the sase-core pin move.'
- id: compiler
  title: 'Python instruction compiler: layers, overlays, facts, manifest assembly,
    render cache'
  size: medium
  depends_on:
  - memory-units
  - manifest-wire
  description: 'compiler: compose bundles in Python from the units with fixed layers,
    section ids, layout, and root/helper/interactive/export overlays; validate facts;
    assemble normalized manifests; add the content-addressed store and the input-digest
    render cache with its 250 ms warm budget.'
- id: render-cli
  title: '`sase instructions render` preview and legacy parity checks'
  size: medium
  depends_on:
  - compiler
  description: 'render-cli: add the render subcommand (agent, fact, json, no-cache,
    parity, sections) and the legacy parity checker, with CI parity tests for sase-,
    bob-cli-, and actstat-shaped fixtures and this repo''s committed AGENTS.md.'
- id: invocation-hook
  title: Shadow render at every root provider invocation, behind one boundary
  size: medium
  depends_on:
  - compiler
  description: 'invocation-hook: route all three root provider.invoke sites through
    one fail-open boundary that writes per-invocation bundle and manifest artifacts,
    the agent_meta summary, and SASE_INSTRUCTIONS_FILE behind the instruction_shadow_render
    sunset flag, guarded by an architecture test and route tests.'
- id: scoreboard
  title: Scoreboard manifest coverage and intended-vs-observed section diff
  size: medium
  depends_on:
  - invocation-hook
  description: 'scoreboard: add the manifest coverage column, the coverage view, the
    per-section intended-vs-observed diff, and the instructions.coverage doctor check,
    leaving every E1 column unchanged.'
- id: acceptance
  title: Live coverage, parity, latency, budget baseline, and acceptance record
  size: small
  depends_on:
  - render-cli
  - scoreboard
  description: 'acceptance: confirm live coverage, unchanged observed columns, render
    and parity checks in all three projects, warm latency, and the budget baseline;
    attach the acceptance record and JSON to the epic.'
proposed_by: bbugyi200.athena.0xc
create_time: 2026-10-06 12:43:22
status: wip
bead_id: sase-1h3
---

- **PROMPT:** [prompts/202610/e2_instruction_bundles_shadow_mode.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/e2_instruction_bundles_shadow_mode.md)
- **BEAD:** [sase-1h3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h3/README.md)

# Plan: E2 — Instruction bundles in shadow mode

## Context

### Where this comes from

- **Research (primary input):**
  `research:202610/memory_built_instruction_migration_epics/memory_built_instruction_migration_epics.md`,
  sections "E2 Instruction bundles in shadow mode", "The yardstick and what it should
  show after each epic", "Issues in the three prior reports", "Rust and Python
  boundary", and "Checklist for every epic plan". Read it with `sase artifact read`, not
  by opening files.
  - It splits the migration to memory-built, per-invocation instructions into five
    ordered epics (E1–E5), plus optional E6 and conditional E7. **This plan is E2
    only.**
  - Its researcher reports sit in the same directory (`__cld`, `__cdx`, `__mus`,
    `__grk`, `__gem`). `__cdx` "A common contract that keeps the epics compatible" and
    issues 1, 4, and 6 define the manifest invariants and the actor/purpose facts.
    `__mus` issues 5 and 10 explain why the fact vocabulary and the budget baseline are
    frozen here.
  - Bundle anatomy (section ids, `common_digest`): R-ledger,
    `research:202610/instruction_bundles_provider_parity_and_memory_ledger.md` §2 and
    §4.
  - Layers and launch facts: R-delivery,
    `research:202610/sase_md_instruction_delivery/sase_md_instruction_delivery.md`
    ("Launch-fact schema", "Layers", "Bundle lifecycle and failure policy").
  - Root and helper overlays: R-fresh,
    `research:202610/fresh_clone_bootstrap_and_native_subagent_instructions/fresh_clone_bootstrap_and_native_subagent_instructions.md`
    ("One shared core and two lifecycle overlays").
- **E1 has landed** (epic bead `sase-1gu`, closed; plan
  `plan:202610/e1_instruction_scoreboard_and_stopgaps.md`). It built
  `sase instructions verify` (the scoreboard), the Grok `--rules` stopgap, the Claude
  helper template and guard, and the ownership inventory
  (`docs/instruction_inventory.md`). Its after-scoreboard JSON
  (`e1_after_scoreboard_v2.json`) and acceptance record are attached to `sase-1gu`. Read
  the bead with `sase bead read`.
- **Why E2 is its own epic:** a renderer bug would reach every agent, including the
  agents that would fix it. E2 renders and records the bundle for every invocation
  **without delivering it**, and proves it matches today's files. E3 (delivery cutover)
  then switches channels on that proven renderer. Nothing in E2 changes what any
  provider loads, so E2 needed no soak after E1.

### Goal

For **every** root provider invocation, SASE renders the instruction bundle the agent
_would_ get, writes it and an **instruction manifest** next to the run's artifacts, and
delivers nothing. A preview command renders any audience on demand. The scoreboard shows
manifest coverage and, per section, intended versus observed delivery. Parity tests show
the bundle carries every section of today's root and home files, and the contract
exactly once.

### Facts verified while planning (2026-10-06, sase at `9e4b9767d2`)

- **Root invocation sites.** Exactly three production `provider.invoke(` calls exist:
  - `src/sase/llm_provider/_invoke.py`, `invoke_agent()` (the main turn). The execution
    provider label, `artifacts_dir` (may be `None`; the TUI fix-hook path passes none),
    `agent_type`, and the resolved model are in scope. `SASE_FINAL_TURN_NONCE` and the
    sync-ceiling variables are exported before the call and restored in `finally`; copy
    that pattern.
  - `src/sase/finalizers/declaration_recovery.py`,
    `ensure_final_declaration_or_recover()`, inside `with finalizer_owned_turn():`.
  - `src/sase/finalizers/commit_repair_conflict.py`, `run_conflict_repair_turn()`,
    inside `with finalizer_owned_turn():`. It has `attempt` and `repo`.
  - The two finalizer sites get only `provider`, `model_tier`, `model_override`,
    `artifacts_dir`, and `options`; derive the provider name with
    `provider.provider_name()` or `agent_meta.json["exec_llm_provider"]`.
  - Non-calls to allowlist: each adapter's `llm_invoke` hookimpl
    (`return self.invoke(...)` in `claude.py`, `codex.py`, `grok.py`,
    `muse_provider.py`, `agy.py`, `qwen.py`, `opencode.py`, `fakey.py`), and
    `_NoModelProvider.invoke` in `src/sase/monitor/host_completion_execute.py` (it
    raises).
  - Monitor and gate turns, `sase tmux-agent`, chops, and `%dispatch` call no provider
    here. A dispatched run goes through `invoke_agent` on the target host, so it renders
    there.
- **Provider fallback.** `src/sase/axe/run_agent_exec_retry.py` re-runs the workflow in
  the same artifacts dir. A fallback sets `SASE_MODEL_OVERRIDE`, which may resolve a
  different provider on the next `invoke_agent`. `snapshot_attempt()` moves only
  `live_reply.md`, its timestamps, and `continuation/` into `attempts/NN/`.
- **Legacy launch evidence.** `capture_instruction_snapshot` in
  `src/sase/axe/launch_evidence.py` records blob OIDs of the workspace `AGENTS.md`, its
  shims, and the first home file once per launch, into
  `agent_meta.json["instruction_snapshot"]`. The TUI memory-version views read it. It
  keeps running unchanged in E2; E4 retires it.
- **`agent_meta.json` writers.** `update_meta_field` / `update_meta_fields` in
  `src/sase/axe/run_agent_helpers.py` (patchable wrappers) do read-modify-write and
  swallow I/O errors.
- **Env var names** live in `src/sase/env_contracts.py`. `tests/conftest.py` snapshots
  and restores every `SASE_*` variable, so a new one needs no test plumbing.
- **Directives today.** Claude `_SINGLE_TURN_DIRECTIVE` (`claude.py`); Codex
  `_codex_single_turn_directive()` (`codex.py`); Grok `_GROK_SINGLE_TURN_DIRECTIVE`
  (`grok.py`); Muse `_muse_single_turn_directive(synchronous=...)`
  (`_muse_directive.py`; the mode comes from the `muse_synchronous_shell` flag); agy
  `_AGY_PRINT_MODE_DIRECTIVE` (`agy.py`). Qwen, OpenCode, and fakey have none.
- **Helper template.** `src/sase/llm_provider/templates/claude_helper_instructions.md`
  (first line `# SASE Helper Instructions`), resolved by `helper_template_path()` in
  `src/sase/llm_provider/_claude_helper_channel.py`.
- **Today's generator** (all Python; `sase_core_rs` is not involved):
  - `src/sase/main/init_memory/root_planning.py::memory_root_context` is the no-write
    seam. It drives web rosters, the generated `sase.md` body
    (`root_rendering_notes.py::render_generated_sase_memory_body`, a Jinja render of
    `templates/memory-sase.template.md` with `project_name` and `linked_repo_entries`),
    the generated reference notes, and `src/sase/amd/_memory.py::plan_amd_memory_sync` →
    `_render_managed_agents` → `render_agents_template` →
    `number_agent_document_sections`.
  - Core notes are inlined by `inline_memory_section` (`src/sase/amd/inline_memory.py`)
    as `### {H1} ({stem})`, with body headings shifted down two levels, in
    `(priority, path)` order (generated `sase.md` has priority 10).
  - Reference entries come from `render_long_memory_entries`
    (`src/sase/memory/notes.py`). A reference note with no `description:` falls back to
    the description in the **existing** `AGENTS.md`
    (`_existing_agents_long_descriptions`), so output depends on the previous output.
  - Hidden `Path.cwd()` inputs: the project config path (`init_memory/config.py`), the
    required-plugin set for task types (`task_types/snapshot.py`), and the
    `plan_amd_memory_sync` root default. Other inputs: `HOME`, `CONFIG_DIR`,
    `CHEZMOI_HOME`, `markdown.print_width`, and the task-type registry (entry points).
  - The H1 comes from `resolve_amd_h1_title` (`src/sase/amd/_config.py`); sase sets
    `memory.h1_title` in `sase/sase.yml`. bob-cli and actstat derive
    `"<name> - Agent Instructions"`.
  - Memory init renders the home layer from the **chezmoi source** home. Audited
    `sase memory read` resolves home memory from `Path.home()`
    (`src/sase/memory/selector.py`): project first, then home.
  - Warm in-process planning of both roots takes about 0.35 s. Importing
    `init_memory_handler` alone takes about 0.58 s. Hotspots: repeated
    `discover_memory_notes`, pure-Python YAML, and per-file `git rev-parse`.
  - Test helpers for arbitrary tmp projects:
    `tests/main/init_memory_handler_helpers.py`.
    `tests/main/test_init_memory_committed_drift.py` runs the planner against this repo.
- **Other projects' shapes** (read-only look via `sase repo open`):
  - **bob-cli:** a derived title, the `sase.md` core note, reference notes, and the
    `decisions`, `glossary`, and `task_types` webs.
  - **actstat:** a derived title, the `sase.md` core note, the generated reference
    notes, and only the `task_types` web.
  - Each tracks five generated files.
- **No plugin contributes instruction text today.** Task types from `sase_task_types`
  feed the generated `task_types` web only. So E2 has no plugin layer content.
- **Token estimates.** `sase memory list` uses `ceil(len(text) / 4)`
  (`src/sase/memory/inventory_references.py`). Use the same estimate and call it
  `tokens_est`.
- **sase-core conventions** (`sase repo open sase-core`; read its `AGENTS.md`):
  - Wire types go in `<module>/wire.rs` as `*Wire` structs, with a
    `<DOMAIN>_WIRE_SCHEMA_VERSION` const and getter.
  - Bindings are dicts in, dicts out, in a per-domain
    `crates/sase_core_py/src/<domain>/`. Recent example: `prose_diff.rs` plus
    `crates/sase_core_py/src/prose_diff/`.
  - No new root `pub use` and no `core_*` prelude aliases.
  - Add a "bindings are registered" test.
  - `finalizer/digest.rs` exposes a public `canonical_json_sha256`.
  - Cross-language golden fixtures are byte-identical copies in both repos, with a sase
    parity test (`tests/macro/test_macro_choice_projection_parity.py`).
  - Gate: `sase tool run check` in the sase-core checkout.
- **Pin moves.**
  - When one declaration commits both repos, the host commits sase-core first and writes
    its pushed SHA into `sase-core-revision.txt` (`docs/rust_backend.md`, "Moving the
    pin").
  - CI's `tools/check_sase_core_rs_bindings` fails if sase calls a binding the pin
    lacks.
  - `just check` rebuilds `sase_core_rs` from the linked checkout when its source
    changed.
  - Do not touch the `sase-core-rs` version range in `pyproject.toml`.
- **Architecture-test precedents:** `tests/test_proc_submission_static_invariants.py`
  and `tests/test_timezone_display_guard.py`, which use an AST walk with a
  `relpath:symbol` allowlist.
- **Known blocker for probes:** task `sase-1h2` (READY) is a circular import that broke
  LaunchApproval dispatch during E1's acceptance. Acceptance must not depend on probes
  alone.

## Design decisions (owned by this plan; phase workers do not re-litigate them)

1. **Shadow only.** E2 changes no provider argv, prompt, env-visible instruction file,
   or native file. The only runtime additions are:
   - files under `<artifacts>/instructions/`;
   - the bundle store and render cache under the SASE home;
   - an `agent_meta.json["instructions"]` summary;
   - the `SASE_INSTRUCTIONS_FILE` env var.

   Nothing reads manifests to make a decision. **The shadow render never fails an
   invocation.**

2. **Rust/Python split.** `sase-core` owns the instruction manifest: the closed
   vocabulary, wire types, validation invariants, and the `common_digest` definition.
   Those are the contract that E3 conformance, E4 history, E6 ledger, and E7 evaluator
   all consume. Markdown composition stays in Python: the memory renderer it reuses is
   entirely Python, and only Python needs composition today (launch and CLI preview).
   - **Reopen** when a non-Python front end (editor, web view) needs to preview a
     render. Then move composition into Rust rather than keep two copies.
   - E3's decision record records this split.
   - E2 moves the `sase-core-revision.txt` pin once, in `manifest-wire`.
3. **One generator, not two.** The compiler reuses the legacy renderer through a
   structured "memory units" API that the legacy `AGENTS.md` path is rebuilt on. Legacy
   output stays byte-identical. The compiler reads `memory-sase.template.md` directly
   and treats the generated `sase/memory/sase.md` notes (project and home) as
   **superseded input**: excluded, with reason `superseded_input`.
4. **Fact vocabulary (closed; frozen in the v1 wire).** The host sets facts. Project
   content cannot set or override them, and cannot select a lifecycle section for the
   wrong actor.

   | Fact       | Values                                                    | E2 source at the hook                                           |
   | ---------- | --------------------------------------------------------- | --------------------------------------------------------------- |
   | `actor`    | `sase_root` \| `native_helper` \| `interactive`           | always `sase_root`                                              |
   | `mode`     | `runtime` \| `interactive` \| `export`                    | always `runtime`                                                |
   | `purpose`  | `ordinary` \| `declaration_recovery` \| `conflict_repair` | per site                                                        |
   | `provider` | execution provider name (identifier string)               | the execution provider, never the requested one                 |
   | `project`  | project memory name or null                               | `project_memory_name(project_root)`; null outside a project     |
   | `host`     | short hostname                                            | `socket.gethostname()` up to the first `.`                      |
   | `vcs`      | VCS provider name or null                                 | `agent_meta.json["vcs_provider"]`, or detection for CLI renders |
   - **Valid combinations:** `mode=runtime` requires `actor` to be `sase_root` or
     `native_helper`; `mode=interactive` and `mode=export` require `actor=interactive`.
   - Monitor and gate are **not** purposes: they never call a provider.
   - Model, effort, agent name, and agent type are recorded in delivery identity, not
     used as facts.
   - Python validates `provider` against `registered_provider_names()`; Rust validates
     only its identifier shape.
   - In E2 only `actor`, `mode`, `provider`, and `project` change content. `host` and
     `vcs` are recorded for E5 and E7.

5. **Layers and section ids.** The fixed layer order is package → plugin → home →
   project → launch. Every byte of a bundle belongs to exactly one section:

   | Layer     | Section ids                                                                                                                                                                       | Source                                                          |
   | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
   | `frame`   | `frame.title`, `frame.core`, `frame.reference`, `frame.webs`                                                                                                                      | compiler-generated headings and the existing intro paragraphs   |
   | `package` | `pkg.sase.heading`, `pkg.sase.memory`, `pkg.sase.workspaces`, `pkg.sase.repos`, `pkg.sase.<slug>`, `pkg.root.final_declaration`, `pkg.helper.contract`, `pkg.provider.<provider>` | contract template, helper template, adapter directive           |
   | `plugin`  | `plugin.<dist>.<slug>`                                                                                                                                                            | reserved; no content in E2                                      |
   | `home`    | `home.core.<stem>`, `home.ref.<stem>`, `home.web.<stem>`                                                                                                                          | home memory at `Path.home()`, the same root audited lookup uses |
   | `project` | `proj.core.<stem>`, `proj.ref.<stem>`, `proj.web.<stem>`                                                                                                                          | the project root's memory (the workspace checkout at runtime)   |
   | `launch`  | `launch.<slug>`                                                                                                                                                                   | reserved; no content in E2 (`corpus-before-mechanism`)          |
   - Template H2 headings map to fixed ids: "SASE Memory" → `pkg.sase.memory`, the
     "Ephemeral … Workspace Directories" heading → `pkg.sase.workspaces`, "Repositories"
     → `pkg.sase.repos`, "SASE Final Declaration" → `pkg.root.final_declaration`.
   - Any other H2 in an override template (`memory.sase_template`, or the home user
     template) gets `pkg.sase.<slug of heading>` and is lifecycle-neutral.
   - **Repositories** are one section: project entries first, then home-only entries
     tagged as home configuration. Duplicate names collapse; the project wins.
   - **Shadowing** matches audited lookup: a project flat note shadows a home note with
     the same relative path. The home section is recorded as excluded with reason
     `shadowed`, plus `shadowed_by`.

6. **Bundle layout** (Markdown; no positional heading numbers, so overlays and providers
   don't renumber the document):

   ```
   # <project H1 title, or home title, or "SASE Agent Instructions">   frame.title
   Home: <home H1 title>     (one line, only when a home layer contributes)
   ## Core Memory + intro                                              frame.core
   ### SASE = Structured Agentic Software Engineering (sase)            pkg.sase.heading
   #### SASE Memory                                                     pkg.sase.memory
   #### Ephemeral `<project>_<N>` Workspace Directories                 pkg.sase.workspaces   (project only)
   #### Repositories                                                    pkg.sase.repos
   #### SASE Final Declaration                                          pkg.root.final_declaration   (runtime + sase_root)
   #### SASE Helper Instructions                                        pkg.helper.contract          (runtime + native_helper)
   #### <Provider> Single-Turn Instructions                             pkg.provider.<provider>      (runtime + sase_root, when a directive exists)
   ### <note H1> (<stem>)                                               home.core.*, then proj.core.*
   ## Reference Memory + intro                                         frame.reference
   <one entry per note; home paths shown as ~/sase/memory/…>            home.ref.*, then proj.ref.*
   ## Memory Webs + intro                                              frame.webs
   ### <web title> (<stem>)                                             home.web.*, then proj.web.*
   ```

   - Within each kind group, layer order applies: home before project.
   - Empty groups omit their frame section.
   - Section bytes are `text.rstrip("\n") + "\n\n"`, and the bundle is their
     concatenation, so offsets are contiguous.
   - Each section is formatted with the same Markdown formatter and print width as
     today's generated files.
   - **Never embed absolute paths, workspace names, timestamps, or hostnames in bundle
     bytes.** Identical inputs from any workspace must give identical bytes, so bundles
     deduplicate across `sase_<N>` workspaces.

7. **Lifecycle overlays** (each included section carries `lifecycle`: `neutral` | `root`
   | `helper`):

   | Render                      | Adds                                                             | Excludes (recorded reason)                                                 |
   | --------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------- |
   | `runtime` + `sase_root`     | `pkg.root.final_declaration` (root), `pkg.provider.<p>` (root)   | `pkg.helper.contract` (`overlay`)                                          |
   | `runtime` + `native_helper` | `pkg.helper.contract` (helper; body from the E1 helper template) | final declaration and provider directive (`overlay`)                       |
   | `interactive`               | nothing                                                          | every root and helper section (`mode`)                                     |
   | `export`                    | nothing                                                          | every root and helper section, every `home.*` section, `launch.*` (`mode`) |
   - Only the root render contains the `SASE Final Declaration` heading. E1's contract
     fingerprint counts it exactly once there.
   - `export` is the frozen vocabulary slot E4 builds on. E4 owns the stub's budget and
     bootstrap clause; E2 only guarantees that export carries no turn obligations and no
     home content.
   - Rendering for `native_helper` is preview-only in E2. The hook renders roots only;
     E3 adds the helper slot.

8. **Instruction manifest v1** (wire owned by `manifest-wire`; names are exact; this is
   a shape sketch, not a byte fixture):

   ```jsonc
   {
     "schema_version": 1,
     "compiler": { "name": "sase-instructions", "version": 1, "sase_version": "…" },
     "facts": { "actor": "sase_root", "mode": "runtime", "purpose": "ordinary",
                "provider": "codex", "project": "sase", "host": "athena", "vcs": "github" },
     "bundle": { "sha256": "<64 hex>", "common_digest": "<64 hex>", "bytes": 18742,
                 "lines": 310, "tokens_est": 4686, "store_path": "/…/bundles/ab/<sha>.md" },
     "budget": { "by_layer": { "frame": {"bytes": 0, "tokens_est": 0}, "package": {…}, "home": {…}, "project": {…} } },
     "sections": [
       { "id": "proj.core.gotchas", "layer": "project", "status": "included",
         "lifecycle": "neutral", "required": false, "provider_specific": false,
         "offset": 1234, "length": 410, "sha256": "<64 hex>", "tokens_est": 103,
         "sources": [ { "scope": "project", "kind": "memory_note",
                        "path": "sase/memory/gotchas.md", "sha256": "<64 hex>", "blob_oid": "<40|64 hex>" } ] },
       { "id": "proj.core.sase", "layer": "project", "status": "excluded",
         "reason": "superseded_input", "sources": [ … ] }
     ],
     "delivery": { "status": "shadow", "channel": null, "invocation_id": "<uuid4>",
                   "invocation_seq": 0, "attempt": 1, "agent_name": "…", "agent_type": "…",
                   "model": "…", "parent_invocation_id": null, "session_ids": [],
                   "provider_cli_version": null, "rendered_at": "<RFC 3339>",
                   "render_ms": 12.4, "cache": "hit" },
     "observation": { "status": "unobserved" }
   }
   ```

   - **Closed enums** (snake_case):
     - layer: `frame` `package` `plugin` `home` `project` `launch`
     - section status: `included` `excluded`
     - reason: `superseded_input` `shadowed` `overlay` `mode` `no_directive` `empty`
     - lifecycle: `neutral` `root` `helper`
     - source scope: `package` `plugin` `home` `project` `launch`
     - source kind: `package_template` `helper_template` `provider_directive`
       `memory_note` `memory_web` `memory_strands` `config` `generated`
       `legacy_fallback`
     - delivery status: `preview` `shadow` `explicit` `inherits_native` `inherits_root`
       `none`. The last four are frozen now for E3, unused in E2.
     - cache: `hit` `miss` `bypass`
     - observation status: `unobserved` `observed` `partial` `unavailable`
   - **Strictness:**
     - Every wire struct uses `deny_unknown_fields`.
     - Readers accept `schema_version` in `1..=current`.
     - Adding a field or an enum value bumps the version, and that bump is a breaking
       `feat!:` in sase-core.
   - **Invariants enforced by `normalize_instruction_manifest`:**
     - Section ids are unique, match
       `^(frame|pkg|plugin|home|proj|launch)(\.[A-Za-z0-9_-]+)+$`, and the prefix
       matches `layer`. The `pkg.` prefix means `package`; `proj.` means `project`.
     - Included sections have contiguous `offset`/`length` from 0 that sum to
       `bundle.bytes`. Excluded sections have no offset or length and do have a
       `reason`.
     - Digests are lowercase hex: sha256 is 64 characters; blob OIDs are 40 or 64.
     - Fact combinations are valid (decision 4).
     - A `root` section appears only for `runtime` + `sase_root`. A `helper` section
       appears only for `runtime` + `native_helper`.
     - A `provider_specific` section is exactly `pkg.provider.<facts.provider>`.
     - A `required` section is never excluded, except by `overlay` or `mode`.
   - **`common_digest`** is sha256 of the canonical JSON array `[[id, sha256], …]` over
     included sections with `provider_specific == false`, in bundle order. Use the
     sorted-key, no-trailing-newline canonical form from `finalizer/digest.rs`.
     `normalize` fills it in when it is null and errors when a supplied value differs.
     Equal `common_digest` across providers means "same instructions except the provider
     section".

9. **Artifacts layout.**
   - Each root invocation writes `<artifacts>/instructions/NN-<provider>.md` (the
     bundle) and `NN-<provider>.json` (the normalized manifest, `indent=2`, sorted
     keys). `NN` is a two-digit per-run sequence. Allocate it with `O_EXCL` on the `.md`
     and retry on a collision.
   - The bundle is also written once to a content-addressed store:
     `<sase home>/instructions/bundles/<sha[:2]>/<sha>.md`, mode 0444, written
     atomically and skipped if present. The render cache is the store's first reader.
   - A shadow failure writes `NN-<provider>.error.json` (exception type, message,
     `rendered_at`) best-effort, and the invocation continues.
   - `agent_meta.json["instructions"] = {"schema_version": 1, "count": n, "latest": {"seq", "provider", "purpose", "sha256", "common_digest", "manifest": "instructions/NN-<provider>.json"}}`.
   - `SASE_INSTRUCTIONS_FILE` holds the `.md` path during the call and is restored
     afterwards.
   - `instructions/` is not moved by `snapshot_attempt`. The manifest's `attempt` is
     `1 + count(attempts/*)`.
10. **Render cache.**
    - **Key:** sha256 over
      - `COMPILER_VERSION`;
      - the sase version;
      - a per-process code fingerprint: path, mtime, and size of every `.py` file in
        `sase.instructions`, `sase.amd`, `sase.memory`, and `sase.main.init_memory`,
        found via `importlib.util.find_spec` without importing;
      - the facts that change content;
      - the sha256 of the provider directive text;
      - the sorted `(relative path, content sha256)` of every input file in the
        enumerated input classes:
        - project and home `sase/memory/**` (legacy `memory/` too);
        - project config files;
        - global config `sase.yml` and its `sase_*.yml` overlays;
        - the resolved template files, including overrides;
        - the helper template;
        - the project root `AGENTS.md` (the legacy description fallback);
        - the installed task-type plugin distributions and their versions.
      - The key holds **no absolute roots**, so the cache is shared across workspaces.
    - **Entry:** `<sase home>/instructions/cache/<key>.json` holds the compiled section
      table. The bundle bytes live in the store. A missing or corrupt entry or store
      blob is a miss.
    - **Hit path:** it must not import `sase.amd`, `sase.main.init_memory`, or
      `sase.memory.web`. That is what keeps warm renders cheap.
    - **Bounds:** prune to the newest 512 entries on write. The store is not pruned in
      E2; watch its size.
    - **Budget:** warm (hit) render p95 ≤ **250 ms**, including key computation. Every
      manifest records `render_ms` and `cache`.
    - **Correctness invariant (tested):** every source path recorded in a manifest
      belongs to an input class in the key.
11. **Kill switch.** One `sunset` flag, `instruction_shadow_render` (default on),
    created with `sase flag new instruction_shadow_render -k sunset …`.
    - **Off:** invocations do exactly what they do today: no `instructions/` files, no
      `agent_meta` summary, and no `SASE_INSTRUCTIONS_FILE`.
    - **Remove when:** E3 makes the rendered bundle the delivered instructions (the
      render can no longer be skipped), or the E2 readout shows zero shadow failures and
      warm p95 within budget for 7 days.
    - Rollback is one `sase flag disable instruction_shadow_render` per execution host
      (athena, apollo, mac).
    - No `beta` flags.
12. **CLI shape** (per `cli_rules`). The group is `sase instructions`, with subcommands
    `list` (default), `render` (new), and `verify`. Options are sorted, and every long
    option has a short alias.
    - The research's `render --as <agent|facts>` becomes `-a/--agent NAME` plus a
      repeatable `-f/--fact KEY=VALUE`. They combine: start from an agent's facts and
      override one.
    - Coverage is `verify -c/--coverage`.
    - There is no doctor subcommand; a deep check `instructions.coverage` joins the
      existing `instructions` doctor group.
13. **Shared literals.** Phases that run in parallel use these exact strings:

    | Literal                         | Value                                                                        | Owner             |
    | ------------------------------- | ---------------------------------------------------------------------------- | ----------------- |
    | Manifest dir                    | `instructions/` under the artifacts dir                                      | `invocation-hook` |
    | Bundle / manifest / error names | `NN-<provider>.md` / `.json` / `.error.json`                                 | `invocation-hook` |
    | agent_meta key                  | `instructions`                                                               | `invocation-hook` |
    | Env var                         | `SASE_INSTRUCTIONS_FILE`                                                     | `invocation-hook` |
    | Flag key                        | `instruction_shadow_render`                                                  | `invocation-hook` |
    | Compiler name                   | `sase-instructions`                                                          | `compiler`        |
    | Bindings                        | `instruction_manifest_wire_schema_version`, `normalize_instruction_manifest` | `manifest-wire`   |
    | Contract fingerprint            | `SASE Final Declaration` (E1 `CONTRACT_HEADING`)                             | existing          |
    | Helper marker                   | `# SASE Helper Instructions` (E1)                                            | existing          |

14. **Memory edits: none.**
    - No phase creates, edits, or deletes a memory file or regenerates a tracked
      instruction file. The contract template is not edited.
    - If a phase finds memory that should change, it records a `PROPOSED FOLLOW-UP:`
      note on its own phase bead.
    - Phase workers record discovered work as `PROPOSED FOLLOW-UP:` notes and do not
      create beads, except the flag bead that `sase flag new` creates.
15. **The home layer is read, never written.** No phase edits `~/AGENTS.md`, the chezmoi
    source, or any other project's files. bob-cli and actstat are opened read-only
    through `sase repo open`.

## Phase details

### memory-units (medium)

- **Goal.** One structured, side-effect-free API that both the legacy `AGENTS.md`
  renderer and the new compiler consume. The legacy output stays byte-identical.
- **API.** Add a small module, for example `src/sase/amd/memory_units.py`, that returns
  a frozen `MemoryRootUnits` for one root:
  - `title`: the resolved H1, or None.
  - `contract_inputs`: the resolved contract template path (including overrides),
    `project_name`, and the structured linked entries (name, description, origin config
    path).
  - `core`: ordered core units, each with stem, relative path, note H1, the
    `inline_memory_section` text, source path, and source bytes. The generated `sase.md`
    is flagged `generated_contract=True`.
  - `references`: ordered top-level reference units, each with stem, relative path, the
    resolved description, whether it used the legacy `AGENTS.md` fallback, and a
    per-entry render with no positional number.
  - `webs`: ordered web units, each with stem, title, the rendered `### Title (stem)`
    block (roster included), the descriptor source, and the strand files that fed the
    roster.
  - `intros`: the existing Core, Reference, and Webs intro texts.
- **Rebuild `_render_managed_agents` on these units.** Today's `AGENTS.md`, shims, home
  render, and chezmoi `.tmpl` path must stay byte-identical.
- **Thread `project_root` explicitly** through the hidden `Path.cwd()` reads that the
  units path touches: config path resolution, the required-plugin set for task types,
  and the `plan_amd_memory_sync` default. A compile from any cwd then sees the same
  inputs. Keep each CLI's cwd defaulting.
- **Expose the home-root choice** as a parameter. Memory init keeps the chezmoi source;
  the compiler passes `Path.home()` (decision 5).
- **Tests:**
  - A golden test: for a tmp fixture project and home (via
    `tests/main/init_memory_handler_helpers.py`), the units-based legacy render is
    byte-identical to the pre-refactor output. Capture the expected text in the test
    before refactoring.
  - Units are identical when computed from a different cwd.
  - The fallback flag is set when a reference note lacks `description:`.
  - The existing init-memory suites and `test_init_memory_committed_drift.py` stay
    green.
  - `sase memory init --check` is clean for this repo.
- **Done when:** `just check` passes, and no tracked file changes when
  `.venv/bin/sase memory init --check` runs.

### manifest-wire (medium; crosses sase-core)

- **Open sase-core** with `sase repo open sase-core -r "<why>"`, read its `AGENTS.md`,
  and work only in the printed path. This phase has commit obligations in **both**
  repos.
- **sase-core:**
  - Add module `crates/sase_core/src/instruction_manifest/`:
    - `mod.rs`: `mod` and `pub use` lines only.
    - `wire.rs`: decision 8's types,
      `INSTRUCTION_MANIFEST_WIRE_SCHEMA_VERSION: u32 = 1`, and
      `instruction_manifest_wire_schema_version()`.
    - `normalize.rs`: parsing, invariants, `common_digest`, and an
      `InstructionManifestError` (thiserror, with `UnsupportedSchema(u32)`).
    - `tests.rs`.
  - Register `pub mod instruction_manifest;` in `lib.rs`. No root `pub use`.
  - Reuse or promote the canonical-JSON sha256 helper; do not add another copy.
  - Fixtures (`include_str!`): valid root, helper, and interactive manifests, plus one
    invalid fixture per invariant in decision 8.
- **Binding:** add domain `crates/sase_core_py/src/instruction_manifest/` with
  `instruction_manifest_wire_schema_version()` and
  `normalize_instruction_manifest(manifest: dict) -> dict`. It raises `ValueError` with
  the invariant that failed. Add a registered-bindings test and a round-trip test.
  Register it in `crates/sase_core_py/src/lib.rs`.
- **Golden cross-language fixture:** a valid manifest JSON stored byte-identically at
  `crates/sase_core/tests/fixtures/instruction_manifest_v1.json` (sase-core) and
  `tests/fixtures/instruction_manifest_v1.json` (sase). Add a sase parity test modeled
  on `test_macro_choice_projection_parity.py`, which skips when no core checkout is
  present.
- **sase adapter:** `src/sase/core/instruction_manifest.py`.
  - Mirror constants: the schema version, plus tuples for every closed enum in decision
    8 and the fact names and values in decision 4.
  - `normalize_instruction_manifest(dict) -> dict` wraps the binding via
    `require_rust_binding` and raises `InstructionManifestError(ValueError)`.
  - Tests:
    - the schema version equals the binding's;
    - the golden fixture normalizes unchanged;
    - each enum tuple accepts its values and rejects a typo (`provider_specific` on a
      non-provider id, `actor: codx`);
    - `common_digest` is filled when null, rejected when wrong, and unchanged by a
      provider-only change.
- **Docs:** add the shipped operation and adapter rows to `docs/rust_backend.md`.
- **Verify:** `sase tool run check` in the sase-core checkout and `just check` in sase.
  `just check` rebuilds `sase_core_rs` from the linked checkout.
- **Pin:** commit both repos in this phase's declaration, so the host commits sase-core
  first and moves `sase-core-revision.txt`. Confirm afterwards that
  `tools/check_sase_core_rs_bindings` passes against the new pin. If the automatic move
  did not happen, run `just ratchet-core-revision` after the sase-core commit lands.

### compiler (medium; depends on memory-units, manifest-wire)

- **Package.** Add these modules under `src/sase/instructions/`. Keep each under the
  repo's file-size gate.
  - `facts.py`: `InstructionFacts` (frozen), parsing and validation per decision 4, and
    defaults: `sase_root`, `runtime`, `ordinary`, and the configured default provider,
    with project, host, and vcs detected from the given project root.
  - `directives.py`: `provider_directive(provider, *) -> str | None`. It lazily imports
    the adapter symbols listed in the verified facts, and selects Muse's mode the way
    the adapter does. It is not a copy of the text.
  - `compile.py`:
    `compile_bundle(facts, *, project_root, home_root, use_cache=True) -> CompiledBundle`.
    That carries the bundle bytes, the ordered sections (included and excluded, as in
    decision 8), totals, `render_ms`, and the cache result.
  - `manifest.py`: `build_manifest(compiled, facts, delivery) -> dict`. It assembles the
    wire dict, calls `normalize_instruction_manifest` (which fills `common_digest`), and
    returns the normalized dict.
  - `cache.py`: decision 10, plus the content-addressed bundle store (decision 9 path).
- **Composition** follows decisions 5–7 exactly. Package sections come from rendering
  the contract template once, with:
  - `project_name` from the project root;
  - the merged linked entries from project and home;
  - a split on H2 headings into the fixed ids.

  Core, reference, and web sections come from `memory-units` for the project root and
  for the home root (`Path.home()`). Then apply shadowing, then exclusions with reasons.

- **Out of scope:** no truncation and no budget gate. Record the bytes and `tokens_est`
  per section and per layer only.
- **Docs:** create `docs/instruction_bundles.md` and add it to the `mkdocs.yml` nav. It
  covers the concepts (bundle, manifest, layers, section ids, overlays, facts,
  `common_digest`), the cache and store, the Rust/Python split, and that E2 is shadow
  only. Link it from `docs/agent_providers.md`. `render-cli` and `invocation-hook`
  append their own sections; expect a trivial rebase between them.
- **Tests** in `tests/instructions/`, on tmp fixture projects and homes:
  - Identical inputs give byte-identical bundles and manifests, ignoring `delivery`
    timing, across two cwds and two project-root paths with the same content.
  - Codex and Grok renders differ only in `pkg.provider.*`, and their `common_digest` is
    equal.
  - Only the root render contains `SASE Final Declaration`. Only the helper render
    contains `# SASE Helper Instructions`. The interactive and export renders contain
    neither.
  - Export has no `home.*` included.
  - Facts cannot attach a lifecycle section to the wrong actor: invalid combinations are
    rejected by both Python validation and Rust normalize.
  - A project note shadows a same-path home note, which is recorded as `shadowed`. The
    generated `sase.md` is excluded as `superseded_input`.
  - A missing home memory dir gives an explicit absence, not an error.
  - Fact parsing rejects unknown keys and values, with the valid choices in the message.
  - **Cache:**
    - A hit returns identical bytes.
    - Mutating any input class causes a miss.
    - A corrupt entry or blob causes a miss.
    - The hit path imports none of the heavy modules; check this in a subprocess via
      `sys.modules`.
    - Every manifest source belongs to a key input class.
  - **Latency:** a test, marked like the repo's other timing tests, does 20 warm renders
    of a realistic fixture and asserts p95 ≤ 250 ms.
  - Compiling writes nothing under the project root (check `git status --porcelain` in a
    tmp git repo).
- **Done when:** `just check` passes.

### render-cli (medium; depends on compiler)

- **CLI:** add a `render` subcommand in `src/sase/main/parser_instructions.py`, with the
  handler in `src/sase/main/instructions_handler.py`. Use `RawDescriptionHelpFormatter`
  with examples. Options, alphabetical:
  - `-a/--agent NAME`: start from that agent's latest manifest facts. For pre-E2 runs,
    derive them from `agent_meta.json` (provider = `exec_llm_provider` or
    `llm_provider`, actor `sase_root`, mode `runtime`, purpose `ordinary`). Reuse
    `verify -a`'s agent lookup. Sources are always the **current** project and home.
    stderr then says whether the bundle sha256 matches the agent's recorded bundle and,
    if it doesn't, lists the changed section ids.
  - `-f/--fact KEY=VALUE`: repeatable; also accepts comma-separated pairs. Setting
    `mode=interactive` or `mode=export` implies `actor=interactive`; a conflicting
    explicit actor is an error.
  - `-j/--json`: print the normalized manifest (`delivery.status: preview`) instead of
    the bundle.
  - `-N/--no-cache`: bypass the cache (`cache: bypass`).
  - `-p/--parity`: compare against the legacy native files (see below). Print a table,
    and exit 1 on any missing unit or a contract count other than one.
  - `-s/--sections`: a Rich table of id, layer, status or reason, lifecycle, bytes,
    `tokens_est`, and source, instead of the bundle.
  - **Default output:** bundle Markdown, raw, to stdout. One summary line goes to
    stderr: sha256 prefix, `common_digest` prefix, section count, bytes, `tokens_est`,
    cache, and ms.
  - Add completion for agent names and fact keys and values, refresh the completion
    snapshot (`just sync-completion-spec`), and update
    `tests/main/test_parser_command_help.py`.
- **Parity:** `src/sase/instructions/parity.py`,
  `legacy_parity(compiled, project_root, home_root) -> ParityReport`.
  - Parse the legacy root `AGENTS.md` and `~/AGENTS.md` with
    `parse_amd_agents_document`.
  - Every legacy core path, reference path, and web path must map to an included or
    shadowed bundle section. The legacy `sase.md` maps to the `pkg.sase.*` and
    `pkg.root.final_declaration` sections.
  - Every repository name in either legacy Repositories list must appear in
    `pkg.sase.repos`.
  - For root renders, exactly one included section contains `SASE Final Declaration`.
  - Report extra bundle sections (for example `pkg.provider.*`) as additions, not
    failures.
- **CI parity tests:**
  - Build three fixture project shapes plus a home with the init-memory test helpers,
    and generate their legacy files with the real memory-init code in tmp. Never commit
    generated fixtures.
    - **sase-like:** a configured title, several core notes, reference notes with a
      parent child, three webs including `task_types`, and linked repos.
    - **bob-cli-like:** a derived title, `sase.md` only, references, and the
      `decisions`, `glossary`, and `task_types` webs.
    - **actstat-like:** a derived title, `sase.md`, the generated references, and only
      `task_types`.
    - **home:** an H1 title, the `sase.md` core note, two reference notes, and a home
      linked repo.
  - Each fixture must pass `legacy_parity` for the root render.
  - A real-repo test, modeled on `test_init_memory_committed_drift.py`, renders this
    checkout's project layer with an isolated empty home and passes parity against the
    committed `AGENTS.md`. `git status --porcelain` must be unchanged afterwards.
- **Docs:**
  - A "Previewing bundles" section in `docs/instruction_bundles.md`.
  - `docs/cli.md` instructions rows.
  - `docs/configuration.md` CLI flag tables.
- **Done when:** `just check` passes, and `sase instructions render -p` passes in this
  checkout.

### invocation-hook (medium; depends on compiler)

- **Boundary.** Add `src/sase/llm_provider/_instruction_boundary.py` with
  `invoke_with_instructions(provider, prompt, *, purpose, artifacts_dir, provider_name=None, agent_type=None, **invoke_kwargs)`.
  - It is the **only** place that calls `provider.invoke` for a root invocation.
  - With the flag on and an `artifacts_dir`, it builds facts (decision 4), runs
    `compile_bundle`, and builds the manifest with `delivery.status: shadow`. It writes
    the files (decision 9), updates the `agent_meta` summary, and sets
    `SASE_INSTRUCTIONS_FILE`. Then it calls `provider.invoke` and restores the env in
    `finally`.
  - **Every** exception from the shadow work is caught. It logs one warning, writes an
    `.error.json`, and the call proceeds. A `KeyboardInterrupt` or `SystemExit` is not
    swallowed.
  - With the flag off or no `artifacts_dir`, it calls `provider.invoke` directly.
  - Declare `SASE_INSTRUCTIONS_FILE` in `src/sase/env_contracts.py`.
- **Call sites.** Route all three sites through it:
  - `_invoke.py`, purpose `ordinary`, passing the execution provider label;
  - declaration recovery, purpose `declaration_recovery`;
  - conflict repair, purpose `conflict_repair`.

  Keep `finalizer_owned_turn()`, nonces, and every existing argument exactly as they
  are.

- **Reader.** Add `src/sase/instructions/manifests.py`,
  `read_run_manifests(artifacts_dir) -> list[RunManifest]`. It is sorted by `NN`,
  tolerant of missing and corrupt files (returns them flagged), and validates through
  the adapter. `scoreboard` consumes it; round-trip test it against the writer here.
- **Flag.** Create `instruction_shadow_render` with `sase flag new -k sunset`, using
  decision 11's three sentences, and paste the registry entry. The Off branch is today's
  behavior exactly.
- **Architecture test** in `tests/instructions/test_invoke_boundary.py`:
  - AST-scan `src/sase/**/*.py` for calls whose function is an attribute named `invoke`.
  - Fail on any call outside the boundary module, except the allowlisted
    `relpath:function` entries. Each entry needs a one-line reason: the adapters'
    `llm_invoke` self-delegations, plus any unrelated `.invoke` receivers the scan
    finds.
  - Also assert that the three sites call `invoke_with_instructions`.
- **Route tests** (fake provider, tmp artifacts dir):
  - **Ordinary:** `00-<p>`, purpose `ordinary`.
  - **Declaration recovery:** the next sequence number, purpose `declaration_recovery`.
  - **Conflict repair:** purpose `conflict_repair`.
  - **Fallback to another provider:** the retry path with `SASE_MODEL_OVERRIDE`
    resolving a different provider gives a second manifest with that provider, and its
    `attempt` increments.
  - **No `artifacts_dir`:** no files and no error.
  - **Flag off:** no files, no meta key, no env var (the both-states test).
  - **Compiler raises:** the invocation still happens, `.error.json` exists, and no
    partial `.json` exists.
  - **Env:** `SASE_INSTRUCTIONS_FILE` is set during the call and restored after.
  - **Meta:** the `agent_meta` summary counts manifests and points at the latest.
  - **Legacy evidence:** `capture_instruction_snapshot` behavior is unchanged.
- **Docs:**
  - A "Shadow manifests" section in `docs/instruction_bundles.md`: artifacts layout, env
    var, kill switch, and failure posture.
  - A note in `docs/agent_providers.md` "Instruction delivery" that E2 records but does
    not deliver.
- **Done when:** `just check` passes. Then run one local smoke invocation through the
  fake provider path, and record its manifest path and `render_ms` (cold and warm) as a
  phase bead note.

### scoreboard (medium; depends on invocation-hook)

- **Coverage column.** `sase instructions verify` adds `manifest` (`k/N`) to every
  provider row.
  - `N` is the observed root sessions in the scored runs.
  - `k` counts the sessions whose run has a manifest for the same execution provider
    with `rendered_at` ≤ the session start, allowing 5 s of skew.
  - For agy, which has no observable sessions, `N` is runs and `k` is runs with at least
    one agy manifest.
  - **Every E1 column keeps its exact meaning and value.** The existing baseline and
    after-state fixture tests must pass unchanged.
- **`-c/--coverage`.** Prints a coverage table per provider and purpose: manifests,
  sessions, covered, and up to 10 uncovered `(agent, session)` pairs. Also lists
  shadow-failure counts (`.error.json`) and the warm and cold `render_ms` p50/p95 from
  manifests in the window.
- **Section diff.** With `-a NAME`, show an intended-vs-observed table for each observed
  root session and its matching manifest (same provider, latest `rendered_at` ≤ session
  start).
  - Rows are included sections. Columns: id, layer, observed count, and channels
    (`native`, `explicit`).
  - The observed count is the number of loaded sources whose whitespace-flattened text
    contains the section's flattened body, with its heading line stripped. Use the
    `flatten_ws` helper from `src/sase/instructions/fingerprints.py`.
  - Frame and heading-only sections are skipped.
  - Status is `◌` when the observation is partial or unavailable (agy).
  - The parsers keep loaded source texts in memory only when a diff is requested.
- **JSON.** Keep `schema_version: 1` (the changes are additive). Add per-row `coverage`,
  a top-level `coverage` block for `-c`, and `section_diff` per observation for `-a`.
- **Doctor.** Add deep check `instructions.coverage` in
  `src/sase/doctor/checks_instructions.py`:
  - Newest 10 runs per provider over 24 h.
  - WARN, naming them, on uncovered sessions in runs that started after the earliest
    manifest in the window.
  - WARN on any `.error.json`.
  - SKIP when the window has no manifests.
  - Add the id to the deep set in `tests/main/test_doctor_command.py`.
- **Fixtures:** extend `tests/instructions/fixtures/` with synthetic artifacts dirs.
  Each holds a manifest and bundle written by the real writer in a test helper, next to
  the existing session fixtures. Cover:
  - **Full coverage:** E1-baseline shapes, where the diff shows Claude contract sections
    observed 2×, Muse home sections 0, and Grok home sections 0.
  - **An uncovered session.**
  - **A fallback** that covers two providers.
  - **An agy run.**
- **Docs:**
  - `docs/agent_providers.md` "Verifying instruction delivery": the coverage column,
    `-c`, and the diff.
  - `docs/cli.md` and `docs/configuration.md` flag rows.
  - Refresh the completion snapshot.
- **Done when:** `just check` passes and `sase instructions verify -c --since 24h` runs
  against real data.

### acceptance (small; depends on render-cli, scoreboard)

- **Precondition.**
  - Find the primary checkout with `sase repo list`. Confirm with
    `git merge-base --is-ancestor` that it contains the `invocation-hook`, `scoreboard`,
    and `render-cli` commits.
  - Note when `invocation-hook` landed (`--since` below).
  - If the commits are missing, note that on the epic and stop.
- **Live coverage.**
  - Run `sase instructions verify -c -j --since <invocation-hook landing>`.
  - Every provider with runs since landing must show coverage `N/N`, including any
    recovery or repair manifests present, with zero `.error.json`.
  - If Claude, Codex, Grok, or Muse has no run since landing, use `/sase_run` to submit
    one LaunchApproval for xsmall `+sase` probes, pinned to that provider's cheapest
    model. The probe replies `OK` and finishes with `/sase_final`. Preflight with
    `sase macro expand`.
  - If dispatch fails (known `sase-1h2`), record it, keep the providers that do have
    evidence, and list the missing ones as a declared gap. Do not invent coverage.
- **Observed columns unchanged.** Compare the observed columns of
  `sase instructions verify -j --since <landing>` with E1's
  `e1_after_scoreboard_v2.json` (attached to `sase-1gu`). Explain any difference; only a
  provider-version change is acceptable.
- **Render checks**, from the primary checkout:
  - `diff <(sase instructions render -f provider=codex) <(sase instructions render -f provider=grok)`
    shows only the provider section, and both `-j` manifests share `common_digest`.
  - The root, helper, interactive, and export renders meet decision 7. Use `grep -c` for
    `SASE Final Declaration` and `# SASE Helper Instructions`.
  - Two renders compare equal with `cmp`.
  - `sase instructions render -a <a recent agent>` reports "matches" for an agent
    launched after the last memory change.
- **Live parity** in all three projects.
  - Run `sase instructions render -p` in the sase primary checkout, and in read-only
    bob-cli and actstat checkouts opened with `sase repo open <project> -r "<why>"`.
  - Run `git status --porcelain` before and after in each; the output must be identical.
    Modify nothing.
- **Latency.** Report warm and cold p50/p95 of `render_ms` from the live manifests since
  landing (via `verify -c`), plus 20 local warm renders. Warm p95 must be ≤ 250 ms.
- **Budget baseline.** Render the matrix as `-j` manifests for the sase project:
  - root × {claude, codex, grok, muse, agy};
  - helper;
  - interactive;
  - export.

  Collect bytes, lines, and `tokens_est` per layer and per section into
  `instructions_budget_baseline.json`. No truncation or gate; E7 ratchets from this.

- **Acceptance record.** Write a short Markdown file containing:
  - source revisions for sase and sase-core, and the pin;
  - provider CLI versions;
  - the commands run;
  - expected versus observed results for every exit criterion;
  - latency numbers;
  - declared gaps.

  Attach it, the coverage JSON, and the baseline JSON to the epic bead with
  `sase bead attach <epic-id> <file> -N <name> -n "<summary>"`.

- If a criterion fails because the scoreboard or render CLI is wrong, fix it in this
  phase. Record any other failure as `DISCOVERED ISSUE:` on this phase bead, leave it
  open, and name the failed criterion.

## Exit criteria (checked at landing)

- [ ] `just check` passes in sase, and `sase tool run check` passes in sase-core.
      `sase-core-revision.txt` includes the `manifest-wire` commit, and the CI binding
      check passes.
- [ ] `diff <(sase instructions render -f provider=codex) <(sase instructions render -f provider=grok)`
      shows only `pkg.provider.*`, and the two `-j` manifests have equal
      `common_digest`.
- [ ] Root, helper, interactive, and export renders: only root contains
      `SASE Final Declaration`; only helper contains `# SASE Helper Instructions`;
      export includes no `home.*` section. Python fact validation and Rust normalize
      both reject a lifecycle section for the wrong actor.
- [ ] The architecture test passes. Route tests cover ordinary, declaration recovery,
      conflict repair, fallback to another provider, no artifacts dir, flag off, and
      compiler failure.
- [ ] Parity passes in CI for the sase-, bob-cli-, and actstat-shaped fixtures and for
      this repo's committed `AGENTS.md`, and live (`render -p`) in all three projects.
      Rendering changes no tracked file.
- [ ] Identical inputs give byte-identical bundles. Warm p95 ≤ 250 ms in the benchmark
      test and in live manifests, and every manifest records `render_ms` and `cache`.
- [ ] `sase instructions verify` shows the `manifest` coverage column. Coverage since
      the hook landed is `N/N` for every provider that ran, with zero `.error.json`, and
      E1's observed columns are unchanged.
- [ ] `sase flag list` shows the `instruction_shadow_render` sunset flag, with
      both-state tests.
- [ ] `sase memory init --check` is clean, and no tracked instruction or memory file
      changed in this epic.
- [ ] `sase doctor -D -C instructions` runs and includes `instructions.coverage`.
- [ ] The acceptance record, coverage JSON, and budget baseline JSON are attached to the
      epic bead.

## Watch metrics (after landing; not landing criteria)

- Shadow failures: the `.error.json` count should stay 0. Any non-zero count needs a
  fix, or `sase flag disable instruction_shadow_render`.
- Warm and cold `render_ms`, and the end-to-end launch latency of agents.
- Manifest coverage stays `N/N` as new providers and routes appear.
- Bundle store size under the SASE home (E2 does not prune it).
- The E1 watch metrics continue unchanged: Grok final-declaration rate, and Claude
  helper attempts and acceptances.

## Confirm it yourself (about 10 minutes, after deployment)

```bash
sase instructions render | head -40                      # what a root agent here would get
sase instructions render -a <recent-agent> -s            # its sections, and "matches" on stderr
diff <(sase instructions render -f provider=codex) <(sase instructions render -f provider=grok)
sase instructions verify -c --since 24h                  # manifest coverage N/N
sase instructions render -p                              # parity with today's AGENTS.md + ~/AGENTS.md
```

## Expected scoreboard after this epic

Every E1 column is unchanged. New: `manifest` reads `N/N` for each provider that ran
since landing. `-a <agent>` shows the intended-vs-observed diff:

- Claude and Codex: contract sections 2×.
- Muse and Grok: home sections 0.
- agy: `◌`.

## Rollback

Run `sase flag disable instruction_shadow_render` on each execution host, or revert the
hook commit. Nothing reads manifests, so rolling back changes no agent behavior. The
`memory-units` refactor is byte-identical by test. The wire module and the pin are
additive.

## Memory edits

None (decision 14).

## Out of scope

- Delivering any bundle, suppressing native files, changing any provider argv or prompt,
  and switching Claude auto-memory or Grok memory v2 (E3).
- Rendering helper bundles at runtime (E3's subagent slot).
- Export stub budgets and the bootstrap clause, `sase instructions export`, `sync`, the
  last-known-good store, and retiring `capture_instruction_snapshot` (E4).
- Home files and other projects' files (E5).
- TUI surfaces and `sase instructions show` (E6).
- `when:` conditions, budgets as gates, `sase instructions check --matrix`, and plugin
  or launch-layer content (E7).
- Fixing `sase-1h2` (its own task).
