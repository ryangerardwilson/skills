# Architecture guardrails

Enforce four structural disciplines on every design, schema, and code change. Mistakes in these areas are paid for in every future change, so they are held to a stricter standard than style, framework choice, or micro-optimization.

1. **Data model** - everything else inherits its shape.
2. **State ownership** - each fact has exactly one owner and one write path.
3. **Boundaries and dependency direction** - the core domain is independent of UI, database, and frameworks.
4. **UI components and UX** - one design vocabulary, every state designed, usable by everyone.

## How to apply this skill

**Before writing code** for any change that adds or alters an entity, a field, stored or cached state, a module, a cross-module call, or a UI component, screen, or user flow:

1. Read the existing schema, the module layout, the places where the affected state is written, and the existing component library. Do not design from assumptions.
2. State in a few lines: which entities change, who owns each new fact, which module the code belongs in, and which existing components will be reused.
3. Run the checks below. Resolve any hard-rule violation before implementing.

**When reviewing code or a design**, run the same checks and report findings in the format under "Reporting".

**Severity:**

- **Hard rules** (marked MUST): do not write or approve code that violates them. If the user's request requires a violation, stop, explain the cost in one or two sentences, and propose the compliant alternative. Proceed with the violation only if the user explicitly accepts it, and record it as an exception (see "Exceptions").
- **Defaults** (marked SHOULD): follow unless there is a concrete reason not to; state the reason when deviating.

Match the existing codebase's conventions where they already satisfy these rules. Do not refactor unrelated code to comply; flag pre-existing violations separately from the change at hand.

## 1. Data model

Goal: the smallest model that represents the domain truthfully. A wrong entity or relationship leaks into every query, API, and screen.

**Hard rules**

- Every entity MUST correspond to a real domain concept that can be named and defined in one sentence. No entities that exist only to serve one screen or one endpoint.
- Every fact MUST be stored in exactly one place. Derived values (totals, counts, status computed from other fields) are computed, not stored, unless there is a measured performance need, and then the stored copy is explicitly marked as derived with one documented refresh path.
- Every entity MUST have a stable, immutable identifier that carries no business meaning. Never use mutable or meaningful values (email, name, composite of user-editable fields) as primary keys.
- Relationships MUST be explicit and enforced: foreign keys, correct cardinality, and a defined behaviour on delete. No relationships held only in application code or in ID lists inside a text or JSON field.
- Invariants MUST be enforced at the lowest layer that can enforce them: NOT NULL, unique constraints, check constraints, and enums or lookup tables in the schema; anything the schema cannot express in the domain layer. Never rely on the UI alone.
- Illegal states MUST be unrepresentable where practical. Prefer one status field with defined transitions over several booleans that can contradict each other (for example `is_active`, `is_deleted`, `is_archived`).
- Schema changes MUST ship as versioned, reviewable migrations that are safe to run against existing data. State the backfill and rollback plan for any change to existing columns.

**Defaults**

- Columns SHOULD be non-nullable by default. Each nullable column needs a stated meaning for null.
- Use precise types: timestamps with time zone stored in UTC, exact decimals or integer minor units for money, never floats for money or strings for dates.
- Avoid generic catch-all structures (untyped JSON blobs, entity-attribute-value tables, `metadata` columns) for data that is queried, validated, or relied on by logic.
- Name things in the domain's language, consistently, one name per concept across schema, code, and API.
- Do not add fields or tables for speculative future needs. Add them when the need exists.
- Index for the access paths that actually exist; every foreign key used in joins or lookups gets an index.

**Red flags**

- The same value appears in two tables or two columns.
- A field whose meaning depends on another field's value.
- A table named after a screen, report, or endpoint.
- Several booleans describing one lifecycle.
- An API payload shaped differently from the model with no stated reason.

## 2. State ownership

Goal: for every fact, anyone can answer "who owns it and how does it change?" with one answer. This applies to database rows, in-memory state, client or UI state, caches, and data held in external services.

**Hard rules**

