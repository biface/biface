# CI/CD Pipeline Documentation

← [Test automation](../automation.en.md)

## Introduction

This documentation explains in detail the GitHub Actions workflows (or simply "pipelines") used in my
projects. These pipelines automate the critical aspects of the development cycle: code quality, tests,
coverage, and build/publish.

**Note:** _Version française → [pipelines.fr.md](pipelines.fr.md)_

## Purpose

The pipelines described in this documentation serve several purposes:

- **Automation**: reduce manual intervention by automating repetitive tasks, [like testing](../automation.en.md),
  and build/publish
- **Quality assurance**: guarantee code quality and compatibility before any merge
- **Continuous integration**: automatically validate every change to catch problems early
- **Consistency**: maintain standardized processes across all branches
- **Reliability**: give developers fast feedback on the state of their changes

## Two architectures

The pipelines in this repository follow one of two architectures — see
[Two pipeline architectures](../architecture/ci-models.en.md) for the full conceptual detail and
comparison. In summary:

1. **[workflow-run/](workflow-run/pipelines.en.md)** — event-driven multi-file chaining
   (`workflow_run`). Each step (Quality, Tests, Coverage) lives in its own workflow file, triggered by the
   completion of the previous one.
2. **[needs/](needs/pipeline.en.md)** — single-file dependency chaining (`needs:`). All steps live in a
   single file, with two implementations documented side by side to show that it's the length of the
   chain — not the language — that varies from one project to another.

## How to use this documentation

Each pipeline is documented with:

- An overview of its purpose
- The trigger conditions (when it runs)
- A detailed step-by-step explanation of each action
- The expected results

Browse the two subfolders above to explore each architecture in detail. The `tox` configuration shared by
the Python pipelines is documented separately in [Tox configuration](../tests/python/tox.en.md).
