# CI/CD Pipelines Documentation

## Introduction

This documentation provides a comprehensive explanation of the GitHub Actions Workflows (or simply pipelines)
implemented and used in my projects. These pipelines automate critical aspects of the development workflow, including
testing, code quality checks, and deployment processes.

**Remarque :** _Les explications détaillées sont disponibles en anglais et en français._<br>
**Note:** _Detailed explanations are available in both English and French._

## Purpose

The pipelines described in this documentation serve several key objectives:

- **Automation**: Reduce manual intervention by automating repetitive tasks [such as testing](../automation.md) and deployment
- **Quality Assurance**: Ensure code quality and compatibility across different environments before merging changes
- **Continuous Integration**: Validate every code change automatically to catch issues early in the development cycle
- **Consistency**: Maintain standardized processes across all branches and contributions
- **Reliability**: Provide fast feedback to developers about the status of their changes

## Pipeline Overview

The following pipelines are documented in this repository:

1. **Python CI - Tests** [Français](tests.fr.md) [English](tests.en.md): Automated [testing across
   multiple Python versions](../../shared/tests.txt) to ensure compatibility and code quality
2. **Python CI - Coverage** [Français](../coverage/python/coverage.fr.md) [English](../coverage/python/coverage.en.md): Automated
   [measures of a Python project's code coverage](../../shared/coverage.txt) and automatically uploads reports to Codecov

_(Additional pipelines will be documented as they are added)_

## How to Use This Documentation

Each pipeline is documented with:

- An overview of its purpose
- Trigger conditions (when it runs)
- Detailed step-by-step explanations of each action
- Expected outcomes and results

Navigate through the sections below to explore each pipeline in detail. Some of these processes will require additional
information about the third-party tools used (such as tox with python), which I will publish shortly.
