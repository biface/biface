# Tox Configuration — Reference Guide

← [Automation](../../automation.en.md)

## Overview

**Tox** is an automation tool that creates isolated virtual environments to run reproducible commands.
Combined with **tox-uv**, which replaces pip with uv for dependency installation, it forms the foundation
of the reference quality workflow.

The `tox.ini` file centralises the definition of all environments: tests, quality checks, formatting,
security, and complete workflows. A single source of truth, usable identically locally and in CI/CD.

> **Ready-to-copy template**: [`shared/tox.ini`](../../../shared/tox.ini) and
> [`shared/tox-config/`](../../../shared/tox-config/) in this repository. To adopt this configuration in a
> project: copy both, rename `tox-config/` to `.tox-config/` (with the leading dot), replace
> `<package_name>` with the actual package name in `tox.ini`, `coverage.sh.txt`/`test.sh.txt` (to be renamed
> to `.sh`), and adjust the Python versions in `envlist`, `[gh-actions]`, and `.tox-config/versions.txt`.

---

## The `.tox-config/` architecture

Configuration is separated from `tox.ini` into a dedicated directory. This allows Python versions or
dependencies to be changed without modifying `tox.ini`.

```text
.tox-config/
├── versions.txt          ← Python versions to test (one per line, # to comment out)
├── coverage-version.txt  ← Python version for the coverage report
├── requirements/
│   ├── base.txt          ← pytest, pytest-cov
│   ├── full.txt          ← all tools combined
│   ├── format.txt        ← black, isort
│   ├── linter.txt        ← flake8
│   ├── security.txt      ← bandit
│   └── type-check.txt   ← basedpyright
└── scripts/
    ├── test.sh           ← sequential multi-version execution
    └── coverage.sh       ← coverage report generation
```

---

## Section by section

### `[tox]` — General configuration

```ini
[tox]
minversion = 4.0
requires = tox-uv
toxworkdir = {toxinidir}/.tox
```

- `minversion = 4.0`: guarantees compatibility with the features used
- `requires = tox-uv`: installs the uv plugin automatically if absent
- `toxworkdir`: working directory for the virtual environments

### `[gh-actions]` — GitHub Actions integration

```ini
[gh-actions]
python =
    3.10: py310
    3.11: py311
    3.12: py312
    3.13: py313
    3.14: py314
```

Maps the Python versions from the GitHub Actions matrix to tox environments. When GitHub Actions uses
Python 3.12, tox automatically activates the `py312` environment.

### `[testenv]` — Default test environment

```ini
[testenv]
uv_seed = true
setenv =
    PYTHONPATH = {toxinidir}/src:{toxinidir}/tests
deps =
    -r {toxinidir}/.tox-config/requirements/base.txt
commands =
    pytest --import-mode=importlib --cov=<package_name> --cov-report=term-missing
```

- `uv_seed = true`: seeds the uv environment with pip — necessary for tools that call pip directly
- `PYTHONPATH`: lets imports work without manipulating `sys.path`
- `--import-mode=importlib`: modern import mode, avoids a class of silent bugs related to module resolution

This environment applies to every Python environment (`py310`, `py311`, etc.) that has no specific
configuration of its own.

---

## Verification environments

### `basedpyright` — Type checking

```ini
[testenv:basedpyright]
deps =
    -r {toxinidir}/.tox-config/requirements/type-check.txt
allowlist_externals = bash
commands =
    bash -c "basedpyright src; code=$?; [ $code -le 1 ]"
```

**basedpyright** is a strict static type checker, based on Microsoft's Pyright engine. Stricter than mypy,
it detects type mismatches, non-existent attributes, and incorrect calls.

The bash wrapper is necessary because basedpyright distinguishes two output levels:
- Code `0`: no issues
- Code `1`: warnings (non-blocking)
- Code `2`: errors (blocking)

`[ $code -le 1 ]` tolerates warnings while still blocking on errors.

