- **PLAN:** [../202607/bead_store_merge_convergence.md](../bead_store_merge_convergence.md)

 Something very strange is going on with `sase beads` (see the output
below and recent git commits related to beads for context). Can you help me
diagnose the root cause of this issue and fix it?

- I like the idea of us syncing the primary workspace bead git repo for enabled
  sase projects (~/projects/github/sase-org/sase/sase/repos/plans on this
  machine) periodically, so the user can always (subject to a delay and assuming
  that `sase axe` is running on that machine) see what beads are open without
  needing to sync their GitHub repo. I think we are maybe already doing this but
  something is going wrong.
- We should also continue to support the new claimed bead status, which every
  bead with a waiting/queued agent associated with it should have.
- When an agent stops waiting and starts working a bead, the status should go
  from claim to in progress for that bead only.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 

```
❯ sase bead
No subcommand provided for 'sase bead'; delegating to 'sase bead list'.
◐ sase-93 · Restore green GitHub Actions CI for sase
◐ sase-95 · `sase task`: durable, session-aware background tasks
◐ sase-9k · Make %wait(priority=N) effective, observable, and editable
○ sase-9p · Align sase's sase-core-rs window with the released placeholder-source core
○ sase-9q · Raw `<placeholder>` tags become prompt input arguments
◐ sase-9q.5 · Wire collection into the prompt-bar launch path ← sase-9q
◐ sase-9q.7 · Documentation, help popup, and end-to-end check ← sase-9q
○ sase-9r · Serialize bead-store writes with SDD sidecar integration
◐ sase-9r.8 · Concurrent-writer soak exercise ← sase-9r
○ sase-9s · Detached background tasks and a single epic-launch path
○ sase-9s.6 · Collapse every epic approval onto the detached launch ← sase-9s
○ sase-9s.8 · End-to-end verification with and without a TUI ← sase-9s
○ sase-9t · Require descriptions for every AXE lumberjack and chop
◎ sase-9t.2 · Plumb optional descriptions through sase and describe the builtin lumberjacks ← sase-9t
○ sase-9t.3 · Describe every lumberjack and chop in chezmoi and bugyi-chops ← sase-9t
○ sase-9t.4 · AXE tab description banner ← sase-9t
○ sase-9t.5 · Surface descriptions in the AXE CLI listings ← sase-9t
○ sase-9t.6 · Enforce required descriptions and document the contract ← sase-9t

bryan in 🌐 athena in sase on  master is 📦 v0.11.1 via  v22.14.0 via 🐍 v3.11.13
❯ sase bead
No subcommand provided for 'sase bead'; delegating to 'sase bead list'.
◐ sase-93 · Restore green GitHub Actions CI for sase
◐ sase-95 · `sase task`: durable, session-aware background tasks
◐ sase-9k · Make %wait(priority=N) effective, observable, and editable
○ sase-9p · Align sase's sase-core-rs window with the released placeholder-source core
○ sase-9q · Raw `<placeholder>` tags become prompt input arguments
◎ sase-9q.7 · Documentation, help popup, and end-to-end check ← sase-9q
○ sase-9r · Serialize bead-store writes with SDD sidecar integration
◐ sase-9r.8 · Concurrent-writer soak exercise ← sase-9r
○ sase-9s · Detached background tasks and a single epic-launch path
◐ sase-9s.5 · Launch approved epics as one detached sase bead work task ← sase-9s
○ sase-9s.6 · Collapse every epic approval onto the detached launch ← sase-9s
○ sase-9s.8 · End-to-end verification with and without a TUI ← sase-9s
◎ sase-9t · Require descriptions for every AXE lumberjack and chop
◎ sase-9t.1 · Rust core accepts lumberjack descriptions and can require them ← sase-9t
◎ sase-9t.2 · Plumb optional descriptions through sase and describe the builtin lumberjacks ← sase-9t
◎ sase-9t.3 · Describe every lumberjack and chop in chezmoi and bugyi-chops ← sase-9t
◎ sase-9t.4 · AXE tab description banner ← sase-9t
◎ sase-9t.5 · Surface descriptions in the AXE CLI listings ← sase-9t
◎ sase-9t.6 · Enforce required descriptions and document the contract ← sase-9t

bryan in 🌐 athena in sase on  master is 📦 v0.11.1 via  v22.14.0 via 🐍 v3.11.13
❯ sase bead
No subcommand provided for 'sase bead'; delegating to 'sase bead list'.
Traceback (most recent call last):
  File "/home/bryan/.local/bin/sase", line 10, in <module>
    sys.exit(main())
             ~~~~^^
  File "/home/bryan/projects/github/sase-org/sase/src/sase/main/entry.py", line 122, in main
    handler(args)
    ~~~~~~~^^^^^^
  File "/home/bryan/projects/github/sase-org/sase/src/sase/bead/cli_query.py", line 39, in handle_bead_list
    issues = view.list_issues(
        statuses=statuses, issue_types=issue_types, tiers=tiers
    )
  File "/home/bryan/projects/github/sase-org/sase/src/sase/bead/project.py", line 165, in list_issues
    return rust_beads.list_issues(
           ~~~~~~~~~~~~~~~~~~~~~~^
        self.beads_dir,
        ^^^^^^^^^^^^^^^
    ...<2 lines>...
        tiers=tiers,
        ^^^^^^^^^^^^
    )
    ^
  File "/home/bryan/projects/github/sase-org/sase/src/sase/core/bead_read_facade.py", line 37, in list_issues
    payload: list[dict[str, Any]] = binding(
                                    ~~~~~~~^
        str(beads_dir),
        ^^^^^^^^^^^^^^^
    ...<2 lines>...
        tier_values(tiers),
        ^^^^^^^^^^^^^^^^^^^
    )
    ^
ValueError: validation: invalid bead event stream /home/bryan/projects/github/sase-org/sase/sase/repos/plans/beads/events/streams/sase-9t.jsonl line 16: expected value at line 1 column 1
```