# Triage: Issue #102

> feat: add synthesis layer diagnostics to dewey doctor

## Verdict

| Dimension | Result |
|---|---|
| **Verdict** | VALID (5/5 unanimous) |
| **Category** | feature |
| **Objectivity** | objective (5/5 unanimous) |
| **Split** | none (5/5 unanimous) |
| **Duplicates** | none found |
| **Confidence** | HIGH |
| **Scope** | small |
| **Spec path** | OpenSpec |

## Panel Assessments

### Adversary

- **Verdict**: VALID (feature, objective)
- No new attack surface — extends existing diagnostic-only CLI command
- Mirrors established Embedding Layer pattern (`autoStart=false`, bounded timeouts)
- **Key concern**: Ensure OAuth tokens are never displayed for Vertex providers; report only configuration metadata (endpoint, project, region, model)
- SSRF not a concern — endpoints are locally configured by the user
- Existing `ollamaHealthCheck` with 2s timeout prevents resource exhaustion

### Architect

- **Verdict**: VALID (enhancement, objective)
- Strong structural symmetry with existing Embedding Layer section (cli.go:1466–1517)
- All building blocks exist: `llm.ReadSynthesisConfig()`, `ResolveSynthesisEndpoint()`, `OllamaSynthesizer.Available()`, `VertexSynthesizer.Available()`
- Maps to single function modification (`runDoctorChecks` in `cli.go`), ~30-50 lines
- Existing helpers reusable: `doctorCounter`, `printCheck`, `section()`, `ensureOllama()`
- Convention adherent: Cobra CLI, charmbracelet/log, standard library testing, no global state

### Guard

- **Verdict**: VALID (enhancement, objective)
- Directly supports Constitution Principle III (Observable Quality): system must be auditable
- Explicit follow-up from OpenSpec `synthesis-endpoint-env` risk R3 (deferred with "A separate issue should be filed if needed")
- Narrow, well-bounded scope — no gate modifications, no governance changes

### SRE

- **Verdict**: VALID (feature, objective)
- Real observability gap worsened post-#71 decoupling of synthesis/embedding endpoints
- Closes monitoring gap for `dewey compile`, `dewey curate`, and `store_compiled` MCP tool
- Minimal performance impact: one additional HTTP health check with 5s timeout
- `Available()` methods already implement caching
- Doctor is diagnostic-only (`autoStart=false`), no subprocess spawning risk

### Tester

- **Verdict**: VALID (feature, objective)
- HIGH testability rating
- Existing test patterns: `runDoctorChecks(w io.Writer, ...)` enables output assertions via `strings.Contains()`
- `NoopSynthesizer` test double already exists with `Avail` and `Model` fields
- Mock Ollama server pattern (`newMockOllamaServer()` in `main_test.go`) supports connectivity checks
- **Clarification areas** (implementation details, not validity concerns):
  1. Zero-config behavior: should section be omitted, INFO/PASS, or WARN?
  2. Vertex connectivity check isolation: `VertexSynthesizer.Available()` calls actual Vertex API, conflicting with Constitution IV (no external services in tests)
- LOW regression risk — additive section, does not modify existing sections

## Implementation Guidance

1. **Spec workflow**: OpenSpec recommended (tactical, single package, <3 user stories)
2. **Vertex connectivity**: Report config metadata only; skip live API probe (requires GCP auth, violates Constitution IV testability). Use `Available()` for Ollama; report config completeness for Vertex.
3. **Zero-config**: Gracefully report "not configured" when no synthesis provider is set
4. **Token safety**: Display endpoint, project, region, model — never OAuth tokens or credentials
5. **Test coverage**: Mirror existing embedding layer test patterns in `cli_test.go`/`main_test.go`; monitor CRAPload on `runDoctorChecks()` (~335 lines, adding ~30-50 more)
6. **Dual-endpoint probing**: If embedding and synthesis point to different Ollama instances, doctor needs two separate probes — one per endpoint

## Recommended Labels

- `triage/valid`
- `type/feature`
- `scope/small`
- `spec/openspec`

## Metadata

- **Triaged**: 2026-08-20
- **Issue author**: jflowers
- **Assignee**: yvonnedevlinrh
- **Existing labels**: `next-release`
- **Related**: #71 (DEWEY_SYNTHESIS_ENDPOINT), OpenSpec `synthesis-endpoint-env` risk R3