### `flake8` — Lint

```ini
[testenv:flake8]
commands =
    flake8 src tests
```

Checks style conventions not covered by black: cyclomatic complexity, unused imports, unused variables.
Modifies nothing.

### `black-check` and `isort-check` — Formatting verification

```ini
commands =
    black --check --diff --target-version py310 src tests
    isort --check-only --diff src tests
```

Verification-only modes (`--check`, `--check-only`). Used in CI to fail if the code isn't correctly
formatted, without modifying files.

### `bandit` — Security

```ini
commands =
    bandit -r src
```

Scans `src/` for dangerous patterns: calls to `eval`, use of deprecated cryptography modules, potential
injections. The analysis doesn't cover `tests/`.

---

## Auto-fix environments

### `black` and `isort` — Automatic formatting

```ini
commands =
    black --target-version py310 src tests
    isort src tests
```

Modify files in place. Use locally, never in CI.

### `format` — Complete formatting in one command

```ini
commands =
    black --target-version py310 src tests
    isort src tests
```

Convenient alias to apply black and isort in a single invocation.

### `coverage` — Coverage report

```ini
[testenv:coverage]
basepython = python3.12
commands =
    pytest --import-mode=importlib \
    --cov=<package_name> \
    --cov-append \
    --cov-report=term-missing \
    --cov-report=xml:{toxinidir}/coverage/coverage.xml \
    --cov-report=html:{toxinidir}/coverage/coverage_html tests
```

Generates three report formats:
- Terminal: uncovered lines visible immediately
- XML (`coverage/coverage.xml`): for CI tools (Codecov, SonarQube)
- HTML (`coverage/coverage_html/`): for interactive local reading

`--cov-append` accumulates results when several runs are chained.

---

## Complete workflows

### `pre-push` — Local quality gate

```ini
[testenv:pre-push]
commands =
    black --target-version py310 src tests   # 1. formatting
    isort src tests
    bash -c "basedpyright src; ..."          # 2. types
    flake8 src tests                          # 3. lint
    bandit -r src                             # 4. security
    bash .tox-config/scripts/test.sh         # 5. multi-version tests
    bash .tox-config/scripts/coverage.sh     # 6. coverage
```

The order is deliberate: formatting comes first to avoid flake8 false positives. On failure at any step,
execution stops.

```bash
tox -e pre-push
```

### `ci-quality` — CI verification

Same logic as `pre-push` but without automatic formatting: uses `--check` and `--check-only`. Designed to
fail if submitted code isn't compliant.

```bash
tox -e ci-quality
```

### `ci-tests` — CI tests

Runs pytest with coverage only. Used by the GitHub Actions matrix to test each Python version in
parallel.

```bash
tox -e ci-tests
```

---

## Tool configuration

### flake8

```ini
[flake8]
max-line-length = 88
extend-ignore = E203, W503, E501
```

- `88` characters: black's standard
- `E203`, `W503`, `E501`: rules incompatible with black's style, ignored

### isort

```ini
[isort]
profile = black
line_length = 88
known_first_party = <package_name>
multi_line_output = 3
include_trailing_comma = true
```

The `black` profile guarantees compatibility between the two tools. `known_first_party` should be adapted
to your package's name.

---

## Reference commands

```bash
tox -e pre-push          # full local workflow
tox -e py312             # tests on Python 3.12 only
tox -e format            # automatic formatting
tox -e check             # quick check, no modification
tox -e ci-quality        # CI verification
tox -e ci-tests          # CI tests
tox -e coverage          # coverage report only
```

---

## See also

- [Configuration tox — version française](tox.fr.md)
- [test.sh script](../shell/tox-uv-test-script.en.md)
- [coverage.sh script](../shell/tox-uv-coverage-script.en.md)
- [Wiki — Unit tests](https://gitlab.com/biface/biface/-/wikis/en/controlled-delivery-software/test-management/testing)
