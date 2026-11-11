# TerraScale shared CI

Reusable workflows for every TerraScale repository.

Call workflows from `TerraScale/.github/.github/workflows/<file>.yml` and pin
each call to an immutable full 40-character commit SHA. Update that pin through
a pull request when adopting a shared workflow change.

| Workflow | Checks |
| --- | --- |
| `ci-time.yml` | Runs after each CI run of the calling repository and opens or updates a `ci-time` issue when the run took more than 30 s (by default) longer than the median of earlier runs with the same jobs, or when a job took more than 30 s longer than the slowest of its earlier runs. Never blocks a merge; see [CI time](#ci-time). |

This repository's own CI only lints workflow syntax. Its actionlint v1.7.12
download is verified against the pinned Linux amd64 archive SHA-256. Run the
same syntax check on a dev host with:

```sh
mise exec actionlint@1.7.12 shellcheck@0.11.0 -- actionlint
```

## CI time

`ci-time.yml` compares the time of every CI run, and of each of its jobs, with
earlier runs of the same workflow. It runs in its own workflow after a CI run
has completed, and only for a run that succeeded on its first attempt: failed,
cancelled and re-run CI runs are never measured or used as history.
A run whose head is in another repository, such as a pull request from a fork,
is never measured or used as history either, because the fork decides the
workflow file, the branch name and the job names.

The earlier runs are the first-attempt runs whose head is in the repository
among the `scan-limit` newest
successful runs of the same workflow and event, newest first; for any event
other than `pull_request` they must also be on the same branch. The check reads
the jobs of all of them in one GraphQL query, so it makes at most two REST
calls, for the run's own jobs and the list of earlier runs, and one GraphQL
query, whatever `scan-limit` is, plus the calls that open or update the issue.

A run's CI time is the time from its creation, queue included, until its last
job completed. Path filters and matrices make runs of one workflow run
different jobs, so the run is compared only with runs in which the same set of
jobs succeeded and no job ended other than successful or skipped. Its baseline
is the median CI time of the latest `baseline-runs` of those runs.

Each job that succeeded is compared with the same job, by name, in the latest
`baseline-runs` earlier runs in which it succeeded, whatever other jobs those
runs had. A job's time runs from its start until it completed. Skipped jobs are
never measured.

With fewer than 3 earlier runs, the run or the job has no baseline yet and the
job summary says so. A run in which a job neither succeeded nor was skipped has
no run baseline; its jobs that succeeded are compared as usual.

The job summary shows the run's CI time against its baseline and the
threshold, then one row per job that succeeded: the seconds it was queued, its
time, the median and the slowest of its earlier times, and how many runs they
come from. A job is created once the jobs it needs have finished and is queued
until it starts, so its queued seconds are the wait for a runner or a
concurrency lock, never the wait for the jobs it needs: a slow run whose jobs
were queued long waited rather than worked. When the run has a baseline, links
to the runs with the same jobs follow.

The check opens or updates the issue in two cases: the run took more than
`threshold-seconds` longer than its baseline, or a job took more than
`threshold-seconds` longer than the slowest of its earlier runs. A job is held
to its slowest earlier run rather than their median so that the normal spread
of one job from run to run does not open issues. The check then creates or
updates the `ci-time` label and opens the issue
`CI time regression: <workflow> (<event>)` with that label, or comments on that
issue while it is open. The issue or comment says which case happened, names
each slow job with its time, its slowest and median earlier times and its
queued seconds, and includes the job summary. Investigate it and fix the cause.

The check runs as two jobs. `compare` measures the run and writes the job
summary; only in those two cases does `report` start, to open the issue or
comment on it. Only one `report` of the same workflow and event runs at a time,
and GitHub keeps one more waiting and cancels it when a third arrives: three
regressions that finish together end as one issue, one comment and one
cancelled `report`, whose comparison stays in its job summary.

The check never blocks a merge: it runs after CI has finished, in its own
workflow; never make it a required check.

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
  checks: read
  issues: write
jobs:
  ci-time:
    uses: TerraScale/.github/.github/workflows/ci-time.yml@<full-40-character-commit-SHA>
```

| Input | Default | Meaning |
| --- | --- | --- |
| `threshold-seconds` | `30` | Seconds the run may take beyond its baseline, or a job beyond the slowest of its earlier runs, before the issue opens or gets a comment. |
| `baseline-runs` | `5` | Earlier runs the run and each job are compared with. |
| `scan-limit` | `25` | Newest successful runs of the workflow and event whose jobs the check reads, all in one GraphQL query, to find them. It never paginates. |

The caller must grant three permissions. `compare` requests `actions: read`,
to read the run's own jobs and the list of earlier runs, and `checks: read`, to
read the jobs of those earlier runs in the GraphQL query, which a private
repository answers only with this permission; `report` requests
`issues: write`, to manage the `ci-time` label and open or comment on the
issue. A caller that leaves one out gets a run that fails at startup with no
job started, and its run page shows "Invalid workflow file" with the job and
the permission, for example
`The nested job 'compare' is requesting 'checks: read', but is only allowed 'checks: none'.`
The check uses the caller's GitHub token and declares no secrets; do not add
`secrets: inherit`. This repository runs the check on its own CI from
`ci-time-self.yml`.
