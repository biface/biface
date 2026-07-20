# Pipeline — single-file dependency chaining (`needs:`)

← [Pipelines](../pipelines.en.md) | _Version française → [pipeline.fr.md](pipeline.fr.md)_

## Overview

This model groups every pipeline step into a **single** workflow file. Each job explicitly declares which
job(s) it depends on via `needs:`; GitHub Actions builds the execution graph from these declarations. The
general principle (advantages, limitations, comparison with `workflow_run`) is explained in
[Two pipeline architectures](../../architecture/ci-models.en.md#model-2--dependency-chaining-needs).

Two real implementations are documented below, side by side: a **long chain** (six steps, Python) and a
**short chain** (three steps, Rust). The difference between the two is **not** the language — it's the
number of steps the project needs to chain; a long chain in Rust or a short chain in Python would have
worked just as well. The complete files live in `shared/github-ci/needs/`; the excerpts below only show
what illustrates the model.

## Generic skeleton

```yaml
name: <pipeline name>

on:
  push:
    branches:
      - "**"
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+"
      - "v[0-9]+.[0-9]+.[0-9]+rc[0-9]+"
  pull_request:
    branches:
      - "**"

jobs:
  <job_1>:
    name: <readable label>
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ... environment setup + running <job_1>

  <job_2>:
    name: <readable label>
    runs-on: ubuntu-latest
    needs: <job_1>
    # if: <optional condition, e.g. tags only>
    steps:
      - uses: actions/checkout@v4
      # ... environment setup + running <job_2>
```

All jobs share the same Git trigger (`on:`) — unlike the `workflow_run` model, there's only one entry
point for the whole chain.

## Implementation A — long chain (six steps, Python)

> **Ready-to-copy template**:
> [`shared/github-ci/needs/python-ci.yaml`](../../../shared/github-ci/needs/python-ci.yaml). Replace
> `<package_name>` with the actual package name, adjust the Python version matrix, and adapt or remove the
> example service container (`httpbin`) depending on the integration tests' needs.

Sequence: `quality` → `test-unit` → `test-integration` → `coverage` → `build` → `publish-pypi` /
`publish-testpypi`. The last four steps are restricted to tags
(`if: startsWith(github.ref, 'refs/tags/')`) — they only consume CI minutes during a release.

```yaml
jobs:
  quality:
    name: Quality checks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install uv
        uses: astral-sh/setup-uv@v4
      - run: uv pip install --system tox tox-uv
      - run: tox -e ci-quality

  test-unit:
    name: Unit tests (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest
    needs: quality
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.10", "3.11", "3.12", "3.13"]
        experimental: [false]
        include:
          - python-version: "3.14"
            experimental: true
    continue-on-error: ${{ matrix.experimental }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: |
          PYVER=$(echo "${{ matrix.python-version }}" | tr -d '.')
          tox -e py${PYVER}

  test-integration:
    name: Integration tests
    runs-on: ubuntu-latest
    needs: test-unit
    if: startsWith(github.ref, 'refs/tags/')
    # EXAMPLE service container — httpbin (ghcr.io/mccutchen/go-httpbin) is
    # used here as a concrete illustration. Replace <service-name>/
    # <service-image>/<port> with your own, or remove this block entirely if
    # your integration tests need no external service.
    services:
      <service-name>:            # e.g. httpbin
        image: <service-image>   # e.g. ghcr.io/mccutchen/go-httpbin
        ports:
          - <port>:<port>        # e.g. 8080:8080
    steps:
      - uses: actions/checkout@v4
      - run: tox -e integration

  coverage:
    name: Coverage
    runs-on: ubuntu-latest
    needs: test-integration
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - uses: actions/checkout@v4
      - run: tox -e coverage
      - uses: codecov/codecov-action@v5
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/coverage.xml
          fail_ci_if_error: true

  build:
    name: Verify version and build package
    runs-on: ubuntu-latest
    needs: coverage
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - uses: actions/checkout@v4
      - name: Verify version consistency
        id: version
        run: |
          # Compares the Git tag, pyproject.toml and src/<package_name>/__static__.py
          # — see the template file for the full script
      - run: python -m build
      - uses: actions/upload-artifact@v4
        with:
          name: python-package-distributions
          path: dist/
    outputs:
      version: ${{ steps.version.outputs.version }}
      tag: ${{ steps.version.outputs.tag }}

  publish-pypi:
    name: Publish to PyPI
    runs-on: ubuntu-latest
    needs: build
    if: startsWith(github.ref, 'refs/tags/') && !contains(github.ref_name, 'rc')
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: python-package-distributions
          path: dist/
      - uses: pypa/gh-action-pypi-publish@release/v1
        with:
          password: ${{ secrets.PYPI_API_TOKEN }}
          skip-existing: true
      - uses: softprops/action-gh-release@v3
        with:
          tag_name: ${{ needs.build.outputs.tag }}
          prerelease: false
          files: dist/*
```

**Notes on this implementation:**

- `publish-testpypi` (not shown above, see the template file) is a separate job, same structure as
  `publish-pypi`, which publishes to TestPyPI when the tag contains `rc` (`if: ... &&
  contains(github.ref_name, 'rc')`) — both `publish-*` jobs depend on `build` and differ only by their
  `if:` condition;
- coverage is **a job in the chain** (`needs: test-integration`), not a separate workflow; it's only
  computed on tag, over the full test suite;
- the `build` job exposes its outputs (`outputs:`) so the `publish-*` jobs can retrieve the tag without
  recomputing it;
- the `httpbin` service is a concrete example, not a prescription — see the adaptation comment at the top
  of the template file.

## Implementation B — short chain (three steps, Rust)

> **Ready-to-copy template**:
> [`shared/github-ci/needs/rust-ci.yml`](../../../shared/github-ci/needs/rust-ci.yml). Replace
> `<feature_name>` with the crate's actual Cargo feature(s) (or drop the corresponding steps if the crate
> has no optional features), and `<crate_name>` with the workspace's default crate for the documentation
> redirect.

