---
tier: tale
title: Expand document artifact refs through their declared expansion format
goal:
  A document artifact reference expands to the prompt text its provider spec declares,
  so `@research:<path>` becomes "the `<path>` file in the research sidecar repo" instead
  of an absolute path inside a 257 MB clone that the launch must first materialize and
  then fail on.
size: medium
proposed_by: bbugyi200.athena.z9
create_time: 2026-08-13 09:10:46
status: wip
---

# Plan: Expand document artifact refs through their declared expansion format

## Problem

`sase-kp.land.w0.r0` failed to launch in 22 s
(`~/.sase/workflows/202608/gh_sase-org__sase_ace-run-260813_082804.txt`):

```
Materializing 'research' sidecar for @research references...

❌ ERROR: The following artifact reference(s) could not be resolved:
  - @research:202608/monitor_command_substrate.md (missing: hint: searched document
    root: .../sase_11/sase/repos/research)

⚠️ Artifact reference validation failed. Terminating workflow.
```

The prompt was
`#gh:sase Can you help me improve the recently added sase monitor functionality …? Review @research:202608/monitor_command_substrate.md for context … #plan #m_opus`.

The launch cloned the ~257 MB `research` sidecar onto the critical path, resolved the
citation to `missing`, called `sys.exit(1)` from `src/sase/artifact_ref_prompt.py:277`,
and left workspace #11 held as a visible failed run. The agent never started.

## Root cause

Two independent causes stack; the fix for the second one dissolves the first.

### 1. The cited path was wrong, and a wrong path is launch-fatal

Research reports live in a directory per report. The real file is
`202608/monitor_command_substrate/monitor_command_substrate.md`; the prompt cited
`202608/monitor_command_substrate.md`. That is an ordinary authoring slip — the kind of
slip that should cost the agent one `ls`, not the whole run.

It is fatal because `_expand_artifact_references()` treats any resolution status outside
`{exact, drifted, vcs_backed}` as a failure, prints the batch, and exits
(`src/sase/artifact_ref_prompt.py:223-242`, `:275-277`). For a document kind the
resolution is a plain filesystem probe against
`<workspace>/sase/repos/<role>/<argument>`, so "the agent's workspace does not have this
exact path right now" ends the run.

This is the second launch killed by this code path in two days. The `yl` failure fixed
by `plans:202608/lazy_sidecar_ref_materialization.md` (landed 2026-08-12 as `d6f435c7a`)
had a _correct_ path and a _missing clone_; that plan added the launch-time
materialization pre-pass. Today's failure is the mirror image: the clone was made (the
pre-pass worked, and cost ~15 s of the 22 s) and the path was wrong. Both are the same
underlying bet — that a launch should stake itself on a document ref resolving to a
local file — and that bet keeps losing.

### 2. `expansion_format` is declared, validated, documented, and ignored

The `sase-js` epic (`plans:202608/artifact_ref_contract.md`, status `done`, §3.4)
specifies a closed expansion formatter for provider specs, and uses the research
provider as its worked example:

```yaml
expansion:
  format: "the {checkout_path} file in the {sidecar_role} artifact repo"
```

Everything that quote needs exists and works:

- the closed formatter and its 12-placeholder vocabulary are in Rust
  (`sase-core:crates/sase_core/src/artifact_ref/expansion.rs:12-71`), exposed as
  `artifact_ref_expansion_validate` / `artifact_ref_expansion_render`
  (`src/sase/artifact_ref_operations.py:234-246`);
- the field is in the config schema (`src/sase/config/sase.schema.json`), in the
  configuration table (`docs/configuration.md:1721`), and in the inline-spec example in
  `docs/artifact_references.md:133`;
- the installed `sase-research` plugin ships exactly that string
  (`sase-research:src/sase_research/provider.py:31`).

