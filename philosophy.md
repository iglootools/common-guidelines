# Philosophy

This document describes the general philosophy that underpins the [coding guidelines](coding.md). 
The guidelines are the concrete rules; this is the reasoning behind them.

## Workflow

### Shorten every feedback loop

**The faster you can see the effect of a change, the faster you can iterate.**

Every minute spent waiting for feedback is a minute of lost context. If a workflow feels slow, treat that as a bug worth fixing.

- Invest in making feedback instant wherever possible. 
- Separate fast tests (unit, in-memory) from slow ones (integration, filesystem, network) so the **common case runs in seconds**. 
- Build demo commands that exercise the functionality end-to-end with fake data — no manual setup required. 
- For formatting functions and visual components, generate sample output with representative data so changes can be reviewed at a glance. 
- Provide tooling that seeds test environments instantly rather than requiring manual setup steps. 

### Automate the toil

If you do something manually more than twice, automate it. 
Formatting, linting, type checking, and test execution should run automatically — in CI, in pre-commit hooks, or in editor integrations. 
Humans should not have to remember to run the formatter or check for lint errors; 
the system should enforce standards so that code review can focus on design and correctness, not style. 
The same applies to environment setup, data seeding, and deployment: every manual step is a step that will eventually be skipped or done wrong.

Further reading on coding philosophy:
- [A Philosophy of Software Design](https://web.stanford.edu/~ouster/cgi-bin/book.php) by John Ousterhout
- [Simple Made Easy](https://www.infoq.com/presentations/Simple-Made-Easy/) by Rich Hickey (talk)
- [The Value of Values](https://www.infoq.com/presentations/Value-Values/) by Rich Hickey (talk)
- [Out of the Tar Pit](http://curtclifton.net/papers/MosesSchonworkerTarPit.pdf) by Moseley & Marks (paper)


### Do fewer things, do them completely

**Shipping one fully finished feature is better than shipping three half-done ones.** 
Half-done features accumulate unhandled corner cases that are hard to reason about, 
cause things to break while building something else, and erode trust in the system.

A feature that works but lacks tests, docs, and error handling is unfinished — it just has the illusion of progress.
Resist the temptation to chase "low-hanging fruit" — a backlog full of quick wins that are each 20% done is worse than a short list of things done properly. 
Reduce scope aggressively so that what you do ship meets the full bar of quality. 
This applies to sprints, pull requests, and individual commits: each unit of work should be complete, not just functional.

**Define the quality bar and enforce it.**
Define standards and maintain implementation checklists that cover the full scale of concerns: documentation, tests, testability (can the feature be tested in isolation?), 
introspection tooling (can someone inspect what happened?), manual testing workflows, error messages, dry-run support, logging, operations, security, and configuration. 
Without a checklist, these concerns get addressed unevenly — the first feature gets thorough tests, the fifth one ships with none. 
The checklist makes thoroughness a habit rather than a heroic effort.

### Small, reviewable changes

Large, messy PRs don't get reviewed well — reviewers skim, miss problems, and rubber-stamp. 
Keep changes focused: one concern per commit, one intent per PR. Invest in the Git commit history.
This pairs naturally with "do fewer things completely" — small scope, fully finished. 

### No broken windows workflow

Treat bugs as unfinished features, not as a separate category of work to be triaged later. 
A known bug that stays open signals that low quality is acceptable — and that signal compounds. 
Fix small things as you encounter them: a misleading error message, a missing test, an outdated comment, a confusing name, missing tooling for introspection. 
Leave every file you touch in a better state than you found it. 
Investigate any strange behavior, even if things seem to "work" — oddities often hide real problems that surface later in worse conditions.
Technical debt is not paid down in dedicated sprints; it is kept in check by refusing to walk past problems.

### Reproducible environments

Pin dependencies, use lock files, and script the development setup. "Works on my machine" is a symptom of environment drift.
If the build depends on implicit system state, it will break for someone else. 
A new contributor should be productive within minutes, not days. 
CI should run in the same environment as local development whenever possible - when environments diverge, bugs hide in the gap.

Further reading on workflow and craft:
- [The Pragmatic Programmer](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/) by Hunt & Thomas — the original "broken windows" metaphor for software
- [Accelerate](https://itrevolution.com/product/accelerate/) by Forsgren, Humble & Kim — research backing short feedback loops, small batches, and continuous delivery


## Architecture and System Design

### High cohesion, low coupling — at every boundary

Draw boundaries so that related things live together and unrelated things communicate through narrow, explicit interfaces. 
This applies to functions, modules, packages, and repositories alike.

A monorepo that lumps everything together does not solve coupling — it hides it. 
Conversely, splitting into many repos with tangled dependencies just distributes the monolith across git remotes, 
requiring coordinated PRs in five places for every feature.

The goal is that 80% of changes affect a single repository — not because everything is in one place, but because dependencies are handled mindfully. 
Each unit should own a coherent slice of the domain, expose a stable contract, and evolve independently. 
When a change routinely fans out across many boundaries, the boundaries are in the wrong place.

### Event-driven, append-only, replayable

**Events as the source of truth.** Prefer event sourcing and append-only logs over mutable state in a database. 
The sequence of events *is* the data — current state is a derived view that can be rebuilt at any time. 
This gives you a complete audit trail, makes debugging straightforward (replay to reproduce), 
and keeps the door open for new projections without migrating existing state.

**Separate reads from writes.** Follow CQRS: the path that accepts commands and produces events should be distinct from the path that serves queries. 
Write models optimize for consistency and validation; read models optimize for the queries callers actually need. 
This separation keeps each side simple and independently scalable, and avoids the compromises of a single model that tries to serve both.

**Append-only over in-place mutation.** Treat storage as a log. Append new entries rather than updating or deleting existing ones. 
Write-ahead logs (WALs), event logs, and immutable file structures all follow this pattern. 
Append-only storage is inherently safer (no data loss from a failed update), simpler to replicate, and naturally supports point-in-time recovery.

**Derive state, don't store it.** Materialized views, caches, and aggregates should be rebuildable from the event log. 
If a projection becomes stale or wrong, the fix is to replay — not to patch the derived data. 
This keeps the event log authoritative and reduces the cost of schema changes: add a new projection, backfill from history, done.

**Make processing idempotent.** Running the same operation twice should produce the same result as running it once. 
This applies at every level — event handlers, file imports, data migrations, API calls. 
Idempotency makes retry safe, crash recovery straightforward, and eliminates an entire class of "was this already applied?" bugs.

**Preserve data, build tooling around it.** Never destroy information for convenience. 
Keep the full-fidelity source and build views, summaries, or simplified formats on top. 
Use merges instead of squashes in git — the full commit history is preserved and can always be summarized later, 
but a squash permanently discards the individual steps. 
The same principle applies everywhere: keep originals, derive what you need, and invest in tooling 
that makes the rich data easy to work with rather than throwing it away to make the simple case simpler.

**Design for replay and recovery.** If a process crashes mid-way, restarting from the last checkpoint should produce the correct result.
This is a natural consequence of append-only, event-driven, idempotent design — lean into it rather than bolting it on.

For a deeper treatment of these ideas — stream processing, derived data, and logs as the fundamental abstraction:
- [Designing Data-Intensive Applications](https://dataintensive.net/) by Martin Kleppmann
- [I Heart Logs](https://www.oreilly.com/library/view/i-heart-logs/9781491909379/) by Jay Kreps
- [The Log: What every software engineer should know about real-time data's unifying abstraction](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) by Jay Kreps
- [Domain-Driven Design](https://www.domainlanguage.com/ddd/) by Eric Evans
- [Implementing Domain-Driven Design](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/) by Vaughn Vernon
- [Release It!](https://pragprog.com/titles/mnee2/release-it-second-edition/) by Michael Nygaard


### Errors as first-class values

Treat errors as data, not as control flow. Return structured error values rather than throwing exceptions that propagate invisibly through the call stack. 
When errors are values, the type system and function signatures make it clear what can fail — nothing is hidden. 

Unhandled errors should halt the system, not silently continue. 
A crash is a forcing function: it makes you deal with the problem explicitly instead of letting it compound unnoticed until it corrupts state or breaks something unrelated. 
Silent failures are the most expensive kind — they hide problems behind a facade of "working" until the damage is too deep to trace back to the cause.

### Observability is not optional

Logging, metrics, and tracing should be designed in from the start, not bolted on after the first production incident. 
If you can't tell what the system is doing from the outside, you can't operate it. 
Structured logs, correlation IDs, and health checks are not nice-to-haves — they are the interface between the system and the people responsible for keeping it running. 
When something goes wrong at 3 AM, observability is the difference between a five-minute diagnosis and a multi-hour guessing game.

### Security as a default, not a phase

Security is not a separate review step or a pre-launch checklist — it is a property of how things are built. 
Secrets stay out of code and version control. 
Access follows least privilege. Inputs are validated at system boundaries. Dependencies are scanned and updated. 
These are not extra work on top of the feature; they are part of the feature being done. 
Treating security as an afterthought guarantees that it will be addressed unevenly — the critical paths get reviewed, the internal tooling doesn't, and that's where the breach comes from.

### Assume failure in distributed systems

Networks partition, processes crash, clocks drift. These are not edge cases — they are the normal operating mode. 
Design for the failure case first: timeouts on every external call, retries with exponential backoff, circuit breakers to avoid cascading failures. Treat "the happy path worked" as insufficient evidence that the system is correct. The question is not *if* a component will fail, but *when* — and whether the rest of the system degrades gracefully or falls over with it.

## Coding

### Correctness over convenience

The default should be safe. Destructive operations require explicit opt-in. Dry-run is always available. 
Errors are surfaced early with clear messages, never swallowed. If the tool is unsure, it asks rather than guesses.

### Explicit over implicit

Functions receive what they need as parameters rather than reaching for global or ambient state. 
Timestamps, platform details, and configuration are passed in, not discovered internally. This makes behavior predictable and testable without mocks or patches.

### Functional style preferred

Prefer pure functions, immutable data, and expressions over procedures, mutable state, and statements.
Build values in a single expression rather than accumulating into mutable containers.
For instance, in Python, frozen data classes, comprehensions, and unpacking produce code where the shape of the result is visible at the definition site.
Side effects should be pushed to the edges — core logic transforms data, the outer layer performs I/O.
Mutation is reserved for cases where it genuinely simplifies the logic.

### Document the why, not the what

Code shows what happens; comments should explain why. Don't restate what the code already says.
document the constraints, trade-offs, and non-obvious decisions that would otherwise be lost. 
For architectural choices that affect multiple components, use Architecture Decision Records (ADRs) or equivalent. 
The goal is that someone reading the code six months later can understand not just what it does, but why it does it that way instead of the obvious alternative.

### Name things precisely

Names are the primary documentation. A well-chosen name makes the code self-explanatory; a misleading name is worse than a bad one because it actively hides the truth. 
Rename aggressively when understanding evolves — the cost of a rename is near zero, the cost of a misleading name compounds every time someone reads the code. 
If you struggle to name something, that is often a signal that the concept itself is unclear and needs to be rethought.

### Test behavior, not implementation

Tests should assert on observable outcomes — return values, state changes, output — not on internal mechanics like which methods were called or in what order. 
Implementation-coupled tests break on every refactor and provide false confidence: they verify that the code does what it does, not that it does what it should. 
When tests assert on behavior, refactoring becomes safe and tests remain meaningful.

### Make the right thing easy and the wrong thing hard

API design, defaults, and project structure should guide users toward correct usage. 
If misuse is common, the design is wrong — fix the interface, don't add documentation warnings. 
Safe defaults, restrictive types, and clear naming all reduce the surface area for mistakes. 
The goal is that doing the right thing requires less effort than doing the wrong thing.

### Design for deletion

Write code that is easy to remove, not just easy to extend. Small, decoupled modules with clear boundaries can be ripped out when requirements change. 
If removing a feature requires touching dozens of files, the coupling is too tight. 
The measure of good architecture is not how easy it is to add code, but how easy it is to remove it.

### Make everything inspectable

Prefer plain text formats (TOML, UTF-8 text, directories) when the data model allows it — they are readable with `ls`, `cat`, and a text editor, 
with no special tooling required.
When binary formats or databases are necessary, invest in tooling that makes their contents easy to inspect and debug — dump commands, human-readable export, 
structured logging.
The goal is that no part of the system should be a black box: anyone should be able to understand what happened and why without reverse-engineering the storage format.

