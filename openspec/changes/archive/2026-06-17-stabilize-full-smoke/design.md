## Context

`tests/smoke/run.sh` grew incrementally with each shipped capability, but its ingestion fixture remained `anthropics/anthropic-cookbook`. That repository currently produces 333 documents and roughly 3,458 chunks. Since `add-knowledge-graph`, every non-code chunk invokes synchronous chat extraction after vector upsert. The default smoke therefore makes thousands of external requests, inherits provider latency and cost, and cannot reach its assertions in a useful feedback window.

The development compose profile also sets `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318` without defining that service. Every span batch retries DNS resolution before failing. Separately, unconstrained `qdrant-client>=1.12` resolved to 1.18 while the server remains 1.12.4, producing a compatibility warning on each client construction.

The smoke is a mandatory release gate. It must validate the integrated behavior of the stack, but it must not depend on the size or availability of an external repository or paid model provider.

## Goals / Non-Goals

**Goals:**

- Complete the default full smoke in five minutes or less on an already-built healthy development stack.
- Exercise ingestion, embeddings, vector retrieval, audit, deterministic code graph extraction, LLM-shaped text extraction, graph review, traversal, promotion, and graph audit.
- Make default smoke runtime and model usage deterministic and locally reproducible.
- Bound every smoke subprocess and terminate descendants when a deadline or assertion fails.
- Validate that expected OTel spans reach a real local collector.
- Isolate assertions and mutations to records created by the current smoke run.
- Provide a separate, opt-in and strictly bounded real-cloud canary.

**Non-Goals:**

- Moving text extraction out of the synchronous ingestion loop.
- Implementing a durable extraction queue, retry scheduler, or dead-letter store.
- Load, soak, or corpus-quality testing.
- Replacing Ollama Cloud or changing production provider defaults.
- Adding the complete production observability stack from `add-foundation`.

## Decisions

### D1: Commit fixture source files and construct a local Git repository per run

The repository will contain a small source tree under `tests/smoke/fixtures/knowledge-repo/`, including prose with a unique retrieval phrase and code with multiple symbols and dependencies. The smoke runner will copy those files into the ingestion container, initialize and commit a local Git repository, and ingest its local path.

The generated repository directory includes the smoke run ID. Because the connector derives `source_uri` from the repository basename, every run receives a unique `git://hive-mind-smoke-<run-id>/...` namespace. Assertions can therefore identify current-run entities and evidence precisely.

Alternative considered: keep using a small public GitHub repository. Rejected because external mutation, network availability, and rate limits still make the release gate nondeterministic.

### D2: Use an Ollama-compatible deterministic stub for default text extraction

A small HTTP service in the smoke compose profile will implement the `/api/chat` response shape used by `OllamaChat`. It will return schema-valid concepts and a `related_to` candidate edge with fixed token counts. The ingestion command will override only the chat base URL/model/key for its process; embeddings continue through the real local Ollama service.

This validates the provider adapter, structured extraction parser, token accounting, graph writer, and review workflow without requiring an external key. It is not a mock inside application code: traffic still crosses HTTP and uses the production client.

Alternative considered: disable text extraction in smoke and rely on existing graph rows. Rejected because it would stop testing the feature that caused the regression and make graph assertions state-dependent.

### D3: Keep real-provider verification as an opt-in bounded canary

`SMOKE_CHAT_MODE=cloud` will select the configured cloud endpoint and require a key. The canary will ingest at most one prose document and two chunks using new CLI bounds, then assert a successful provider response, parseable extraction result, and non-zero token accounting. It will retain the same overall deadline.

The default mode remains `stub`; CI and routine pre-push checks MUST NOT call a paid provider.

Alternative considered: run one cloud request during every smoke. Rejected because credentials, provider availability, latency, and spend would remain mandatory release dependencies.

### D4: Add explicit ingestion bounds shared by the CLI runner

