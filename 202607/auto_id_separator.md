---
create_time: 2026-07-13 07:44:38
status: wip
prompt: 202607/prompts/auto_id_separator.md
tier: tale
---
# Generalized `@` Auto-ID Separators

## Goal

Move separator handling into the shared agent-name template contract so callers can write the natural template shape and
still get readable names when an auto-ID starts with a letter. Then remove the hard-coded dash from the standard fork,
wait, and retry templates:

- fork/resume: `<base>.f@`
- wait: `<base>.w@`
- retry: `<base>.r@`

The renderer will insert one dash immediately before a letter-leading auto-ID only when the `@` marker has a left-hand
neighbor and that neighbor is neither `-` nor `.`. Digit-leading IDs remain adjacent to the prefix. A marker at the
start of a template remains unmodified because there is no preceding name segment to disambiguate.

| Template | Token | Concrete name | Reason                                            |
| -------- | ----- | ------------- | ------------------------------------------------- |
| `foo.f@` | `0`   | `foo.f0`      | digit-leading IDs need no separator               |
| `foo.f@` | `a`   | `foo.f-a`     | `@` follows `f`, so a dash is inserted            |
| `foo.f@` | `0a`  | `foo.f0a`     | only the first token character controls insertion |
| `foo.f@` | `a0`  | `foo.f-a0`    | the first token character is a letter             |
| `foo-@`  | `a`   | `foo-a`       | the template already supplies a dash              |
| `foo.@`  | `a`   | `foo.a`       | the template already supplies a dot               |
| `@.cld`  | `a`   | `a.cld`       | a leading marker has no ambiguous left segment    |

Rendering, matching, latest-name lookup, allocation, and namespace reservation must all agree on these shapes. This is
not only a presentation change: matching must recover `a` from `foo.f-a`, reject non-canonical variants such as
`foo.fa`, and continue to recover numeric tokens from names such as `foo.f0`.

## Plan

1. **Change the authoritative template contract in `sase-core`.**
   - Update `crates/sase_core/src/agent_name_template.rs` so `AgentNameTemplate::render` computes the conditional
     separator from the template prefix and the first character of the validated token.
   - Make `AgentNameTemplate::match_token` the exact inverse: separator-protected prefixes (`-`/`.`) accept the normal
     token shape; other non-empty prefixes accept an adjacent digit-leading token or a dash-prefixed letter-leading
     token; start-of-template markers retain their existing direct token shape.
   - Keep parsing, token generation/order, namespace-template derivation, and the PyO3 API signatures unchanged. Because
     namespace names are rendered through the same primitive, they will automatically inherit the new rule.
   - Expand the Rust unit matrix for numeric and alphabetic single-character tokens, mixed multi-character tokens,
     leading markers, existing dash/dot separators, matching round trips, and rejection of missing or spurious dashes.

2. **Release and consume the breaking core behavior deliberately.**
   - Treat the changed render/match semantics as a breaking `sase-core` change so release-plz produces the next
     compatibility series; do not manually edit core workspace/crate versions.
   - After that core release is available, update SASE's `sase-core-rs` dependency range in `pyproject.toml` to require
     the new series (expected `>=0.4.0,<0.5.0`) and regenerate `uv.lock`. This prevents SASE from emitting `.f@`, `.w@`,
     or `.r@` templates against an older binding that would render letter tokens without the new separator.

3. **Use the generalized renderer consistently from SASE.**
   - Update the Python adapter documentation and `tests/test_agent_name_templates.py` to mirror the Rust behavior,
     including render/match round trips, allocation across the `9` to `a` boundary, latest-name resolution, and
     namespace reservation for templates whose marker follows an ordinary character.
   - Replace the remaining direct `str.replace("@", token)` suffix rendering in the standard plan-chain evaluator and
     question fallback path with `render_agent_name_template`, so every agent-family/template path observes the same
     separator rule rather than maintaining local substitution semantics.
   - Retain current behavior for existing family suffix templates such as `--@`, `--code-@`, and `--plan-@`, because
     their markers already follow a dash.

4. **Remove the standard derived-name dash and propagate the new concrete names.**
   - Change `resume_agent_name_template`, `wait_agent_name_template`, and `retry_agent_name_template` to return `.f@`,
     `.w@`, and `.r@`, and update their docstrings plus repeat-launch documentation.
   - Update affected behavior tests across extraction/directive handling, repeat and multi-prompt planning, model/alt
     fan-out, planned launch environment values, fork/wait prompt rewriting, mobile retry flows, and ACE retry editing.
     Numeric examples should now use `foo.f0`, `foo.w0`, and `foo.r0`; tests that exhaust `0` through `9` should prove
     the next names are `foo.f-a`, `foo.w-a`, and `foo.r-a`.
   - Update `docs/xprompt.md` and any nearby user-facing examples to describe the new standard sequence.

5. **Preserve historical artifacts without introducing alias matching.**
   - Do not rename or migrate existing agent artifacts. Registry entries remain exact names.
   - Assert the intentional compatibility consequences: historical no-dash numeric names such as `foo.f1` now occupy the
     corresponding new token because they are exact canonical renderings; dash-based numeric names such as `foo.f-1`
     remain distinct and do not reserve `foo.f1`; dash-based letter names such as `foo.f-a` do reserve token `a` because
     that is the new canonical letter rendering.
   - Apply the same coverage to wait and retry names, including descendant namespace reservations, so allocation never
     creates an exact collision while avoiding loose legacy aliases that would break render/match symmetry.

6. **Verify both repositories and the integrated contract.**
   - In `sase-core`, run formatting, Clippy with warnings denied, and the full Cargo workspace test suite.
   - Build/install the linked core binding in SASE, run focused agent-template and derived-name tests first, then run
     `just install` followed by the required `just check` for the complete Python/TUI suite.
   - Finish with repository-wide searches for stale `.f-@`/`.w-@`/`.r-@` producers, direct `@` token substitution, and
     obsolete `.f-0`/`.w-0`/`.r-0` expectations, retaining old spellings only in explicit compatibility fixtures.

## Delivery sequencing

This requires two coordinated repository changes. Land and publish the breaking `sase-core` contract first; then land
the SASE change that requires that released core version and switches the standard templates. Keeping that ordering
avoids a release window in which the Python code requests the new template shapes while an allowed older core silently
renders the wrong letter-leading names.
