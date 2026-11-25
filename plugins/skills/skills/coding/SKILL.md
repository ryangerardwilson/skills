# Coding

Guardrails for seven structural areas: performance-critical paths (PF), data model (DM), state ownership (SO), boundaries and dependency direction (BD), UI components and UX (UI), tests (TS), specs and documentation (DC).

The rules serve the system's requirements, including speed. No rule justifies a regression against a stated requirement or performance budget.

Start at step 1. Most changes exit there or use one section of the rules. Do not read or apply sections you are not routed to.

## 1. Triage (always first)

Pick the tier that fits.

| Tier | Fits when | Do |
|---|---|---|
| Skip | No change to behaviour or to a performance-critical path, and no new or changed entity, field, stored or cached state, module, cross-module call, component, screen, flow, test, spec, or doc. Examples: styling that uses existing tokens and components, a comment, an internal rename. | Stop. Apply nothing from this skill and do not mention it. |
| Inline | The change touches one or two areas in a handful of files. | Route (step 2) and apply the routed sections yourself. No subagents. |
| Orchestrate | A feature spanning several modules, a schema redesign, or a review of a large diff. | Route (step 2), then delegate (step 4). |
| Audit | Checking or fixing an existing codebase, test suite, or docs tree against these rules. | Follow step 5. |

Unsure between Skip and Inline: choose Inline. Unsure between Inline and Orchestrate: choose Inline.

## 2. Route

| The change... | Section |
|---|---|
| touches a path with a stated performance budget, or adds, removes, or replaces a cache, in-memory store, precomputed data, or background write | PF |
| adds or alters an entity, field, relationship, constraint, or migration | DM |
| adds or alters stored, cached, shared, or client-side state, or adds a write to existing state | SO |
| adds a module, a cross-module import or call, or an external integration | BD |
| adds or alters a component, screen, form, or user flow | UI |
| adds or alters tests, fixes a bug, or changes behaviour that existing tests cover | TS |
| adds, removes, or alters observable behaviour, or touches `openspec/`, `README.md`, `docs/`, or any Markdown file | DC |

Apply only the routed sections.

- **Writing code:** apply the MUST rules of each routed section. Consult its SHOULD rules only when you are designing something new in that area.
- **Reviewing:** apply both MUST and SHOULD rules of each routed section.

**Order.** When several sections are routed, apply them in the order a change is made: the spec rules first (DC-1 to DC-5), then PF, DM, SO, BD, UI, TS, and the docs rules last (DC-6 to DC-13). Under PF, read the budget and take the baseline measurement before changing any code, and take the second measurement once the change is complete.

**Precedence.** A stated requirement or performance budget outranks a structural rule. A MUST outranks a SHOULD. When two MUST rules conflict, stop and ask.

## 3. Working rules

- **Scope your reading.** Read only the entities, write paths, modules, components, tests, specs, and docs this change touches. Do not survey the codebase. Do not design from assumptions about the parts you do touch.
- **Read the routers first.** For what the system must do, open the relevant spec under `openspec/specs/`. For how it is built or run, follow `docs/index.md` to the one relevant doc. Then search the code.
- **Check the docs on every change.** For any Inline or Orchestrate change, apply DC-11 even when DC is not routed, and DC-13 when PF, DM, SO, or BD is routed. End your summary with `Docs: updated <files>` or `Docs: no impact`.
- **Tooling first.** If the project has linters, type checks, import-boundary rules, accessibility or token linters, schema constraints, link checkers, benchmarks, `openspec validate`, or architecture tests, run them and trust the result. Reason only about rules they do not cover. If a rule that tooling could check keeps being violated, suggest adding the check, once.
- **MUST rules block.** Do not write or approve code that violates one. If the request requires a violation, stop, state the cost in one or two sentences, and propose the compliant alternative. Proceed only if the user explicitly accepts, and record it (see Exceptions).
- **SHOULD rules are defaults.** Follow them unless there is a concrete reason not to; state the reason in one line.
- **Stay on the change.** Match existing conventions where they satisfy the rules. Do not refactor unrelated code to comply.
- **Clean up after yourself.** Before finishing, delete any scratch scripts, debug output, one-off repro files, and working notes you created.
- **Keep output short.** Cite rule IDs instead of restating rules. Do not narrate the triage or list rules that passed.