Nothing reads it. Only the two Python-owned builtin _entry_ kinds render through the
formatter (`src/sase/artifact_providers/builtin_entry_stitch.py:34-41,94-100`, and the
patch equivalent). Every document kind falls through to `_legacy_replacement_text()`,
which hardcodes `@{resolved_path}` (`src/sase/artifact_ref_prompt_rendering.py:56-62`) —
the "hardcoded rendering" the epic's `retire` phase left behind when it deleted the
`#ref/<kind>` xprompt renderer. The builtin `plan` spec's declared `{kind}:{argument}`
(`src/sase/artifact_providers/_builtin.py:21`) and the default
`DEFAULT_DOCUMENT_REF_EXPANSION_FORMAT` (`src/sase/sidecar_ref_config.py:39`) are dead
strings too: neither matches what document refs actually expand to.

So the provider surface Bryan configured — "expand a research citation to prose naming
the file and its repo" — has been inert since it shipped, and the hardcoded path
expansion it was meant to replace is what forces resolution, forces materialization, and
kills launches.

## Approach

Make the declared format the contract, and let the format decide how much work expansion
has to do.

1. **A document ref expands through its provider spec's `expansion_format`.** The
   placeholder set is closed and Rust-validated already; expansion binds values for the
   document subset of it.
2. **A format that uses `{checkout_path}` is _path-bound_.** It needs a resolved local
   file, so it keeps today's behavior exactly: resolve, materialize a missing sidecar,
   and fail the launch with a diagnostic when the document is not there. `@plan` stays
   on this path, byte-identical.
3. **A format that uses no path placeholder is a _pointer_.** It renders from the
   reference itself, so expansion needs no clone and no filesystem: it never
   materializes a sidecar, and an unresolvable pointer never fails a launch.
4. **`@research` becomes a pointer kind**, declaring
   `"the {repo_relative_path} file in the {sidecar_role} sidecar repo"`, so
   `@research:202608/artifacts_pane_contract/artifacts_pane_contract.md` expands to
   `the 202608/artifacts_pane_contract/artifacts_pane_contract.md file in the research sidecar repo`.

The agent then opens the sidecar the same way every other non-workspace repo is opened —
`sase repo open research`, per the `/sase_repo` rule in `CLAUDE.md` §4.2 — and pays the
257 MB clone only if it actually reads the report. Today's expansion hands the agent a
path into a clone it did not ask for and bypasses that rule entirely.

Under this change today's prompt launches, the agent reads
`the 202608/monitor_command_substrate.md file in the research sidecar repo`, opens the
sidecar, finds the real report one directory down, and does the work.

### Why not the alternatives

- **Just fix the hint / keep failing.** The failure mode is the problem, not its
  wording. A citation is context, not a precondition; killing a `#plan` run and holding
  a workspace because one context pointer has a typo is the wrong trade.
- **Special-case `@research` in the expansion code.** Rejected: the spec field for
  exactly this already exists, is documented, and is what the plugin ships. Hardcoding a
  second research-shaped branch would leave three sources of truth (`_builtin.py`, the
  plugin spec, and the renderer) and still leave the field dead.
- **Make every document kind a pointer / flip the default.** Rejected: `@plan:` refs are
  read directly by agents, the `plans` sidecar is `auto_clone: true`, and the default
  document root has a machine-global fallback (`~/.sase/plans`). Path expansion is right
  there; the plan provider keeps it by declaring it.
- **Retire the launch-time materializer now that research no longer needs it.** Rejected
  here, deliberately. It is one day old, it is still correct for any path-bound document
  sidecar that is not `auto_clone`, and gutting it is a separate decision from this one.
  It becomes unreachable for `research` and stays reachable for `plan`.
- **Warn but keep resolving strictly for pointer kinds.** Rejected as the primary
  behavior — a pointer's whole point is that expansion does not depend on local state.
  Step 5 keeps the _signal_ (one non-fatal stderr note when the sidecar is already
  cloned and the path is not in it) without the failure.

## Implementation

All paths are repo-relative to the `sase` repo unless prefixed with `sase-research:`.

### 1. Carry each document kind's expansion policy on the resolution context

`src/sase/artifact_ref_models.py`, `src/sase/artifact_ref_context.py`

Add a frozen dataclass and a Python-only context field:

```python
@dataclass(frozen=True, slots=True)
class ArtifactRefDocumentExpansion:
    kind: str
    role: str
    expansion_format: str
    is_pointer: bool
```

