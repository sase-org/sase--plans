---
tier: tale
title: Stop tale gate settlement from silently dropping the coder follow-up
goal:
  Approving a tale gate launches its coder agent again, and any future settlement-time
  shell-block parse failure is recorded and visible instead of silent.
size: small
proposed_by: bbugyi200.athena.0f4
create_time: 2026-09-09 20:00:31
status: wip
---

# Plan: Stop tale gate settlement from silently dropping the coder follow-up

## Problem

Approving a tale (`approve+commit` or `approve`) settles the gate shell to
`TALE APPROVED` / `PLAN APPROVED` and then launches **nothing**. No `--code` member is
created, no `gate_followup_error` is recorded, no warning is logged, and the ACE agent
panel has nothing to show. Every tale approval since
`32da1f3d2 feat(plan): add shell-backed approval handoff` has been affected: the
gate-shell path is now the _only_ path that launches a tale's coder, so the capability
is fully broken, not degraded.

Epic gates and question gates are unaffected (see "Blast radius" below).

## Root cause

The gate `shell` block is parsed twice with **different strictness**, and the
settlement-time parse is stricter than the creation-time parse.

1. **Creation** — `src/sase/notification_gates/model_request.py:249-253` parses the
   shell block with `allow_branch_subsets=kind in {"epic_plan", "plan"}`. For a tale,
   the compiled branch list is `[["approve","commit"], ["reject"], ["feedback"]]` (the
   `approve`/`commit` options share the `TALE_PLAN_SUBMIT_GROUP` declared at
   `src/sase/plan_gate.py:160`). Subsets are allowed, so the shell block's `approve`,
   `commit`, **and** `approve+commit` branch keys all validate.

2. **Settlement** — `src/sase/gate_shell/followup_policy.py:155-168` (`_parse_shell`)
   re-parses that _same, already-validated_ envelope as
   `GateShellSpec.from_mapping(raw_shell, branches=branches)`, i.e. with the default
   `allow_branch_subsets=False`. `_valid_branch_keys` then only accepts the full joined
   key `approve+commit`, so the singleton `approve` and `commit` keys raise:

   ```
   GateError: shell branch key must be a compiled branch or timeout, stopped, or failed
   ```

   `_parse_shell`'s bare `except Exception: return None` swallows it.

Everything downstream then does exactly what "no shell block" means:

- `resolve_gate_followup(...)` returns `None`.
- `resolve_gate_branch_presentation(...)` returns `(None, None)`.
- `settlement._apply_branch_policy` takes its `policy is None` arm and **deletes**
  `gate_next_action` (`src/sase/gate_shell/settlement.py:345-347`).
- `shells/settlement.settle_shell_claim_and_followup` keys its entire launch decision
  off `meta.get("gate_next_action")` (`src/sase/shells/settlement.py:55,74`). With the
  field gone it skips both the launch arm _and_ every `_record_followup_outcome` /
  `_record_followup_error` call, and just releases the claim. Nothing is written, so
  nothing is shown.
- `resolve_shell_next_action` / `plan_next_action` / `prepare_accepted_plan_successor`
  are never reached, so the coder prompt is never even composed.

### Reproduction against the user's live gate

The `0f2` tale (project `gh_bobs-org__bob-cli`, gate
`773c2f50-61a2-4431-bf3f-11825dd52ff8`) is a verbatim instance. Its `response.json`
carries `selected_option_ids: ["approve","commit"]` and a merged result of
`{"action":"approve","commit_plan":true,"run_coder":true}`, and against its real bundle
envelope:

```
key= 'approve+commit'
shell parsed: False
policy: None
presentation: (None, None)
```

