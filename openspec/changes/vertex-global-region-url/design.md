## Context

Both Vertex AI providers (`VertexSynthesizer` in `llm/vertex.go` and `VertexEmbedder` in `embed/vertex.go`) construct endpoint URLs using the pattern `https://{region}-aiplatform.googleapis.com/...`. This format is correct for regional endpoints (e.g., `us-east5`) but invalid for the `global` endpoint, which uses `https://aiplatform.googleapis.com/...` without a region prefix.

The `locations/{region}` path segment in the URL correctly uses `global` in both cases — the bug is only in the hostname construction.

## Goals / Non-Goals

### Goals
- Support `region: global` in Vertex AI config for both synthesis and embedding
- Maintain identical behavior for existing regional endpoints
- Add test coverage for `global` region URL construction

### Non-Goals
- Validating whether a given region string is a real GCP region
- Changing the Vertex AI config schema or adding new config fields
- Updating the `unbound-force` gateway (that repo has its own fix via PR #125)

## Decisions

### D1: Conditional hostname construction (not a separate URL template)

**Decision**: Add a conditional check in each URL builder method: if `region == "global"`, use `aiplatform.googleapis.com` as the hostname; otherwise, use `{region}-aiplatform.googleapis.com`.

**Rationale**: This is the minimal change that fixes the bug. An alternative would be extracting a shared helper function, but since the two URL builders are in different packages (`llm/` and `embed/`) with slightly different URL paths (`rawPredict` vs `predict`, `anthropic` vs `google` publisher), a shared helper would add coupling without meaningful deduplication. Each method stays self-contained and readable.

**Constitution alignment**: Composability First — each package remains independently testable with no cross-package dependency introduced.

### D2: Extract helper variable, not a function

**Decision**: Use a local `host` variable in each URL builder rather than extracting a `vertexHost(region string)` function.

**Rationale**: The logic is two lines. A function would be over-engineering for a simple conditional. If more region-specific logic emerges later, extraction can happen then.

### D3: Config update is out of scope for the code fix

**Decision**: The `.uf/dewey/config.yaml` update (setting correct project/region) is a local configuration change, not a code change. It will be done as a separate task but is not part of the tested/reviewed code change.

## Coverage Strategy

- **Type**: Unit tests only (no integration or e2e needed — URL construction is a pure function with no I/O)
- **Target**: 100% branch coverage of the `rawPredictURL()` and `predictURL()` methods
- **Regression**: Both `global` and regional cases covered per method to prevent regression (TC-006)

## Risks / Trade-offs

- **Risk**: The `global` endpoint may not support all Vertex AI model types or APIs. **Mitigation**: This is a Google API concern, not a Dewey concern. If Google doesn't support a model in the global endpoint, the user will get a clear HTTP error from Google, not a DNS failure from a malformed URL.
- **Trade-off**: No shared helper between `llm/` and `embed/` means the fix is duplicated in two places. Accepted because the duplication is minimal (3 lines) and avoids cross-package coupling.
