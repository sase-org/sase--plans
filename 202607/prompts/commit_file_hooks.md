- **PLAN:** [../202607/commit_file_hooks.md](../commit_file_hooks.md)

 Can you help me add sase configuration support for hooks that run
whenever a sase agent adds / modifies / deletes files that match the user's
specifications?

- These hooks should enable custom, user-defined commands to run against certain
  artifacts/files, matching the user's specification.
- These hooks should be run by `sase commit`. As a consequence, we only consider
  file additions / changes / removals that are commited to VCS OR created via
  the `sase artifact create` command.
- The configured hook command should be run once for each file match. The file
  path should be provided to the hook command as a CLI argument.
- The user should be able to specify the following fields to filter when the
  hook command should run (i.e. the configured hook command should only run on
  files that match):
  - `projects`: A list of sase projects this hook applies to.
  - `sidecars`: A list of sidecar repo names (e.g. `research`, `plans`, `beads`,
    etc...) that this hook applies to.
  - `globs`: A list of file path globs. One of these must match every file that
    is passed to the hook commands. This list can contain negative globs, which
    are specified by prefixing the string/glob with "!". If ANY negative glob
    matches a file path, then that file should not be matched by the hook.
  - `ops`: An enum/string that supports the values `ADD`, `MODIFY`, and
    `REMOVE`. This controls what file operations this hook applies to.
- Our first use-case:
  - For our first use-case, I would like to use the following configuration to
    filter files:
    ```
    sidecars: [research]
    globs: ["20*/*/*.md", "!20*/*/*__*.md"]
    ops: [ADD]
    ```
  - We should add a new `bob highlights create <md_file>` command where
    `<md_file>` is a markdown file path of the form `some/path/to/<name>.md` to
    my bobs-org/bob-cli GitHub repo to support this new functionality. This is
    the command we should configure (in the sase.yml file in my chezmoi repo) as
    the hook for this use-case.
  - This command should convert the given markdown file into an excellent PDF
    file, which is easy to read and includes an indexed table-of-contents.
  - This PDF should be written to a new file in the ~/bob/lib/chat/ directory
    named `<name>.pdf`.
  - We should add the appropriate Highlights marker note to this PDF file so the
    `bob highlights scan` command triggers the creation of a new Obsidian ref
    note file.
  - Make sure this command has excellent output for both success and failure
    (see the next bullet for why this is important).
  - Verify this command's functionality by running it against the
    202607/sase_beads_close_integrity_and_capture/sase_beads_close_integrity_and_capture.md
    file that already exists in the research sidecar repo. After you verify that
    the PDF file was created in the right location and looks the way it should,
    run the `bob highlights scan` command to make sure the appropriate Obsidian
    reference note gets created.
- We should send a custom sase notification to the user once this hook command
  succeeds or fails. On failure, display all of the error results to the user in
  this notification. On success, this notification should contain the command
  output too but should also contain other useful information related to this
  hook command run (use your best judgement here).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 