---
tier: epic
title: Rust core contracts for Plan Decisions
goal: Complete sase-1hi.1 with typed decision validation, frozen definitions, deterministic
  resolution, human-quote matching, and shared reviewer and implementer output exposed
  through tested Python bindings.
phases:
- id: grammar
  title: Validate decisions and preserve additive plan wire compatibility
  depends_on: []
  size: medium
  description: 'grammar: implement all decisions diagnostics, source lines, Archived
    mode, branch callouts, the additive validated wire, and schema and binding parity
    tests.'
- id: resolve
  title: Freeze definitions and resolve one accepted answer vector
  depends_on:
  - grammar
  size: medium
  description: 'resolve: define host facts, effective defaults, the definitions digest,
    strict value resolution and the agent memory boundary, with bindings and tests.'
- id: quotes
  title: Match human quotes with Unicode normalization and useful suggestions
  depends_on:
  - resolve
  size: small
  description: 'quotes: implement the normalized three-word contiguous matcher, deterministic
    closest-sentence suggestions, dependency integration, and the Python binding and
    tests.'
- id: sheet
  title: Build the Decision Sheet, summaries, and implementer instructions
  depends_on:
  - quotes
  size: medium
  description: 'sheet: build the shared sheet, both summary forms, all implementer
    audiences and inherited instructions, and complete the seven-binding integration
    coverage.'
proposed_by: bbugyi200.apollo.sase-1hi.1
parent_bead: sase-1hi.1
create_time: 2026-10-07 19:00:02
status: wip
bead_id: sase-1hi.1.1
---

- **PROMPT:** [prompts/202610/core_plan_decisions.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/core_plan_decisions.md)
- **PARENT:** [202610/plan_decisions.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions.md)
- **BEAD:** [sase-1hi.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.1.1.md)

# Rust core contracts for Plan Decisions

## Scope, sources, and ownership

This child epic completes the Rust backend work assigned to **sase-1hi.1**, the `core`
phase of **sase-1hi**. SASE's proposal handler attaches the child epic to the active
phase automatically. The outer epic remains owned by its land agent.

The authoritative design is **plan:202610/plan_decisions.md**, especially Sections 1.6,
2, 3, 4, and 6.1. Its accepted background reports are
**research:202610/plan_frontmatter_decisions/plan_frontmatter_decisions.md** and
**research:202610/plan_decisions_cross_surface_ux/plan_decisions_cross_surface_ux.md**.
Read these with `sase artifact read <ref> "<reason>"`. The approved design wins where
the reports differ: the whole definition is frozen, the wire change is additive, and
`decided_by` uses `reviewer | auto | agent`.

Every implementer opens the linked repo with
`sase repo open sase-core -r "Implement the Rust contracts for sase-1hi.1"` from their
own sase workspace, then reads the printed checkout's `AGENTS.md`. Use that returned
path throughout. All domain behavior and serde contracts belong in `crates/sase_core`;
PyO3 wrappers belong in `crates/sase_core_py`.

The Python host supplies resolved memory identities and human-authored texts. It owns
provenance gathering, note resolution and overlap detection, receipt acceptance,
revision checks, stamping, inheritance lookup, and frontend widgets in the later outer
phases. The outer `gate` phase moves the sase-core revision pin when it starts calling
these new bindings. This child epic delivers the core API and its registration and
tests.

The four phases are intentionally sequential: they extend the same plan facade, binding
registration, and wire records. Each is a bounded direct implementation unit. This keeps
intermediate commits buildable and gives each following worker the exact contracts it
consumes.

## Existing seams and implementation structure

The existing code was inspected before authoring this plan:

- `crates/sase_core/src/plan/validate.rs` owns `Validator`, `SourceIndex`,
  `PlanValidationMode`, the normalized wire, and field-schema metadata. It is already
  over the new-file guideline; add short hooks and required field/mode changes there,
  with new parsing and behavior in separate modules.
