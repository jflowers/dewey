## Context

The `dewey doctor` command provides diagnostic output organized into
sections: Environment, Workspace, Database, Content Sources, Content
Sanitization, Embedding Layer, and MCP Server. Each section uses the
`doctorCounter` + `printCheck()` pattern to report pass/warn/fail/info
checks.

Since spec 016-pluggable-providers and the synthesis-endpoint-env
OpenSpec (#71), embedding and synthesis are independently configurable
provider stacks with separate endpoints, models, and provider types.
However, `dewey doctor` only reports the Embedding Layer — leaving the
Synthesis Layer completely undiagnosed.

The existing Embedding Layer section (`cli.go:1466-1518`) demonstrates
the exact pattern to follow:
1. Read config → `embed.ReadEmbeddingConfig(deweyDir)`
2. Section header with model + endpoint
3. Ollama connectivity → `ensureOllama(endpoint, false, nil)`
4. Model availability → `embedder.Available()`
5. Additional checks (legacy model advisory, embedding count)

## Goals / Non-Goals

### Goals

- Add a "Synthesis Layer" section to `dewey doctor` between the
  existing Embedding Layer and MCP Server sections
- Report resolved endpoint, provider type, model, connectivity
  status, and model/credential availability
- Handle all three provider states: ollama, vertex, unconfigured
- Maintain test coverage parity with the Embedding Layer section

### Non-Goals

- Modifying the Synthesizer interface or adding new methods
- Adding synthesis diagnostics to the `health` MCP tool (separate
  concern)
- Live API calls to Vertex AI for connectivity checks (violates
  Constitution IV — no external services in tests)
- Exposing OAuth tokens or credentials in doctor output

## Decisions

### D1: Mirror the Embedding Layer pattern exactly

The synthesis section will follow the same structure as the embedding
section: read config, print section header, check connectivity,
check model availability. This provides consistency for users and
minimizes implementation risk.

**Rationale**: The embedding section is proven, tested, and familiar
to users. Diverging from the pattern would create unnecessary
cognitive load.

### D2: Provider-specific connectivity checks

- **Ollama provider**: Reuse `ensureOllama(synthEndpoint, false, nil)`
  for connectivity, then `OllamaSynthesizer.Available()` for model
  availability. This is identical to the embedding pattern.
- **Vertex provider**: Report config completeness (project, region,
  model set) without making a live API call. Use
  `NewSynthesizerFromConfig()` — if it returns an error, required
  fields are missing. If it succeeds, report "configured" status. Do
  NOT call `VertexSynthesizer.Available()` from doctor because it
  invokes `tokenFn(ctx)` which requires real GCP credentials.
- **Unconfigured**: When `ReadSynthesisConfig()` returns a zero-value
  `ProviderConfig` (empty provider, empty model), report "not
  configured (optional)" with PASS status. Synthesis is not required
  for core functionality (Composability First).

**Rationale**: Ollama connectivity is cheap (local HTTP GET). Vertex
connectivity requires OAuth credentials which may not be available in
all environments and would violate the diagnostic-only contract of
`dewey doctor`. Config completeness validation catches the most
common Vertex misconfiguration (missing project/region).

### D3: Construct synthesizer inline (no parameter injection)

Like the embedding section which constructs
`embed.NewOllamaEmbedder()` inline, the synthesis section will call
`llm.ReadSynthesisConfig()` + `llm.NewSynthesizerFromConfig()`
inline. No new parameters to `runDoctorChecks()`.

**Rationale**: Consistency with existing pattern. The `NoopSynthesizer`
test double is available for integration tests but isn't needed here —
tests use mock HTTP servers and environment variables, not injected
dependencies.

### D4: Section position

Insert the synthesis section immediately after the Embedding Layer
section (`cli.go:1518`) and before the MCP Server section
(`cli.go:1520`). The two provider sections are logically grouped.

## Risks / Trade-offs

### R1: Vertex credential validation gap (ACCEPTED)

By not calling `VertexSynthesizer.Available()`, doctor cannot verify
that GCP application-default credentials are actually valid. Users
with correct config but expired/missing credentials will see
"configured" in doctor but fail at `dewey compile` time. This is
the same trade-off the embedding section makes for Ollama model
availability when Ollama is unreachable (it reports "skipped" rather
than failing).

**Mitigation**: The doctor output clearly indicates it's reporting
configuration status, not full end-to-end readiness.

### R2: Large function growth (LOW)

`runDoctorChecks()` is ~335 lines and will grow by ~40-50 lines.
This is still within reasonable bounds for a sequential diagnostic
function. The function's complexity is linear (no branching depth
increase), so CRAPload impact is modest.

**Mitigation**: Adequate test coverage for all branches (ollama
reachable/unreachable, vertex configured/misconfigured, unconfigured).
<!-- scaffolded by uf vdev -->
