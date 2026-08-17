---
tier: tale
size: small
title: Fix sase-telegram CI by building the flag_triage gate with its real factory
goal:
  Make sase-org/sase-telegram CI green again by replacing the forged flag_triage gate
  spec in tests/test_custom_gates.py with a gate built through
  sase.bead.flag_gate.create_flag_triage_gate, so the trusted flag_triage validation
  contract is satisfied.
proposed_by: bbugyi200.athena.04q
create_time: 2026-08-17 09:10:22
status: done
---

# Fix sase-telegram CI: build the `flag_triage` gate with its real factory

## Diagnosis

`actstat --repo sase-org/sase-telegram` reports the newest settled commit on `master` as
failing:

```
✘ 0bda964 feat(beads): render flag sections in Telegram · master · failure
    ✘ CI · #133 · failure
        ⊘ check (3.13)   step 9: Install dependencies
        ✘ check (3.12)   step 11: Run tests
```

Run: <https://github.com/sase-org/sase-telegram/actions/runs/31979787960>

### There is exactly one real failure

`check (3.12)` ends with `1 failed, 580 passed`. `Lint` (which only runs on 3.12)
passed. The single failure is:

```
FAILED tests/test_custom_gates.py::test_registry_declared_generic_forms_render_keyboards
  - sase.notification_gates.model_validation.GateError: flag triage gates require flag_triage
```

with this traceback tail:

```
tests/test_custom_gates.py:612: in test_registry_declared_generic_forms_render_keyboards
    flag = create_gate(flag_spec)
.sase-deps/sase/src/sase/notification_gates/service.py:68: in create_gate
    validate_gate_spec(spec, adapter)
.sase-deps/sase/src/sase/notification_gates/validation.py:226: in validate_gate_spec
    validate_flag_triage_spec(spec)
.sase-deps/sase/src/sase/notification_gates/kind_validation/flag_triage.py:20: in validate_flag_triage_spec
    _validate_flag_triage_structure(spec)
E   GateError: flag triage gates require flag_triage
```

`check (3.13)` is **collateral, not a second defect**. Its log ends with
`##[error]The operation was canceled.` while it was still `Compiling sase_core_py`
inside `Install dependencies`, and the orphan processes terminated were `just`,
`maturin`, `cargo`, and `rustc`. The `check` matrix has no `fail-fast: false`, so the
3.12 test failure cancelled the in-flight 3.13 job. `actstat` renders it with `⊘`
(cancelled) rather than `✘`. Fixing the 3.12 test clears both rows; there is nothing to
repair in the 3.13 job itself.

### Root cause

Commit `0bda964` added a `flag_triage` entry to the fixture map in
`test_registry_declared_generic_forms_render_keyboards` by **forging** the spec — taking
the `custom` gate spec built by the `_custom_spec()` helper and overwriting one field:

```python
flag_spec = _custom_spec(request_id="telegram-registry-flag")
flag_spec["kind"] = "flag_triage"
flag = create_gate(flag_spec)
```

That cannot ever succeed. In sase, `flag_triage` is a **trusted, human-only** gate kind:
`sase/notification_gates/kind_validation/flag_triage.py` reconstructs the whole spec
from the registered adapter in `sase.bead.flag_gate` and rejects any divergence. It
checks, in order:

1. `continuation_mode == FLAG_TRIAGE_CONTINUATION_MODE` — **this is the check that
   trips**, because `_custom_spec()` carries no flag-triage continuation mode.
2. `query == FLAG_TRIAGE_QUERY` and
   `branches == (("remove",), ("extend",), ("keep",), ("close",))`.
3. No `groups` and no `operations` — but `_custom_spec()` declares a `groups` entry.
4. Every option rebuilt whole from `flag_triage_option_spec` and compared for equality
   against the `remove`/`extend`/`keep`/`close` option ids.
5. Exactly the adapter's preview and per-option command resources, with the command
   scripts compared byte for byte against `flag_triage_gate_command_script`.
6. Presentation fields pinned to `sender="bead"`, `icon="⚑"`, `tags=["bead", "flag"]`,
   `panel="beads"`, and the note produced by `flag_triage_presentation_note`.
7. The preview file re-rendered from its payload and compared byte for byte.

So the `continuation_mode` error is only the _first_ of seven walls a forged custom spec
would hit. Patching that one field would just move the failure; the spec has to come
from the real adapter.

