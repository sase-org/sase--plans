---
tier: tale
title: Fix artifact read resolution for plan refs
goal:
  "`sase artifact read plan:<path>` resolves and records an audited read from a bob-cli
  workspace, and the sidecar ref-config warning that misled the retry names the config
  key it is about."
size: small
proposed_by: bbugyi200.athena.bob-cli-11
bead: bob-cli-11
create_time: 2026-09-09 20:00:18
status: wip
---

- **BEAD:** bob-cli-11

# Fix artifact read resolution for plan refs (bob-cli-11)

## Problem

`sase artifact read plan:<path> "<reason>"` fails from a bob-cli workspace:

```
'plan' is missing its plugin prefix; use 'builtin@plan'
Error: Error: artifact reference 'plan:202608/block_id_prompt_first_responder.md' resolved with status 'unknown_kind'
```

Retrying with the suggested spelling fails differently:

```
$ sase artifact read 'builtin@plan:202608/block_id_prompt_first_responder.md' "..."
Error: malformed artifact reference: validation: artifact reference kind must match [a-z][a-z0-9_-]*
```

`sase plan show 202608/block_id_prompt_first_responder.md` succeeds, so the plan file is
present and the plans sidecar is cloned. Agents therefore cannot record audited artifact
reads or consumption edges for plan refs.

## Root cause (confirmed)

Two independent defects combine.

### 1. bob-cli's project config declares the plan ref provider without its plugin prefix

`sase/sase.yml` in this repo contains:

```yaml
repos:
  sidecar:
    builtin:
      plans:
        auto_clone: true
        ref:
          use: plan
```

`sase.plugins.qualified_id.parse_plugin_qualified_id` requires `<plugin>@<id>`, so
`use: plan` raises `PluginQualifiedIdError`.
`sase._sidecar_ref_normalization._normalize_document_ref_spec` records a
`missing_use_prefix` diagnostic and returns `None`, `policy_for_role` returns `None`,
and the `plans` role is dropped from the sidecar ref policy map. With no `plans` policy
there is no `plan` document root in the artifact ref context, so `plan:<path>` resolves
`unknown_kind`.

Verified directly against the installed `sase` package:

```
--- current (use='plan') ---
diagnostics: [('repos.sidecar.<bucket>.plans.ref.use', 'missing_use_prefix', "'plan' is missing its plugin prefix; use 'builtin@plan'")]
policy roles: ['agents', 'beads', 'research']
plans policy ref_kind: None | provider: None

--- fixed (use='builtin@plan') ---
diagnostics: []
policy roles: ['agents', 'beads', 'plans', 'research']
plans policy ref_kind: plan | provider: plan
```

`builtin@plan` is the correct value:
`get_artifact_provider_registry().ref_providers_by_id` exposes provider id `plan` with
`provenance.builtin=True`. The sase project's own `sase/sase.yml` already uses
`use: builtin@plan`; bob-cli's copy was written by
`e9f97ae chore: initialize SASE repositories` and was never migrated.
`sase doctor -C config.repos` already reports this as a WARN with the fully qualified
key `repos.sidecar.builtin.plans.ref.use`.

### 2. The runtime warning drops the config key, so it reads as artifact-ref advice

`sase.sidecar_ref_config.effective_sidecar_ref_policies` logs each diagnostic with
`log.warning("%s", diagnostic.message)`. No logging is configured for the
`sase artifact read` path, so Python's last-resort handler prints the bare sentence to
stderr immediately before the artifact-ref error. `SidecarRefPolicyDiagnostic.key` and
the `source_path` argument both identify the offending project-config key, and both are
discarded.

The result is that a message about `repos.sidecar.builtin.plans.ref.use` in `sase.yml`
looks like a correction to the reference the user just typed — which is what produced
the `builtin@plan:<path>` retry, and `builtin@plan` is not legal artifact-reference kind
grammar.

## Scope

Two repositories change.

- **bob-cli** (this workspace checkout): fix the stale project config. This is what
  unblocks plan artifact reads today.
- **sase** (`gh:sase-org/sase`, open with `sase repo open sase -r "<reason>"`): make the
  fail-soft warning self-locating so this config error can no longer be misread as
  artifact-reference syntax advice.

Out of scope: the Rust artifact-ref kind catalog (`plan` is intentionally absent from it
and is supplied by the builtin ref provider), the `builtin@<id>` grammar itself, and the
`<bucket>` placeholder inside `SidecarRefPolicyDiagnostic.key`.

## Implementation

