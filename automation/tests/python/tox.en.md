# Tox Configuration - Complete Guide

## What is Tox?

**Tox** is a Python test automation tool that allows you to:

- Create **isolated virtual environments** for each test configuration
- Test your code across **multiple Python versions** simultaneously
- Execute **standardized command suites** (tests, linting, formatting)
- Guarantee test **reproducibility** between local development and CI/CD

### Why Use Tox?

#### For Local Testing

- **Complete isolation**: Each test environment is independent, avoiding
  dependency conflicts
- **Consistency**: You test in exactly the same conditions as CI/CD
- **Time-saving**: A single command (`tox`) to run all tests and checks
- **Multi-version**: Easily test your code on Python 3.9, 3.10, 3.11, and 3.12
  locally

#### For Remote Repository (CI/CD)

- **Standardization**: The same Tox commands work locally and in GitHub Actions
- **Simplified maintenance**: Modifying a test = modifying only `tox.ini`, not
  GitHub workflows
- **Traceability**: Developers can reproduce CI tests exactly locally
- **Flexibility**: Different Tox environments for different needs (tests,
  quality, security)

---

## General Configuration

```ini
[tox]
minversion = 3.24.5
envlist = py39, py310, py311, py312
```

- **minversion**: Minimum required Tox version (3.24.5)
- **envlist**: List of Python environments to test by default (Python 3.9 to
  3.12)

### GitHub Actions Integration

```ini
[gh-actions]
python =
    3.9: py39
    3.10: py310
    3.11: py311
    3.12: py312
```

This section **maps** GitHub Actions Python versions to Tox environments. When
GitHub Actions uses Python 3.10, Tox automatically activates the `py310`
environment.

---

## Main Test Environment

### [testenv] - Unit Tests with Coverage

```ini
[testenv]
description = Run unit tests with coverage
```

This default environment applies to all Python environments (py39, py310, py311,
py312).

#### PYTHONPATH Configuration

```ini
setenv =
    PYTHONPATH = {toxinidir}/src:{toxinidir}/tests
```

Sets the Python path for imports to work correctly:

- `{toxinidir}`: Project root directory
- Adds `src` and `tests` to PYTHONPATH

#### Dependencies

```ini
deps =
    pytest
    pytest-cov
```

Installs the necessary tools for:

- **pytest**: Testing framework
- **pytest-cov**: Code coverage measurement plugin

#### Execution Commands

```ini
commands =
    pytest --import-mode=importlib \
           --cov=ndict_tools \
           --cov-report=term-missing \
           --cov-report=xml:coverage/coverage.xml \
           --cov-report=html:coverage/coverage_html tests
```

Runs pytest with:

- `--import-mode=importlib`: Modern and recommended import mode
- `--cov=ndict_tools`: Measures coverage of the `ndict_tools` package
- `--cov-report=term-missing`: Displays uncovered lines in terminal
- `--cov-report=xml`: Generates XML report (used by Codecov)
- `--cov-report=html`: Generates interactive HTML report

---

## Individual Verification Tools

### [testenv:mypy] - Type Checking

```ini
[testenv:mypy]
description = Type checking with mypy
```

Checks Python type annotations consistency to detect typing errors before
execution.

### [testenv:flake8] - Linting

```ini
[testenv:flake8]
description = Lint the code with flake8
```

Analyzes code to detect:

- Syntax errors
- PEP 8 convention violations
- Excessive complexity
- Dead or unused code

### [testenv:black-check] - Format Checking

```ini
[testenv:black-check]
description = Check code formatting with black (no changes)
commands =
    black --check --diff src tests
```

Checks if code respects Black formatting **without modifying files**:

- `--check`: Verification mode only
- `--diff`: Shows differences if formatting is incorrect

### [testenv:isort-check] - Import Checking

```ini
[testenv:isort-check]
description = Check import sorting with isort (no changes)
```

Checks that imports are sorted according to conventions **without modifying
files**.

### [testenv:bandit] - Security Analysis

```ini
[testenv:bandit]
description = Run security analysis with bandit
commands =
    bandit -r src
```

Scans code for common security vulnerabilities:

- Use of dangerous functions
- Cryptographic security issues
- Potential injections
- `-r`: Recursive analysis

---

## Automatic Correction Tools

### [testenv:black] - Automatic Formatting

```ini
[testenv:black]
description = Auto-format code with black
commands =
    black src tests
```

Automatically reformats code according to Black style (without `--check`).

### [testenv:isort] - Automatic Import Sorting

