# TerraScale shared CI

Reusable workflows for every TerraScale repository.

Call workflows from `TerraScale/.github/.github/workflows/<file>.yml` and pin
each call to an immutable full 40-character commit SHA. Update that pin through
a pull request when adopting a shared workflow change.

| Workflow | Checks |
| --- | --- |
| `ci-time.yml` | Runs after each CI run of the calling repository and opens or updates a `ci-time` issue when a run is slow for the second time in a row: its working time is more than 30 s (by default) over the median of its earlier runs, or a job is more than 30 s over the slowest of its own. Time spent waiting for a runner is not counted. Never blocks a merge; see [CI time](#ci-time). |

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

A run's wall time is the time from its creation until its last job completed,
queue included. Its working time is the longest chain of jobs: each job counts
from the moment it started until it completed, and the wait before it started
does not count. GitHub creates a job when the jobs it needs have finished, so a
job continues the longest chain among the jobs that had completed by the time it
was created; jobs created at the start of the run start their own chain, and
jobs that run side by side take the longer chain. A job that waited in the queue
is not chained onto a job that merely finished while it waited. The same wait
is what the wall time reports as spent waiting, so a run whose jobs did the
same work and only waited longer for a runner has the same working time.
Path filters and matrices make runs of one workflow run
different jobs, so the working time is compared only with runs in which the
same set of jobs succeeded and no job ended other than successful or skipped.
Its baseline is the median working time of the latest `baseline-runs` of those
runs, measured the same way.

Each job that succeeded is compared with the same job, by name, in the latest
`baseline-runs` earlier runs in which it succeeded, whatever other jobs those
runs had. A job's time runs from its start until it completed. Skipped jobs are
never measured.

With fewer than 3 earlier runs, the run or the job has no baseline yet and the
job summary says so. A run in which a job neither succeeded nor was skipped has
no run baseline; its jobs that succeeded are compared as usual.

The job summary shows the wall time, the working time and how much of the wall
time was waiting, the working time against its baseline and the threshold, then
one row per job that succeeded: the seconds it was queued, its time, the median
and the slowest of its earlier times, and how many runs they come from. A job
is created once the jobs it needs have finished and is queued until it starts,
so its queued seconds are the wait for a runner or a concurrency lock, never
the wait for the jobs it needs. When the run has a baseline, links to the runs
with the same jobs follow, each with its working time.

A run is slow when its working time took more than `threshold-seconds` longer
than the median of its earlier runs, or when a job took more than
`threshold-seconds` longer than the slowest of its own earlier runs. One slow
run is not a regression: a slow runner or a slow download usually does not
repeat, and a change that costs time does. The check opens or updates the issue
only when the slowness repeats, this run slow and the previous run with the
same jobs slow by the same two tests against the runs before both. A run of
other jobs in between does not count, and a run with no earlier run of the same
jobs is not a repeat. A run that is slow once writes "slower once (...):
reported if the next run with the same jobs is slow too" in its job summary and
opens nothing. A job is held to its
slowest earlier run rather than their median so that the normal spread of one
job from run to run does not count as slow. Waiting for a runner never does
either, because that wait is not part of the working time and not part of a
job's time. The check then creates or updates the `ci-time` label and opens the
issue `CI time regression: <workflow> (<event>)` with that label, or comments on
that issue while it is open. The issue or comment gives the working time when
the run has a baseline, how much of the wall time was waiting, each slow job
with its time, its slowest and median earlier times and its queued seconds, and
the previous run with the same jobs that was slow too, and includes the job summary. Investigate it
and fix the cause.

The check runs as two jobs. `compare` measures the run and writes the job
summary; `report` starts only when the issue opens, to open it or comment on
it. Only one `report` of the same workflow and event runs at a time,
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
| `threshold-seconds` | `30` | Seconds the working time may take beyond the median of its earlier runs, and a job beyond the slowest of its earlier runs. A run over either is slow, and an issue opens only when the previous run with the same jobs was slow too. |
| `baseline-runs` | `5` | Earlier runs the working time and each job are compared with. |
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
