# Coding Guidelines

See [philosophy.md](philosophy.md) for the reasoning behind these guidelines.

**These are defaults, not dogma.** A project is free to deviate from any rule here, or add its own,
as long as the deviation and its rationale are documented — project-wide ones in the project's
`docs/guidelines.md`, local ones in a comment at the point of deviation. A justified exception is fine;
silent drift is not. Where the exception depends on a condition that may change, name the condition that
would retire it — a written rationale is what lets the exception be re-evaluated later instead of becoming
permanent by default. See [Applying These Guidelines](philosophy.md#applying-these-guidelines) for the bar
a rationale has to meet.

**Functional Style**:
- Prefer functional programming style over procedural style. Use pure functions and avoid mutability when possible.

**Function size**: Keep functions short and focused. When a function grows beyond ~30 lines or handles multiple concerns (e.g. scanning, processing, and persisting), extract named helpers. Each function should do one thing at one level of abstraction. Prefer reading like a high-level outline that delegates to well-named helpers over a single long procedure.

**Code comments**: When making changes to the codebase, explain the reasoning when the implementation is non-obvious, and document any non-trivial design decisions or trade-offs that were made.

**Charsets**:
- UTF-8 everywhere.

**Time Management**
- UTC for all timestamps
- Do not generate the current timestamps directly inside the core logic: pass the timestamps from the higher-level functions, tests, and other entry points.

**Mocks**
- Prefer passing values as explicit parameters (with sensible defaults) over reading global/ambient state internally. This makes functions testable without mocking. For example, pass `now: datetime` instead of calling `datetime.now()` internally, pass `platform: str = sys.platform` instead of reading `sys.platform` internally. Tests should pass these values explicitly rather than patching modules.
**Console Output**
- Do not hardcode indents in strings, compute the indent at the call site

**Version Management**
- Pin specific versions of all dependencies or use a lock file (e.g. `uv.lock`, `mise.lock`) to ensure reproducible builds and avoid issues with breaking changes in dependencies.

  ```bash
  # examples
  mise use --pin uv@0.12.3
  ```
- A lock file only helps if something asserts that the environment matches it. Provide both
  assertions, because they catch different failures: `uv lock --check` asks whether the lock is
  consistent with the manifest, while `UV_LOCKED=1` in CI asks whether installing would *change*
  the lock. Prefer the environment variable over a `--locked` flag on the command, so the install
  command stays identical locally and in CI and only the strictness differs.
- Watch for pins that no updater reads. `[tool.uv] build-constraint-dependencies` is the clearest
  example: `uv.lock` does not cover PEP 517 build dependencies, and no built-in Renovate or
  Dependabot manager parses that field, so those pins rot silently unless a custom manager is
  added for them. A pin nothing updates is a pin nothing tells you about.

**Command Line**
- When calling external commands, build the command lines as lists of arguments instead of strings to avoid issues with quoting and escaping.
- Make CLIs discoverable: commands should reference each other (e.g. a `check` command suggests `troubleshoot`, which suggests `rebuild`). 
  The user should be able to navigate the tool by following its output.
- Display paths relative to the base directory for readability. 
  When suggesting sample commands, use paths relative to the current working directory so they can be copy-pasted and run as-is.

**Testability**
- Expose exceptions/errors as structured data classes and perform the assertions on the structured output in tests instead of matching against raw error message strings. This allows for more robust tests that are not brittle to changes in error message formatting.

**No Silent Failures**
- Avoid silent failures and ensure that all errors are surfaced with clear messages. This includes validating inputs and configurations early, and providing informative error messages when something goes wrong.