Add `document_expansions: tuple[ArtifactRefDocumentExpansion, ...] = ()` to
`ArtifactRefContext` (`src/sase/artifact_ref_models.py:357-371`) and a
`document_expansion_for(kind)` lookup helper. Do **not** add it to `to_wire()`
(`:387-403`) — this is context for Python-side rendering only, exactly like the existing
`selected_project` field.

Populate it in `artifact_ref_context()` (`src/sase/artifact_ref_context.py:32-70`),
which already has `policies` in hand at line 45 and already loops the document roles at
lines 51-70. One entry per document policy, `role` from `policy.role`, `kind` from
`policy.ref_kind`.

Do **not** extend `ArtifactRefDocumentRoot`: it has a `to_wire()` and crosses into the
Rust resolver, so extending it would drag in a `sase-core` wire change for no benefit
(the same reasoning `plans:202608/lazy_sidecar_ref_materialization.md` step 2 used).

### 2. Classify formats, and make the defaults honest

`src/sase/sidecar_ref_config.py`

- Add the document placeholder vocabulary and the path-bound marker:

  ```python
  DOCUMENT_REF_EXPANSION_PLACEHOLDERS = frozenset({
      "kind", "argument", "canonical_argument", "display_label",
      "repo_relative_path", "sidecar_role", "checkout_path",
  })
  DOCUMENT_REF_PATH_PLACEHOLDERS = frozenset({"checkout_path"})
  ```

  These are a subset of the Rust vocabulary
  (`sase-core:crates/sase_core/src/artifact_ref/expansion.rs:12-25`); the excluded names
  (`project`, `repository`, `captured_revision`, `captured_digest`, `logical_path`) have
  no document-ref binding and must be rejected rather than rendered empty.

- In `_normalize_document_ref_spec()` (`:239-314`), after
  `validate_ref_provider_spec()`, run
  `artifact_ref_expansion_validate(expansion_format)` and reject a spec whose
  placeholders are not a subset of `DOCUMENT_REF_EXPANSION_PLACEHOLDERS` through the
  existing `_diagnostic(...)` / return-`None` fail-soft channel. A bad format is a
  config error surfaced by `sase doctor -C config.repos`, not a launch-time render
  crash.

- Expose the format and its classification on `SidecarRefPolicy` (`:75-92`) as
  `expansion_format` and `is_pointer_expansion` properties derived from `spec`.

- Change `DEFAULT_DOCUMENT_REF_EXPANSION_FORMAT` (`:39`) from `"{kind}:{argument}"` to
  `"@{checkout_path}"`, and the builtin plan spec
  (`src/sase/artifact_providers/_builtin.py:21`) to the same. This is not a behavior
  change: it writes down what document refs have always expanded to. `@plan:` and every
  unconfigured document sidecar keep byte-identical expansion.

### 3. Render document refs through the declared format

`src/sase/artifact_ref_prompt_rendering.py`, `src/sase/artifact_ref_prompt.py`

In `artifact_ref_replacement()` (`:21-46`), when the reference's `kind_type` is
`document` and the context carries an `ArtifactRefDocumentExpansion` for its kind,
render through `artifact_ref_expansion_render()` instead of
`_legacy_replacement_text()`, with:

| placeholder          | value                                                                                                                          |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `kind`               | `reference.kind`                                                                                                               |
| `argument`           | the authored argument (fragment stripped)                                                                                      |
| `canonical_argument` | `render_artifact_ref(replace(reference, fragment=None))` payload                                                               |
| `repo_relative_path` | the sidecar-relative POSIX path — identical to `argument` for a document kind, bound separately so the semantic name is usable |
| `display_label`      | `Path(argument).name`                                                                                                          |
| `sidecar_role`       | the expansion entry's `role`                                                                                                   |
| `checkout_path`      | the resolved absolute document path — bound **only** for path-bound formats, where resolution already guarantees it            |

Keep `_fragment_annotation()` (`:112-125`) appended after the rendered text for both
classes, so `@research:…#L10-L20` still says `(lines 10-20)`.

