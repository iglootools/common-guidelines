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