## 4. Orchestrate

Use this only if you can spawn subagents. Otherwise work through the routed sections yourself, one at a time.

- **You keep the judgment:** triage, routing, design decisions, verification, and the final report.
- **One subagent per routed section, started in parallel.** If you can choose the model, use the smallest reliable one for these.
- **Brief each subagent with only:**
  1. that section's rules, copied verbatim from below;
  2. the diff or the exact file paths to check;
  3. the task: "list violations of these rules in these files; read-only; do not fix";
  4. the output format from Reporting, a cap of 10 findings, no preamble, and the single word "none" if there are none.
- **Delegate lookups too.** Questions such as "where is X written", "is there an existing component for Y", "is there already a test for Z", "which docs mention W", or "what imports V" go to a small-model search subagent that returns the conclusion with file and line, not file contents.
- **Verify before reporting.** Open the cited location for every MUST finding and drop any you cannot confirm. Spot-check SHOULD findings.
- **Do not over-delegate.** If only one section is routed and the diff is small, do it inline.

## 5. Audit

Find in parallel, fix in sequence. Finding follows step 4 across every section. Fixing follows the rules below, so that complying with one section does not break another.

- **Fix in this order.**
  1. Gaps: report missing specs, budgets, and declarations (DC-1, PF-1, PF-2), and agree them with the user before fixing anything.
  2. PF: take the baseline measurements.
  3. TS safety net: bring the tests around the code to be changed up to TS-1, TS-2, TS-3, and TS-11, and get them passing.
  4. DM, then SO, then BD.
  5. UI.
  6. TS, the remaining rules.
  7. DC-6 to DC-13, once, at the end.
- **Earlier sections are frozen.** A fix under one section satisfies every MUST of the sections already completed. If it cannot, stop and ask.
- **Fixes preserve behaviour.** A fix that would change observable behaviour is not an audit fix. Raise an OpenSpec change for it (DC-2).
- **Remove before repair.** Within a section, first apply the rules that delete or merge (DM-1, DM-2, DM-12, SO-8, BD-9, BD-10, UI-2, TS-7, TS-10, TS-14, TS-20, DC-10, DC-11), then the rules that correct what remains. PF-5 still applies.
- **Gate each section.** After each section, run the tests, the benchmarks, and the tooling. Do not start the next section until they pass. One section per commit.
- **Re-check stale findings.** Findings gathered in parallel go stale once earlier fixes land. Re-check a section's findings against the current code before fixing them.
- **Docs are claims until verified.** During an audit, trust the code over the README and docs, skip the per-change docs check, and apply the docs rules once at the end.

## Rules

### PF - Performance-critical paths

Speed is a requirement, not an optimization. A design that exists to meet a budget is load-bearing, and the source of truth need not be the durable store.

MUST
- **PF-1** Every performance-critical path has a stated budget (latency or throughput), recorded as a requirement in its spec.
- **PF-2** Every design that exists to meet a budget is declared where it lives: what it is, the budget it serves, which store is the source of truth, and what is lost if the process dies before a background write completes.
- **PF-3** No change to a declared path without measuring before and after. A change that misses the budget is rejected, however much it improves DM, SO, or BD.
- **PF-4** Compliance on a declared path is achieved inside the design, not by replacing it: name the owner, keep one write path, and define how each copy is derived.
- **PF-5** A structure that looks like it exists for speed but has no stated reason is never removed on inference. Ask first.
- **PF-6** Each budget is guarded by a repeatable benchmark or performance test, kept apart from the deterministic tests.

SHOULD
- **PF-7** The fast path sits behind a narrow interface, so its internals can break the default patterns without the rest of the code depending on them.
- **PF-8** No optimization without a measurement either: the same evidence standard applies to adding speed as to removing it.

### DM - Data model

MUST
- **DM-1** Every entity is a real domain concept definable in one sentence. None that exist only to serve one screen or endpoint.
- **DM-2** Every fact is stored once. Derived values are computed, not stored, unless there is a measured performance need; then the copy is marked derived with one documented refresh path.
- **DM-3** Every entity has a stable, immutable identifier with no business meaning. Never email, name, or user-editable values as a primary key.
- **DM-4** Relationships are explicit and enforced: foreign keys, correct cardinality, defined on-delete behaviour. None held only in application code or as ID lists in text or JSON.
- **DM-5** Invariants are enforced at the lowest layer that can: schema constraints (NOT NULL, unique, check, enum or lookup table) first, the domain layer for the rest, never the UI alone.
- **DM-6** Illegal states are unrepresentable where practical: one status field with defined transitions, not several booleans that can contradict.
- **DM-7** Schema changes ship as versioned migrations that are safe on existing data, with a backfill and rollback plan for any change to existing columns.

