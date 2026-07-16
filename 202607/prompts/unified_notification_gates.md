---
plan: 202607/unified_notification_gates.md
---
 #fork:research.e.final Can you use your /sase_questions skill to ask me all of the design questions proposed by the previous agent and then help me implement this functionality by designing the required work using your /sase_plan skill? If a set of questions and answers are below this text, skip the sase question skill step and go straight to designing.

%xprompts_enabled:false
### Questions and Answers

#### Q1: CLI role

> Operationally, how should sase notify create participate in opening plan, question, and launch gates?

- [x] **Shared service plus CLI (Recommended)** — The CLI and in-process producers use one Python gate constructor; internal producers do not shell out.
- [ ] **CLI-only front door** — All producers invoke sase notify create as a subprocess.
- [ ] **Shared service only** — Existing typed commands share a constructor, while notify create remains a raw row writer.

#### Q2: Abstraction

> What should the common abstraction own?

- [x] **Gate plumbing plus adapters (Recommended)** — The common gate owns request and response plumbing; typed adapters own continuation and side effects.
- [ ] **Entire gate lifecycle** — The common layer also owns handoff, waiting, slot reacquisition, prompt reconstruction, and async behavior.
- [ ] **Notification row only** — Only row construction is unified; request bundles and continuations stay separate.

#### Q3: Protocol

> Should this initiative unify only constructors or also replace typed action protocols and change NotificationWire?

- [x] **Typed projections, no wire change (Recommended)** — Keep PlanApproval, UserQuestion, LaunchApproval, and HITL action types; add a versioned request envelope in Python first.
- [ ] **Typed plus Rust schema** — Keep typed actions but add row-level schema or envelope support in sase-core now.
- [ ] **One generic protocol** — Version and migrate Rust, mobile, and Telegram to one generic interaction action now.

#### Q4: HITL scope

> How should HITL relate to this work?

- [x] **Design now, migrate later (Recommended)** — Make the abstraction accommodate HITL, but leave its migration to a later phase.
- [ ] **Migrate now** — Include HITL as a fourth gate in this implementation.
- [ ] **Exclude entirely** — Do not shape the abstraction around HITL.

#### Q5: Sender scope

> Which actionable notification senders are in scope?

- [x] **Blocking gates only (Recommended)** — Cover plan, question, and launch; leave navigation, memory review, and error actions alone.
- [ ] **All actionable senders** — Generalize memory review, navigation, mentor review, and error report actions too.
- [ ] **Gates plus memory review** — Include memory review because it already participates in Python pending actions.

#### Q6: Role removal

> How much of the improve_plan and tester custom lifecycle-role feature should be removed?

- [x] **Whole subsystem L1+L2+L3 (Recommended)** — Remove examples, generic lifecycle-role machinery, and plan-gate member options.
- [ ] **Examples plus gate UI L1+L3** — Remove examples and plan approval selection, but retain config-driven lifecycle roles.
- [ ] **Examples only L1** — Delete only the inactive example definitions and prompt bodies.

#### Q7: Role defaults

> If plan-gate member selection is removed but lifecycle roles remain, should those roles continue running from config defaults?

- [x] **Not applicable; remove L2 (Recommended)** — The whole lifecycle-role subsystem is removed, so no default-driven custom roles remain.
- [ ] **Keep default execution** — Configured custom roles continue without gate-time selection.
- [ ] **Keep L2 but disable execution** — Retain parsing and metadata only, with no automatic execution.

#### Q8: Keep family

> Which independent agent-family capabilities must remain after lifecycle-role removal?

- [x] **Manual attachment (Recommended)** — Keep percent-n parent and suffix attachment.
- [x] **Arbitrary suffixes (Recommended)** — Keep generic names such as tester and reviewer without lifecycle semantics.
- [x] **Family launch approvals (Recommended)** — Keep agent-initiated family launches through LaunchApproval.

*Multi-select*

#### Q9: Old seams

> What should happen to role_completed and custom-role metadata or label seams after subsystem removal?

- [ ] **Stop writes, keep historical readers (Recommended)** — Remove runtime generation while preserving read-only compatibility for existing artifacts.
- [x] **Delete everything** — Remove events, metadata, labels, and historical readers immediately.
- [ ] **Retain the seam** — Keep dormant event and metadata support for a future controller.

#### Q10: Role cleanup

> What compatibility and documentation posture should lifecycle-role removal use?

- [x] **Hard error plus replacement docs (Recommended)** — Reject kind agent_family with a migration message, replace onboarding examples with supported family workflows, and regenerate snapshots and goldens.
- [ ] **One-release deprecation** — Warn for one release before removing support; regenerate affected artifacts.
- [ ] **Hard removal, no replacement** — Delete support and examples without a new onboarding path.

#### Q11: Create owns

> Should sase notify create own the response directory, typed request file, and stable JSON result containing IDs, paths, and continuation mode?

- [x] **Yes, own full bundle (Recommended)** — A successful gate create returns a stable machine-readable descriptor.
- [ ] **Directory only** — Create the owned directory but require callers to write request payloads.
- [ ] **Row only** — Keep bundle creation in each producer.

#### Q12: Gate layout

> What request-directory layout should new gates use?

