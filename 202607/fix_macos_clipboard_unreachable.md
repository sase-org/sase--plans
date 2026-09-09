---
tier: tale
goal:
  Eliminate the macOS unreachable-code warning in clipboard capture while preserving
  clipboard source selection, command arguments, and failure behavior on every supported
  target.
create_time: 2026-09-09 19:53:12
status: wip
---

# Plan: Fix macOS clipboard control-flow warning

## Context and diagnosis

`clipboard_command_output` first honors `BOB_CLIPBOARD_CMD`, then conditionally compiles
an unconditional `return` for macOS. The Linux-specific block is removed on macOS, but
the shared tmux and terminal-error statements remain in the compiled function after the
macOS `return`. Rust therefore correctly reports those statements as unreachable during
a macOS build. This is a compile-time control-flow layout problem; suppressing
`unreachable_code` would hide it without making the target-specific intent explicit.

The correction must retain the behavior introduced with clipboard capture:

- A non-empty `BOB_CLIPBOARD_CMD` overrides all platform tools.
- macOS invokes `pbpaste` with its existing arguments and error labeling, and returns
  that result directly.
- Linux chooses `wl-paste` under Wayland or `xclip`/`xsel` under X11 exactly as it does
  now.
- tmux remains available only on target/environment paths that can currently reach it;
  this change must not introduce a new fallback after a selected platform command fails.
- Unsupported-target and no-source diagnostics continue to list the tools relevant to
  the compiled target.

## Implementation

Reshape the platform-selection portion of `src/native/capture_clip.rs` into mutually
exclusive compile-time paths. Keep the environment override in shared code, but place
the macOS terminal result and the non-macOS Linux/tmux/error flow behind complementary
`#[cfg]` boundaries (or equivalently small target-specific helpers) so statements that
cannot execute on macOS are not compiled into that function body. Prefer an explicit
target-shaped return value over a lint allowance, and avoid changing command execution
helpers or clipboard normalization.

Keep the patch narrowly scoped. If a small helper extraction is used, retain the
existing command names, arguments, labels, and diagnostic text verbatim. Do not revise
the documented precedence unless investigation reveals an actual behavior mismatch
outside this warning; that would be a separate product decision.

## Verification

1. Reproduce the relevant compiler condition with a macOS target and make the warning
   fatal, for example
   `RUSTFLAGS="-D unreachable-code" cargo check --locked --target <installed-apple-darwin-target>`.
   If running on macOS, also rerun the reported
   `cargo install --path . --locked --force` command and confirm the warning is absent.
2. Compile and test on the host Linux target to exercise the non-macOS branch. Run the
   focused clipboard/capture tests, then the complete `cargo test` suite.
3. Run `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and
   `git diff --check`. Distinguish any pre-existing Clippy warnings from new
   diagnostics, and introduce no new warnings.
4. Review the final diff against each source-selection branch to confirm the patch
   changes compile-time reachability only: override, macOS, Wayland, X11 with
   `xclip`/`xsel`, tmux, and no-source errors must retain their prior outcomes.

## Acceptance criteria

- macOS compilation emits no unreachable-statement warning from clipboard capture,
  including when `unreachable_code` is denied.
- Existing clipboard capture tests and the full test suite pass.
- Linux and other non-macOS builds still compile through their existing selection paths.
- No clipboard command precedence, arguments, error handling, output normalization, or
  user-facing diagnostics change as a side effect.
