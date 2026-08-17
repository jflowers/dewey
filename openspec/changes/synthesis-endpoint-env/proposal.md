## Why

The synthesis provider in `llm/config.go` reads `DEWEY_EMBEDDING_ENDPOINT` (lines 85 and 117) to resolve its Ollama endpoint, conflating embedding and synthesis concerns. This prevents users from running embedding and synthesis on separate Ollama instances via environment variables — a real deployment scenario (different GPUs, different machines, different resource profiles).

The embedding path (`embed/config.go`) already has a well-structured resolution chain (`DEWEY_EMBEDDING_ENDPOINT` → `OLLAMA_HOST` → default) via `ResolveOllamaEndpoint()`. The synthesis path must have its own equivalent.

This was explicitly deferred as a non-goal during the `ollama-host-fallback` OpenSpec change (design.md lines 22, 25, 141-146) and tracked as GitHub issue #71.

## What Changes

- Introduce `DEWEY_SYNTHESIS_ENDPOINT` environment variable for synthesis endpoint resolution
- Add `ResolveSynthesisEndpoint()` function in `llm/config.go` mirroring `embed.ResolveOllamaEndpoint()`
- Update both synthesis config paths (legacy `compile_model` at line 85, `synthConfigFromEnv` at line 117) to use the new resolver
- Add `OLLAMA_HOST` fallback to the synthesis endpoint resolution chain (parity with embedding)
- Document the new env var in `AGENTS.md`

## Capabilities

### New Capabilities
- `DEWEY_SYNTHESIS_ENDPOINT`: Environment variable that controls the synthesis provider's Ollama endpoint independently from the embedding endpoint

### Modified Capabilities
- `ReadSynthesisConfig()`: Uses `DEWEY_SYNTHESIS_ENDPOINT` instead of `DEWEY_EMBEDDING_ENDPOINT` for endpoint resolution
- `synthConfigFromEnv()`: Same change — reads `DEWEY_SYNTHESIS_ENDPOINT` with `OLLAMA_HOST` fallback

### Removed Capabilities
- None

## Impact

- **`llm/config.go`**: Two call sites modified (lines 85, 117) + new `ResolveSynthesisEndpoint()` function
- **`llm/provider_test.go`**: New endpoint resolution tests (6-7 cases mirroring `embed/config_test.go`)
- **`AGENTS.md`**: New synthesis endpoint resolution documentation
- **Backward compatibility**: Users who don't set `DEWEY_SYNTHESIS_ENDPOINT` get the same default behavior. Users who set `DEWEY_EMBEDDING_ENDPOINT` will no longer have it silently affect synthesis — this is the intended behavioral correction.
- **No schema changes**: `config.yaml` `synthesis.endpoint` already works correctly and is unaffected.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Composability First

**Assessment**: PASS

This change improves composability by allowing independent configuration of embedding and synthesis endpoints via env vars. Users can run Dewey with separate Ollama instances for different workloads without being forced to use `config.yaml`. The fix removes an unintended coupling between two independent concerns.

### II. Autonomous Collaboration

**Assessment**: N/A

No MCP tool contracts are modified. The synthesis endpoint resolution is an internal configuration concern that does not affect tool input/output schemas or inter-agent communication.

### III. Observable Quality

**Assessment**: PASS

This change directly improves observability — the env var name `DEWEY_SYNTHESIS_ENDPOINT` accurately describes what it controls, unlike the current state where `DEWEY_EMBEDDING_ENDPOINT` silently controls synthesis behavior. Users can now audit their configuration and understand which env var controls which provider.

### IV. Testability

**Assessment**: PASS

The fix follows the existing test pattern from `embed/config_test.go` (8 endpoint resolution tests). New tests will cover the full synthesis endpoint resolution chain using `t.Setenv()` — no external services required. The existing test gap (endpoint value never asserted) will be closed.
