# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains a single reusable GitHub Actions workflow (`.github/workflows/workflow.yml`) for testing Python packages that use `uv` + `invoke`. It is consumed by other repositories via `workflow_call`; there is no Python package, application code, or test suite of its own to build/run here — the workflow can only be truly exercised by a caller repo's own GitHub Actions run.

## Commands

The only local checks available are on the workflow file itself:

```bash
# YAML well-formedness (PyYAML fetched ad hoc via uv; no venv/lockfile in this repo)
uv run --with pyyaml python3 -c "import yaml; yaml.safe_load(open('.github/workflows/workflow.yml'))"

# Static analysis of workflow syntax/expressions, if actionlint is installed locally
actionlint .github/workflows/workflow.yml
```

## Workflow Design

The workflow exposes the following inputs to callers:

| Input | Default | Purpose |
|---|---|---|
| `support-python-versions` | `[ '3.14', '3.13', '3.12', '3.11', '3.10' ]` | JSON array of Python versions to test against |
| `support-os` | `[ 'ubuntu-latest', 'windows-latest' ]` | JSON array of runner labels for the main `test` matrix; Ubuntu/Windows labels supported, other runners run tests via the non-Windows branch but are untested. Does not affect the legacy 3.7 job |
| `is-supporting-python-3-7-in-linux` | `false` | Enables a separate legacy job on Ubuntu 22.04 for Python 3.7 |
| `setup-command-linux` | `''` | Optional command run on Linux runners (bash) after checkout, before `uv sync` — installs system tools (e.g. ffmpeg) tests depend on; also used by the legacy 3.7 job |
| `setup-command-windows` | `''` | Optional command run on Windows runners (PowerShell) after checkout, before `uv sync` — installs system tools tests depend on |
| `post-sync-command-linux` | `''` | Optional command run on Linux runners (bash) after `uv sync`, before the test step — runs with the project venv and its console scripts available (e.g. `uv run <project-cli> install`); also used by the legacy 3.7 job |
| `post-sync-command-windows` | `''` | Optional command run on Windows runners (PowerShell) after `uv sync`, before the test step |

The two hook pairs bracket `uv sync` deliberately: `setup-command-*` runs before it because it installs *system* tools that native builds during dependency resolution might need, while `post-sync-command-*` runs after it because it needs the project's own virtualenv and installed console scripts to exist — those only appear once `uv sync` has completed.

### Jobs

- **`test`** — matrix across `support-os` (default `[ubuntu-latest, windows-latest]`) × `support-python-versions`. Order: checkout → `setup-command-*` hook (if non-empty) → `astral-sh/setup-uv` → `uv sync` → `post-sync-command-*` hook (if non-empty) → `uv run invoke test.all`. On Windows the shell is forced to `cmd` because Invoke does not support PowerShell; on any non-Windows runner the test step runs in the default shell (`runner.os != 'Windows'` guard, so unlisted runners like macOS never silently skip tests).
- **`test_legacy_linux`** — runs only when `is-supporting-python-3-7-in-linux: true`. Uses `actions/setup-python` (not `setup-uv`) because uv has limited Python 3.7 support; installs uv via the shell script instead. Runs `setup-command-linux` after checkout and `post-sync-command-linux` after `uv sync --python ...`, both if non-empty (Linux-only job, so no Windows hook here).

### Key constraint

Both jobs call `uv run invoke test.all` — the calling repository must define that Invoke task.

## Caller Usage

Minimal caller workflow:

```yaml
jobs:
  call:
    uses: yukihiko-shinoda/reusable-workflow-invoke-lint-test/.github/workflows/workflow.yml@v1
```

With custom Python versions:

```yaml
jobs:
  call:
    uses: yukihiko-shinoda/reusable-workflow-invoke-lint-test/.github/workflows/workflow.yml@v1
    with:
      support-python-versions: "[ '3.13', '3.12', '3.11' ]"
```

## Versioning

Tags follow semver (`v1.0.0`, `v1.0.1`, `v1.1.0`, ...), plus a floating major tag `v1` that always points at the latest `v1.x.y` commit — callers pin `@v1` (see README Quickstart) to auto-track the newest compatible release without a manual bump. Patch releases are for things like Dependabot action bumps (`v1.0.0` → `v1.0.1`); minor releases are for backward-compatible input additions (`v1.0.0` → `v1.1.0`). When cutting a new release, tag `vX.Y.Z`, then move the floating `vN` tag to the same commit and push both.

## Maintenance

Dependabot is configured to keep `github-actions` dependencies updated weekly (`.github/dependabot.yml`). Action versions in `workflow.yml` (`actions/checkout`, `astral-sh/setup-uv`, `actions/setup-python`) are the ones callers inherit, so keep them current.
