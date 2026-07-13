# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains a single reusable GitHub Actions workflow (`.github/workflows/workflow.yml`) for testing Python packages that use `uv` + `invoke`. It is consumed by other repositories via `workflow_call`.

## Workflow Design

The workflow exposes two inputs to callers:

| Input | Default | Purpose |
|---|---|---|
| `support-python-versions` | `[ '3.14', '3.13', '3.12', '3.11', '3.10' ]` | JSON array of Python versions to test against |
| `is-supporting-python-3-7-in-linux` | `false` | Enables a separate legacy job on Ubuntu 22.04 for Python 3.7 |
| `setup-command-linux` | `''` | Optional command run on Linux runners (bash) after checkout, before uv — installs system tools (e.g. ffmpeg) tests depend on; also used by the legacy 3.7 job |
| `setup-command-windows` | `''` | Optional command run on Windows runners (PowerShell) after checkout, before uv — installs system tools tests depend on |

### Jobs

- **`test`** — matrix across `[ubuntu-latest, windows-latest]` × `support-python-versions`. Runs the optional setup-command hook (if non-empty), then `uv sync`, then `uv run invoke test.all`. On Windows the shell is forced to `cmd` because Invoke does not support PowerShell.
- **`test_legacy_linux`** — runs only when `is-supporting-python-3-7-in-linux: true`. Uses `actions/setup-python` (not `setup-uv`) because uv has limited Python 3.7 support; installs uv via the shell script instead. Runs `setup-command-linux` (if non-empty) after checkout.

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

## Maintenance

Dependabot is configured to keep `github-actions` dependencies updated weekly (`.github/dependabot.yml`). Action versions in `workflow.yml` (`actions/checkout`, `astral-sh/setup-uv`, `actions/setup-python`) are the ones callers inherit, so keep them current.