- `crates/sase_core/src/plan/mod.rs` exports the domain; import new items as
  `sase_core::plan::...`, without adding root or prelude aliases.
- `crates/sase_core/tests/plan_validate_parity.rs` pins an exact legacy JSON fixture at
  schema version **3**. Keep that fixture byte-compatible.
- `crates/sase_core_py/src/plans/mod.rs` owns `register_plans`, error translation, and
  the JSON bridge. Its existing `tests.rs` demonstrates registration and round-trip
  checks through a real `sase_core_rs` module.
- `crate::finalizer::canonical_json_sha256` already canonicalizes nested object keys and
  preserves array order. Reuse it, mapping errors into `PlanError`, rather than making a
  second canonical serializer.

Create `crates/sase_core/src/plan/decisions/` with a facade `mod.rs`, domain wire,
grammar, callout, resolver, quote, and presentation modules as needed. Keep new files at
or under 1,500 lines, and tests beside their implementation. If source indexing needs to
be extracted, move it mechanically into a sibling module and preserve its phase-index
behavior. Diagnostics must use the existing diagnostic envelope and
`Validator::push`/`push_direct` conventions; do not run a second top-level YAML parse
just to validate decisions.

Add `crates/sase_core_py/src/plans/decisions.rs`, or split it into a directory with a
facade when necessary. Use the existing JSON bridge, typed serde input records,
`PyValueError` for invalid binding arguments, and normal serialization for outputs.
Domain errors are `PlanError`/typed errors, not panics.

## Grammar

Implement all of Section 2 of the outer design on both tale and epic plans. `decisions`
is optional, ordered, and a map with at most five entries. Duplicate YAML keys must
remain rejected; every accepted vector preserves YAML author order, including choice
order. An absent or empty map yields no new serialized plan fields.

| Input                       | Validation and diagnostic                                                                                                                                                                                                                                                                                                                                          |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Decision fields             | Allow only `ask`, `choices`, `default`, `why`, `memory`, `requested`, and the system-written `answer`; unknown fields produce `decision-unknown-field`. More than five decisions produces `decision-limit`.                                                                                                                                                        |
| Decision id                 | Match `^[a-z][a-z0-9_]*$`, maximum 32 characters (`decision-id-invalid`). Ban `y n yes no on off true false null`, and the reserved names `approve commit reject feedback coder_prompt coder_model wait epic_launch_mode capacity` (`decision-id-reserved`).                                                                                                       |
| `ask`                       | Required non-empty one-line text of at most 120 characters (`decision-ask-invalid`); missing trailing `?` warns with `decision-ask-not-question`. Count Unicode characters, not UTF-8 bytes.                                                                                                                                                                       |
| Choice                      | Presence of `choices` selects the choice kind. Require a map with 2–5 entries (`decision-choices-count`). Keys use the id spelling and YAML-word exclusions, maximum 24 characters; host reserved names need not be banned here. Labels are non-empty one-line consequences of at most 100 characters (`decision-choice-invalid`).                                 |
| Toggle                      | Absence of `choices` selects the toggle kind. Its default must be an actual YAML boolean. Reject strings, including `yes`/`on`, with a true/false remedy.                                                                                                                                                                                                          |
| `default`                   | Required (`decision-default-missing`). A choice default must exactly name an authored key; otherwise `decision-default-invalid`. A toggle accepts only booleans with the same invalid-default code.                                                                                                                                                                |
| `why`                       | Optional non-empty one-line text of at most 100 characters. Presence on any memory decision is `decision-why-on-memory`.                                                                                                                                                                                                                                           |
| `memory`                    | Toggle only (`decision-memory-on-choice`). Require a non-empty list of non-empty one-line read selectors (`note.md`, `web`, `web:keyword`), with non-empty web/keyword components (`decision-memory-selector-invalid`). Match the host selector forms, including nested relative note paths and keyword aliases; core checks syntax, the host resolves identities. |
| `requested`                 | Memory only (`decision-requested-not-memory`), one line, 3–300 characters. Required when the authored memory default is true (`decision-requested-missing`). The three-word minimum is additionally enforced by the quote matcher.                                                                                                                                 |
| `answer`                    | Forbidden in Authoring (`decision-answer-forbidden`). In Launch/Archived validate it against the decision kind and choices, with actual boolean values for toggles.                                                                                                                                                                                                |
| `decided_by`, `decided_via` | Top-level system fields, forbidden in Authoring with a diagnostic pointing at the exact field. `decided_by` is `reviewer`, `auto`, or `agent`; `decided_via` is `tui`, `telegram`, `mobile`, or `cli`, and must be absent for auto. Validate values in Launch/Archived.                                                                                            |
| `phases[].when`             | Always fail with `phase-when-reserved`, even if its value is false or null.                                                                                                                                                                                                                                                                                        |

