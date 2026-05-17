# Custom new-api Builder

This repository is a small builder for official `QuantumNous/new-api` releases.
It is not a fork of the upstream source tree.

## What It Does

The GitHub Action:

1. Resolves an official upstream tag from `QuantumNous/new-api`.
   - Manual runs can specify a tag.
   - Scheduled runs use the latest upstream tag, excluding `alpha` and `nightly`.
2. Skips the build if this repository already has a Release with the same tag.
3. Checks out the upstream source.
4. Applies the two local patches in `patches/`.
5. Builds frontend assets.
6. Builds three pure-Go binaries with `CGO_ENABLED=0`:
   - `linux/amd64`
   - `linux/arm64`
   - `darwin/arm64`
7. Publishes `.tar.gz` packages and `checksums.txt` to a GitHub Release named
   with the official upstream tag you entered.

No Docker image is built.

## Patches

- `0001-custom-upstream-user-agent-env-fallback.patch`
  - Adds upstream User-Agent env support:
    - `UPSTREAM_USER_AGENT_OPENAI`
    - `UPSTREAM_USER_AGENT_CLAUDE`
    - `UPSTREAM_USER_AGENT_GEMINI`
  - Keeps channel/header override User-Agent higher priority.
  - Falls back to the incoming request User-Agent when no env User-Agent is configured.

- `0002-custom-request-user-agent-log-display.patch`
  - Stores the incoming request User-Agent once in `logs.other.request_user_agent`.
  - Shows it in the default web usage-log detail dialog for all users.
  - Does not add or migrate any database columns.

## Version Handling

The workflow sets the runtime version to the official upstream tag that you enter
when starting the action.

It uses the full Go module path:

```text
github.com/QuantumNous/new-api/common.Version
```

This matches the module path in upstream `go.mod` and avoids the empty-version
problem caused by using a short import path in linker flags.

## Static Build Notes

Linux builds use:

```text
CGO_ENABLED=0
```

For this project that is the preferred static-compatible mode because the SQLite
driver is pure Go (`github.com/glebarez/sqlite` / `modernc.org/sqlite`). It avoids
linking to the host system glibc and improves compatibility with older Linux
servers.

## Triggers

- Manual: run the workflow from the GitHub Actions page. You can enter an
  upstream tag, or leave it empty to use the latest upstream tag.
- Scheduled: runs daily at `00:00 UTC+8` (`16:00 UTC`) and builds only when the
  matching Release does not already exist in this repository.

## How To Run Manually

1. Open the GitHub repository page.
2. Go to **Actions**.
3. Select **Build custom new-api**.
4. Click **Run workflow**.
5. Enter the official upstream tag, for example:

```text
v1.0.0
```

You can also leave the tag empty to build the latest upstream tag detected by
the workflow.

6. Download the generated files from the GitHub Release with the same tag name.

If upstream code changes conflict with these patches, the action fails at
`git apply --check`. In that case, refresh the patches against the new upstream
tag.
