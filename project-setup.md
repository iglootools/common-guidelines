# Project Setup Guidelines

See [philosophy.md](philosophy.md) for the reasoning behind these guidelines, and
[Applying These Guidelines](philosophy.md#applying-these-guidelines) for how to deviate from them —
these are defaults, and a documented, justified exception is always allowed.

Language- and tool-specific setup lives alongside this file: [python-tooling.md](python-tooling.md)
for uv, mise and packaging, and [ide.md](ide.md) for editor and Claude Code configuration.

## GitHub Workflows
- Whenever safe (i.e. not affecting production), enable `workflow_dispatch` and `repository_dispatch` to allow manual triggering of workflows from the GitHub UI or CLI, which is useful for testing and debugging.
- Use OpenID Connect (OIDC) authentication for publishing to PyPI, and set up a separate workflow for testing releases to Test PyPI. This allows testing the release and publish process without affecting the real PyPI index, and provides more detailed logs for debugging.
  - When the Test PyPI workflow synthesizes a throwaway version, derive it from the **base
    release** version, not from the full VCS-derived one. Appending `.devYYYYMMDDHHMMSS` to a
    version that already carries `.dev0` produces two dev segments and is invalid PEP 440 — a bug
    that stays hidden as long as the workflow is only ever dispatched from a tag. Reading the tag
    (`git describe --tags --abbrev=0`) avoids it and needs no build tool.
- **Prefer tools whose mise backend is lockable.** Check with `mise registry <tool>`: a backend
  such as `aqua:` records per-platform URLs and checksums in `mise.lock`, while `vfox:` records
  none, and `mise lock` cannot generate them. An unlockable tool makes `mise install --locked`
  fail on a fresh runner, which forces `install: false` plus a bare `mise install` into every
  workflow — quietly giving up checksum and attestation verification for *all* tools to
  accommodate one. The one legitimate use of `install: false` is a workflow that runs on branches
  where `mise.toml` is deliberately ahead of `mise.lock`, such as one that regenerates the lock
  for Renovate branches.
- **Pair `install: false` with `env: false`.** After setup, `jdx/mise-action` exports mise's
  `[env]` into the job — and the `UV_PYTHON = "{{ tools.python.path }}"` pin that
  [python-tooling.md](python-tooling.md#configuration) requires resolves through the *installed*
  toolset. With nothing installed, the template fails with ``Field `python` is not defined`` and
  takes the whole setup step down, before the workflow's own steps ever run. The two inputs are
  therefore a pair, not independent knobs: skipping the install means skipping the export. Nothing
  is lost, because a lock-regenerating workflow needs the mise binary and the config, not a tool
  environment.
- **Name every workflow after its own file.** `name: test` in `test.yml`, `name:
  renovate-mise-lock` in `renovate-mise-lock.yml` — the filename's stem, verbatim, kebab-case
  included. GitHub exposes both identifiers and neither one substitutes for the other: the
  status-badge URL, the REST `/actions/workflows/{file}` endpoint, and `gh workflow run` address
  a workflow by filename, while the Actions UI, the `workflows:` list of a `workflow_run`
  trigger, and `${{ github.workflow }}` address it by name. Letting them diverge means holding
  a mapping between the two in your head every time you read a badge, a trigger, or a run
  listing.

  Two failures make this more than tidiness. A workflow with no `name:` at all gets
  `github.workflow` set to its *path*, so the concurrency group below silently changes shape
  — matching the filename keeps the group readable whether or not the key is set. And two
  workflows that share a `name:` share a `github.workflow`, which puts them in the *same*
  concurrency group: with `cancel-in-progress: true` they cancel each other, for no reason
  visible in either file. Filenames cannot collide within a directory, so deriving the name
  from the file makes that collision impossible by construction.

- **Give every workflow a concurrency group, and pick the variant by whether a half-finished
  run can be abandoned and redone.** There are two. The test is not whether the workflow writes
  to something outside the repository — plenty of external writes are perfectly safe to
  cancel — but whether killing it partway leaves state that a later run cannot simply redo from
  scratch.

  Supersede the run in flight whenever abandoning it costs nothing but the compute already
  spent. That covers anything that only reads and reports — test, lint, link-check, build — and
  equally anything that writes somewhere re-writable: a container tag or docs site the next run
  overwrites wholesale, a preview or dev environment rebuilt from scratch on every deploy. The
  external write is not the problem; a cancelled run there leaves nothing that re-running does
  not replace.

  ```yaml
  concurrency:
    group: ${{ github.workflow }}-${{ github.ref == 'refs/heads/main' && format('main-{0}', github.event.workflow_run.head_sha || github.sha) || github.ref }}
    cancel-in-progress: true
  ```

  A per-sha group on main means no push to main is ever cancelled, so every commit gets a
  complete run; a per-ref group everywhere else means a new push supersedes the run still in
  flight for that pull request or tag. `cancel-in-progress: true` is what does the cancelling,
  and making the group unique per commit is what exempts main from it.

  Queue instead when a partial run leaves state the next run cannot repair:

  ```yaml
  concurrency:
    group: ${{ github.workflow }}
    cancel-in-progress: false
  ```

  Publishing to PyPI is the clearest case: uploaded files are immutable, and a deleted filename
  cannot be re-uploaded, so a run cancelled between two artifact uploads leaves that version
  permanently half-populated — the only way out is burning a version number. A deployment
  cancelled mid-rollout leaves part of the fleet on the new build and the rest on the old, which
  no re-run reconstructs because it cannot know how far the killed run got. Creating tags or
  releases is the same shape. What these share is not that the write is external but that it is
  not replayable: the second attempt cannot start from a clean slate. Waiting is cheap by
  comparison.

  The key deliberately carries no ref or sha, so two runs cannot overlap even when they carry
  different ones — a publish workflow reachable by tag push, by `workflow_run`, and by manual
  dispatch can have more than one trigger aiming at the same version. Note that GitHub keeps at
  most one run *pending* per group: a third arrival supersedes the pending run, never the
  running one.

  Two traps to avoid, both of which were live in these repos:

  - **Do not interpolate `github.workflow` inside the `format()` as well as outside it.** The
    group then comes out as `test-test-main-…`, and if both arms of the conditional also carry
    a literal `-main-` infix, a pull request's group reads
    `test-test-main-refs/pull/95/merge`. Harmless but misleading, and it makes the groups hard
    to recognise in the API.
  - **Prefer `github.event.workflow_run.head_sha` over `github.sha` for the main arm.** For a
    `workflow_run` event GitHub sets `GITHUB_REF` to the default branch and `GITHUB_SHA` to the
    *last commit on the default branch* — not the commit that triggered the run. Keying on
    `github.sha` therefore drops every `workflow_run`-triggered run into a single group for as
    long as the default branch head does not move, and `cancel-in-progress` kills the run
    already going. The term is inert in workflows with no `workflow_run` trigger, so the same
    expression can be used verbatim everywhere.

  `actionlint` will not catch the second one: it validates expression syntax but treats
  `github.event.*` as loosely typed, and exits 0 even on
  `github.event.workflow_run.head_shaa`. Confirm payload property names against the
  workflow-run object in the REST API instead.

  Two established spellings of the queueing variant are worth recognising rather than
  "fixing": a GitHub Pages deployment uses the fixed `group: pages`, which is the documented
  convention for that action, and a lock-regenerating workflow keyed on `github.ref` alone is
  fine because it only ever runs on branches and one run per branch is the point.

- Set `timeout-minutes` on every job. Without it, a hung step (a stalled `apt-get`, a network call that never returns) runs until GitHub's 6-hour default before the job is killed, wasting CI minutes and delaying feedback. A tight job-level guard (e.g. `timeout-minutes: 10`, sized to the job) fails fast and legibly. Prefer a single job-level timeout over per-step timeouts: one guard covers the whole job with no per-step bookkeeping.

## All Projects

- `renovate.json`: group all dependency updates into a single PR, delay
  updates by 14 days (`minimumReleaseAge`) to avoid adopting broken
  releases and limit risk of supply chain attacks
- Github build/test/release workflows
- `git config user.email "<email>"` and `git config user.name "<name>"` in the project.

### Split Renovate and Dependabot by job, not by ecosystem

The 14-day delay above is a supply-chain measure: it protects you from a release that turns out
to be broken or malicious, and it works by waiting. Waiting is exactly the wrong response to a
published advisory, where the fix is already known and every day of delay is another day of
exposure.

One tool cannot be both slow and immediate, so give the two tools opposite latencies:

| Tool | Handles | Latency |
|---|---|---|
| Renovate | routine version updates, grouped into one PR | delayed 14 days (`minimumReleaseAge`) |
| Dependabot | security updates only | immediate |

Renovate takes the routine half because it is also the only one of the two with managers for
`mise.toml` and `customManagers` regex entries. Dependabot takes the security half because its
advisory-driven updates are exempt from delay.

Do **not** let both do version updates. That produces duplicate PRs, and Dependabot's half
ignores the 14-day delay — quietly defeating the measure the delay exists for.

`open-pull-requests-limit: 0` is what implements the split. It is GitHub's documented way to
switch off *version* updates for an ecosystem, and security PRs are exempt from it (and from
`cooldown`), so setting it to zero leaves exactly the security half running:

```yaml
updates:
  - package-ecosystem: "uv"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 0
```

**`package-ecosystem` must track the build tool.** A stale value fails *silently*: `pip` reads
`poetry.lock` and understands neither `uv.lock` nor `[dependency-groups]`, so after a migration
it keeps running, finds nothing, and reports green — indistinguishable from working coverage.

### Ignore OS and editor cruft in the committed `.gitignore`

Not in `.git/info/exclude` or a personal `core.excludesfile`. Those are not shared on clone, so
they look like coverage to whoever set them up while giving none to anyone else. This is not
cosmetic: hatchling reads only `.gitignore` when selecting files to package, so a `.DS_Store`
inside the package directory that is ignored only locally gets published inside the wheel.

## Python Projects

See [python-tooling.md](python-tooling.md) for the build backend, the mise and uv configuration,
and the task set.

## CLI Projects

Consider providing:
- `asciinema` demo
- auto-generated `clidocs`/`clidocs-check` tasks for CLI documentation
- Provide preflight checks wherever applicable and troubleshooting instructions with the specific commands that need to be executed to fix the problem.