### Why the failure appeared on this commit

Two things converged:

- sase `master` registers a `flag_triage` adapter with `generic_form=True` (see the
  `GateAdapter(kind="flag_triage", ...)` entry in
  `src/sase/notification_gates/adapters.py`). There are now four `generic_form` kinds:
  `task_triage`, `bead_snooze`, `flag_triage`, and `custom`.
- The test loops over every registered kind and indexes the fixture map for each kind
  whose adapter declares a generic form:

  ```python
  for kind in registered_gate_kinds():
      adapter = adapter_for_kind(kind)
      if not adapter.generic_form:
          continue
      _text, keyboard, _attachments = format_notification(notifications[kind])
      assert keyboard is not None, kind
  ```

- The immediately preceding commit,
  `d29f358 build: install local sase checkouts in development`, wired CI to check out
  `sase-org/sase` and `sase-org/sase-core` into `.sase-deps/` and install sase from that
  source. sase-telegram CI therefore tracks sase `master`, so the newly registered kind
  became mandatory for the loop.

The new adapter turned the missing map entry into a `KeyError`. The commit silenced that
`KeyError` with a forged spec instead of a real gate, trading one failure for another.

### The right fix

Build the gate through its own factory, exactly as the same test already does for the
other two bead gates (`create_task_triage_gate`, `create_bead_snooze_gate`). sase
exports `create_flag_triage_gate` from `sase.bead.flag_gate`, and it returns the same
`create_gate` result object, so `.notification_id` works like the others.

No `sase_telegram` **source** change is needed. `src/sase_telegram/` contains no
`flag_triage` branch and does not need one — the Telegram formatter drives keyboards off
the registry's generic-form declaration, which is the property this test exists to pin.

## Prerequisites

- Open the repo with the `/sase_repo` skill before touching anything:

  ```bash
  sase repo open sase-telegram -r "Fix CI: build the flag_triage test gate with its real factory"
  ```

  Use the printed path as the only path for reads, writes, and commands. Note that
  `sase repo open` cleans the checkout, so open it **before** creating any file there.

- sase itself does not need modification. Read sase's reference implementations from the
  current sase workspace checkout (`src/sase/bead/flag_gate.py`,
  `src/sase/bead/model.py`, `tests/test_bead/test_flag_gate.py`).

## Steps

### 1. Replace the forged spec with a real flag triage gate

In the sase-telegram checkout, edit
`tests/test_custom_gates.py::test_registry_declared_generic_forms_render_keyboards`.

Delete these three lines:

```python
flag_spec = _custom_spec(request_id="telegram-registry-flag")
flag_spec["kind"] = "flag_triage"
flag = create_gate(flag_spec)
```

Replace them with a `create_flag_triage_gate(...)` call. Match the surrounding fixtures'
style: keyword arguments, one per line, and values that read as deliberate test data
rather than placeholders. The factory signature is:

```python
create_flag_triage_gate(
    *,
    request_id: str,
    bead_id: str,
    project: str,
    title: str,
    flag: FlagRecord,
    due_state: str,
    due_as_of: str,
    release: str,
    definition: Mapping[str, str] | None = None,
    description: str = "",
    notes: str = "",
    created_by: str = "",
    created_at: str = "",
    size: str | None = None,
    refs: Sequence[str] = (),
    producer: Mapping[str, Any] | None = None,
) -> Any
```

Required-argument guidance, all of it validated by sase:

- `request_id`: keep `"telegram-registry-flag"` so the id stays consistent with the
  `telegram-registry-*` ids the rest of the test uses.
- `bead_id` / `project` / `title`: mirror the neighbouring fixtures — e.g.
  `bead_id="sase-registry-flag"`, `project="sase"`, and a title in the same voice as
  `"Exercise the snooze generic form"`.
- `flag`: a `FlagRecord(key=..., remove_by_date=..., remove_by_release=...)` imported
  from `sase.bead.model`. `FlagRecord.validate()` requires `key` to be non-empty
  `snake_case`, `remove_by_date` to be a calendar `YYYY-MM-DD` date, and
  `remove_by_release` to be a release string. `sase`'s own
  `tests/test_bead/test_flag_gate.py` uses
  `FlagRecord(key="prettier_enabled", remove_by_date="2026-08-01", remove_by_release="0.16.0")`
  as a known-good shape.
