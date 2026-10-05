# workflows

Reusable GitHub Actions workflows for every repo of **dragos-catalin** (and the personal `dragoscv` repos). Public on purpose: a public repo's reusable workflows can be called from any account or org, and contain no secrets.

| Workflow                                           | What it runs                                                                                                                                                                                                                                                                                                                                                        | Runner          |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| [`node-ci.yml`](.github/workflows/node-ci.yml)     | pnpm install (store cached via restore/save, so red builds still warm it) → engines check (`.nvmrc` / `engines.node` / `packageManager` agree) → each task (`lint typecheck test` by default) reported separately → optional `extra` commands. `turbo: true` runs only packages affected since the PR base / previous push, with the turbo cache kept between runs. | `ubuntu-latest` |
| [`python-ci.yml`](.github/workflows/python-ci.yml) | uv (cached) → ruff check (+ optional `ruff format --check`) → pytest.                                                                                                                                                                                                                                                                                               | `ubuntu-latest` |
| [`security.yml`](.github/workflows/security.yml)   | gitleaks CLI (only the new commits on push/PR, full history on schedule) + osv-scanner over every lockfile ecosystem (pnpm, npm, Cargo, pip/uv, Gradle, Go), failing at CVSS ≥ `fail-severity` (default 7) + licence summary or allowlist. Checksums verified for both binaries.                                                                                    | `ubuntu-slim`   |

## Use

```yaml
# .github/workflows/ci.yml
name: ci
on:
  pull_request:
  push:
    branches: [main]
    paths-ignore: ["**.md", "docs/**"]
  schedule:
    - cron: "17 3 * * 1" # weekly full-history secret scan + fresh CVE data
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read

jobs:
  node:
    uses: dragos-catalin/workflows/.github/workflows/node-ci.yml@v1
    with:
      tasks: lint typecheck test
  security:
    uses: dragos-catalin/workflows/.github/workflows/security.yml@v1
    with:
      full-history: ${{ github.event_name != 'push' && github.event_name != 'pull_request' }}
```

Callers pin `@v1` (moving major tag) or a commit SHA. Breaking input changes bump the major.

## Cost model (why it is shaped like this)

- Private repos pay per job, **rounded up to the minute** ($0.006/min Linux 2-core, $0.002/min `ubuntu-slim`). One verify job with `if: !cancelled()` per check reports every failure at a third of the cost of separate lint/typecheck/test jobs that each reinstall.
- Security jobs need no CPU → `ubuntu-slim`.
- `cancel-in-progress` only on PRs: cancelling main runs means caches are never saved.
- Docs-only pushes skip via `paths-ignore` in the caller; turbo repos run only affected packages.
- Exceptions to a finding go in the caller repo: `.gitleaksignore` (fingerprints) and `osv-scanner.toml` (`[[IgnoredVulns]]` with `reason` and `ignoreUntil`), never by lowering the gate.

## Local equivalent

`gitleaks git . --redact`, `osv-scanner scan source -r .` — same versions as `GITLEAKS_VERSION` / `OSV_VERSION` in `security.yml`.