The fingerprints are all over its settled `agent_meta.json`: no `gate_next_action`, no
`gate_followup_*` field at all, `gate_next_fork` still `family` and
`gate_next_raw_prompt` still `False` (the creation-time `shell.next` values, never
overwritten by the `approve+commit` branch's `none` / `True`), and `gate_accent` still
`#FF87AF` instead of the branch's `#00D7D7`. Its `done.json` likewise carries no
`gate_followup_*` keys. The agent name registry has `0f2`, `0f2--plan`, and `0f2--gate`
— and no `0f2--code`.

### Why the tests did not catch it

Every gate-shell settlement and policy test builds its member from **single-option**
compiled branches — `(("cleanup",), ("reject",))` in
`tests/gate_shell/test_settlement_followup.py:90`, `(("submit",),)` in the
question-shell tests, `(("cleanup",),)` in
`tests/gate_conformance/test_gate_shell_conformance.py:97`. A multi-option compiled
branch with subset keys — the tale's exact shape — is never settled in any test.
`tests/plan_shell/test_create.py` _does_ use the tale shape but only ever parses it with
`allow_branch_subsets=True`, mirroring the creation side and therefore reproducing the
divergence instead of catching it.

## Blast radius

`allow_branch_subsets=True` is granted to `plan` and `epic_plan`, but subset keys only
_exist_ when a kind compiles a multi-option branch, and `plan_gate.py:160` is the only
place in `src/` that declares gate `groups`. So:

- **tale plan gates**: broken — every branch key set includes the singleton `approve`
  and `commit`, so `_parse_shell` always raises.
- **epic plan gates**: unaffected — `groups` is empty for `epic`, branches are all
  singletons, and the shell block declares exactly those keys.
- **question / HITL / launch-approval / custom gates**: unaffected — single-option
  branches only.

Verified directly:

```
tale allow_subsets=True  -> OK      tale allow_subsets=False  -> FAIL
epic allow_subsets=True  -> OK      epic allow_subsets=False  -> OK
```

## Approach

Two independent defects, fixed together because the second is what made the first
invisible.

**A. Make settlement parse the shell block exactly as creation did.** Not "make
settlement permissive" — make the two agree by construction, so a future kind that opts
into subsets cannot re-open this.

**B. Never let a settlement-time parse failure be silent.** Keep returning `None` rather
than raising (settlement must not crash — that contract is stated in the
`followup_policy` and `kind_next_action` module docstrings), but log it and record it on
the shell's own metadata so ACE can show it.

## Implementation

### 1. One shared subset-branch policy

In `src/sase/notification_gates/model_shell.py`, add the constant and a kind-keyed
helper next to `GATE_SHELL_RESERVED_BRANCHES`:

```python
#: Gate kinds whose shell block may key a branch by a subset of a compiled
#: multi-option branch (a tale's ``approve`` / ``commit`` under the compiled
#: ``approve+commit``). Creation and settlement MUST agree on this set: a
#: settlement parse stricter than the creation parse silently voids the
#: branch's whole follow-up policy.
SUBSET_BRANCH_GATE_KINDS = frozenset({"epic_plan", "plan"})


def subset_branches_allowed(kind: object) -> bool:
    """Return whether *kind*'s shell block may declare subset branch keys."""
    return isinstance(kind, str) and kind in SUBSET_BRANCH_GATE_KINDS
```

Export both from `sase.notification_gates` if that package re-exports shell model
symbols; follow whatever the existing `model_shell` exports do.

### 2. Creation uses the helper

In `src/sase/notification_gates/model_request.py:249-253`, replace the inline
`kind in {"epic_plan", "plan"}` with
`allow_branch_subsets=subset_branches_allowed(kind)`. Behaviour is identical; the
literal set now lives in exactly one place.

### 3. Settlement uses the helper, keyed off the envelope's own kind

Rewrite `_parse_shell` in `src/sase/gate_shell/followup_policy.py`:

- Read `kind = envelope.get("kind")` and pass
  `allow_branch_subsets=subset_branches_allowed(kind)` to `GateShellSpec.from_mapping`.
- Replace `except Exception: return None` with an `except Exception:` that logs
  `logger.warning(...)` with `exc_info=True` — include the gate kind and the branch
  keys, because the message is the whole diagnosis — and _then_ returns `None`.
- Add a module `logger = logging.getLogger(__name__)`.

Update the module docstring: it currently justifies the swallow as "a malformed or
absent `shell` block always resolves to no policy ... creation-time validation would
have rejected a bad block". That reasoning is exactly what broke — state that settlement
must parse under the same policy as creation, and that a swallow here is a _bug report_,
not an expected path.

### 4. Distinguish "no policy" from "could not resolve a policy"

Give `followup_policy` a way to say the shell block was present but unparseable, so
settlement can record it. Smallest shape that works: a
`shell_block_unparseable(envelope) -> bool` predicate (present `shell` dict +
`_parse_shell` returning `None`), used by
`settlement._apply_branch_policy`/`settle_gate_shell` to set
`meta["gate_followup_error"]` to something like
`"gate shell block did not parse at settlement; follow-up policy was skipped"` before
the done marker is written. Prefer a shape that does not re-parse twice; returning a
small result object from `_parse_shell` is acceptable if it reads cleaner than a second
predicate.

`_done_marker` already copies `gate_followup_error` into `done.json`
(`settlement.py:206-224`) and `_agent_gate_section.py` already renders gate follow-up
state, so no display work should be needed — confirm rather than assume.

Do **not** record an error for the ordinary `policy is None` case (`prompt: null` on
`commit` / `reject`, or an unmapped reserved key): that is the designed "no follow-up"
path and is correct.

## Tests

Regression tests, in order of what they defend:

1. **`tests/gate_shell/test_followup_policy.py`** — resolve policy and branch
   presentation from a real tale envelope (`plan_gate_shell_block("tale")` plus
   `branches=[["approve","commit"],["reject"],["feedback"]]`) with
   `selected_option_ids=["approve","commit"]`. Assert the resolved policy has `prompt`
   set, `fork == "none"`, `role == "code"`, `suffix == "--code"`, `raw_prompt is True`,
   and that presentation resolves to `("TALE APPROVED", "#00D7D7")`. Also cover the
   `["approve"]`-only selection resolving the `approve` branch to `PLAN APPROVED` /
   `#00D7AF`. Both of these fail on the current tree.

2. **The invariant, not just the instance** — a test that asserts creation and
   settlement never diverge: for each gate kind that declares a shell block, build the
   creation-validated envelope and assert `_parse_shell` on it returns a non-`None` spec
   whose branch keys equal the declared ones. Put it wherever it will be run against
   every kind — `tests/gate_conformance/` is the natural home given that suite's stated
   purpose. This is the test that would have caught the original divergence and will
   catch the next one.

3. **`tests/gate_shell/test_settlement_followup.py`** — settle a gate shell whose
   envelope carries an unparseable shell block and assert `gate_followup_error` is set
   on both `agent_meta.json` and `done.json`. Guards defect B on its own.

4. **End-to-end tale settlement** — extend
   `tests/gate_shell/test_settlement_followup.py` (or the conformance shell dimension)
   with a member built from the tale shape and assert settlement reaches the launch arm:
   `gate_next_action` is present and `gate_followup_outcome` is recorded. Use the
   suite's existing launch-stubbing convention rather than spawning a real agent.

Follow each file's existing fixture style; do not introduce a new harness.

## Verification

- `just check` must pass. Run `just install` first if this workspace is stale.
- Confirm the four test groups above fail on `master` and pass after the change — a
  regression test that passes before the fix is not a regression test.
- Re-run the live reproduction against the `0f2` bundle
  (`/home/bryan/.sase/interaction_requests/plan/773c2f50-61a2-4431-bf3f-11825dd52ff8`):
  `_parse_shell` must now return a spec, the resolved policy must carry `role="code"` /
  `suffix="--code"`, and presentation must be `("TALE APPROVED", "#00D7D7")`.

## Out of scope

- **Re-driving already-settled gates.** Every tale approved between `32da1f3d2` and this
  fix lost its coder, including `0f2`. `_is_terminal_meta` short-circuits
  `settle_gate_shell`, so there is no re-settle path and building one is a separate
  change with its own risk. Instead: after the fix lands, report to the project owner
  which settled plan gate shells have `gate_state` in `{answered, completed}`, a
  `plan`-kind bundle, a `run_coder: true` result, and no `gate_followup_*` field — a
  read-only scan, no new tooling — and let them decide whether to relaunch. `0f2`'s plan
  was archived and committed (`plan_archive_state: archived`,
  `plan:202608/single_hyphen_em_dash_trigger.md`), so relaunching its coder from the
  archived plan is a normal agent launch.
- Changing the tale gate's option/group structure, or removing subset branch keys from
  the plan shell block. Both would also fix the symptom, but they change the
  reviewer-facing gate contract to work around a parser divergence.

## Notes

The `sase-ud` epic is still in progress (`sase-ud.13` and `sase-ud.14` open). This
defect is caused by that epic — `sase-ud.7` introduced the settlement-time
`_parse_shell` with the default strictness, and `sase-ud.11` introduced both
`allow_branch_subsets` and the tale shell block that depends on it. A note recording
this is being left on the `sase-ud` epic bead.