- [x] **Neutral layout with legacy fallback (Recommended)** — Write new interaction_requests kind and id bundles; keep readers for old per-kind paths during migration.
- [ ] **Keep per-kind layouts** — Share the constructor but let adapters retain current plan, question, and launch directories.
- [ ] **Immediate neutral cutover** — Move all readers and writers at once with no legacy fallback.

#### Q13: Waiting

> How should response waiting be unified?

- [x] **Generic poller with adapter hooks (Recommended)** — Share polling and cancellation; keep plan auto-approval checks and typed response handling in adapters, and mechanically observe launch responses.
- [ ] **Plan and question only** — Share their polling while launch remains an unobserved async exception.
- [ ] **Keep separate pollers** — Do not unify continuation waiting.

#### Q14: Launch flow

> What behavior should the launch gate have after unification?

- [x] **Typed exception, close response loop (Recommended)** — Approval still dispatches host-side, but the requester mechanically observes the result and rejection; non-agent sase run stays exempt.
- [ ] **Keep current async behavior** — The approver dispatches and no code polls launch_response.json.
- [ ] **Make launch producer-driven** — Approval only authorizes; the requesting producer performs dispatch after receiving the response.

#### Q15: Choices

> How should gate choices be represented?

- [ ] **Typed per kind (Recommended)** — Keep closed Rust choice enums and coordinate core changes when a new gate kind is added.
- [ ] **Data-driven now** — Version the wire so each request declares arbitrary choices.
- [ ] **Hybrid** — Keep built-in typed choices but allow optional untyped extensions.
- [x] **Other:** "Every option that a user has for a custom notification like this (ex: approve, reject, etc...) should be represented by a command that will be run if the user selects that option. There is at least one exception to this rule, which is when one of the options is to edit a file. We'll need special support for this (ex: to edit plan files). The question notification action should be generalized as a way to inject user input into the command that is run. In the case of the question notification we should write a script that will be executed with the proper user inputs when all questions are answered."

#### Q16: Auto policy

> Where should automatic approval or answer policies live?

- [ ] **Kind-intrinsic adapters (Recommended)** — Preserve plan auto-approve, question first-option, and launch mandatory approval in typed adapters.
- [ ] **Common configurable field** — Represent auto behavior uniformly in the gate envelope.
- [ ] **Remove auto behavior** — Require explicit user responses for every gate.
- [x] **Other:** "This should be a common field that any notification type should be able to configure. Also we should improve this directive so it accepts a generic argument, which is defined by the notification type. Also since we now require a plan files tier to be specified in the frontmatter as a property, we can distinguish between normal plan files and epic plan files. Let's create a brand new notification type for epic approvals. When 'auto' is used for an agent that proposed an epic plan file, the epic should be auto-approved (as an epic)."

#### Q17: Atomicity

> What durability guarantee should gate creation provide?

- [x] **Success guarantee with compensation (Recommended)** — Return success only when bundle, notification, and pending action are durable; clean up or clearly mark partial failures.
- [ ] **Best effort plus cleanup** — Allow success with missing pending state and repair asynchronously.
- [ ] **Row is authoritative** — Only notification durability is guaranteed.

#### Q18: Trust

> What trust rules should privileged gate creation enforce?

- [x] **Owned paths, adapters, and hashes (Recommended)** — Confine bundles to SASE directories, validate kind and filenames through registered adapters, hash reviewed payloads or previews, and bar raw privileged rows.
- [ ] **Owned paths and validation** — Enforce directories and adapters but do not add content hashes.
- [ ] **Permissive raw creation** — Allow callers to mint privileged actions pointing at arbitrary files.

#### Q19: Staleness

> What should the 24-hour pending-action expiry mean?

- [x] **Transport-advisory only (Recommended)** — Expiry hides stale remote actions but does not terminate a waiting producer; gate timeout is separate and explicit.
- [ ] **Per-kind terminal expiry** — Each adapter configures a timeout that ends the producer wait.
- [ ] **Global terminal expiry** — All gates fail when the common 24-hour threshold is reached.

#### Q20: Compatibility

> How should mixed versions and in-flight legacy requests be handled?

- [x] **Legacy readers for one release plus grace (Recommended)** — New consumers answer old bundles through at least one release and beyond the pending-action stale window.
- [ ] **Long-term legacy readers** — Keep legacy request layouts answerable indefinitely.
- [ ] **Drain on upgrade** — Require pending actions to be cleared before deployment.

#### Q21: Bug scope

> Which latent notification bugs and debts should this initiative fix?

- [x] **Fix gate-critical items (Recommended)** — Fix TUI question handled-state, Rust launch dismissal, launch response observation, and directly touched notify, summary, and docs debt; defer unrelated mappings.
- [ ] **Fix every listed item** — Also unify Rust string mappings and resolve memory_review during this initiative.
- [ ] **Separate all bug fixes** — Keep the implementation narrowly structural and file follow-ups for every existing bug.

#### Q22: Public CLI

> What should remain agent-facing after the shared constructor exists?

- [x] **Keep typed front doors (Recommended)** — Plan propose, questions, and launch request remain public; notify create exposes the shared low-level gate API without replacing generated skill workflows.
- [ ] **Migrate agents to notify create** — Generated skills invoke the generic command; typed commands become temporary aliases.
- [ ] **Internal API only** — Do not expose gate creation through notify create to agents or users.

---

> **Global Note:** Answered via Telegram

%xprompts_enabled:true