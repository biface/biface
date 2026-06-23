# Python Quality Pipeline (CI)

```yaml
name: Python CI - Quality
```

← [Pipelines](./pipelines.en.md)

## Overview

This GitHub Actions pipeline is the **first link** in the CI/CD chain. It checks that the code meets basic
rules — typing, lint, formatting, security — **before** any other pipeline triggers. The
[Tests](./tests.en.md) pipeline only runs if this one succeeds: there's no point running a test suite on
code that wouldn't even pass a style check.

## Pipeline trigger

```yaml
on:
  push:
    branches:
      - '**'
  pull_request:
    branches:
      - '**'
```

Unlike the pipelines further down the chain, this one triggers directly on **every push and every pull
request, on every branch** — it's the sole entry point of the CI chain. Every subsequent pipeline (Tests,
Coverage) triggers in cascade (`workflow_run`), not on a direct Git event.

## Pipeline architecture

```yaml
jobs:
  quality:
    name: Code Quality Checks
    runs-on: ubuntu-latest
```

A single job, which sequentially runs the five checks defined in the `ci-quality` tox environment — see
[Tox configuration](../tests/python/tox.en.md#ci-quality--ci-verification) for the detail of each tool.

## Detailed pipeline steps

### Step 1: Checking out the source code

```yaml
- name: Checkout code
  uses: actions/checkout@v6
```

No particular `ref` parameter here — this pipeline triggers directly on the Git event, so it naturally
checks out the right commit. It's only the following pipelines, triggered by `workflow_run`, that need to
explicitly specify which commit to check out.

### Step 2: Setting up Python

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.12'
```

A single Python version — the quality check (typing, lint, formatting) doesn't need to be repeated across
every supported version: the source code is the same regardless of which version will later run it.

### Step 3: Installing uv and tox

```yaml
- name: Install uv
  uses: astral-sh/setup-uv@v4

- name: Install tox
  run: |
    uv pip install --system tox tox-uv
```

`uv` replaces `pip` for dependency installation — faster, especially for a pipeline run frequently (on every
push). `tox-uv` is the plugin that lets `tox` delegate its installs to `uv` rather than `pip`.

### Step 4: Running the quality checks

```yaml
- name: Run quality checks
  run: |
    tox -e ci-quality
```

`tox -e ci-quality` runs, in this order, basedpyright (typing), flake8 (lint), black and isort in
verification-only mode (formatting), bandit (security) — see
[Tox configuration](../tests/python/tox.en.md#verification-environments) for the detail of each tool. No
automatic fix is applied here: this is a check, not a repair. If the code isn't compliant, the pipeline
fails and reports precisely which tool failed.

### Step 5: Keeping the report

```yaml
- name: Upload quality report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: quality-report
    path: |
      .tox/ci-quality/log/
    retention-days: 30
```

`if: always()`: this report is kept **even on failure** — that's precisely when the pipeline has failed
that the log detail is needed to understand why. Kept for 30 days, then automatically deleted by GitHub.

## Result

If the five checks pass, the pipeline is marked **successful** ✅ and automatically triggers the
[Tests](./tests.en.md) pipeline via `workflow_run`. If any of the five fails, the pipeline is marked
**failed** ❌ — and no following pipeline triggers: tests, coverage, and publishing stay inactive until the
code is fixed.

## See also

- [Tests pipeline](./tests.en.md) — triggered by this one
- [Tox configuration — ci-quality environment](../tests/python/tox.en.md)
- [Quality pipeline — version française](./quality.fr.md)
