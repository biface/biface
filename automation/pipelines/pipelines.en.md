# CI/CD Pipelines Documentation

## Introduction

This documentation explains in detail the GitHub Actions workflows (or simply "pipelines") used in my
projects. These pipelines automate the critical aspects of the development cycle: code quality, tests,
coverage, and — eventually — deployment.

**Note:** _Version française → [pipelines.fr.md](pipelines.fr.md)_

## Purpose

The pipelines described in this documentation serve several purposes:

- **Automation**: reduce manual intervention by automating repetitive tasks, [like testing](../automation.en.md),
  and eventually deployment
- **Quality assurance**: guarantee code quality and compatibility before any merge
- **Continuous integration**: automatically validate every change to catch problems early
- **Consistency**: maintain standardised processes across every branch
- **Reliability**: give developers fast feedback on the state of their changes

## Pipeline overview

The following pipelines are documented in this repository, in the order they chain together:

1. **Python CI - Quality** → [quality.en.md](quality.en.md) — type checking, lint, formatting, and
   security checks, the first link in the chain; every following pipeline depends on it
2. **Python CI - Tests** → [tests.en.md](tests.en.md) — runs the test suite across several Python
   versions, only triggered if Quality succeeded
3. **Python CI - Coverage** → [../coverage/python/coverage.en.md](../coverage/python/coverage.en.md) —
   measures code coverage and publishes it to Codecov, only triggered on `staging/**` and the main branch

_(Build and publishing pipelines will be documented in a future iteration.)_

## How to use this documentation

Each pipeline is documented with:

- An overview of its purpose
- Trigger conditions (when it runs)
- A detailed, step-by-step explanation of each action
- Expected outcomes

Browse the pages above to explore each pipeline in detail. The `tox` configuration shared by these three
pipelines is documented separately in
[Tox configuration](../tests/python/tox.en.md).
