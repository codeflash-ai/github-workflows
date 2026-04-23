# github-workflows

Shared reusable GitHub Actions workflows for [codeflash-ai](https://github.com/codeflash-ai) repositories.

## Available workflows

### `ci-python-uv.yml`

Reusable CI for Python projects using [uv](https://docs.astral.sh/uv/). Runs up to four parallel jobs (lint, typecheck, test, extra) -- each is optional.

**Inputs:**

| Input | Default | Description |
|---|---|---|
| `python-version` | `"3.12"` | Python version |
| `working-directory` | `"."` | Working directory (for monorepo subdirs) |
| `sync-command` | `"uv sync --all-packages"` | Dependency install command |
| `lint-command` | `""` | Lint command (empty to skip) |
| `typecheck-command` | `""` | Type-check command (empty to skip) |
| `test-command` | `""` | Test command (empty to skip) |
| `test-env` | `"{}"` | JSON object of extra env vars for test |
| `extra-command` | `""` | Additional check (empty to skip) |
| `extra-command-name` | `"extra"` | Display name for extra check |
| `test-python-versions` | `""` | JSON array of Python versions for the test matrix (overrides `python-version` for test job) |
| `test-os` | `'["ubuntu-latest"]'` | JSON array of runner OSes for the test matrix |
| `test-fail-fast` | `true` | Stop remaining test matrix jobs on first failure |

> **Secrets:** For jobs that need secrets as env vars, keep those as local jobs in your caller workflow and use this reusable workflow for secret-free jobs.

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

**Example -- multi-version test matrix:**

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  ci:
    uses: codeflash-ai/github-workflows/.github/workflows/ci-python-uv.yml@main
    with:
      sync-command: "uv sync"
      test-command: "uv run pytest tests/"
      test-python-versions: '["3.9", "3.10", "3.11", "3.12"]'
      test-fail-fast: false
```

**Example -- monorepo hybrid pattern (shared workflow + local job for secrets):**

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

  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: django/aiservice
    steps:
      - uses: actions/checkout@v6
      - uses: astral-sh/setup-uv@v8.0.0
        with:
          python-version: "3.12"
          enable-cache: true
      - run: uv sync
      - name: Test
        run: uv run pytest
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

### `tessl-update.yml`

Reusable workflow for keeping [tessl](https://tessl.io) tiles up to date. Updates existing tiles and attempts to install tiles listed in `.tessl/missing-tiles.txt`. Opens a PR when changes are detected.

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `TESSL_TOKEN` | Yes | Tessl API token for the workspace |
| `CI_BOT_APP_ID` | Yes | GitHub App ID for `codeflash-ci-bot` |
| `CI_BOT_PRIVATE_KEY` | Yes | Private key for `codeflash-ci-bot` |

**Missing tiles file (`.tessl/missing-tiles.txt`):**

One tile per line, `#` comments supported. Successfully installed tiles are removed automatically. When the file is empty it gets deleted.

```
# PyPI tiles not yet in the registry
tessl/pypi-tree-sitter-javascript
tessl/pypi-ruff
```

**Example:**

```yaml
name: Tessl Tile Updates

on:
  schedule:
    - cron: "0 9 * * 1"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  tessl:
    uses: codeflash-ai/github-workflows/.github/workflows/tessl-update.yml@main
    secrets:
      TESSL_TOKEN: ${{ secrets.TESSL_TOKEN }}
      CI_BOT_APP_ID: ${{ secrets.CI_BOT_APP_ID }}
      CI_BOT_PRIVATE_KEY: ${{ secrets.CI_BOT_PRIVATE_KEY }}
```

---

## Adding a new workflow

1. Create a new `.yml` file in `.github/workflows/`
2. Use `on: workflow_call` with typed inputs
3. Document it in this README
4. Callers reference it as `codeflash-ai/github-workflows/.github/workflows/<name>.yml@main`
