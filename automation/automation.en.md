# Test automation

← [README](../README.md) | _Version française → [automation.fr.md](automation.fr.md)_

## Overview

Test automation means using software tools to run tests automatically, giving fast feedback on code
quality without manual intervention. This automation is essential in modern development workflows,
particularly in CI/CD pipelines where every code change needs to be validated quickly and reliably.

## Test-Driven Development (TDD)

**Test-Driven Development** is a methodology that places automated tests at the very core of the
development process. Rather than writing tests after the code, TDD reverses that relationship.

### The TDD cycle: Red-Green-Refactor

```mermaid
flowchart LR
    P(["Write a failing test<br>(Red)"]) --> n1(["Write the code<br>(Green)"]) --> n2(["Refactor<br>(Refactor)"])
```

1. **Red**: write a test that fails, defining the expected behaviour
2. **Green**: write the minimal code needed to make the test pass
3. **Refactor**: improve the code while keeping the tests green

### Why TDD benefits automation

- **Natural coverage**: every line of production code answers to a test — coverage naturally reaches high
  levels, with no forgotten case
- **Better design**: writing the test first forces thinking about interfaces and APIs before the
  implementation
- **Living documentation**: tests describe what the code must do, always up to date
- **Confidence in refactoring**: a complete test suite immediately catches regressions
- **Faster debugging**: failures are caught at the very moment the code is written

## Unit tests and integration tests

### Unit tests — the foundation

Test an isolated component (function, method, class), run in milliseconds, need no external dependency.
Fast, precise in locating a failure, parallelisable.

### Integration tests — interactions between components

Verify that several components work correctly together: data flow between modules, respected API
contracts. Slower than unit tests, faster than end-to-end tests.

```text
Code change
    ↓
Commit & Push
    ↓
CI pipeline triggered
    ↓
1. Quality (seconds)        ────→ immediate feedback
    ↓
2. Unit tests (minutes)     ────→ component validation
    ↓
3. Coverage report          ────→ quality indicator
    ↓
Success/failure notification
```

---

## Two pipeline architectures

The GitHub Actions pipelines documented in this repository follow one of two architectures —
**event-driven multi-file chaining** (`workflow_run`) or **single-file dependency chaining** (`needs:`).
The choice between the two isn't tied to the project's language: both models coexist on the Python side
as well as the Rust side. The detail of both architectures, with their respective advantages and
limitations, is explained in [Two pipeline architectures](architecture/ci-models.en.md).

---

## Structure of this repository

```text
biface/biface
├── shared/                          ← reusable configuration files
│   ├── tox.ini                      ← reference tox configuration (Python)
│   ├── tox-config/                  ← tox usage configuration
│   │   ├── requirements/
│   │   ├── versions.txt
│   │   ├── coverage-version.txt
│   │   └── scripts/
│   │       ├── test.sh
│   │       └── coverage.sh
│   ├── github-ci/                   ← reference GitHub Actions workflows
│   │   ├── workflow-run/
│   │   │   ├── python-ci-quality.yaml
│   │   │   └── python-ci-tests.yaml
│   │   ├── needs/
│   │   │   ├── python-ci.yaml
│   │   │   └── rust-ci.yml
│   │   └── coverage/
│   │       ├── python-ci-coverage.yaml
│   │       └── rust-coverage.yml
│   └── issue-templates/
│       └── github/                  ← GitHub issue templates
└── automation/
    ├── automation.en.md             ← this page
    ├── architecture/
    │   └── ci-models.en.md          ← the two pipeline architectures, independent of language
    ├── pipelines/
    │   ├── pipelines.en.md          ← overview of GitHub Actions pipelines
    │   ├── workflow-run/            ← model 1: event-driven multi-file chaining
    │   │   ├── quality.en.md
    │   │   ├── tests.en.md
    │   │   └── test-uv.en.md        ← reinforced model (.tox-config), kept as an example
    │   └── needs/                   ← model 2: single-file dependency chaining
    │       └── pipeline.en.md
    ├── coverage/
    │   ├── python/
    │   │   └── coverage.en.md       ← coverage pipeline, workflow_run model
    │   └── rust/
    │       └── coverage.en.md       ← coverage pipeline, independent workflow
    └── tests/
        ├── python/
        │   └── tox.en.md           ← tox configuration explained
        └── shell/
            ├── tox-uv-test-script.en.md      ← test.sh script explained
            └── tox-uv-coverage-script.en.md  ← coverage.sh script explained
```

## How to use this repository

The files under `shared/` are **real, ready-to-copy templates** — not documentation to transcribe by hand:

- `tox.ini` + `tox-config/`: reference tox configuration for a Python project — see
  [Tox configuration](tests/python/tox.en.md) for the full procedure (copying, adapting the package name,
  the versions).
- `github-ci/`: the GitHub Actions workflows themselves, one per architecture and per implementation
  (`workflow-run/`, `needs/`, `coverage/`) — each page in [pipelines/](pipelines/pipelines.en.md) explains
  the pipeline in detail **and** links to the corresponding real file to copy into `.github/workflows/`.

Every file under `github-ci/` carries a header comment block listing precisely what to adapt for the
target project (package name, versions, Cargo features…), on the same principle as `tox.ini`.

---

## Validated on

| Project | Registry | CI model |
| --- | --- | --- |
| [ndt](https://github.com/biface/ndt) | [PyPI](https://pypi.org/project/ndict-tools/) | `workflow_run` |
| [i18n](https://github.com/biface/i18n) | [PyPI](https://pypi.org/project/pyi18t-tools/) | `needs:` |
| [oxiflow](https://github.com/biface/oxiflow) | [crates.io](https://crates.io/crates/oxiflow) | `needs:` |
| [sds](https://github.com/skyfrigate/sds) | — | — |

---

## Wiki documentation

The wiki explains the **why** and the **what**; this repository shows the **how**.

- [Testing — Overview](https://gitlab.com/biface/biface/-/wikis/en/controlled-delivery-software/test-management)
- [Unit tests](https://gitlab.com/biface/biface/-/wikis/en/controlled-delivery-software/test-management/testing)
- [Code coverage](https://gitlab.com/biface/biface/-/wikis/en/controlled-delivery-software/test-management/coverage)

---

## License

[CC BY-NC 4.0](../LICENSE.txt) — documentation and configuration files.
