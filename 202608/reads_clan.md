---
tier: tale
title: Put the
goal:
  "The #sase/reads swarm carries no obsolete %g:read directive and every agent of one
  invocation belongs to the same rootless clan named with the indexed reads-@ shape,
  with no tribe assigned."
size: small
proposed_by: bbugyi200.athena.yr
create_time: 2026-08-12 13:16:11
status: wip
---

# Plan: Put the `#sase/reads` swarm in one `reads-@` clan

## Goal

Fix the `#sase/reads` xprompt definition (`sase/xprompts/reads.md`) so that:

1. the obsolete `%g:read` directive is gone from all four segments, and
2. all four agents of one invocation are members of the same rootless agent clan named
   with the indexed `reads-@` shape, with **no tribe** assigned.

## Background: why `%g:read` is a real bug, not just noise

- `%g` / `%group` is a removed directive spelling. The live grammar in
  `src/sase/xprompt/_directive_types.py` lists only
  `auto, clan, effort, hide, model, id, repeat, wait` in `_KNOWN_DIRECTIVES`, with
  aliases `a, c, e, h, i, m, n, r, t, w` — there is no `g`. `%g` is not even in
  `_DEPRECATED_DIRECTIVE_MESSAGES`, so it raises no migration error.
- Unknown directives are _not_ stripped from the prompt. Verified live:

  ```python
  extract_prompt_directives("%id:reads.1.agy\n%model:claude/opus\n%g:read\nFind me articles.")
  # -> cleaned == "%g:read\nFind me articles."
  ```

  So today every one of the four reads agents receives a stray `%g:read` line at the top
  of its prompt, and no grouping happens at all. Removing it is a behavior fix.

- The modern replacement for grouping parallel agents is an **agent clan**: `%clan` on
  the one declaring segment plus `%id(<id>, clan=<clan>)` on every other member
  (`docs/xprompt.md`, "Clans" section around the `%clan` table entry).

## Current state (`sase/xprompts/reads.md`, lines 44-69)

```text
%id:reads.{@1}.agy
%model:agy/gemini-3.6-flash-high
%g:read
#_article_search_agent

---

%id:reads.{@1}.cld
%model:claude/opus
%g:read
#_article_search_agent

---

%id:reads.{@1}.cdx
%model:codex/gpt-5.6-sol
%g:read
#_article_search_agent

---

%id:reads.{@1}.final
%wait:reads.{@1}.agy
%wait:reads.{@1}.cld
%wait:reads.{@1}.cdx
%g:read
```

The frontmatter (`description`, `input`, the `_article_search_agent` local helper) and
all prose bodies below these directive lines are **unchanged by this plan**.

## Target state

Replace exactly those directive lines with:

```text
%id:reads-{@1}.agy
%clan:reads-{@1}
%model:agy/gemini-3.6-flash-high
#_article_search_agent

---

%id(cld, clan=reads-{@1})
%model:claude/opus
#_article_search_agent

---

%id(cdx, clan=reads-{@1})
%model:codex/gpt-5.6-sol
#_article_search_agent

---

%id(final, clan=reads-{@1})
%wait:reads-{@1}.agy
%wait:reads-{@1}.cld
%wait:reads-{@1}.cdx
```

Nothing else in the file changes. After the edit the file must contain **no** `%g:`
occurrence and **no** `reads.{@1}` occurrence.

Resulting agent identities for one invocation (token allocated per invocation):

| Segment   | Agent name      | Clan role                         |
| --------- | --------------- | --------------------------------- |
| 1 (agy)   | `reads-0.agy`   | declares clan `reads-0` (`%clan`) |
| 2 (cld)   | `reads-0.cld`   | joins via `clan=`                 |
| 3 (cdx)   | `reads-0.cdx`   | joins via `clan=`                 |
| 4 (final) | `reads-0.final` | joins via `clan=`, waits on 1-3   |

## Design decisions and the rules they satisfy

1. **The clan name is written `reads-{@1}`, i.e. the `reads-@` shape spelled with a
   keyed agent-name marker.** Do **not** write a bare `%clan:reads-@`. Epic `sase-aq`
   (plan `202607/agent_name_key_markers.md`) migrated this very file off bare `@`
   markers because bare markers resolve _latest-wins_ and one swarm launch can steal
   another's hood; its acceptance record states that no swarm body may still use a bare
   `@` agent-name marker. `{@1}` is resolved once per dispatch, before any name
   planning, so every `%clan` / `%id` / `clan=` / `%wait` occurrence of one invocation
   collapses onto the same freshly allocated `reads-<token>` hood.
2. **The separator changes from `reads.{@1}` to `reads-{@1}`** because the requested
   clan name is `reads-@`. Clan members must live in the `<clan>.<suffix>` hood
   (`prepare_clan_launches` in `src/sase/agent/multi_prompt_launch_plan.py` rejects
   anything else), so the member ids become `reads-<token>.agy` and friends rather than
   `reads.<token>.agy`. Both `reads-{@1}` and `reads-{@1}.agy` reserve the same
   namespace `reads-{@1}`, so one token covers the clan container and all four members.
3. **One declaration, three joins.** `%clan` is create-only, requires a hood-qualified
   `%id`, and may appear once per resolved clan per launch; every other member uses
   `%id(<id>, clan=<clan>)`, which derives `<clan>.<id>`. Combining `%clan` with
   `%id(..., clan=...)` in a single segment is a hard error
   (`_validate_clan_directive_contract` in `src/sase/xprompt/_directive_collect.py`),
   which is why segment 1 keeps a full `%id:` and segments 2-4 use the paren join form.
