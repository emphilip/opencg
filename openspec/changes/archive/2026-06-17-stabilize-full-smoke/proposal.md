## Why

The full smoke test no longer behaves like a smoke test: its default repository expands to thousands of chunks, each non-code chunk can trigger a paid cloud extraction call, and missing telemetry infrastructure adds repeated retry delays. The latest run processed only about 160 chunks in 18 minutes and projected hours of runtime, making the mandatory pre-push verification unreliable and financially unbounded.

## What Changes

- Add a committed, deterministic Git fixture that exercises text ingestion, Graphifyy code extraction, vector retrieval, candidate relationship review, traversal, and audit behavior with fewer than 20 chunks.
- Make the default smoke independent of paid or external chat providers by adding an Ollama-compatible deterministic chat stub for structured graph-extraction responses.
- Add an optional cloud canary mode that processes a strictly bounded number of chunks and verifies real provider credentials, structured output, and token accounting without becoming part of the default smoke.
- Add hard overall and per-stage deadlines, child-process cleanup, stage timing, and actionable failure output to the smoke runner.
- Add an OpenTelemetry Collector to the development stack and verify that ingestion and graph-extraction spans are accepted during the smoke. Offline runs may explicitly disable export.
- Align the Qdrant client/server versions and fail readiness clearly when they are incompatible instead of emitting a warning throughout ingestion.
- Make smoke assertions target only data created by the current run, so repeated executions do not tombstone or promote unrelated catalogue records.
- Keep synchronous ingest-time extraction unchanged in this change. Decoupling extraction into a queue remains a separate architectural change.

## Capabilities

### New Capabilities

- `smoke-testing`: Defines the deterministic fixture, bounded execution contract, provider modes, observability checks, version compatibility, run isolation, and cleanup behavior for end-to-end verification.

### Modified Capabilities

- `ingestion`: Requires the Git connector and ingestion CLI to support a deterministic local fixture path and a bounded document/chunk selection used by smoke and provider canary runs.

## Impact

- `tests/smoke/`: new fixture repository, deterministic chat stub, smoke helpers, and a rewritten runner.
- `services/ingestion/`: bounded selection options shared by CLI and smoke/canary execution.
- `infra/compose/docker-compose.yml`: OpenTelemetry Collector and smoke chat-stub services/profiles, health checks, and service wiring.
- `infra/otel/`: local collector configuration with a testable trace sink.
- Python Qdrant dependency and compose image pins are aligned.
- `.env.example`, `Makefile`, and `docs/OPERATIONS.md`: document default deterministic smoke, optional cloud canary, deadlines, and offline telemetry.
- No production API contract, database schema, retrieval behavior, or synchronous extraction ordering changes.
