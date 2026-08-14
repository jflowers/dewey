## Why

The Vertex AI URL builders in both `llm/vertex.go` and `embed/vertex.go` construct endpoint URLs by prepending `{region}-` to `aiplatform.googleapis.com`. This works for regional endpoints (e.g., `us-east5-aiplatform.googleapis.com`) but produces an invalid URL when `region` is set to `global`: `global-aiplatform.googleapis.com` instead of `aiplatform.googleapis.com`.

The `global` endpoint is a valid and useful Vertex AI location — it routes requests to the nearest available region. Users configuring `region: global` in their `config.yaml` get silent failures (DNS resolution errors or unexpected HTTP errors) with no indication that the URL is malformed.

Related: PR #125 in `unbound-force/unbound-force` addressed the same `global` region issue in the gateway's Vertex provider by rejecting it with an error. Dewey should instead support it properly, since Dewey's Vertex providers use `rawPredict`/`predict` endpoints that do work with the global endpoint when the URL is constructed correctly.

## What Changes

Fix the URL construction in both Vertex providers to detect `region == "global"` and use `aiplatform.googleapis.com` (no region prefix) instead of `global-aiplatform.googleapis.com`.

## Capabilities

### New Capabilities
- None

### Modified Capabilities
- `VertexSynthesizer.rawPredictURL()`: Correctly handles `global` region by omitting the region subdomain prefix
- `VertexEmbedder.predictURL()`: Same fix — correctly handles `global` region

### Removed Capabilities
- None

## Impact

- **Files**: `llm/vertex.go`, `embed/vertex.go`, `llm/vertex_test.go`, `embed/vertex_test.go`
- **Behavior**: Users can now set `region: global` in their Vertex AI config and get working synthesis/embedding. Regional endpoints (e.g., `us-east5`) continue to work identically.
- **Config**: Update `.uf/dewey/config.yaml` with correct project/region values for local development.
- **Backward compatible**: No API changes. Existing regional configurations are unaffected.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: N/A

This is a bug fix to URL construction logic. No changes to artifact-based communication or MCP tool interfaces.

### II. Composability First

**Assessment**: PASS

Dewey remains independently installable. The fix improves standalone functionality by supporting a broader range of valid Vertex AI configurations without external workarounds.

### III. Observable Quality

**Assessment**: PASS

No changes to output format or provenance metadata. The fix ensures Vertex AI requests reach the correct endpoint, improving reliability of synthesis and embedding operations.

### IV. Testability

**Assessment**: PASS

New unit tests will verify URL construction for both `global` and regional endpoints. Tests are isolated (no external service calls) and verify the specific URL format produced by each region value.