Use specific decision diagnostics for malformed map/text/answer shapes not assigned a
code above, following the existing `decision-...-invalid` vocabulary; document and pin
each in tests. Accumulate independent problems rather than abandoning validation after
the first malformed decision.

Add `PlanValidationMode::Archived` and accept the binding mode string `"archived"`. It
retains Authoring's strict size and structural validation, while allowing system-written
answers. An unstamped archived plan is valid. If any answer/provenance stamp exists,
Archived requires `decided_by` and a valid `answer` on every decision together
(`decision-answer-incomplete`). Test partial vectors, top-level-only stamps, answer-only
stamps, invalid provenance, and auto with a transport field. Launch continues its
existing legacy size normalization and warnings while accepting valid answers. Preserve
unrelated Authoring rules for pre-existing SASE-managed fields.

Extend `SourceIndex` to point `decisions.<id>` and nested fields, including choice keys,
to the actual document lines in block YAML; flow mappings use the nearest containing
field fallback. Include quoted keys, comments, CRLF, and source offsets in tests. Body
diagnostics and callout spans use original Markdown line numbers, including frontmatter,
rather than body-relative offsets.

Parse callouts outside backtick and tilde fenced code. A header is `> [!decision] <id>`,
`<id> = <key>`, or `<id> = no`, and may have prose after the branch marker. A bare
toggle selects yes; `= no` selects no. Choices must name a defined key. Unknown
ids/branches fail with `decision-branch-unknown`. Return inclusive original-document
blockquote spans, ending at the blockquote boundary, including multiline and adjacent
callout cases. The parser identifies markers; it does not remove or execute text.

Warn with `decision-unreferenced` only when an id has neither a valid callout nor a
whole-word mention in the body. Ignore fenced examples for executable branch validation.
Add `decision-memory-uncovered` for apparent memory edits: `sase memory init`, or a
named `sase/memory/` path on a line with an edit verb (add, edit, update, create,
delete, remove, rewrite) without a covering memory decision. Read-only references must
stay quiet. Syntax coverage is an advisory heuristic; durable identity overlap and
authorization stay with the host.

Tune that heuristic using at least 20 actual archived memory-editing plans from the
accepted report's archive population, plus read-only controls. Discover via the plan
CLI, read each selected artifact with `sase artifact read`, and commit compact
representative regression inputs with source refs; do not copy whole sidecar reports
into tests. Record which artifacts and false-positive cases were checked in the phase
note.

Expose the following additive validated-plan fields with serde defaults and
`skip_serializing_if` for empty vectors/absent options:

```text
decisions: Vec<PlanDecisionWire>
decision_callouts: Vec<PlanDecisionCalloutWire>
decided_by: Option<reviewer | auto | agent>
decided_via: Option<tui | telegram | mobile | cli>

PlanDecisionWire = {
  id, kind: toggle | choice, ask, why?,
  choices: [{key, label}], default: boolean | choice-key,
  memory?: {selectors}, requested?, answer?
}
PlanDecisionCalloutWire = {
  id, key?: string, branch: yes | no | choice, start_line, end_line
}
```

