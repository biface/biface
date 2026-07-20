# Two pipeline architectures

← [Test automation](../automation.en.md) | _Version française → [ci-models.fr.md](ci-models.fr.md)_

## Overview

Every pipeline documented in this repository follows one of these two architectures. The choice between
the two has **nothing to do with the project's language** — Python and Rust coexist within each of the two
families. What actually distinguishes these architectures is **how the steps chain together**:

- **event-driven multi-file chaining** (`workflow_run`) — one workflow file per step, each file triggered
  by the completion of the previous one;
- **single-file dependency chaining** (`needs:`) — a single workflow file, whose jobs explicitly declare
  their internal dependencies.

A given project adopts one or the other — not both — but that same project can then host a third,
completely independent workflow (coverage, documentation publishing, repository mirroring…) that triggers
on its own, outside any chain. That case is handled separately in each implementation.

## Model 1 — event-driven chaining (`workflow_run`)

```yaml
# file B: triggers when file A completes
on:
  workflow_run:
    workflows: ["<workflow A name>"]
    types:
      - completed
    branches:
      - "**"

jobs:
  <job>:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - uses: actions/checkout@v6
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
```

Each step of the chain lives in its **own** `.github/workflows/*.yaml` file, with its own workflow name.
GitHub Actions links them via the `workflow_run` trigger, which listens for the completion of another
workflow named explicitly.

**Points to watch with this model:**

- the file triggered by `workflow_run` doesn't automatically receive the right commit — the checkout must
  specify `ref: ${{ github.event.workflow_run.head_sha }}`, otherwise GitHub fetches the default branch
  instead;
- the `if: github.event.workflow_run.conclusion == 'success'` condition must be repeated in **every**
  following file — forgetting it lets the job run even if the previous step failed;
- each file shows up separately in the Actions tab, giving a finer-grained read of the history (one badge
  per step), at the cost of a coupling by workflow name string between files (renaming a workflow silently
  breaks the chain).

→ documented implementation: [pipelines/workflow-run/](../pipelines/workflow-run/pipelines.en.md)

## Model 2 — dependency chaining (`needs:`)

```yaml
# a single file, all jobs
jobs:
  <job_a>:
    steps: [...]

  <job_b>:
    needs: <job_a>
    steps: [...]

  <job_c>:
    needs: <job_b>
    if: startsWith(github.ref, 'refs/tags/')
    steps: [...]
```

All steps live in a **single** file. GitHub Actions computes the execution graph from the `needs:`
declared by each job — a job waits for everything listed in its `needs:` to succeed before it starts.

**Points to watch with this model:**

- the commit is correct by construction (a single Git event triggers the whole file) — no `head_sha` to
  manage;
- restricting a job to certain conditions (for example: only build/publish on tag) is done with `if:`,
  independently of `needs:` — the two combine freely;
- a single status badge for the whole chain in the Actions tab; the per-step detail is read from the jobs
  of the run, not from a list of separate workflows;
- `needs:` accepts a list (`needs: [job_a, job_b]`) when a job depends on several predecessors — something
  `workflow_run` cannot express natively.

→ documented implementation: [pipelines/needs/](../pipelines/needs/pipeline.en.md), with two real examples
(a long chain and a short chain) to show that it's the length of the chain — not the language — that
varies from one project to another.

## Which one to choose?

| | `workflow_run` | `needs:` |
|---|---|---|
| Number of files | one per step | a single one |
| Reading a run | one badge per step | one global badge, detail per job |
| Coupling risk | by workflow name (string) | none, everything is in the same file |
| Adding a step | new file + `workflow_run` trigger | new job + `needs:` |
| Multiple dependencies | not natively expressible | `needs: [a, b]` |

The single-file model (`needs:`) is the one recommended for new projects — see DD-36 in the design
decisions of the relevant project. The multi-file model stays documented and maintained where it's already
in place, with no forced migration.

---

## License

[CC BY-NC 4.0](../../LICENSE.txt) — documentation and configuration files.