Sequence: `fmt` → `clippy` **and** `test` (in parallel, both depending on `fmt`) → `doc` (depends on both
of the above, restricted to the main branch).

```yaml
jobs:
  fmt:
    name: Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo fmt --check

  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    needs: fmt
    steps:
      - uses: actions/checkout@v4
      - run: cargo clippy -- -D warnings
      - run: cargo clippy --features <feature_name> -- -D warnings

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: fmt
    steps:
      - uses: actions/checkout@v4
      - run: cargo test
      - run: cargo test --features <feature_name>
      - run: cargo test --doc
      - run: cargo test --doc --features <feature_name>

  doc:
    name: Documentation
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/master' && github.event_name == 'push'
    needs: [clippy, test]
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - run: cargo doc --no-deps --document-private-items
      - run: |
          echo '<meta http-equiv="refresh" content="0; url=<crate_name>/index.html">' \
            > target/doc/index.html
      - uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./target/doc
```

**Notes on this implementation:**

- `clippy` and `test` share the same `needs: fmt` — they run **in parallel** with each other, each only
  waiting on `fmt`; this is the concrete example of a single-predecessor `needs:` used twice in a fan-out,
  rather than a strictly linear chain;
- `doc` illustrates `needs: [clippy, test]` — a dependency on **multiple** jobs, something the
  `workflow_run` model can't express natively (it would require triggering `doc` on the completion of two
  different workflows, with two parallel `conclusion` checks in the same file);
- each command (`clippy`, `test`) is repeated once per declared Cargo feature, rather than a single
  `--all-features` run — see the detailed rationale in
  [Coverage — independent workflow](../../coverage/rust/coverage.en.md#step-5--generating-coverage-once-per-feature-combination),
  which applies the same principle;
- here, **coverage isn't part of this file** — see below.

## Where does coverage go, in this model?

The two implementations above illustrate two different ways of placing coverage, both compatible with the
`needs:` model:

- **coverage built into the chain** (implementation A, Python): a `coverage` job, with its own `needs:`,
  restricted to tags — one more link in the same file;
- **coverage as an independent workflow** (implementation B, Rust): a second workflow file
  (`shared/github-ci/coverage/rust-coverage.yml`), which depends on nothing in `rust-ci.yml` — it triggers
  on its own, on its own push conditions. This is **not** a case of `workflow_run` (no file waits for the
  completion of another): the two files are simply independent. See
  [Coverage — independent workflow](../../coverage/rust/coverage.en.md) for the detail of this variant.

The choice between the two mainly depends on how often coverage needs to run: if it only needs to run on
tag like the rest of the release, folding it into the chain avoids an extra file; if it needs to run on
every push to a working branch (fast feedback, independent of the release cycle), a separate workflow is
more appropriate. Nothing prevents the opposite either (chained coverage in Rust, independent workflow in
Python) — the choice follows the project's need, not its language.

---

## License

[CC BY-NC 4.0](../../../LICENSE.txt) — documentation and configuration files.