- `due_state`: `"due"`. This gate is only ever raised for a due flag bead.
- `due_as_of` and `release`: an ISO date and a release string. Keep them consistent with
  the `flag` record and with the fixed dates already used in this test file (the snooze
  fixture pins `2026-08-09` / `2026-08-06`), so nothing depends on the wall clock.

Leave `definition`, `description`, and `notes` to taste — they only affect the rendered
preview, which the factory and validator render through the same code path.

Keep the fixture map entry unchanged in shape:

```python
"flag_triage": _stored_notification(flag.notification_id),
```

### 2. Fix up imports

Add to the existing `sase.*` import block at the top of `tests/test_custom_gates.py`, in
the file's alphabetical ordering (`sase.bead.flag_gate` sorts before `sase.bead.model`,
which is already imported for `SnoozeRecord`):

```python
from sase.bead.flag_gate import create_flag_triage_gate
from sase.bead.model import FlagRecord, SnoozeRecord
```

Then confirm `create_gate` is still used elsewhere in the file before assuming the
import can stay — it is used by several other tests, so it should remain. Do not remove
an import that is still referenced.

### 3. Verify locally

The sase-telegram checkout has no `.venv`, and `just install` builds `sase_core_py` with
`maturin develop --release`. In CI that Rust build alone ran for roughly four minutes,
so **do not run it inline**. Hand it to the `/sase_monitor` skill with a `--next`
action, e.g.:

```bash
sase monitor start --command 'just install && just check' --next '<follow-up action>'
```

`just check` runs `lint` (ruff + mypy) then `test` (pytest). The `Justfile` resolves the
local sase source automatically when the repo is opened inside a SASE linked workspace,
so no `SASE_TELEGRAM_SASE_SOURCE_DIR` override should be needed.

Expected result: the previously failing test passes and the suite reports 581 passed.
Once the venv exists, the targeted re-run is cheap:

```bash
just test tests/test_custom_gates.py::test_registry_declared_generic_forms_render_keyboards
```

Also re-run the whole file, since it shares the `gate_home` fixture and the notification
store across tests:

```bash
just test tests/test_custom_gates.py
```

### 4. Commit

Commit in the sase-telegram checkout using the `/sase_git_commit` skill. The repo
enforces conventional commit titles via `.github/workflows/pr-title.yml`, and this is a
test-only correctness fix, so use a `fix(tests):` or `test:` title describing that the
flag triage fixture is now built by its registered factory. Do not amend or rewrite
`0bda964`; land this as a new commit on top.

### 5. Confirm CI

After the commit reaches `master`, re-check with:

```bash
actstat --repo sase-org/sase-telegram -n 1
```

Both `check (3.12)` and `check (3.13)` must be green. `check (3.13)` will finally run
its test step for the first time on this change, because it was cancelled before
reaching it on run #133 — treat any failure there as new information and diagnose it
rather than assuming the matrix is symmetric.

## Verification

- `just check` passes in the sase-telegram checkout (ruff, mypy, and pytest).
- `tests/test_custom_gates.py::test_registry_declared_generic_forms_render_keyboards`
  passes, and the suite total goes from `1 failed, 580 passed` to `581 passed`.
- The test now constructs the `flag_triage` gate through
  `sase.bead.flag_gate.create_flag_triage_gate`; no test in the file forges a gate spec
  by mutating `kind` on another kind's spec.
- `actstat --repo sase-org/sase-telegram -n 1` shows the new head commit settled green
  with both matrix jobs passing.

## Non-goals

- **Do not weaken sase's `flag_triage` validation.** The seven-part reconstruction in
  `sase/notification_gates/kind_validation/flag_triage.py` is the trusted contract for a
  human-only gate that can remove a feature flag. The test was wrong, not the validator.
- **Do not add `flag_triage` branching to `src/sase_telegram/`.** Registry-driven
  generic-form rendering is the intended design and is what this test pins.
- **Do not add `fail-fast: false` to the CI matrix.** Losing the 3.13 signal when 3.12
  fails is a real annoyance, but it is a separate workflow-design question. If it is
  worth changing, file it through `/sase_new_task` instead of folding it into this fix.
- **Do not touch the Node 20 deprecation warnings** from `actions/checkout@v4`,
  `astral-sh/setup-uv@v4`, and `extractions/setup-just@v2`. They are warnings, they did
  not cause this failure, and bumping action majors deserves its own change.