### Step 1 — bob-cli: add the plugin prefix

In `sase/sase.yml`, change the `plans` sidecar entry:

```yaml
plans:
  auto_clone: true
  ref:
    use: builtin@plan
```

Change only that one scalar. Do not touch the surrounding `repos.sidecar` structure and
do not run `sase memory init` — `sase/sase.yml` is project config, not a memory note,
and this edit does not change any tier-1 memory content.

### Step 2 — sase: qualify the fail-soft diagnostic warning

In `src/sase/sidecar_ref_config.py`, `effective_sidecar_ref_policies` currently ends
with:

```python
    for diagnostic in report.diagnostics:
        log.warning("%s", diagnostic.message)
    return report.policies
```

Emit the diagnostic's config key, and the config source path when one was supplied, so
the line names the file and key it is about. Keep it a single `log.warning` call per
diagnostic and keep `diagnostic.message` verbatim at the end (the existing message text
is asserted by `tests/test_sidecar_ref_config.py`). Something like:

```python
    location = "" if source_path is None else f" ({source_path})"
    for diagnostic in report.diagnostics:
        log.warning("sidecar ref config%s %s: %s", location, diagnostic.key, diagnostic.message)
```

The resulting stderr line becomes, e.g.:

```
sidecar ref config (/path/to/sase/sase.yml) repos.sidecar.<bucket>.plans.ref.use: 'plan' is missing its plugin prefix; use 'builtin@plan'
```

Do not change `SidecarRefPolicyDiagnostic`, `policy_for_role`,
`_normalize_document_ref_spec`, or the doctor check — the diagnostic data and the
`missing_use_prefix` behavior are already correct and tested.

### Step 3 — sase: cover the warning shape with a test

Add a test to `tests/test_sidecar_ref_config.py`, next to
`test_sidecar_ref_use_without_plugin_prefix_fails_soft`, that calls
`effective_sidecar_ref_policies` with
`{"repos": {"sidecar": {"builtin": {"plans": {"ref": {"use": "plan"}}}}}}`,
`roles=("plans",)`, and an explicit `source_path`, using `caplog` at `logging.WARNING`.
Assert the emitted record text contains the config key (`.plans.ref.use`), the
`source_path`, and the original
`'plan' is missing its plugin prefix; use 'builtin@plan'` sentence. This is the
regression guard for the misleading bare warning.

## Verification

Run from the bob-cli workspace checkout after Step 1:

1. `sase doctor -C config.repos` — was WARN with one problem keyed
   `repos.sidecar.builtin.plans.ref.use`; must now be OK with no problems.
2. `sase artifact read plan:202608/block_id_prompt_first_responder.md "Verify bob-cli-11 fix"`
   — must exit 0, print the plan body with no `missing its plugin prefix` line on
   stderr, and record the read. Confirm the audit edge landed, e.g. with
   `sase artifact read plan:202608/block_id_prompt_first_responder.md ... --format json`
   showing `"recorded_link": true`, or by checking the artifact link store for a `read`
   row targeting the plan ref.
3. `sase artifact path plan:202608/block_id_prompt_first_responder.md` (or
   `sase artifact show`) resolves to the file under the plans sidecar rather than
   `unknown_kind`.
4. Confirm `sase plan show 202608/block_id_prompt_first_responder.md` still succeeds, so
   the config change did not regress the plan CLI.

For the sase repo changes, the checkout opened by `sase repo open` has no virtualenv.
Run the focused test with the sase project's own interpreter, e.g.

```
PYTHONPATH=<sase-checkout>/src /home/bryan/projects/github/sase-org/sase/.venv/bin/pytest \
  <sase-checkout>/tests/test_sidecar_ref_config.py -q
```

Confirm `sase` is imported from the checkout (an editable install of the live source may
shadow `PYTHONPATH`); if it is shadowed, run the file from the live sase checkout after
copying the change there, or skip to the equivalent assertion with a direct
`caplog`-free call and `logging` capture. Whole-file green is required, not just the new
test.

Note that the `sase` CLI on `PATH` is an editable install of the live sase source tree,
not the `sase repo open` checkout, so Step 2's message change will not appear in the
verification output from Step 1's commands. That is expected.

## Commit obligations

Both repositories are modified this turn, so the final declaration must carry a `commit`
decision for the bob-cli workspace checkout and for the `sase` external repo opened via
`sase repo open`.

## Close-out

Close `bob-cli-11` with `sase bead close bob-cli-11 --note "<what you verified>"`,
naming the doctor result and the successful audited plan read.