SHOULD
- **DM-8** Columns are non-nullable by default; each nullable column has a stated meaning for null.
- **DM-9** Precise types: UTC timestamps with time zone, exact decimals or integer minor units for money. No floats for money, no strings for dates.
- **DM-10** No catch-all structures (untyped JSON, entity-attribute-value tables, `metadata` columns) for data that is queried, validated, or used by logic.
- **DM-11** Names use the domain's language, one name per concept across schema, code, and API.
- **DM-12** No fields or tables for speculative future needs.
- **DM-13** Index the access paths that exist, including every foreign key used in joins or lookups.

### SO - State ownership

Applies to database rows, in-memory state, client or UI state, caches, and data held in external services.

MUST
- **SO-1** Every piece of state has exactly one owning module or service. Only the owner writes it.
- **SO-2** Every fact has exactly one write path, where validation and invariants are applied. No writes that bypass it.
- **SO-3** Other modules read through the owner's interface, never its tables or internals.
- **SO-4** Every copy (cache, denormalized column, client copy, search index, replica) is derived: it names its source, is never written independently, and has a defined invalidation or refresh mechanism.
- **SO-5** Client or UI state does not hold server state as a second source of truth. Server data on the client is a cache; only local-only state (drafts, selection, open panels) lives locally.
- **SO-6** Changes that must be consistent happen in one transaction or through an explicit eventual-consistency mechanism (outbox, idempotent events, reconciliation). Never two independent writes.
- **SO-7** Writes that may be retried are idempotent.

SHOULD
- **SO-8** Derive rather than synchronize: if a value can be computed from owned state, compute it.
- **SO-9** Keep state as local as possible: local variable, then component, then shared store, then global.
- **SO-10** No mutable globals or module-level singletons; pass state and dependencies explicitly.
- **SO-11** If two modules both need to write a fact, the ownership is wrong: move the fact, or send one module's changes to the owner as requests.

### BD - Boundaries and dependency direction

MUST
- **BD-1** Dependencies point inward: UI, handlers, persistence, and integrations depend on the core domain. The core imports no UI code, ORM or database client, web framework types, or vendor SDKs.
- **BD-2** The core reaches the outside world only through interfaces it defines, implemented by outer layers and passed in.
- **BD-3** The module dependency graph is acyclic. On a cycle, extract the shared concept or invert one dependency.
- **BD-4** Modules interact only through public interfaces. No imports of another module's internal files, no reads of its tables.
- **BD-5** Framework, transport, and storage types (request objects, ORM rows, SDK responses) are translated at the boundary and never pass into or out of the core.
- **BD-6** Business rules live in the core domain, not in controllers, UI components, SQL triggers, or integration glue.

SHOULD
- **BD-7** Interfaces are narrow, in domain terms, and hide how the work is done.
- **BD-8** Side effects (I/O, network, clock, randomness) sit at the edges; core logic is deterministic and testable without infrastructure mocks.
- **BD-9** Each module has one responsibility that fits in a sentence; otherwise split it.
- **BD-10** No pass-through layers, and no single-implementation abstractions unless they protect a boundary that changes independently.
- **BD-11** Organize by domain capability rather than technical type where the codebase allows.
- **BD-12** Encode dependency rules in tooling where it exists, so violations fail automatically.

### UI - Components and UX

Independent of language, framework, and platform. Use the project's existing component library and conventions.

