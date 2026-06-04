# Custom new-api Builder

Builds official `QuantumNous/new-api` source with local patches and publishes
static binary releases. This repository is not a fork of the upstream source
tree.

## Build Modes

- Patch Mode
  - `patched` builds upstream source with all files under `patches/*.patch`
    applied. This is the default and keeps the existing release tags.
  - `upstream` builds the same upstream source without applying any local patch.
    It publishes to separate `upstream-*` releases so comparison builds never
    overwrite patched releases.

- Stable
  - Manual runs can specify an upstream tag or leave it empty to use the latest
    stable upstream tag.
  - Scheduled runs check the latest stable upstream tag, excluding `alpha` and
    `nightly`.
  - Release tag, release name, and runtime version all use the upstream tag.
  - The build is skipped when the matching Release already exists.
  - Manual runs can enable force rebuild to update an existing Release.

- Alpha
  - Manual runs can enter `alpha`.
  - Scheduled runs also check `alpha` daily.
  - Builds the latest upstream default-branch commit.
  - Release tag is fixed to `alpha`.
  - Release name and runtime version use `alpha-<upstream commit short hash>`.
  - The build is skipped when the existing `alpha` Release already has the same
    release name.
  - Before updating `alpha`, all uploaded assets in that Release are deleted.
    The Release and tag are kept.
  - Manual runs can enable force rebuild to rebuild the current alpha source.

## Workflow

1. Resolve the upstream source.
2. Check out upstream source.
3. Apply `patches/*.patch` when patch mode is `patched`; skip this step when
   patch mode is `upstream`.
4. Print the patched source diff summary.
5. Build default and classic frontend assets.
6. Build pure-Go binaries with `CGO_ENABLED=0`:
   - `linux/amd64`
   - `linux/arm64`
   - `darwin/arm64`
7. Publish binaries and `checksums.txt` to a GitHub Release with a short build
   summary in the Release body.

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
  - Shows it as the last row in the classic web usage-log expanded details.
  - Does not add or migrate any database columns.

- `0003-show-cache-read-rate-in-usage-logs.patch`
  - Shows cache read token rate beside the usage-log list cache read value in both frontends.
  - Uses `cache_tokens / prompt_tokens` for OpenAI-style usage.
  - Uses `cache_tokens / (prompt_tokens + cache_tokens)` for Claude/Anthropic usage.
  - Does not add or migrate any database columns.

- `0004-show-running-request-count-in-usage-logs.patch`
  - Adds current running relay request count to `/api/log/stat` and `/api/log/self/stat`.
  - Shows the value as `RUN` beside usage-log statistics in both frontends.
  - Refreshes usage-log statistics every 5 seconds in both frontends.
  - Changes the default usage-log end time to the end of the current day.
  - Does not add or migrate any database columns.

- `0005-force-default-relay-ipv4.patch`
  - Forces the default relay HTTP client to dial upstream hosts through IPv4.
  - Keeps HTTP/HTTPS and SOCKS proxy clients on their existing proxy-specific dialing behavior.
  - Does not add any runtime environment variable.

- `0006-response-header-timeout-and-error-body-cleanup.patch`
  - Adds `RELAY_RESPONSE_HEADER_TIMEOUT` for bounding upstream response-header wait time.
  - Adds `RELAY_NON_STREAM_TIMEOUT` for bounding total non-stream relay request time.
  - Uses Go's native `http.Transport.ResponseHeaderTimeout`.
  - Applies the timeout only to relay requests already known to be streaming.
  - Also applies `RELAY_RESPONSE_HEADER_TIMEOUT` as a per-stream first-response timeout after the upstream attempt starts.
  - Cancels the upstream stream attempt when no response begins within the configured timeout.
  - Classifies both response-header and first-response timeout failures as `upstream error: response header timeout`.
  - Avoids treating downstream client cancellation as an upstream first-response timeout.
  - Keeps normal non-stream relay clients without a response-header timeout.
  - Applies non-stream total timeout through request context and cancels it when the returned response body is closed.
  - Keeps stream requests out of `RELAY_NON_STREAM_TIMEOUT`; stream total timeout still follows `RELAY_TIMEOUT`.
  - Uses `RELAY_TIMEOUT` as the non-stream fallback when `RELAY_NON_STREAM_TIMEOUT` is unset, and as the hard cap when both are set.
  - Binds upstream relay requests to the downstream request context, so client disconnects cancel the active upstream attempt.
  - Applies the same relay timeout/client selection to AWS Bedrock relay paths.
  - Uses separate cached clients for stream proxy requests so proxy traffic keeps connection pooling without mutating shared transports.
  - Closes error-response bodies even when `io.ReadAll(resp.Body)` fails in `RelayErrorHandler`.
  - Keeps existing `RELAY_TIMEOUT` global-cap semantics intact.

