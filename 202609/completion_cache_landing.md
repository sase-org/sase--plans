---
tier: tale
title: Finish transactional completion-cache landing
goal:
  Runtime completion refresh failures preserve the last valid generation, with
  end-to-end loader and cache acceptance coverage.
size: medium
proposed_by: bbugyi200.apollo.sase-12o.land
bead: sase-12o
create_time: 2026-09-18 15:20:16
status: wip
---

- **PARENT:**
  [202609/completion_freshness.md](https://github.com/sase-org/sase--plans/blob/main/202609/completion_freshness.md)
- **BEAD:**
  [sase-12o](https://github.com/sase-org/sase--beads/blob/main/pages/sase-12o/README.md)

# Plan: Finish transactional completion-cache landing

## Context

Epic `sase-12o` added runtime-cached Bash, fish, and zsh completion grammars and
portable installed loaders. Its focused regression suite passes and the applied chezmoi
installation has been migrated successfully, but landing review reproduced a
failure-preservation defect: a forced refresh publishes the new grammar before the new
manifest. If manifest publication then fails, the old manifest remains beside the new
grammar, so the last valid generation is lost and diagnostics report a corrupt cache.
This directly violates the epic's requirement that generation, compilation, or
publication failures preserve the previous valid generation.

The same review found that the new loader/cache tests mostly assert rendered strings and
happy-path cache reuse. The acceptance coverage does not directly exercise the
transactional failure above, simultaneous cold starts, paths containing spaces, runtime
partitioning and retention, or real-shell loading of the portable loader. These are
remaining acceptance obligations from the epic, not separate follow-up work.

## Implementation

1. Make one runtime grammar generation a recoverable transaction. Stage the grammar, zsh
   bytecode when applicable, and manifest without replacing the active generation;
   validate the staged artifacts, then publish them as a coherent generation. A failure
   while generating, compiling, staging, or publishing must leave the prior grammar,
   bytecode, and manifest usable. A first-generation failure must leave no cache that
   can be mistaken for current. Keep the bounded per-runtime lock and recheck behavior,
   and clean staging or backup artifacts deterministically.
2. Preserve the existing public cache path and manifest contract unless a small schema
   revision is necessary for safe recovery. Keep warm-cache resolution on the
   pre-argparse path and avoid importing the full parser on a hit. Do not change raw
   `sase completion bash|fish|zsh` export behavior, dynamic-candidate caching, stamped
   install ownership, or loader discovery paths.
3. Add focused tests that fault-inject each publication boundary and prove the previous
   generation remains current. Cover simultaneous cold callers building once, distinct
   runtime identities sharing one `SASE_HOME`, bounded old-runtime retention, and
   loader/cache paths containing spaces. Exercise the rendered Bash and zsh loaders in
   real shells so a first completion sources the ensured grammar and repeated static
   completion does not invoke ensure again. Exercise fish likewise when fish is
   available, with an explicit skip on hosts where it is not installed.
4. Add a warm-path import/latency regression that proves cache hits do not construct the
   parser or import parser/TUI modules, using a generous CPU-time budget rather than a
   shared-host wall-clock threshold. Keep existing shell syntax, alias/prefix, update
   refresh, diagnostics, and CLI snapshot tests passing.

## Verification

- Run focused completion-cache, loader, install/refresh, doctor, update-mode-switch,
  Bash smoke, zsh smoke, and available fish tests.
- Run the linked chezmoi Bash completion test suite to confirm its portable loaders
  remain byte-for-byte compatible with the generator.
- Run `just fix`, then the repository-required exhaustive `just check-full` through the
  SASE monitor workflow.