Keep `PLAN_WIRE_SCHEMA_VERSION = 3`. Update Rust struct literals that construct
validated plans without decisions. Add ordered schema rows with examples for
`decisions`, `.ask`, `.choices`, `.default`, `.why`, `.memory`, and `.requested`, plus
system-field guidance as appropriate. Update
`schema_is_ordered_and_contains_exact_phase_guidance`, legacy parity tests, and the
existing `plan_validate`/`plan_frontmatter_schema` binding round-trips.

## Frozen definitions and resolution

Provide `plan_decisions_payload(validated, host_facts)` taking the normalized validated
plan object, not the outer validation envelope, and returning an ordered
`Vec<PlanDecisionDefinitionWire>`. Define and document a typed host-facts object keyed
by decision id. Each fact carries `requested_verified`, provenance
`asked | not_asked | quote_not_found | inherited`, and resolved memory records:

```text
{ selector, kind: note | web | strand, scope: project | home, path,
  type: core | reference | web | strand, exists, strands? }
```

Preserve the optional frozen strand list and all durable scope/path identities. The host
resolves these records; Rust does not discover or read memory files. Missing
verification facts fail closed. A memory decision authored with default true has
effective default false unless `requested_verified` is true. Preserve its authored
default and quote for explanation, and return the warning provenance needed by the
sheet. Inherited authorization is supplied as trusted host context, not inferred from
quote text or a submitted answer.

The frozen definition contains authored question, ordered choices, why, default, memory
selectors, requested quote, effective default, verification/provenance, and resolved
records. System-written answers are not question definitions. Provide
`plan_decisions_digest(definitions)`: SHA-256 of canonical definition JSON, sorting
object keys recursively while preserving every ordered array. Changing the ask, labels,
why, defaults, selector scope, or frozen strand list changes the digest; object
insertion order does not. Answer vectors, plan prose, and review revision do not
participate. Missing optional fields round-trip consistently so JSON-to-Rust-to-JSON
yields the same digest.

Provide `plan_decisions_resolve(definitions, submitted, caller)` with caller
`human | agent | auto`. Its response is:

```text
{ values: {id: canonical-value},
  rows: [{id, value, source: default | submitted | clamped, changed}],
  errors: [{id, code, message, allowed, default}] }
```

Resolve in author order. Omitted values take the effective default. Toggles require JSON
booleans: strings, integers, and null fail. Choice values must be strings and match a
full key case-insensitively; return the authored spelling, with no prefix matching.
Unknown ids and invalid values are errors containing the allowed values and effective
default. Validate all entries and accumulate errors; an error result cannot be consumed
as a successful accepted vector.

An agent cannot supply true for a memory decision whose effective default is false:
return `memory_decision_requires_human`. An agent may retain an already verified
effective true default or turn it off. A human may enable an unrequested decision. Auto
uses effective defaults only, never an override. Validate any submitted ids and value
shapes, then ignore valid overrides and report sources as default or clamped; unknown
ids and invalid values remain errors. Unknown caller strings are usage errors, never
human callers.

Rows identify clamped omitted memory defaults and calculate changes against the
effective review default. Omitted versus explicitly submitted defaults yield identical
`values`, even though explanatory `source` differs. Later receipt identity uses the
canonical vector, not source labels.

Add bindings `plan_decisions_payload`, `plan_decisions_digest`, and
`plan_decisions_resolve` under the plans binding domain. Pin round-trips for ordinary
toggles, choices, verified and unverified memory, frozen webs and inherited provenance,
missing host facts, case canonicalization, error lists, and invalid binding input/caller
shapes.

## Human quote matcher

Provide `plan_decision_quote_match(quote, texts)` with ordered typed input records
`{source, ref, text}` and output:

```text
{ verified: boolean, matched_source: optional source record,
  closest?: {source, ref, sentence} }
```

