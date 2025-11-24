# Python Code Coverage Pipeline (CI)

```yaml
name: Python CI - Coverage
```

The name of the pipeline in an automated job chaining objective is important, as
it will be used to trigger or not trigger another workflow.

## Overview

This GitHub Actions pipeline measures your Python project's code coverage and
automatically uploads reports to Codecov. It runs **after** the tests pipeline
and only if it succeeds.

## Pipeline Triggers

This pipeline uses a **cascading trigger mechanism** (`workflow_run`), making it
dependent on another pipeline:

```yaml
on:
  workflow_run:
    workflows: ["Python CI - Tests"]
    types:
      - completed
    branches:
      - "updates/**"     # Toutes les branches de versions (updates/1.0.0, updates/1.1.0, etc.)
      - "staging/**"     # Toutes les branches de staging (staging/1.0.x, staging/1.1.x, etc.)
      - main
      - master
```

### Trigger Conditions

1. **Parent pipeline**: Triggers only after the "Python CI - Tests" workflow
   completes
2. **Required status**: The tests pipeline must have succeeded
   (`conclusion == 'success'`)
3. **Concerned branches**:
   - `updates/**`: All version branches (e.g., `updates/1.0.0`, `updates/1.1.0`)
   - `staging/**`: All staging branches (e.g., `staging/1.0.x`, `staging/1.1.x`)
   - `main`: Main branch
   - `master`: Alternative main branch

**Important**: This pipeline **never** triggers directly. It always waits for
the tests pipeline to complete successfully on one of the specified branches.

## Pipeline Architecture

```yaml
jobs:
  coverage:
    name: Upload coverage to Codecov
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

### Execution Condition

```yaml
if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

The `coverage` job only runs if the parent workflow (tests) has completed
successfully. If the tests fail, this pipeline is completely ignored. This
approach optimises the resources consumed in workflow automation. In this case,
there is **no point** in triggering code coverage processing on the GitHub
repository if the tests fail. In this scenario, it is better to return to the
local branches to analyse and process the errors published from the previous
workflow. Experience has shown that, particularly with code completion and
production assistance mechanisms, integrated development environments can add
unwanted directives

### Execution Environment

- **Operating System**: Ubuntu (latest available version)
- **Python Version**: 3.12 (single version, unlike the tests pipeline)
- **Runner**: Virtual machine provided by GitHub Actions

## Detailed Pipeline Steps

### Step 1: Source Code Checkout

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```

Clones the Git repository into the execution environment to access source code
and configuration files.

### Step 2: Python Setup

```yaml
- name: Setup Python
  uses: actions/setup-python@v4
  with:
    python-version: "3.12"
```

Installs Python 3.12 specifically. Unlike the tests pipeline which uses a
matrix, only a single version is needed here to generate the coverage report. We
might wonder why tests need to be repeated when they have already been performed
previously: in fact, as the processes are separate, the tests are no longer
available when this process is launched.

### Step 3: Dependencies Installation

```yaml
- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install tox
```

Prepares the environment by:

1. Upgrading `pip` to the latest version
2. Installing `tox` to manage test execution with coverage measurement

### Step 4: Running Tests with Coverage

```yaml
- name: Run tests with coverage
  run: tox -e gh-ci
```

Runs the tests using the same `gh-ci` environment as the tests pipeline, but
this time also generating code coverage data (percentage of code tested).

### Step 5: Upload to Codecov

```yaml
- name: Upload coverage reports to Codecov
  if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master'
  uses: codecov/codecov-action@v5
  with:
    token: ${{ secrets.CODECOV_TOKEN }}
    fail_ci_if_error: true
```

This final step uploads the coverage report to Codecov, but **only** for main
branches (`main` or `master`).

**Important parameters**:

- `token`: Uses a secret token stored in GitHub secrets to authenticate with
  Codecov
- `fail_ci_if_error: true`: If the upload fails, the entire pipeline fails as
  well, ensuring we detect reporting issues

**Branch-specific behavior**:

- On `main` or `master`: Report is uploaded to Codecov
- On `updates/**` or `staging/**`: Tests are executed with coverage, but the
  report is **not** uploaded (to avoid polluting Codecov with temporary
  branches)

## Complete Workflow

```plaintext
1. Code push to a concerned branch
   ↓
2. "Python CI - Tests" pipeline triggers and executes
   ↓
3. If tests succeed → "Coverage" pipeline triggers
   ↓
4. Tests execution with coverage measurement
   ↓
5. If branch = main/master → Upload to Codecov
```

## Result

- **Success** ✅: Tests pass successfully and coverage report is generated (and
  uploaded if on `main`/`master`)
- **Failure** ❌: Either tests fail or Codecov upload fails (only on
  `main`/`master`)
