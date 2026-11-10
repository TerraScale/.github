# TerraScale shared CI

Reusable workflows for every TerraScale repository.

Call workflows from `TerraScale/.github/.github/workflows/<file>.yml` and pin
each call to an immutable full 40-character commit SHA. Update that pin through
a pull request when adopting a shared workflow change.

| Workflow | Checks |
| --- | --- |
| `command.yml` | Required `command` string, run from the caller's checkout with `bash -euo pipefail`. Used for branch-action, homebrew-tap, and ts-resources. |
| `node.yml` | ts-status's exact Legacy scan, dependency audit, checks, tests, build, and npm cache restore/save policy. |
| `terraform.yml` | terraform-provider-terrascale's exact Legacy scan, Go and advisory caches, advisory checks, tests, build, and snapshot archive/checksum validation. Keeps `GOTOOLCHAIN: local`. |
| `result.yml` | Required `result` string; succeeds only for `success`. Aggregates an existing upstream job without moving its checks. |
| `ci-time.yml` | Runs after each CI run of the calling repository and opens or updates a `ci-time` issue when the run took more than 30 s (by default) longer than the median of earlier runs with the same jobs. Never blocks a merge; see [CI time](#ci-time). |

For example, keep the caller's existing triggers, concurrency, permissions, and
dependencies, and replace only its job implementation:

```yaml
jobs:
  test:
    uses: TerraScale/.github/.github/workflows/command.yml@<full-40-character-commit-SHA>
    with:
      command: python3 scripts/check_resources.py
```

The generic command receives the existing repository check script through the
`CHECK_COMMAND` environment variable. A result gate receives the caller's
upstream result through its `result` input; the caller retains the `needs` and
`if` conditions that make failure, cancellation, and skipped jobs visible.

Every reusable workflow except `ci-time.yml` requests only `contents: read`;
`ci-time.yml` requests only `actions: read` and `issues: write`, which its
caller must grant. They all use the caller's existing GitHub token and declare
no secrets, so calls require no credentials; do not add `secrets: inherit`.
Source checks and existing test policies stay exact. No TerraBase binary is
downloaded or executed here.

This repository's own CI only lints workflow syntax. Its actionlint v1.7.12
download is verified against the pinned Linux amd64 archive SHA-256. Run the
same syntax check on a dev host with:

```sh
mise exec actionlint@1.7.12 shellcheck@0.11.0 -- actionlint
```

## CI time

`ci-time.yml` compares the time of every CI run with earlier runs of the same
workflow. It runs in its own workflow after a CI run has completed, and only
for a run that succeeded on its first attempt: failed, cancelled and re-run CI
runs are never compared.

A run's CI time is the time from its creation, queue included, until its last
successful job completed. Path filters and matrices make runs of one workflow
run different jobs, so two runs are comparable only when the same set of jobs
succeeded and no job ended other than successful or skipped. The baseline is
the median CI time of the last `baseline-runs` comparable first-attempt runs of
the same workflow and event, searched among its `scan-limit` newest successful
runs of that event; for any event other than `pull_request` they must also be
on the same branch. With fewer than 3 such runs the job summary says "no
baseline yet". Otherwise it shows this run's time, the baseline, the difference
and the threshold, the jobs that ran, each job's time against its median, and
links to the baseline runs.

When a run took more than `threshold-seconds` longer than the baseline, the
check creates or updates the `ci-time` label and opens the issue
`CI time regression: <workflow> (<event>)` with that label, or comments on that
issue while it is open. Investigate it and fix the cause.

The check never blocks a merge: it runs after CI has finished, in its own
workflow; never make it a required check. It needs no secrets.

Add this caller workflow to each repository, with `workflows` naming its CI
workflow:

```yaml
name: CI time
on:
  workflow_run:
    workflows: [CI]
    types: [completed]
permissions:
  actions: read
  issues: write
jobs:
  ci-time:
    uses: TerraScale/.github/.github/workflows/ci-time.yml@<full-40-character-commit-SHA>
```

| Input | Default | Meaning |
| --- | --- | --- |
| `threshold-seconds` | `30` | Seconds a run may take beyond the baseline before the issue opens or gets a comment. |
| `baseline-runs` | `5` | Comparable earlier runs whose median CI time is the baseline. |
| `scan-limit` | `25` | Newest successful runs of the workflow and event searched for them. The check reads the jobs of each run it inspects, one API call each, and never paginates. |

The caller must grant `actions: read`, to read run and job times, and
`issues: write`, to manage the `ci-time` label and open or comment on the
issue. This repository runs the check on its own CI from `ci-time-self.yml`.