MUST
- **UI-1** Visual values (colour, spacing, type scale, radius, elevation, motion duration, breakpoints) come from one source of design tokens. No raw values in components; add a token rather than a one-off.
- **UI-2** One component per UI concept. Search for an existing component and extend or compose it before creating one.
- **UI-3** Presentational components contain no business rules or data access. They receive data and emit events.
- **UI-4** Every data-dependent view handles loading, empty, error, partial, and populated. Every interactive element defines default, hover or pressed, focus, disabled, and busy where applicable.
- **UI-5** Every user action gives visible feedback. Slow actions show progress, and the triggering control is protected against duplicate submission.
- **UI-6** Destructive or irreversible actions require confirmation or offer undo, and look distinct from safe actions.
- **UI-7** A failure never loses user input. Error messages say what happened and what to do next, in plain language, next to what failed. No raw error codes or stack traces as the only message.
- **UI-8** Everything operable by pointer is operable by keyboard or the platform's assistive input, in a logical order, with a visible focus indicator.
- **UI-9** Controls use semantic elements or expose equivalent role, name, and state. Every input has an associated label; every meaningful image or icon has a text alternative.
- **UI-10** Text and essential graphics meet contrast minimums (WCAG 2.2 AA or the platform equivalent). Nothing is conveyed by colour alone.
- **UI-11** Overlays manage focus: move it in on open, contain it while open, restore it on close, and close on the platform's standard dismiss action.
- **UI-12** Motion respects the reduced-motion preference. Nothing flashes or auto-plays without a way to stop it.
- **UI-13** Layout adapts to the supported viewport sizes and to enlarged text without clipping, overlap, or loss of function.

SHOULD
- **UI-14** Component interfaces are small: composition over configuration, a named variant over several booleans.
- **UI-15** A component's value and state have one source; no private copy of a value passed in.
- **UI-16** The same action looks, is named, and behaves the same everywhere. One primary action per view.
- **UI-17** Platform conventions come before new patterns.
- **UI-18** Prevent errors before reporting them: constrain inputs, give sensible defaults, disable or explain unavailable actions. Validate inline at the field once the user has finished typing, not per keystroke and not only on submit.
- **UI-19** Acknowledge input immediately; show progress or a skeleton for waits beyond about a second. Optimistic updates only with a defined rollback and error message.
- **UI-20** Touch targets are about 44 by 44 points with adequate spacing.
- **UI-21** Show the common path first and advanced options on request.
- **UI-22** View state (current view, selection, filters, pagination) is restorable and shareable through routing or deep links, and survives back navigation and refresh.
- **UI-23** User-facing text uses the data model's names, is kept separate from logic for translation, and is never built by concatenating fragments. Layouts tolerate longer text, and right-to-left scripts where localisation is in scope.
- **UI-24** Motion is brief and explains a change of state or location.
- **UI-25** Each shared component has examples of all its variants and states.

### TS - Tests

Independent of language and test framework. A test should be readable on its own and fail only when behaviour is wrong.

MUST
- **TS-1** Tests assert observable behaviour through the public interface. Never private functions, internal state, or internal call sequences.
- **TS-2** Each test is independent: it creates the state it needs and passes alone, in any order, and in parallel. No reliance on state left by another test.
- **TS-3** Tests are deterministic: no real clock, real network, sleeps, or unseeded randomness.
- **TS-4** Each test checks one behaviour, and its name states the scenario and the expected outcome.
- **TS-5** The values that matter to the assertion are visible in the test body. Shared builders supply only defaults that are irrelevant to the test.
- **TS-6** Mock or fake only the system's edges (network, clock, third-party services, and persistence where needed). Never the code under test or the core domain.
- **TS-7** No skipped, disabled, commented-out, or expected-to-fail tests without a stated reason and an owner. Delete tests when the code they cover is deleted.
- **TS-8** Production code contains nothing that exists only for tests: no test flags, back doors, or methods made public for a test.
- **TS-9** A bug fix comes with a test that fails without the fix.
- **TS-10** The tests directory contains only tests and their support code. No scratch scripts, debug output, or one-off repros.
- **TS-11** Every test has an assertion that can fail for the behaviour it names. Never weaken or delete an assertion, or regenerate a snapshot, just to make a test pass.

SHOULD
- **TS-12** The tests directory mirrors the source layout, so the test for a module is findable by path.
- **TS-13** Test each behaviour at the lowest level that can prove it. Keep end-to-end tests for a few critical flows, and do not repeat the same check at several levels.
- **TS-14** Before adding a test, look for an existing one to extend or parameterise.
- **TS-15** Test data comes from one set of builders or factories, not hand-rolled objects or large fixture files.
- **TS-16** Shared setup is explicit and close to the test. Avoid fixtures applied automatically or defined far from where they are used, and avoid deep helper layers or test base classes.
- **TS-17** Snapshots are small and reviewed. Prefer explicit assertions on the fields that matter.
- **TS-18** No logic in tests (loops, conditionals, computed expectations). Expected values are written literally.
- **TS-19** Recorded data files (fixtures, captured responses, dumps) state their source and which tests use them; remove them when unused.
- **TS-20** Do not write tests for trivial code or the framework itself to raise coverage.

