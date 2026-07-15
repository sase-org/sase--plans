---
plan: 202607/workspace_local_core_fallback.md
---
 The `just install` command is failing for some reason (see the output below). Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

```
❌130 ❯ just install && just all
[install] Building sase_core_rs from ../sase-core for local dev.
[validate_sase_core_rs_version] sase-core checkout is behind: source version 0.3.4 from ../sase-core/Cargo.toml does not satisfy `sase`'s `sase-core-rs>=0.4.0,<0.5.0` dependency in pyproject.toml. Pull/rebuild `sase-core`, or lower `sase`'s `sase-core-rs` constraint only if that older core is intentional.
error: recipe `rust-install` failed on line 427 with exit code 1
error: recipe `install` failed on line 93 with exit code 1
```