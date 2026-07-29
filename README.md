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
5. Build the frontend assets.
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
  - Does not add or migrate any database columns.

- `0003-show-cache-read-rate-in-usage-logs.patch`
  - Shows cache read token rate beside the usage-log list cache read value in the web frontend.
  - Uses `cache_tokens / prompt_tokens` for OpenAI-style usage.
  - Uses `cache_tokens / (prompt_tokens + cache_tokens)` for Claude/Anthropic usage.
  - Does not add or migrate any database columns.

- `0004-show-running-request-count-in-usage-logs.patch`
  - Adds the current running relay request count to the administrator-only `/api/log/stat` response.
  - Shows the value as `RUN` only in the administrator usage-log view.
  - Refreshes usage-log statistics every 5 seconds only in the administrator view.
  - Changes the default usage-log end time to the end of the current day.
  - Does not add or migrate any database columns.

- `0005-configure-default-relay-network-family.patch`
  - Adds `RELAY_DIRECT_NETWORK_FAMILY=auto|ipv4_only|ipv6_only` for relay clients without an explicit channel proxy.
  - Leaves `auto` untouched; the `only` modes wrap the inherited `DialContext` on every direct HTTP/2 shard without replacing its timeout, keepalive, or context handling.
  - Keeps explicit HTTP/HTTPS and SOCKS channel proxies, plus the SSRF-protected fetch client, on their existing dialing behavior.
  - Treats an environment proxy as the dial target and requires a restart after changing the setting.

- `0006-native-stream-response-header-timeout.patch`
  - Adds `RELAY_RESPONSE_HEADER_TIMEOUT` for streaming relay requests using Go's native `http.Transport.ResponseHeaderTimeout` semantics.
  - Applies the response-header timeout after the request body is written, without adding a separate first-response timer or binding the upstream request to the downstream context.
  - Includes the timeout in the transport policy and proxy-client cache key, and applies it to every configured HTTP/2 shard.
  - Classifies native HTTP/1 and HTTP/2 response-header timeout errors as `upstream error: response header timeout` while preserving the existing relay error code.
  - Adds `RELAY_NON_STREAM_TIMEOUT` as the total `http.Client.Timeout` for non-stream relay attempts; `RELAY_TIMEOUT` remains the fallback and, when set, the upper bound.
  - Keeps stream requests outside `RELAY_NON_STREAM_TIMEOUT`; their total client timeout continues to use `RELAY_TIMEOUT`.
  - Reuses the normalized transport and connection pools when only the client-level total timeout differs.
  - Applies the same client selection to AWS Bedrock and prevents the AWS SDK from internally retrying response-header timeout errors.
  - Leaves the known AWS non-stream limitation documented in code: `RELAY_NON_STREAM_TIMEOUT` is per HTTP attempt and does not yet bound the SDK's complete retry cycle.

- `0007-skip-retry-after-client-disconnect.patch`
  - Skips relay retry when the downstream request context is already done.
  - Prevents extra channel switches after the client has already disconnected.
  - Keeps the current upstream request construction unchanged.

- `0008-responses-terminal-error-event.patch`
  - Emits a Responses-style synthetic `event: error` SSE event only when a stream ends without any terminal event.
  - Records stream inactivity timeout as `stream_inactivity_timeout` instead of the generic `timeout`.
  - Leaves `response.completed` streams without `[DONE]` unchanged.

- `0009-close-relay-error-response-body-on-read-failure.patch`
  - Closes upstream error-response bodies even when reading the body fails.
  - Keeps successful error-body parsing and error classification unchanged.
  - Adds a regression test that verifies the body is closed exactly once on read failure.

- `0010-release-memory-body-storage-buffers.patch`
  - Clears in-memory request-body buffer references when `memoryStorage.Close()` runs.
  - Helps large request bodies become garbage-collectable after request cleanup.
  - Does not change disk-backed body storage behavior or request replay semantics before close.

- `0011-relay-timing-diagnostics.patch`
  - Logs `relay request received` for text, Responses, and Gemini generate relay paths as soon as the route tag middleware runs.
  - Adds millisecond precision to normal application, system, and fatal log timestamps.
  - Adds optional relay preflight and upstream timing logs controlled by `RELAY_TIMING_LOG_ENABLED`.
  - Uses Go HTTP trace hooks on the shared relay request path to capture DNS, connect, TLS, connection reuse, request-write, first-byte, and response-header timing.
  - Logs successful local request-write completion and complete response-header receipt with channel, model, stream mode, retry index, write sequence, status, protocol, and elapsed durations.
  - Treats request-write completion as a local transport event rather than an upstream acknowledgement; an upstream early response may be logged before request writing finishes.
  - Reads late HTTP trace callbacks through a synchronized snapshot so stream probes and the final timing summary can use the newest available attempt data.
  - Splits preflight `model_request` timing into request body storage, body bytes access, body decode, and body reset phases.
  - Logs request body size and whether body storage is memory-backed or disk-backed when timing diagnostics are enabled.
  - Adds optional early SSE line probes controlled by `RELAY_TIMING_LOG_STREAM_PROBE_COUNT`.
  - Keeps `relay request received` always on for the narrowed relay paths so request arrival can be correlated even when timing diagnostics are disabled.
  - Logs failed upstream attempt timing before relay retry overwrites the final attempt timing.
  - Logs timing fields, event kind, line length, path, client IP, model/channel/status, connection reuse data, and upstream write error text when present.
  - Covers requests sent through the shared relay `doRequest` path; SDK-managed and channel-specific auxiliary HTTP clients are outside these two transport event logs.
  - Does not log request bodies, response bodies, Authorization values, API keys, headers, or raw query strings; URL-like error text is masked before logging.

- `0012-record-actual-response-model-in-usage-logs.patch`
  - Captures the model reported by OpenAI Chat Completions and Responses responses/streams.
  - Stores the captured model in usage-log `other.actual_model_name` when it is the actual served model.
  - Keeps the existing mapped-model fallback for older logs and mapped channels.
  - Shows `actual_model_name` before the mapped-model fallback in the web usage-log UI.
  - Does not add or migrate any database columns.

- `0013-add-openai-alpha-search-relay.patch`
  - Extends the upstream `POST /v1/alpha/search` handler to allow ordinary OpenAI channels alongside Sub2API, Codex, and Advanced Custom channels.
  - Reuses the upstream Alpha Search request mapping, OpenAI adaptor, response passthrough, billing, logging, and retry behavior unchanged.
  - Lets an unsupported OpenAI-compatible provider return its upstream error and participate in the configured retry policy.
  - Does not add endpoint-specific channel filtering or Alpha Search test files.

- `0014-limit-log-cleanup-to-consumption-logs.patch`
  - Limits asynchronous history-log cleanup counting and deletion to consumption logs.
  - Preserves top-up, management, system, error, refund, login, and unknown log records regardless of their age.
  - Applies the same filter to regular databases and the ClickHouse manual cleanup mutation.
  - Leaves the cleanup API, task state, frontend, and separately configured ClickHouse automatic TTL unchanged.

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