Validate the binding at import time the way
`src/sase/artifact_providers/builtin_entry_stitch.py:39-41,99` does — a module-level
frozenset of supported names asserted against the values dict — so a future edit that
lets the two drift fails loudly instead of inside a Rust render call.

`_legacy_replacement_text()` keeps serving `chat`, `file`, `bead`, and `agent`; only the
`document` branch moves.

### 4. Pointer refs never resolve-to-fail and never clone

`src/sase/artifact_ref_prompt.py`, `src/sase/artifact_ref_prompt_materialize.py`,
`src/sase/artifact_ref_prompt_resolution.py`

- In `_expand_artifact_references()` (`:187-273`), when a candidate's kind is a pointer
  document kind, a resolution status outside `{exact, drifted, vcs_backed}` is no longer
  appended to `failures`: expand it with `resolved_path=None` and continue. Path-bound
  kinds keep the current strict branch verbatim.
- `artifact_resolved_path()` (`src/sase/artifact_ref_prompt_resolution.py:60-70`) must
  return `None` instead of raising for an unresolved pointer document ref.
- Keep calling the resolver for pointer kinds. It is a local filesystem probe with no
  network and no clone, and when it succeeds the resolved path still feeds artifact
  staging and published permalinks exactly as today. The _expansion text_ never depends
  on its outcome.
- `_stage_artifact_references()` (`:533-561`) and `_record_artifact_ref_consumption()`
  (`:459-504`) skip entries whose `resolved_path` is `None`
  (`ArtifactConsumptionResolutionStatus` is `Literal["exact","drifted","vcs_backed"]`,
  `src/sase/core/artifact_consumption.py:28`, so an unresolved ref has no valid event).
  `_record_artifact_ref_uses()` (`:507-530`) still records the occurrence and its
  rendered prompt text — a pointer citation stays visible in `ref-uses`.
- In `materialize_missing_document_roots()`
  (`src/sase/artifact_ref_prompt_materialize.py:31-90`), skip kinds whose expansion is a
  pointer. Take the kind → role map from the new context field instead of recomputing
  policies, and delete `_document_role_by_ref_kind()` (`:93-121`) with its four-import
  policy rebuild.

### 5. Keep the typo signal, non-fatally

`src/sase/artifact_ref_prompt.py`

When a pointer ref does not resolve **and** its sidecar root exists (so the checkout is
present and the path is genuinely absent), print one stderr note naming the ref and the
searched root, then continue. Do not print when the root is absent — not having the
sidecar cloned is now the expected state, and a note there would be pure noise on every
launch. Reuse `artifact_ref_resolution_hint()`
(`src/sase/artifact_ref_prompt_resolution.py:183-195`) for the text.

### 6. `@research` declares the pointer format

`sase-research:src/sase_research/provider.py:31`

```python
"expansion_format": "the {repo_relative_path} file in the {sidecar_role} sidecar repo",
```

Update the module docstring's ref-provider paragraph and
`sase-research:tests/test_provider_specs.py` to assert the new string and that it uses
no path placeholder. `sase-research:README.md` / `AGENTS.md` need no change.

### 7. Docs

- `docs/artifact_references.md`: add an **Expansion** subsection under _Provider Specs_
  stating the two classes, the document placeholder table, and that a pointer ref never
  clones and never fails a launch. Update the _On-demand document sidecars_ section
  (`:73-87`) — the research example there is now the counter-example — and note under
  _Publication_ that a pointer ref that does not resolve locally publishes unlinked but
  still records its use row.
- `docs/configuration.md:1721`: default becomes `@{checkout_path}`.
- `docs/plugins.md:807`: the example spec's format follows the same default.

## Tests

`tests/artifact_refs/test_rendering.py`,
`tests/artifact_refs/test_prompt_materialization.py`,
`tests/test_linked_repo_sidecar_config.py`, plus the existing
`tests/artifact_refs/helpers.py` context fixtures.

1. **Pointer expansion.** A `research`-kind document policy with the pointer format
   expands `@research:202608/x/x.md` to
   `the 202608/x/x.md file in the research sidecar repo`, with no sidecar clone on disk.