```ini
[testenv:isort]
description = Auto-sort imports with isort
```

Automatically sorts imports according to conventions.

### [testenv:format] - Complete Formatting

```ini
[testenv:format]
description = Auto-format code (black + isort)
commands =
    black src tests
    isort src tests
```

Combined environment that applies **both** Black and isort for complete
formatting.

---

## Complete Workflows

### [testenv:pre-push] - Local Pre-Push Workflow

```ini
[testenv:pre-push]
description = Complete local workflow before push (format + checks + tests)
deps =
    -r requirements.test.txt
    mypy
commands =
    # 1. Auto-formatting
    black src tests
    isort src tests

    # 2. Type checking
    mypy src

    # 3. Linting
    flake8 src tests

    # 4. Security
    bandit -r src

    # 5. Tests with coverage
    pytest --import-mode=importlib \
           --cov=ndict_tools \
           --cov-report=term-missing \
           --cov-report=xml:coverage/coverage.xml \
           --cov-report=html:coverage/coverage_html tests
```

**Complete** environment to execute before pushing code. It combines:

1. **Auto-formatting**: Black + isort
2. **Type checking**: mypy
3. **Linting**: flake8
4. **Security**: bandit
5. **Tests with coverage**: pytest with the production conditions of the various
   elements seen above.

**Usage**: `tox -e pre-push` before each `git push` to guarantee quality.

### [testenv:ci-quality] - CI Checks (No Modifications)

```ini
[testenv:ci-quality]
description = CI quality checks (no auto-fix, only verify)
deps =
    -r requirements.test.txt
    mypy
commands =
    # Vérifications sans modification
    mypy src
    flake8 src tests
    black --check --diff src tests
    isort --check-only --diff src tests
    bandit -r src
```

CI/CD version that **checks** without modifying:

- Uses `black --check` instead of `black`
- Uses `isort --check-only` instead of `isort`

Perfect for a CI pipeline that should fail if code is not compliant.

### [testenv:ci-tests] - CI Tests

```ini
[testenv:ci-tests]
description = CI tests with coverage (used by gh-actions matrix)
deps =
    -r requirements.test.txt
commands =
    pytest --import-mode=importlib \
           --cov=ndict_tools \
           --cov-report=term-missing \
           --cov-report=xml:coverage/coverage.xml \
           --cov-report=html:coverage/coverage_html tests
```

Simplified environment used by GitHub Actions matrix. Executes only tests with
coverage, without quality checks.

---

## Convenient Aliases

### [testenv:check] - Quick Check

```ini
[testenv:check]
description = Quick check (no formatting, only verify)
```

Quick check without automatic formatting:

- mypy
- flake8
- black --check
- isort --check-only

**Usage**: `tox -e check` for a quick check without modifying files.

### [testenv:local] - Compatibility Alias

```ini
[testenv:local]
description = Alias pour pre-push (rétrocompatibilité)
```

Simple alias to `pre-push` to maintain compatibility with old habits.

---

## Tool Configuration

### Flake8

```ini
[flake8]
max-line-length = 88
extend-ignore = E203, W503, E501
```

- **max-line-length**: 88 characters (Black standard)
- **extend-ignore**: Ignores rules incompatible with Black
  - E203: Whitespace before ':'
  - W503: Line break before binary operator
  - E501: Line too long (handled by Black)

### isort

```ini
[isort]
profile = black
line_length = 88
```

Black-compatible configuration:

- **profile = black**: Uses Black profile
- **line_length = 88**: Consistent with Black

### mypy

```ini
[mypy]
python_version = 3.9
warn_return_any = true
check_untyped_defs = true
ignore_missing_imports = true
```

Type checking configuration:

- Targets Python 3.9 as minimum version
- Enables warnings on incomplete types
- Ignores imports without type stubs

---

## Usage Examples

### Local Testing

```plaintext
# Test on all Python versions
tox

# Test on Python 3.12 only
tox -e py312

# Complete workflow before push
tox -e pre-push

# Quick check
tox -e check

# Automatic formatting
tox -e format
```

### In GitHub Actions

```yaml
# The tests pipeline uses
run: tox -e ci-tests
```

```yaml
# A quality pipeline could use
run: tox -e ci-quality
```

---

## Configuration Benefits

1. **Separated environments**: Tests, quality, security are isolated
2. **Flexibility**: Choose environment according to need
3. **Local/remote consistency**: Same commands everywhere
4. **Simplified maintenance**: Single file to maintain
5. **Living documentation**: Descriptions explain each environment