4. **No tribe**, as requested: no `tribe=` on `%clan` and no `%id(..., tribe=...)`. Note
   that `%tribe` / `%t` no longer exists as a directive at all.
5. Directive order inside a segment does not matter to the parser; `%id` then `%clan`
   then `%model` is used above so the identity directives stay together.

### Feasibility already verified against the real code

A read-only probe built the target content in memory (no repo file was modified), loaded
it with `load_xprompt_from_file`, expanded **two** `#reads(...)` references in one
dispatch with `expand_xprompt_swarms_with_metadata`, ran
`resolve_agent_name_key_markers`, and then inspected each segment with
`extract_static_clan_directive` / `extract_static_name_directive`. Result:

- invocation A -> clan `reads-0`, members `reads-0.agy`, `reads-0.cld`, `reads-0.cdx`,
  `reads-0.final`; exactly one segment with `declared=True`, `tribe=None`;
- invocation B -> clan `reads-1` with the same four suffixes;
- the three `%wait:` targets in each `final` segment were rewritten into that
  invocation's own hood.

So the target state parses, resolves, and satisfies the hood rule; the implementation is
a mechanical edit plus a regression test.

## Steps

1. Edit `sase/xprompts/reads.md` exactly as described in **Target state**.
2. Add the regression test below to `tests/test_agent_name_key_markers.py` (that module
   already owns the keyed-marker acceptance tests and the `_configure_allocation`
   helper). Leave
   `tests/test_xprompt_swarm_local_helpers.py::test_checked_in_reads_xprompt_uses_direct_local_helper`
   untouched — it asserts helper/model/prose facts that this change preserves, and it
   must stay green.
3. Run verification (below) and fix anything it reports.

## Regression test to add

Add to `tests/test_agent_name_key_markers.py`, importing
`extract_static_clan_directive`, `extract_static_name_directive`,
`expand_xprompt_swarms_with_metadata`, `load_xprompt_from_file`, and
`tests._xprompt_swarm_helpers.patch_catalog` at module scope (keep ruff/mypy happy):

```python
def test_checked_in_reads_swarm_declares_one_clan_per_invocation(
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    """Every #sase/reads invocation puts its four agents in one reads-<token> clan."""
    reads_path = Path(__file__).resolve().parents[1] / "sase" / "xprompts" / "reads.md"
    source = reads_path.read_text(encoding="utf-8")
    assert "%g:" not in source
    assert "reads.{@1}" not in source

    reads = load_xprompt_from_file(reads_path)
    assert reads is not None
    with patch_catalog({"reads": reads}):
        records = expand_xprompt_swarms_with_metadata(
            ["#reads(episodic agent memory)", "#reads(context rot)"]
        )

    _configure_allocation(monkeypatch, ["0", "1"])
    resolved = resolve_agent_name_key_markers([record.prompt for record in records])
    assert len(resolved) == 8

    for invocation, clan_name in ((resolved[:4], "reads-0"), (resolved[4:], "reads-1")):
        clans = []
        for segment in invocation:
            clan = extract_static_clan_directive(segment)
            assert clan is not None
            clans.append(clan)
        assert {clan.name for clan in clans} == {clan_name}
        assert [clan.declared for clan in clans] == [True, False, False, False]
        assert all(clan.tribe is None for clan in clans)
        assert [extract_static_name_directive(segment) for segment in invocation] == [
            f"{clan_name}.agy",
            f"{clan_name}.cld",
            f"{clan_name}.cdx",
            f"{clan_name}.final",
        ]
        for suffix in ("agy", "cld", "cdx"):
            assert f"%wait:{clan_name}.{suffix}" in invocation[3]
```

`_configure_allocation` must be called before `resolve_agent_name_key_markers` so the
token sequence is deterministic; expansion itself does not touch the allocator.

## Verification

1. `just install` (workspaces are ephemeral, so install before anything else).
2. `just test tests/test_agent_name_key_markers.py tests/test_xprompt_swarm_local_helpers.py`
   — new test passes, existing checked-in-reads test still passes.
3. `sase xprompt show sase/reads` — the definition renders and the directive lines show
   the new `%clan` / `clan=` wiring with no `%g:`.
4. `just check` before replying (whole-repo lint gates plus the diff-scoped test lane).
   `sase/xprompts/reads.md` is a data file, so rely on step 2 for the behavioral proof.
5. Optional, owner-gated live smoke (launches four real agents, so do **not** run it
   unprompted): `sase run '#sase/reads(agent memory systems)'`, then check in ACE that
   one `reads-<token>` clan row contains all four members and that no agent prompt
   contains `%g:read`.

## Out of scope

- No tribe assignment (explicitly requested).
- No clan `summary=` / `summary_script=`. The sibling `research_swarm` xprompt in the
  `sase-research` plugin sets a rich-markup clan summary; adding an equivalent
  `summary=[[[bold]READING REQUEST:[/bold] {{ topic }}]]` here would be a separate,
  optional follow-up.
- No docs change: the `#!sase/reads` section of `docs/development.md` documents inputs
  and behavior only, never agent names or grouping.
- No changes to `~/sase/xprompts/`, the chezmoi source of home xprompts, or the
  `sase-research` plugin's `research_swarm.md`.
- Nothing in `src/` changes; the clan machinery already supports this shape.

## Risks

- If a `reads-<token>` name or hood is already reserved, the key allocator simply skips
  to the next free token; no action needed.
- Two `#sase/reads` invocations in one dispatch get distinct clans (verified above), and
  a keyed marker cannot be stolen by a later launch (`sase-aq`).
- The only thing tests cannot prove is the ACE clan row rendering, which is why the live
  smoke check is listed as optional.