### DC - Specs and documentation

Specs follow OpenSpec and say what the system must do. The README and docs say how the system is built, run, and used now.

MUST
- **DC-1** Every requirement is stored once, in `openspec/specs/`. The README and docs link to a spec and never restate it.
- **DC-2** Every change to observable behaviour starts as an OpenSpec change in `openspec/changes/`, agreed before code is written. A fix that restores specified behaviour needs none.
- **DC-3** Main specs change only through OpenSpec's archive or sync step. No hand edits, and nothing under `openspec/changes/archive/` is edited or deleted.
- **DC-4** A completed change is archived in the same piece of work, and `openspec validate` passes after any edit under `openspec/`.
- **DC-5** Where code and spec disagree, the spec is right until a change says otherwise. Fix the code or raise a change; never edit the spec to match the code.
- **DC-6** The repository root has exactly one Markdown file, `README.md`, plus an optional `LICENSE.md`. Every other doc lives under `docs/`.
- **DC-7** `README.md` holds exactly four things, in order: an opening sentence on what the system does and who it is for, Quick start, Usage, and a link to `docs/index.md`. Plain language, with commands that work on a clean checkout.
- **DC-8** `docs/index.md` is a router with no detail of its own: one line per doc saying what question it answers, covering at least architecture, development, and deployment, plus a link to `openspec/specs/`.
- **DC-9** Every doc is reachable from `docs/index.md`, and every topic has one home. Adding, moving, or deleting a doc updates the index in the same change.
- **DC-10** The README and docs describe only what exists in the code now. Nothing planned, in progress, or removed; no requirements, plans, roadmaps, changelogs, task logs, or session notes.
- **DC-11** Every code change updates the README and docs it makes wrong, in the same change. References to removed or renamed code are removed or renamed, and a doc left describing nothing is deleted.
- **DC-12** No secrets, real credentials, or personal machine setup in any doc.
- **DC-13** `docs/architecture.md` opens with four sections, in order: performance-critical paths with their budgets, data model, state ownership, boundaries and dependency direction. Every statement in it is checked against the code before it is written and is true of the code now. Nothing misstated, overstated, or aspirational; where the code falls short of PF, DM, SO, or BD, the doc says so plainly.

SHOULD
- **DC-14** One capability per spec folder, named as in the data model.
- **DC-15** A change is small enough to implement and archive in one piece of work; otherwise split it.
- **DC-16** The reasoning behind a decision goes in the change's design file, not in `docs/`.
- **DC-17** The README fits in one or two screens; move growth to a doc and link to it.
- **DC-18** Extend an existing doc before creating one. One doc per topic, named in lowercase with hyphens.
- **DC-19** Each doc opens with one sentence on what it covers and who needs it, and gives exact paths, commands, and names from the code.
- **DC-20** Point to code or configuration as the source for values that change, rather than restating them.

## Reporting

Report findings only, one line each:

`ID | where (file:line, table, component, test, spec, or doc) | what goes wrong | smallest fix`

Example: `SO-2 | orders/refund.py:41 | second write path to order.status, so a refund can overwrite a shipment update | call OrderService.set_status`

- MUST findings first, then SHOULD.
- List pre-existing violations separately from those the change introduces, at most five unless a full review or audit was requested.
- If there are none, say "No guardrail findings."

## Exceptions

Rules may be broken deliberately, never silently. When the user accepts a violation:

- add a short comment at the site with the rule ID, the reason, and what would allow its removal;
- keep the violation behind an interface so other code does not depend on it;
- mention it in the summary of the change.

Typical legitimate exceptions: a denormalized column or cache justified by measured performance, a temporary dual write during a migration, a one-off visual treatment for a marketing surface, a characterization test pinned to current behaviour before a refactor, a root Markdown file that a tool requires at that exact path, a prototype explicitly marked as throwaway.