- `0007-skip-retry-after-client-disconnect.patch`
  - Skips relay retry when the downstream request context is already done.
  - Prevents extra channel switches after the client has already disconnected.
  - Keeps the current upstream request construction unchanged.

- `0008-responses-terminal-error-and-cleanup.patch`
  - Closes upstream response bodies before waiting for scanner goroutines to exit.
  - Emits a Responses-style synthetic `event: error` SSE event only when a stream ends without any terminal event.
  - Records stream inactivity timeout as `stream_inactivity_timeout` instead of the generic `timeout`.
  - Leaves `response.completed` streams without `[DONE]` unchanged.

- `0009-response-header-timeout-log-reason.patch`
  - Classifies Go/http2 response-header wait timeouts in relay logs.
  - Keeps the existing `do_request_failed` error code so retry and auto-disable behavior stay unchanged.
  - Changes the hidden upstream error message to `upstream error: response header timeout` for easier log diagnosis.

- `0010-release-memory-body-storage-buffers.patch`
  - Clears in-memory request-body buffer references when `memoryStorage.Close()` runs.
  - Helps large request bodies become garbage-collectable after request cleanup.
  - Does not change disk-backed body storage behavior or request replay semantics before close.

- `0011-relay-timing-diagnostics.patch`
  - Logs `relay request received` for text, Responses, and Gemini generate relay paths as soon as the route tag middleware runs.
  - Adds millisecond precision to normal application, system, and fatal log timestamps.
  - Adds optional relay preflight and upstream timing logs controlled by `RELAY_TIMING_LOG_ENABLED`.
  - Splits preflight `model_request` timing into request body storage, body bytes access, body decode, and body reset phases.
  - Logs request body size and whether body storage is memory-backed or disk-backed when timing diagnostics are enabled.
  - Adds optional early SSE line probes controlled by `RELAY_TIMING_LOG_STREAM_PROBE_COUNT`.
  - Keeps `relay request received` always on for the narrowed relay paths so request arrival can be correlated even when timing diagnostics are disabled.
  - Logs failed upstream attempt timing before relay retry overwrites the final attempt timing.
  - Logs timing fields, event kind, line length, path, client IP, model/channel/status, connection reuse data, and upstream write error text when present.
  - Does not log request bodies, response bodies, Authorization values, API keys, headers, or raw query strings; URL-like error text is masked before logging.

- `0012-record-actual-response-model-in-usage-logs.patch`
  - Captures the model reported by OpenAI Chat Completions and Responses responses/streams.
  - Stores the captured model in usage-log `other.actual_model_name` when it is the actual served model.
  - Keeps the existing mapped-model fallback for older logs and mapped channels.
  - Shows `actual_model_name` before the mapped-model fallback in default and classic usage-log UI.
  - Does not add or migrate any database columns.

## Version Handling

The workflow sets `common.Version` through the full upstream module path:

```text
github.com/QuantumNous/new-api/common.Version
```

- Stable: upstream tag.
- Alpha: `alpha-<upstream commit short hash>`.

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

- Manual: enter an upstream tag, enter `alpha`, or leave empty for the latest
  stable upstream tag. Choose `patched` or `upstream` patch mode. Enable force
  rebuild to update an existing Release.
- Scheduled: runs daily at `00:00 UTC+8` (`16:00 UTC`) for stable and alpha.

## How To Run Manually

1. Open the GitHub repository page.
2. Go to **Actions**.
3. Select **Build custom new-api**.
4. Click **Run workflow**.
5. Enter the official upstream tag, for example:

```text
v1.0.0
```

Leave the tag empty for the latest stable upstream tag, or enter `alpha` for the
latest upstream source commit.

6. Choose the patch mode:

```text
patched
```

for normal custom builds, or:

```text
upstream
```

for an upstream-only comparison build.

7. Download the generated files from the GitHub Release:

```text
alpha                      patched latest upstream source
upstream-alpha             upstream-only latest upstream source
vX.Y.Z                     patched stable/tag source
upstream-vX.Y.Z            upstream-only stable/tag source
```

If upstream code changes conflict with these patches, the action fails at
`git apply --check`. In that case, refresh the patches against the new upstream
tag.
