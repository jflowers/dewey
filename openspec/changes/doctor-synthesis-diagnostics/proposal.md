# Proposal: Add Synthesis Layer Diagnostics to dewey doctor

**Issue**: [#102](https://github.com/unbound-force/dewey/issues/102)
**Provenance**: OpenSpec `synthesis-endpoint-env` risk R3 (deferred)

## Why

The `dewey doctor` command reports comprehensive diagnostics for the
Embedding Layer (endpoint, Ollama state, model availability, embedding
count) but has zero coverage for the Synthesis Layer. Since #71
decoupled synthesis and embedding endpoints via `DEWEY_SYNTHESIS_ENDPOINT`,
users configuring dual-provider setups (e.g., Ollama for embedding,
Vertex AI for synthesis) have no way to verify synthesis configuration
short of attempting a `dewey compile` and observing failure.

This gap was explicitly acknowledged during the `synthesis-endpoint-env`
OpenSpec as risk R3 and deferred with the note: "A separate issue
should be filed if needed."

## What Changes

Add a "Synthesis Layer" section to `dewey doctor` output, inserted
between the existing "Embedding Layer" and "MCP Server" sections in
`runDoctorChecks()`. The section mirrors the embedding diagnostics
pattern and reports:

1. **Resolved endpoint** — the synthesis endpoint after precedence
   resolution (config.yaml > DEWEY_SYNTHESIS_ENDPOINT > OLLAMA_HOST >
   default)
2. **Provider type** — `ollama`, `vertex`, or unconfigured
3. **Model** — the configured synthesis model
4. **Connectivity** — for Ollama: HTTP health check via existing
   `ollamaHealthCheck()`; for Vertex: credential availability check
   (config completeness, no live API call)
5. **Model availability** — for Ollama: model presence via
   `Available()`; for Vertex: config validation (project + model set)

## Capabilities

- Users can verify synthesis provider configuration without
  attempting a compile or curate operation
- Dual-provider setups (different endpoints for embedding vs
  synthesis) are fully diagnosable
- Zero-config state is clearly reported ("not configured")

## Impact

- **Files modified**: `cli.go` (~30-50 lines added to
  `runDoctorChecks()`), test files
- **New packages**: none
- **New MCP tools**: none
- **Schema changes**: none
- **Breaking changes**: none — additive diagnostic output only
- **Performance**: one additional HTTP check with 5s timeout (Ollama)
  or no network call (Vertex/unconfigured)

## Constitution Alignment

### I. Composability First — PASS

No new dependencies introduced. Doctor continues to work with any
vault configuration. Synthesis diagnostics degrade gracefully when
no provider is configured.

### II. Autonomous Collaboration — N/A

No MCP tool changes. Existing tool contracts unchanged.

### III. Observable Quality — PASS

Directly addresses this principle. The system becomes fully auditable
for both provider stacks (embedding + synthesis).

### IV. Testability — PASS

Existing test patterns (`io.Writer` output assertions,
`NoopSynthesizer` double, mock Ollama server) provide complete
test templates. No external services required for testing. Vertex
connectivity check uses config validation only (no GCP auth in tests).