- Every piece of state MUST have exactly one owning module or service. Only the owner writes it.
- There MUST be exactly one write path per fact: a single function, command, or endpoint through which all changes pass, where validation and invariants are applied. No direct writes that bypass it (ad hoc SQL updates from other modules, direct mutation of another module's objects or store).
- Other modules MUST read through the owner's interface or receive the data from it. They do not reach into the owner's tables or internals.
- Any copy of state (cache, denormalized column, client-side copy, search index, replica in another service) MUST be explicitly treated as derived: it names its source, is never written independently, and has a defined invalidation or refresh mechanism.
- Client or UI state MUST NOT duplicate server state as a second source of truth. Server data on the client is a cache of the server's answer; local-only state (form drafts, selection, open panels) stays local.
- Changes that must be consistent MUST happen in one transaction or have an explicit, designed mechanism for eventual consistency (outbox, idempotent events, reconciliation). Never two independent writes that are hoped to both succeed.
- Writes that may be retried MUST be idempotent.

**Defaults**

- Prefer deriving over synchronizing. If a value can be computed from owned state, compute it.
- Keep state as local as it can be: local variable before component state, component state before shared store, shared store before global.
- Avoid mutable global and module-level singletons. Pass state and dependencies explicitly.
- When two modules both seem to need to write the same fact, the ownership is wrong: move the fact, or route one module's changes as requests to the owner.

**Red flags**

- Two code paths that update the same field.
- Code that exists to keep two values in sync.
- A bug described as stale, out of sync, or "sometimes shows the old value".
- A cache with no documented invalidation.
- A component that fetches server data, copies it into local state, and edits the copy.

## 3. Boundaries and dependency direction

Goal: a small change in requirements produces a small, local change in code.

**Hard rules**

- Dependencies MUST point inward: UI, API handlers, persistence, and external integrations depend on the core domain. The core domain MUST NOT import UI code, ORM or database clients, HTTP or web framework types, or vendor SDKs.
- The core MUST reach the outside world only through interfaces it defines (for example a repository or gateway interface), implemented by the outer layers and passed in.
- The module dependency graph MUST be acyclic. If A needs B and B needs A, extract the shared concept or invert one dependency.
- Modules MUST interact only through each other's public interface. No imports of another module's internal files, no reads of another module's tables.
- Framework, transport, and storage types (request objects, ORM rows, SDK responses) MUST be translated at the boundary and never passed into or returned from the core.
- Business rules MUST live in the core domain, not in controllers, UI components, SQL triggers, or integration glue.

**Defaults**

- Keep interfaces narrow: few methods, expressed in domain terms, hiding how the work is done.
- Keep side effects (I/O, network, clock, randomness) at the edges; keep core logic deterministic and testable without mocks of infrastructure.
- Each module has one responsibility that can be stated in a sentence. If it cannot, split it.
- Do not add layers that only pass calls through, or abstractions with a single implementation and no boundary to protect. A boundary earns its place by isolating something that changes independently.
- Organize code by domain capability rather than by technical type when the codebase allows it.
- Where tooling exists (import linting, dependency rules, architecture tests), encode the dependency rules so violations fail automatically.

**Red flags**

- A domain file importing a framework, ORM, or SDK.
- A change to one feature that touches many modules.
- A "utils", "common", or "shared" module that everything depends on and that keeps growing.
- Domain tests that need a database or HTTP server to run.
- An interface that mirrors one implementation's quirks.

## 4. UI components and UX

Goal: the interface is built from one small, consistent vocabulary, every state a user can reach is designed, and everyone can operate it. These rules are independent of language, framework, and platform; apply them using the project's existing component library and conventions.

**Hard rules**

- Visual values MUST come from a single source of design tokens (colour, spacing, type scale, radius, elevation, motion duration, breakpoints). No raw values hard-coded in components. If a needed value is missing, add a token rather than a one-off.
- Each UI concept MUST have exactly one component. Before creating a component, search for an existing one and extend it or compose from it. No parallel buttons, modals, inputs, or tables.
- Presentational components MUST NOT contain business rules or data access. They receive data and emit events; decisions and fetching live in the layers defined in sections 2 and 3.
- Every view that depends on data MUST handle all of its states: loading, empty, error, partial, and populated. Every interactive element MUST define its states: default, hover or pressed, focus, disabled, and busy where applicable.
- Every user action MUST produce visible feedback. Anything that takes noticeable time shows progress, and the triggering control is protected against duplicate submission.
- Destructive or irreversible actions MUST require confirmation or offer undo, and MUST be visually distinct from safe actions.
- A failure MUST NOT lose user input. Error messages MUST say what happened and what the user can do next, in plain language, placed next to the thing that failed. No raw error codes or stack traces as the only message.
- Accessibility is a hard requirement (target WCAG 2.2 AA or the platform equivalent):
  - Everything operable by pointer MUST be operable by keyboard or the platform's assistive input, in a logical order, with a visible focus indicator.
  - Controls MUST use the platform's semantic elements or expose equivalent role, name, and state. Every input has a programmatically associated label; every meaningful image or icon has a text alternative.
  - Text and essential graphics MUST meet contrast minimums. Information MUST NOT be conveyed by colour alone.
  - Overlays (dialogs, menus, popovers) MUST manage focus: move it in on open, contain it while open, restore it on close, and close on the platform's standard dismiss action.
  - Motion MUST respect the user's reduced-motion preference; nothing flashes or auto-plays without a way to stop it.
- Layout MUST adapt to the supported range of viewport sizes and to enlarged text without clipping, overlap, or loss of function.

**Defaults**

- Keep component interfaces small. Prefer composition over configuration, and a named variant over several booleans that can contradict each other (the same principle as the data model).
- Components SHOULD be controlled by a single source for their value and state; avoid components that keep a private copy of a value passed in.
- The same action SHOULD look, be named, and behave the same everywhere. Each view has one clear primary action.
- Follow platform conventions before inventing new patterns. Novel interactions need a reason.
- Prevent errors before reporting them: constrain inputs, provide sensible defaults, disable or explain unavailable actions. Validate inline at the field, after the user has had a chance to finish typing, not on every keystroke and not only on submit.
- Acknowledge input immediately; show a progress indicator or skeleton for waits beyond about a second. Use optimistic updates only where there is a defined rollback and error message.
- Interactive targets SHOULD be comfortably large for touch (about 44 by 44 points) with adequate spacing.
- Reveal complexity progressively: show the common path first and advanced options on request.
- Navigable state (current view, selection, filters, pagination) SHOULD be restorable and shareable through the platform's routing or deep-link mechanism, and survive back navigation and refresh.
- User-facing text SHOULD use the same names as the data model, be kept separate from logic so it can be translated, and never be built by concatenating fragments. Layouts tolerate longer text and right-to-left scripts where localisation is in scope.
- Motion SHOULD be brief and purposeful: it explains a change of state or location, never decorates.
- Each shared component SHOULD have examples of all its variants and states, so reuse is easier than rebuilding.

**Red flags**

- A hex code, pixel value, or font size written directly in a component.
- A second component that does nearly what an existing one does.
- A component that fetches data, applies a business rule, and renders, all in one place.
- A screen with no designed empty or error state, or a spinner with no failure path.
- A clickable element that is not a real control (no keyboard access, no role, no focus style).
- A component with a long list of boolean options.
- An error message that says only "Something went wrong".
- A form that clears itself when submission fails.

## Pre-implementation checklist

Answer each before writing code. A "no" on a hard rule blocks the change.

Data model
- Can each new or changed entity be defined in one sentence in domain terms?
- Is every new fact stored exactly once?
- Are identifiers, relationships, and constraints enforced in the schema?
- Can any contradictory state be represented? If so, can the model prevent it?
- Is there a safe migration with a backfill and rollback plan?

State ownership
- Who is the single owner of each fact touched by this change?
- Is there one write path, and does this change use it?
- Is every copy marked as derived with a refresh or invalidation mechanism?
- Are multi-step writes transactional or explicitly designed for eventual consistency?

Boundaries
- Which module does this belong in, and why?
- Do all new imports point inward, with no cycle introduced?
- Are external types translated at the edge?
- Is the business rule in the core domain?

UI components and UX
- Does an existing component already cover this? If not, why is a new one needed?
- Do all visual values come from tokens?
- Are loading, empty, error, partial, and populated states all designed, along with each control's interactive states?
- Can the whole flow be completed by keyboard and assistive technology, with labels, focus handling, and sufficient contrast?
- Does every action give feedback, and are destructive actions confirmable or undoable?
- Is user input preserved on failure, with an actionable message?
- Is the component free of business rules and data access?

## Reporting

When reviewing, or when a requested change conflicts with a rule, report each finding as:

- **Rule:** the area and the rule violated.
- **Where:** file and line, table, component, or design element.
- **Why it matters:** the concrete failure it leads to (for example "two write paths to `order.status`, so a refund can overwrite a shipment update").
- **Fix:** the smallest change that restores compliance.

Order findings hard rules first, then defaults. Keep pre-existing violations in a separate list from those introduced by the change. If there are no findings, say so in one line.

## Exceptions

Rules may be broken deliberately, never silently. When the user accepts a violation:

- Add a short comment at the site stating what rule is broken, why, and what would allow it to be removed.
- Contain it: keep the violation behind an interface so the rest of the code does not depend on it.
- Mention it in the summary of the change.

Typical legitimate exceptions: a denormalized column or cache justified by measured performance, a temporary dual write during a migration, a one-off visual treatment for a marketing surface, a prototype explicitly marked as throwaway.
