---
tier: tale
title: Align sase's sase-core-rs window with the released placeholder-source core
goal: Once sase-core publishes the release containing the breaking placeholder-candidate
  source change, raise sase's `sase-core-rs` window in `pyproject.toml` to that published
  version so a released `sase` wheel can never resolve a binding whose `placeholder_completion`
  lacks the `common` argument.
create_time: 2026-07-25 15:12:47
status: wip
---

# Plan: Align sase's `sase-core-rs` window with the released placeholder-source core

## Problem

Epic `sase-9m` (saved common placeholder tags) changed the `sase-core` placeholder engine so that
`placeholder_completion` accepts a trailing `common` sequence and returns `{"text", "source"}` candidates. That change
landed in `sase-core` as commit `69504fe` (`feat(editor)!: tag placeholder candidates with a source and accept common
tags (sase-9m.1)`) but has **not been released**: the newest published `sase-core-rs` on PyPI is `0.9.2`, and the
`sase-core` workspace version is still `0.9.2`.

In this repo, `src/sase/xprompt/placeholder_completion.py::placeholder_completion` always calls the binding with four
positional arguments:

```python
payload = binding(text, line, character, list(common))
```

`pyproject.toml` still declares `sase-core-rs>=0.9.1,<0.10.0`. Dev installs are unaffected — `just install` builds
`sase_core_rs` from the local `sase-core` checkout and `just _core-overrides-arg` neutralises the published window — but
a **released** `sase` wheel carrying this code would resolve `sase-core-rs 0.9.2`, whose `placeholder_completion` takes
three arguments. Every `<` typed in the ACE prompt would then raise `TypeError` instead of opening the placeholder menu.

The `sase-9m.1` agent deliberately deferred the version bump the epic plan called for, for two sound reasons that still
hold:

1. `sase-core/AGENTS.md` gives `release-plz` ownership of `[workspace.package].version`; hand-editing it during feature
   work is forbidden without explicit user approval and a `manual-version` PR label.
2. The target version is not published, so raising the floor now makes
   `tools/validate_sase_core_rs_version --published-minimum` (run by `just validate`) fail.

So this is externally blocked on a `sase-core` release, exactly like the earlier `sase-93.7` and `sase-8u.4.2` work.

## Precondition

`sase-core` has published the release containing `69504fe`. Because that commit is a `feat!` on a `0.x` line, `release-plz`
is expected to compute **`0.10.0`**, not `0.9.3`. Confirm the actual published version before editing anything:

```bash
python - <<'EOF'
import json, urllib.request
d = json.load(urllib.request.urlopen("https://pypi.org/pypi/sase-core-rs/json", timeout=10))
print(d["info"]["version"], sorted(d["releases"])[-5:])
EOF
```

If the newest published version still lacks the `common` argument, stop — the precondition is not met.

## Work

1. `pyproject.toml`: raise the `sase-core-rs` requirement from `>=0.9.1,<0.10.0` to `>=<released>,<<next-minor>>` (for a
   `0.10.0` release: `sase-core-rs>=0.10.0,<0.11.0`). This is the only pin in the repo; `Justfile` handles dev installs
   through its overrides file and needs no change.
2. Verify the published binding really carries the argument, rather than trusting the version number alone:
   ```bash
   just install
   .venv/bin/python -c "import sase_core_rs; print(sase_core_rs.placeholder_completion('<a> <b', 0, 6, ['zeta']))"
   ```
   The payload's `candidates` must be `{"text", "source"}` dicts and must include the `common`-sourced entry.
3. Run `just validate` so `tools/validate_sase_core_rs_version --pyproject pyproject.toml --published-minimum` confirms
   the new floor is published, and `just check`.

## Out of scope

- Editing `sase-core`'s version by hand, or any change inside the `sase-core` repo. `release-plz` owns the release.
- Wiring the xprompt LSP to a common-placeholder source; it still passes `&[]` by design.

## Acceptance

`pyproject.toml` names the published core release, `just validate` and `just check` pass, and a fresh install resolves a
`sase-core-rs` whose `placeholder_completion` accepts `common`.
