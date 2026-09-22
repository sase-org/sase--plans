---
tier: tale
title: Make path queries match by substring on Plans and Research panes
goal:
  A path:<fragment> query on the Artifacts Research (and Plans/provider) sub-tabs finds
  documents whose path contains the fragment, while identity reveal keeps working.
size: small
proposed_by: bbugyi200.athena.0p1
create_time: 2026-09-22 07:01:31
status: wip
---

# Make `path:` match by substring on the Plans and document-provider panes

## Problem

On the Artifacts → Research sub-tab, the query `path:multi_cli_orchestration_vs_sase`
returns `0 matches` even though a research report with that name exists. If you type the
same text without the `path:` prefix, it matches.

## Root cause

Commit `df465e063` ("add identity query fields and identity-reveal rung", sase-w3.5)
turned `path` from a search-only free-text field into a filterable **identity field**
with `exact_match=True`. It did this in two schemas:

- `src/sase/ace/query_profile/profiles/_plans.py` (`plans_query_schema`, pane
  `ref:plan`): `path` is in the shared `string_fields` loop, which sets
  `exact_match=True` for every key.
- `src/sase/ace/query_profile/profiles/_provider.py` (`provider_query_schema`, used by
  document-provider panes such as `ref:research` through
  `_artifact_tab_contract_provider.provider_query_profile`): the fallback `path` field
  is added with `exact_match=True`.

The Rust evaluator (`evaluate_many`) gets `exact_match` over the profile wire and treats
it as case-insensitive equality. The Python reference evaluator does the same
(`profile_evaluator_matching._match_text_field`). A row's `path` value is the document's
full path or provider identity (`query_rows.plan_query_entry` → `record.identity`, for
example `/…/sase--research/reports/…/multi_cli_orchestration_vs_sase.md`). As a result,
`path:<basename-fragment>` can never match. Only the full, exact path works. Exact
matching was chosen so that identity reveal (`link_reveal.build_identity_reveal_query`)
could write `path:"<full identity>"`. As a side effect, the user-facing `path:` filter
lost the substring behavior that people naturally expect from a path filter.

I reproduced this against the real Rust binding with a one-row index whose path is
`/home/bryan/x/sase--research/reports/2026/multi_cli_orchestration_vs_sase.md`. The
results were the same under both the `ref:plan` profile and
`provider_query_schema("research", None)`:

- `path:multi_cli_orchestration_vs_sase` → no match
- `path:<full path>` → match
- `multi_cli_orchestration_vs_sase` (free text) → match

## Fix

Make `path` match by substring (`exact_match=False`) in both schemas. Leave everything
else unchanged: `path` stays filterable, searchable, repeatable (Plans), negatable, and
the declared `identity_field`.

This needs no Rust change. `exact_match` is already profile data, and Rust applies
substring matching when it is false. This fits the Rust-core boundary, because the
per-pane dialect schemas are authored in Python by design.

Identity reveal still works. It writes the row's full identity, and a full absolute path
or provider identity as a substring matches that row. In practice it matches only that
row: a second match would need another identity that contains the entire first identity,
such as a `foo.md.bak` sibling. That is an acceptable, harmless widening of the reveal
lens and much better than a broken user-facing `path:` filter.

The profile digest changes as a result. That is the intended stale-query detection
mechanism, and no test pins a literal digest.

## Changes

1. `src/sase/ace/query_profile/profiles/_plans.py`
   - In the `string_fields` generator, set `exact_match=key != "path"` (keep `kind`,
     `status`, `tier`, `project` exact).
   - Update the docstring to say that `path` matches by substring so that
     `path:<fragment>` finds a document by any part of its path, while identity reveal's
     full-path query still names the row.
2. `src/sase/ace/query_profile/profiles/_provider.py`
   - In the fallback `path` `QueryFieldSpec`, remove `exact_match=True` (default
     `False`) and add a short comment explaining why (substring filter; identity reveal
     still works with the full identity).
3. Tests
   - `tests/test_query_profile_plans.py`: change
     `assert profile.field("path").exact_match is True` to `is False` (other plans
     string fields remain exact; optionally assert `kind` is still exact).
   - `tests/test_query_profile_provider.py`: add an assertion that the fallback `path`
     field is not `exact_match`.
   - `tests/test_query_profile_corpus_facade.py` (it already builds Rust-backed indexes;
     follow its existing helpers and fixtures): add a regression test for both
     `plans_query_schema()` and `provider_query_schema("research", None)`. Use one row
     whose `path` is a nested `.../multi_cli_orchestration_vs_sase.md` path plus a
     second, unrelated row. Assert that:
     - `path:multi_cli_orchestration_vs_sase` matches only the first row;
     - `path:MULTI_CLI` matches it too (case-insensitive);
     - `path:"<full path>"` matches only the first row (identity reveal still works);
     - `-path:multi_cli_orchestration_vs_sase` excludes it.
   - `tests/ace/tui/artifacts_contract/goldens/query/profile_cases.json`, `ref:plan`
     section: add a query case `"path:plans/"` whose canonical is `path:plans/` and
     which matches `["plan-proposal", "plan-active"]` (not `plan-archive`). This locks
     in substring semantics in the conformance goldens that both the Python reference
     and Rust evaluators check. If the conformance test regenerates or validates
     canonicals differently, follow its existing convention.
   - Run the identity-reveal tests (`tests/ace/test_link_reveal.py`,
     `tests/ace/tui/test_link_follow_ladder.py`, `tests/test_identity_query_parsers.py`)
     and confirm they still pass. Update expectations only if one asserted that a path
     fragment does _not_ match.

## Verification

- Read the `lint_and_test` reference memory and run the verification recipe it
  prescribes (`just check`).
- Optionally, in the live TUI, open Artifacts → Research, enter
  `path:multi_cli_orchestration_vs_sase`, and confirm the report is listed.

## Out of scope

- Glob or regex path matching.
- Changing any other pane's exact-match identity fields (Files `id`, Stitches `sha`,
  Beads `id`, and so on).
