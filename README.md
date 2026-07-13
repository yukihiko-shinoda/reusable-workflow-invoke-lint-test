# Reusable Workflow Invoke Lint Test

[![Dependabot](https://flat.badgen.net/github/dependabot/yukihiko-shinoda/reusable-workflow-invoke-lint-test?icon=dependabot)](https://github.com/yukihiko-shinoda/reusable-workflow-invoke-lint-test/security/dependabot)

Reusable GitHub Actions workflow for testing Python packages that use [`uv`](https://docs.astral.sh/uv/) + [`Invoke Lint`](https://pypi.org/project/invokelint/).

## Advantage

Testing Python packages across multiple versions and OSes requires per-repo boilerplate that drifts over time. This workflow centralises that logic for projects already using `Invoke Lint` so callers get:

- **Cross-platform coverage out of the box** — matrix across Ubuntu and Windows with a single call.
- **Invoke on Windows just works** — forces `cmd` shell automatically, because Invoke does not support PowerShell.
- **Python 3.7 legacy support when you need it** — opt-in job on Ubuntu 22.04 that falls back from `setup-uv` to `actions/setup-python` + the uv shell installer, working around uv's limited 3.7 support.
- **System-tool installation hook** — optional `setup-command-linux` / `setup-command-windows` inputs run a caller-supplied command (e.g. installing `ffmpeg`) on each runner before tests, so suites that shell out to CLI tools work without forking the workflow.

## Quickstart

Create `.github/workflows/test.yml` in your repository:

```yaml
name: Test
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
permissions:
  contents: read
jobs:
  call:
    uses: yukihiko-shinoda/reusable-workflow-invoke-lint-test/.github/workflows/workflow.yml@v1
```

Your repository must define an `invoke` task `test.all` that runs your test suite.

<!-- markdownlint-disable-next-line no-trailing-punctuation -->
## How do I...

### How do I test against specific Python versions?

Pass a JSON array string to `support-python-versions`:

```yaml
jobs:
  call:
    uses: yukihiko-shinoda/reusable-workflow-invoke-lint-test/.github/workflows/workflow.yml@v1
    with:
      support-python-versions: "[ '3.13', '3.12', '3.11' ]"
```

### How do I install system tools my tests depend on (e.g. ffmpeg)?

Pass an install command per OS via `setup-command-linux` and `setup-command-windows`. Each runs right after checkout and before `uv sync`, only when non-empty:

```yaml
jobs:
  call:
    uses: yukihiko-shinoda/reusable-workflow-invoke-lint-test/.github/workflows/workflow.yml@v1
    with:
      setup-command-linux: sudo apt-get update && sudo apt-get install -y ffmpeg
      setup-command-windows: choco install ffmpeg -y
```

`setup-command-linux` runs in bash; `setup-command-windows` runs in PowerShell (the runners' default shells). When the Python 3.7 legacy job is enabled, it runs `setup-command-linux` too.

### How do I enable the Python 3.7 legacy job?

Set `is-supporting-python-3-7-in-linux: true`:

```yaml
jobs:
  call:
    uses: yukihiko-shinoda/reusable-workflow-invoke-lint-test/.github/workflows/workflow.yml@v1
    with:
      is-supporting-python-3-7-in-linux: true
```

This adds a separate job on `ubuntu-22.04` that installs uv via the shell script and runs `uv run invoke test.all` under Python 3.7.

## API

### Inputs

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `support-python-versions` | `string` | `"[ '3.14', '3.13', '3.12', '3.11', '3.10' ]"` | JSON array of Python versions for the main matrix |
| `is-supporting-python-3-7-in-linux` | `boolean` | `false` | When `true`, enables an additional job on Ubuntu 22.04 for Python 3.7 |
| `setup-command-linux` | `string` | `''` | Command run on Linux runners (bash) after checkout, before uv — for installing system tools like `ffmpeg`. Empty = skipped. Also runs in the Python 3.7 legacy job. |
| `setup-command-windows` | `string` | `''` | Command run on Windows runners (PowerShell) after checkout, before uv — for installing system tools like `ffmpeg`. Empty = skipped. |

### Jobs

| Job | Runner | Condition |
| --- | --- | --- |
| `test` | `ubuntu-latest`, `windows-latest` × each version in `support-python-versions` | Always |
| `test_legacy_linux` | `ubuntu-22.04` with Python 3.7 | Only when `is-supporting-python-3-7-in-linux: true` |