Keep the matching source identity usable by the host: retain its supplied source and ref
in the result. This API receives texts the host already classified as human-authored; it
does not authenticate a self-reported source or search the filesystem.

Normalize both sides with NFKC, full Unicode casefold, straight equivalents for
quotes/dashes, and collapsed Unicode whitespace. Strip leading/trailing punctuation from
the quote. Require at least three word tokens, then match a contiguous complete-word
sequence inside one supplied text. Do not join texts, match partial words, reorder
tokens, or accept fuzzy overlap as verification. Preserve meaningful internal
punctuation such as `.md` and apostrophes.

Use crate-backed normalization and full non-Turkic casefold rather than lowercase as a
substitute. The checked APIs are
[UnicodeNormalization::nfkc](https://docs.rs/unicode-normalization/latest/unicode_normalization/trait.UnicodeNormalization.html)
and
[UnicodeCaseFold](https://docs.rs/unicode-casefold/latest/unicode_casefold/trait.UnicodeCaseFold.html),
which supports full expansions such as `ß` to `ss`. Add the necessary dependency entries
and lockfile through the repo's build workflow, and regenerate the workspace
feature-unification dependencies if the features gate requires it. Follow the repo's
no-bare-cargo rule and its documented feature-repair recipe.

On failure, rank individual original human sentences by overlap with normalized quote
tokens. Make ties stable by input order and sentence order, return original sentence
text with its source/ref, and return no closest result for empty sources or no token
overlap. Sentences and quotes spanning newlines, `.md` filenames, curly punctuation,
combining characters, full-width characters, sharp-s, and Greek sigma need explicit
tests. A match inside a negated sentence remains a lexical match: intent is not this
function's contract.

Register `plan_decision_quote_match` and test both real module registration and JSON
round-trips, including the three-word boundary, noncontiguous text, two texts containing
separate halves, a partial-word false match, suggestions with ties, and empty/legacy
text arrays.

## Decision Sheet and implementer output

Provide `plan_decision_sheet(definitions, values, review_revision)` returning:

```text
PlanDecisionSheetWire = {
  count, memory_count, changed_count, review_revision,
  rows: [{id, kind, ask, why?, choices, default, value, changed,
          memory?: {selectors, resolved, provenance, quote}}]
}
```

Use the resolver's canonical vector and retain author order. Validate the vector rather
than coercing malformed/missing answers into display values. Each row's review `default`
is its effective default, so reset, Enter, and `%auto` agree; the frozen definition
retains the authored default for diagnostics. Copy the frozen resolved identities into
memory rows. `quote_not_found` and inherited provenance remain visible. Counts derive
from the rows, including an empty sheet.

Provide `plan_decision_summary(sheet, verdict, form)` using only supported verdicts
`coder + commit`, `coder`, `commit`, and `epic launch`, and forms `short | full`. The
common output rules are:

- Full: `→ <verdict> · <id>=<value>[ ●] · ... · 🧠 <notes>`. Non-memory decisions appear
  in author order; toggles display `yes`/`no`, choices their keys. End with one ordered,
  deduplicated memory selector clause for enabled memory decisions, or
  `🧠 no memory edits` if all memory decisions are off. Omit the clause when there are
  no memory decisions.
- Short: `defaults`, `1 change`, or `N changes`, followed by ` · 🧠` if any memory
  decision is enabled. Memory changes count toward changed_count too.
- Invalid verdict/form strings are usage errors, not interpolated text.

Provide
`plan_decisions_prompt_block(sheet, decided_by, decided_via, audience, inherited)` with
audience `tale_coder | epic_phase | epic_land` and optional inherited
`{sheet, epic_title}`. Render host-owned instructions from validated ids, enum keys,
booleans, memory selectors, and safely quoted asks/titles. Frontmatter strings must not
introduce extra instruction lines through raw interpolation.

The reviewer header names the actual surface (for example `reviewer via ACE` for `tui`).
The auto header explicitly says no human reviewed the plan, and the agent header
identifies agent approval honestly. For each choice or toggle, name the selected branch
and relevant excluded branches, using the outer design's copy, for example:

```text
Reviewer decisions for this plan (final · reviewer via ACE):
- grouping = mode (planner default: pane). Implement the "grouping = mode" branch; ignore "grouping = pane".
- tui_note = yes 🧠. Memory edits are authorized for tui.md only.
No other memory note may be edited. Implement only the branches selected above.
```

Include the quoted ask as context without treating it as executable text. Enabled memory
rows state their exact authorized selectors; disabled rows state which changes to skip.
Emit inherited grants as `Inherited from epic "<title>": ...`, preserving the epic's
accepted values and scopes. Empty local decisions still carry inherited authorization
when present.

When auto left an unrequested memory change off, instruct a tale coder to use
`/sase_new_task` for the skipped memory change. Phase and land audiences instead record
`PROPOSED FOLLOW-UP:` on their assigned bead. A human-declined change does not get a
follow-up instruction. Unverified quotes must never be portrayed as authorization.
Unknown audience/provenance/transport or malformed inherited sheets fail with an
actionable usage error.

Register `plan_decision_sheet`, `plan_decision_summary`, and
`plan_decisions_prompt_block`. Add exact-output tests for all verdicts, forms, caller
headers, audiences, empty and inherited-only sheets, approved versus declined memory,
multiple notes, changed toggles/choices, and quote/title escaping. A final integration
test calls all seven public bindings via the initialized Python module: validate a tale
with a choice and a memory toggle, compile host facts, digest, resolve, match its quote,
build its sheet, format both summaries, and render the implementer block. Add an
epic/Archived example.

## Verification and closeout

Every phase runs focused tests through `just test -p sase_core <filter>` and
`just test -p sase_core_py <filter>` as appropriate, using the repo's interpreter and
hermetic test wrapper. Run formatting before the final gate. Then run
`sase tool run check` from the opened sase-core checkout; targeted tests do not replace
that gate. If a command needs a long handoff, use `/sase_monitor` and wait for the
monitor-start command itself to exit. Do not finish a turn with a yielded execution
session still running.

The child land agent reviews the merged implementation against every item in outer
Section 6.1, verifies all seven bindings are registered, and checks legacy parity and
new wire round-trips. Re-run the combined check when integration changes or failures
justify it. For any sase-primary tracked change, also read `lint_and_test.md` through
`sase memory read` and run `sase tool run check` there.

Record the stable input/output shapes and test evidence on **sase-1hi.1** so the outer
`gate` phase can consume the APIs without guessing. Include the clamp and
effective-default semantics, digest contents, enum spelling, frozen web record shape,
quote result identity, and the core commit once host finalization provides it. Closing
the phase must not wait for this turn's own commit SHA.

Workers do not create follow-up beads. Record discovered work as
`PROPOSED FOLLOW-UP: <summary — detail>` on the assigned child phase, and the child land
agent carries relevant evidence to **sase-1hi.1** for the outer land agent to triage. A
check failure reproduced identically on the clean base does not keep a phase open:
record reproduction evidence and any existing tracking bead, then complete the phase
normally. Do not weaken assertions to mask it.

Each child worker closes only its assigned child phase after verification, first
checking its `sase bead epic-symbols <assigned-phase>`. Resolve leftover entries or
re-key them to a still-open consuming phase/epic.

After the child epic is complete and its own land workflow has closed it, its land agent
verifies that this delivered the entire assigned outer phase, runs:

```bash
sase bead epic-symbols sase-1hi.1
sase bead close sase-1hi.1 --note "<combined core checks, binding registration and round-trip tests, unchanged legacy parity, and heuristic archive evidence verified>"
```

Retire or re-key every remaining `--epic-symbol` entry before that close. This closes
only the assigned outer phase **sase-1hi.1**; **sase-1hi** and all ancestor plan beads
remain for their existing land agents.
