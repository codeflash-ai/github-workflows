# github-workflows

Shared reusable GitHub Actions workflows for [codeflash-ai](https://github.com/codeflash-ai) repositories.

## Available workflows

### `ci-python-uv.yml`

Reusable CI for Python projects using [uv](https://docs.astral.sh/uv/). Runs up to four parallel jobs (lint, typecheck, test, extra) -- each is optional.

**Inputs:**

| Input | Default | Description |
|---|---|---|
| `python-version` | `"3.12"` | Python version |
| `uv-version` | `"v6"` | `astral-sh/setup-uv` tag |
| `working-directory` | `"."` | Working directory (for monorepo subdirs) |
| `sync-command` | `"uv sync --all-packages"` | Dependency install command |
| `lint-command` | `""` | Lint command (empty to skip) |
| `typecheck-command` | `""` | Type-check command (empty to skip) |
| `test-command` | `""` | Test command (empty to skip) |
| `test-env` | `"{}"` | JSON object of extra env vars for test |
| `extra-command` | `""` | Additional check (empty to skip) |
| `extra-command-name` | `"extra"` | Display name for extra check |

**Example -- standalone repo:**

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  ci:
    uses: codeflash-ai/github-workflows/.github/workflows/ci-python-uv.yml@main
    with:
      lint-command: "uv run ruff check && uv run ruff format --check"
      typecheck-command: >-
        uv run interrogate src/ &&
        uv run mypy src/
      test-command: "uv run pytest tests/ -v"
      test-env: '{"CI": "true"}'
    secrets: inherit
```

**Example -- monorepo subdirectory:**

```yaml
name: AI Service CI

on:
  pull_request:
    paths: ["django/aiservice/**"]

jobs:
  ci:
    uses: codeflash-ai/github-workflows/.github/workflows/ci-python-uv.yml@main
    with:
      working-directory: "django/aiservice"
      sync-command: "uv sync"
      typecheck-command: "uv run mypy --non-interactive --config-file pyproject.toml @mypy_allowlist.txt"
      test-command: "uv run pytest"
    secrets: inherit
```

## Adding a new workflow

1. Create a new `.yml` file in `.github/workflows/`
2. Use `on: workflow_call` with typed inputs
3. Document it in this README
4. Callers reference it as `codeflash-ai/github-workflows/.github/workflows/<name>.yml@main`
