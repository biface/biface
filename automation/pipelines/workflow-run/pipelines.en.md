# Pipelines — event-driven multi-file chaining (`workflow_run`)

← [Pipelines](../pipelines.en.md) | _Version française → [pipelines.fr.md](pipelines.fr.md)_

Reference implementation: [ndt](https://github.com/biface/ndt) (Python). Three separate workflow files,
each triggered by the completion of the previous one:

1. **[Quality](quality.en.md)** — first link, triggers directly on push/pull request
2. **[Tests](tests.en.md)** — triggered by `workflow_run` on the completion of Quality
3. **[Coverage](../../coverage/python/coverage.en.md)** — triggered by `workflow_run` on the completion of
   Tests, only on `staging/**` and the main branch

The general principle of the model (the `workflow_run` trigger, handling `head_sha`, repeating the
`conclusion == 'success'` condition) is explained in
[Two pipeline architectures](../../architecture/ci-models.en.md#model-1--event-driven-chaining-workflow_run).
The three pages above each document the concrete implementation of one step.

A fourth page, **[Tests — reinforced model](test-uv.en.md)**, documents as an example an alternative
approach to the Tests step (managing several environments via `.tox-config`) — abandoned, kept for
reference, not carried over into `shared/github-ci/`.
