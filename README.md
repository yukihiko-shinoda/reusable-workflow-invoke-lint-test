# Reusable Workflow Invoke Lint Test

[![Dependabot](https://flat.badgen.net/github/dependabot/yukihiko-shinoda/reusable-workflow-invoke-lint-test?icon=dependabot)](https://github.com/yukihiko-shinoda/reusable-workflow-invoke-lint-test/security/dependabot)

Reusable GitHub Actions workflow for testing Python packages that use [`uv`](https://docs.astral.sh/uv/) + [`Invoke Lint`](https://pypi.org/project/invokelint/).

## Advantage

Testing Python packages across multiple versions and OSes requires per-repo boilerplate that drifts over time. This workflow centralises that logic for projects already using `Invoke Lint` so callers get:

- **Cross-platform coverage out of the box** — matrix across Ubuntu and Windows with a single call.
- **Invoke on Windows just works** — forces `cmd` shell automatically, because Invoke does not support PowerShell.
- **Python 3.7 legacy support when you need it** — opt-in job on Ubuntu 22.04 that falls back from `setup-uv` to `actions/setup-python` + the uv shell installer, working around uv's limited 3.7 support.

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

### Jobs

| Job | Runner | Condition |
| --- | --- | --- |
| `test` | `ubuntu-latest`, `windows-latest` × each version in `support-python-versions` | Always |
| `test_legacy_linux` | `ubuntu-22.04` with Python 3.7 | Only when `is-supporting-python-3-7-in-linux: true` |