2. **The regression, end to end.** The exact failing prompt shape — a pointer ref whose
   path does not exist, in a workspace with no `sase/repos/research` — expands and
   returns instead of exiting; `ensure_sdd_kind_clone` is never called.
3. **`@plan` is unchanged.** A `plan` ref still expands to `@<absolute path>`, still
   materializes a missing `plans` root, and still fails the launch with today's
   diagnostic when the file is absent. Assert the rendered string byte-for-byte against
   the pre-change expectation.
4. **Path-bound custom kind.** A document policy declaring `@{checkout_path}` behaves
   like `plan` — this is the case `tests/artifact_refs/test_prompt_materialization.py`
   already covers for `research`; re-point those cases at a path-bound kind so the
   materializer keeps its coverage.
5. **Fragments.** `@research:202608/x/x.md#L10-L20` renders the pointer text plus
   `(lines 10-20)`.
6. **Telemetry.** An unresolved pointer ref writes a `ref-uses` row carrying the
   rendered prompt text and writes no consumption event and no staged artifact; a
   _resolved_ pointer ref (sidecar present) writes all three, unchanged from today.
7. **Non-fatal note.** Sidecar root present + path absent prints the note and exits
   zero; sidecar root absent prints nothing.
8. **Spec validation.** A document spec whose format uses `{captured_revision}` is
   rejected with a diagnostic and drops the policy; `{kind}`-only and `@{checkout_path}`
   specs normalize cleanly.
9. **Defaults.** An unconfigured document sidecar gets `@{checkout_path}` and path-bound
   behavior.

`sase-research:tests/test_provider_specs.py` gains the assertion from step 6.

## Verification

- `just install` first — this workspace is ephemeral and dependencies may have moved.
- `just check` inline; then `just check-full` through `/sase_monitor` before landing —
  this touches shared prompt preprocessing that many suites import.
- Re-run the real failure. Relaunch `sase-kp.land.w0.r0`'s prompt verbatim
  (`Review @research:202608/monitor_command_substrate.md …`) and confirm it launches,
  that the agent's prompt shows the pointer text, and that the run's workspace never
  grows a `sase/repos/research` clone.
- Confirm a `@plan:` launch still expands to a path and still materializes a missing
  `plans` sidecar.
- Confirm ACE `@`-completion, `sase artifact show research:<path>`, and the LSP catalog
  are unchanged — none of them route through prompt expansion.

## Rollout

`sase-research` is installed into the `sase` uv tool environment from the git repo, not
from PyPI (`~/.local/share/uv/tools/sase/uv-receipt.toml`:
`{ name = "sase-research", git = "https://github.com/sase-org/sase-research" }`, pinned
at commit `f499469`). The new expansion string is live for launches only after the
plugin change is on `master` **and** the tool environment is refreshed
(`uv tool upgrade sase`). Until then `@research` keeps its `{checkout_path}` format and
therefore today's path-bound behavior — correct, just not yet simplified. Call this out
in the landing note so the behavior change is not mistaken for a regression in either
direction.

## Out of scope

- **Referenced By write-back for unresolved pointer refs.**
  `plan_referenced_by_requests()`
  (`src/sase/agents_sync/referenced_by_planning.py:53-61`) works backwards from a staged
  record's local path, so a research citation made without a local clone will not queue
  a table row. Nothing regresses that is working today — the research sidecar has no
  `.sase/referenced-by/` index and no managed table in any report — but planning the
  request from the logical `research:<repo-path>` instead of a filesystem path is a real
  follow-up. File it through `/sase_new_task`.
- Retiring `artifact_ref_prompt_materialize.py`, changing any sidecar's `auto_clone`, or
  changing the `~/.sase/plans` fallback root.
- Changing `@plan` expansion, or the expansion of the non-document kinds still served by
  `_legacy_replacement_text()`.
- ACE completion, the LSP catalog, `sase artifact *`, and inventory globs — all
  read-only surfaces that resolve documents directly and are unaffected by prompt
  expansion.
- Releasing workspace #11; that is a normal `sase ace` dismissal.
