- **PLAN:** [../202607/sidecar_repo_labels.md](../sidecar_repo_labels.md)

 
%xprompts_enabled:false
# Previous Conversation

**User:**

# Previous Conversation

**User:**

I keep seeing the term SDD thrown around a lot, but that terminoligy should be completely replaced / removed at this point. Can you help me audit this codebase to determine everywhere that this term is used to give me an idea of how difficult it would be to remove/replace this term? Express your answer by setting some sase variables.

**Assistant:**

I'll audit the codebase for SDD usage. Let me start by getting a broad picture.

Audit done. Variables are set on this run (agent `n8`).

## Scalars

| Variable | Value |
|---|---|
| `sdd_total_occurrences` | 8,412 (sase repo, temp dirs excluded) |
| `sdd_files_touched` | 637 |
| `sdd_src_files` | 254 (61 = the `src/sase/sdd/` package, 188 outside it across 24 subpackages) |
| `sdd_test_files` | 316 |
| `sdd_doc_files` | 45 |
| `sdd_paths_to_rename` | 133 |
| `sdd_python_identifiers` | 571 distinct |
| `sdd_import_refs` | 916 `sase.sdd` refs across 261 files |
| `sdd_core_files` / `sdd_core_occurrences` | 29 / 534 (in `sase-core`) |
| `sdd_env_vars` | 9 |
| `sdd_difficulty` | **high** |

Also set: `sdd_scope_summary`, `sdd_hard_surfaces`, `sdd_easy_surfaces`.

## The shape of it

Roughly 90% of the 8,412 hits are mechanical — `src/sase/sdd/` is a self-contained 61-module package with 36 exported names, and the blast radius outside it is concentrated in `ace` (54 files), `main` (21), `bead` (21), `axe` (15). A rename + import rewrite handles that.

The cost is in 9 spots that aren't sed-able:

- **`sdd/` as a real on-disk directory** — 519 path literals (`sdd/beads` 265, `sdd/plans` 256, plus research/prompts/tales/epics). Only applies to `in_tree` storage.
- **`.sase/sdd-store.json`** — persisted state, needs read-both/write-new.
- **`<owner>/<repo>--sdd`** sidecar naming (78 refs) — this renames actual remote GitHub repos.
- **9 `SASE_SDD_*` env vars** and the `sdd:` config block + `commits_toggle_sdd` keymap action.
- **12 persisted record/JSON fields** (`sdd_plan_path`, `sdd_base_sha`, `sdd_sidecar_suffix`, …).
- **8 `Sdd*Wire` types** crossing the Python↔Rust boundary — requires lockstep changes in `sase-core`.
- **Published doc URLs**: `docs/sdd.md`, `docs/sdd_storage.md`, `blog/posts/beads-and-sdd.md`.

Three findings make this cheaper than the raw count suggests:

1. The on-disk plan/artifact-link **syntax carries no `sdd` token** — `artifact_link.rs` only has it in error message text. No file-format migration for plans.
2. **Sidecar repos already store bare `plans/`, `beads/`, `research/`** with no `sdd/` prefix — your primary storage mode is already SDD-free on disk.
3. `sdd_kind_roots()` in `src/sase/sdd/_paths.py:82` **already falls back to bare `<kind>/`** alongside `sdd/<kind>/` — the in-tree compatibility hook you'd need for the directory rename exists today.

There is no `sase sdd` CLI subcommand, and SDD is already gone from the `AGENTS.md`/`CLAUDE.md` glossary — only 3 memory files still mention it.

---

# New Query

What exactly does this concept represent? What is SDD?

**Assistant:**

**SDD = "Spec-Driven Development."** It's documented in `docs/sdd.md:1`.

## What the docs claim it is

A methodology: agent plans are ephemeral, living and dying inside one session's context window. SDD "fixes this" by writing the intent behind agent work to disk as first-class, queryable artifacts — creating a traceable chain from prompt → plan → bead → commit.

## What it actually is in the code

Nearly all 8,412 occurrences don't mean a methodology. They mean **a durable artifact store** — a resolved on-disk location plus the machinery to read, write, commit, and cross-link four artifact kinds:

| Kind | What it holds |
|---|---|
| `prompts/` | Snapshot of the fully expanded prompt the agent received (xprompts resolved, `%directives` stripped, Q&A appended) |
| `plans/` | Approved planning artifacts — `tier: tale` (ordinary) or `tier: epic` (multi-phase, executable via `sase bead work`) |
| `research/` | Exploratory findings, prior art, critiques — longer-lived context the plans depend on |
| `beads/` | The issue tracker that links artifacts to execution |

`SDD_CANONICAL_DIRS` at `src/sase/sdd/_paths.py:8` is literally just `("beads", "plans", "research")`.

The subsystem has three real responsibilities:

1. **Store resolution** — picking where artifacts live. Four provider-owned policies: `in_tree` (checkout's `sdd/`), `local` (`.sase/sdd/`), `separate_repo` (sidecar clone), `sidecar_repos` (auto-cloned `--plans` / `--research` repos). Not user-configurable; the workspace provider owns it.
2. **Artifact persistence** — `YYYYMM/` monthly layout, frontmatter schemas, tier validation, archiving on approval.
3. **Provenance linking** — the bidirectional `PROMPT`/`PLAN`/`PARENT`/`AGENTS`/`COMMITS` header block, re-derived from `SASE_PLAN=`/`SASE_AGENT=` commit footers.

## Why the term grates

The name is a **methodology label doing a storage-subsystem's job.** `SASE_SDD_PLANS_DIR`, `resolve_sdd_dir()`, `SddStoreRecord`, `sdd/beads/`, `--sdd` sidecar — none of those are about a development philosophy. They're all saying "the place where durable agent artifacts live."

There's also an internal inconsistency: "Spec-Driven" is stale vocabulary. The system moved to tales and epics — `_SDD_PROMPT_KINDS = {"prompts", "specs"}` at `_paths.py:7` keeps `specs` only as a legacy read alias, and `write_sdd_spec` / `update_spec_with_qa` still carry the old noun for what is now a prompt snapshot.

So the rename target isn't really one concept — it's an **artifact store** (the ~90% mechanical mass: paths, env vars, `Sdd*` types, sidecar suffix) that happens to be named after a **methodology** (the ~45 doc files, which is the only place the phrase "Spec-Driven Development" is doing honest work). Those two could split cleanly: rename the subsystem to something like `store`/`artifacts`/`plans`, and either drop the methodology framing entirely from docs or keep it as prose without it being a code identifier.

---

%xprompts_enabled:true
# New Query

 Why do we show `sdd` in #sshot for the bead commits / deltas? Is that correct? If not, use your /sase_plan skill to plan the appropriate changes.
 