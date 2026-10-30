# TerraScale shared CI

Reusable workflows for the repositories that remain outside the monorepo.

Call workflows from `TerraScale/.github/.github/workflows/<file>.yml` and pin
each call to an immutable full 40-character commit SHA. Update that pin through
the normal `ts-ship` process when adopting a shared workflow change.

| Workflow | Existing checks |
| --- | --- |
| `command.yml` | Required `command` string, run from the caller's checkout with `bash -euo pipefail`. Used for branch-action, homebrew-tap, and ts-resources. |
| `node.yml` | ts-status's exact Legacy scan, dependency audit, checks, tests, build, and npm cache restore/save policy. |
| `terraform.yml` | terraform-provider-terrascale's exact Legacy scan, Go and advisory caches, advisory checks, tests, build, and snapshot archive/checksum validation. Keeps `GOTOOLCHAIN: local`. |
| `result.yml` | Required `result` string; succeeds only for `success`. Aggregates an existing upstream job without moving its checks. |

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

Every reusable workflow requests only `contents: read`, uses the caller's
existing GitHub token, and declares no secrets. Calls require no credentials or
new permissions; do not add `secrets: inherit`. Source checks and existing test
policies stay exact. No TerraBase binary is downloaded or executed here.

This repository's own CI only lints workflow syntax. Its actionlint v1.7.12
download is verified against the pinned Linux amd64 archive SHA-256. Run the
same syntax check on a dev host with:

```sh
mise exec actionlint@1.7.12 shellcheck@0.11.0 -- actionlint
```
