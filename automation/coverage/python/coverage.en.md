# Python Code Coverage Pipeline (CI)

```yaml
name: Python CI - Coverage
```

← [Pipelines](../../pipelines/pipelines.en.md) | ← [Tests pipeline](../../pipelines/tests.en.md)

The pipeline name matters for the automated chaining — it's the one referenced in the `workflows:` of the
`workflow_run` trigger that follows in the chain.

## Overview

This GitHub Actions pipeline measures code coverage and automatically uploads the report to Codecov. It
runs **after** the [Tests](../../pipelines/tests.en.md) pipeline, and only if that one succeeded — and only
on branches where coverage makes sense for the release decision.

## Pipeline trigger

```yaml
on:
  workflow_run:
    workflows: ["Python CI - Tests"]
    types:
      - completed
    branches:
      - "staging/**"   # release candidates
      - master         # production
```

### Trigger conditions

1. **Parent pipeline**: only triggers after the full run of the "Python CI - Tests" pipeline
2. **Required status**: the test pipeline must have succeeded (`conclusion == 'success'`)
3. **Branches involved**: only `staging/**` (release preparation) and the main branch (production)

**What doesn't trigger this pipeline**: `updates/X.Y.0` and `feature/*` branches. On these branches, the
code is still under development — a coverage measurement there would be partial and worthless for the
release decision. See
[Integration tests](https://gitlab.com/biface/biface/-/wikis/en/controlled-delivery-software/test-management/integrate)
on the wiki side for the general logic of restricting expensive steps to the right moments in the pipeline.

## Pipeline architecture

```yaml
jobs:
  coverage:
    name: Upload coverage to Codecov
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

### Execution environment

- **Python version**: 3.12 only (unlike the test pipeline, which covers five versions) — coverage is
  measured on a stable reference version, not across the whole matrix. See
  [Tox configuration § coverage](../../tests/python/tox.en.md#coverage--coverage-report) for the full
  rationale.

## Detailed pipeline steps

### Step 1: Checking out the source code

```yaml
- name: Checkout repository
  uses: actions/checkout@v6
  with:
    ref: ${{ github.event.workflow_run.head_sha }}
```

Same necessity as for the Tests pipeline: this pipeline is triggered by `workflow_run`, so it must
explicitly specify which commit to check out — `head_sha`, the exact commit that triggered the Tests
pipeline, not just the branch name.

### Step 2: Setting up Python

```yaml
- name: Setup Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.12"
```

A single, fixed version — see the rationale above.

### Step 3: Installing uv and tox

```yaml
- name: Install uv
  uses: astral-sh/setup-uv@v4

- name: Install tox
  run: uv pip install --system tox tox-uv
```

Identical to the Quality and Tests pipelines.

### Step 4: Running the tests with coverage

```yaml
- name: Run tests with coverage
  run: tox -e coverage
```

Reuses the `coverage` tox environment — see
[Tox configuration § coverage](../../tests/python/tox.en.md#coverage--coverage-report). Important: this
pipeline **re-runs** the tests, it doesn't reuse the results of the previous Tests pipeline — each GitHub
Actions job runs in an isolated, sealed environment, results from one job aren't directly accessible to
another without an explicit artifact-transfer step.

### Step 5: Uploading to Codecov

```yaml
- name: Upload coverage reports to Codecov
  uses: codecov/codecov-action@v5
  with:
    token: ${{ secrets.CODECOV_TOKEN }}
    fail_ci_if_error: true
```

No additional condition on this step: the restriction to `staging/**`/`main` branches is already set at the
pipeline trigger level (`on.workflow_run.branches`) — no need to repeat it here.

`fail_ci_if_error: true`: an upload failure fails the entire pipeline. A missed coverage threshold or
Codecov unavailability is therefore a blocking signal, not a silently ignored alert.

## Complete workflow

```text
1. Push to a staging/** or main branch
   ↓
2. "Python CI - Quality" pipeline — must succeed
   ↓
3. "Python CI - Tests" pipeline — must succeed
   ↓
4. "Python CI - Coverage" pipeline triggers
   ↓
5. Upload to Codecov
```

## Result

- **Success** ✅: tests pass and the report is uploaded to Codecov
- **Failure** ❌: either the tests fail in this run, or the Codecov upload fails

## See also

- [Tests pipeline](../../pipelines/tests.en.md) — trigger of this one
- [Tox configuration § coverage](../../tests/python/tox.en.md)
- [Coverage pipeline — version française](./coverage.fr.md)