The Git CLI will accept `--max-documents` and `--max-chunks`. Limits are optional and absent by default, preserving normal ingestion. Selection is deterministic in connector discovery order. Reaching `max_documents` stops yielding new documents; reaching `max_chunks` stops processing before another chunk is inserted or embedded.

These controls are useful for smoke and provider probes without adding a test-only branch to the ingestion pipeline. The final CLI summary will report whether a bound truncated the run.

### D5: Implement process deadlines and cleanup in the shell harness

The smoke runner will define:

- An overall default deadline of 300 seconds, configurable by `SMOKE_TIMEOUT_SECONDS`.
- Per-stage deadlines, including clone/fixture setup, ingestion, retrieval, and telemetry flush.
- A trap on `EXIT`, `INT`, and `TERM` that kills the active subprocess and removes the in-container fixture repository.
- Stage start/end timestamps and durations.

The timeout helper will use Bash process management rather than assuming GNU `timeout`, which is not available by default on macOS.

### D6: Add a local OTel Collector and verify named spans through collector logs

The dev profile will include a pinned OpenTelemetry Collector Contrib service with OTLP/HTTP enabled, a health endpoint, and the debug exporter. Application services will depend on collector health where they export telemetry.

The smoke records its start timestamp, waits for BatchSpanProcessor flush, then inspects collector logs since that timestamp for:

- `service.name: hive-mind-ingestion`
- `pipeline.graph_extract_code`
- `pipeline.graph_extract_text`

Offline operators may set `OTEL_EXPORTER_OTLP_ENDPOINT=none`; in that explicit mode the smoke skips collector assertions with a visible notice. The checked-in default is a working collector, not a dead hostname.

Alternative considered: only disable OTel during smoke. Rejected because observability is a project requirement and the current missing service is a deployment defect.

### D7: Pin Qdrant client and server to the same minor compatibility line

Both Python services will constrain `qdrant-client` to `>=1.12,<1.13`, matching `qdrant/qdrant:v1.12.4`. The lockfile will be regenerated. Readiness/smoke will fail if the client emits an incompatibility condition rather than allowing warning spam to mask drift.

Upgrading server and client together can be proposed later. This change chooses the smallest compatibility correction.

### D8: Query current-run records before mutating them

The smoke will use the unique source URI prefix to select:

- A current-run entity to tombstone.
- A candidate edge whose evidence references a current-run chunk.
- A current-run concept for traversal.

It will then exercise public HTTP endpoints with those IDs. It MUST NOT choose the first global entity or candidate returned by an unfiltered endpoint.

## Risks / Trade-offs

- **The deterministic stub cannot measure real model quality.** → Keep the optional cloud canary and unit tests for malformed/timeout responses.
- **Collector debug logs are a test interface rather than a query API.** → Pin the collector image/config and match stable span/resource fields; a trace backend can replace this when the full observability stack lands.
- **Local Git fixture setup depends on Git inside the ingestion image.** → Git is already a production dependency of that image; add an explicit fixture-setup assertion.
- **Bounds could accidentally affect normal ingestion.** → Defaults remain unlimited and tests cover absent, document-limit, and chunk-limit behavior.
- **Qdrant pinning delays newer client features.** → No current code requires them; upgrade client and server together in a future dependency change.
- **Five minutes may be tight on first-time model/image downloads.** → The contract applies to an already-built healthy stack; bootstrap/download time remains outside the smoke timer and is documented separately.

## Migration Plan

1. Add the deterministic fixture, chat stub, collector config, and compose services.
2. Add ingestion bounds and unit tests.
3. Align Qdrant dependencies and regenerate `uv.lock`.
4. Rewrite the smoke runner around run isolation, deadlines, and telemetry assertions.
5. Update environment examples and operations documentation.
6. Rebuild the stack and run the default smoke twice to prove repeatability.
7. Run the optional cloud canary once with the configured key.

Rollback is limited to reverting the change. No database migration or persistent data conversion is required.

## Open Questions

- None blocking. The full production trace backend and asynchronous extraction architecture remain explicitly deferred.
