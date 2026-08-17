## Context

The synthesis provider (`llm/config.go`) resolves its Ollama endpoint by reading `DEWEY_EMBEDDING_ENDPOINT` — the same env var used by the embedding provider (`embed/config.go`). This conflation was identified during the `ollama-host-fallback` OpenSpec and deferred as a separate issue (GitHub #71).

The embedding path already has the correct design: `embed.ResolveOllamaEndpoint()` implements a three-tier fallback chain (`DEWEY_EMBEDDING_ENDPOINT` → `OLLAMA_HOST` → `DefaultOllamaEndpoint`) with scheme normalization. The synthesis path needs an equivalent.

### Current State

```
Embedding endpoint resolution (embed/config.go):
  DEWEY_EMBEDDING_ENDPOINT → OLLAMA_HOST → http://localhost:11434
  ✅ Correct — dedicated env var, ecosystem fallback, default

Synthesis endpoint resolution (llm/config.go):
  DEWEY_EMBEDDING_ENDPOINT → http://localhost:11434
  ❌ Bug — reuses embedding env var, no OLLAMA_HOST fallback
```

## Goals / Non-Goals

### Goals
- Introduce `DEWEY_SYNTHESIS_ENDPOINT` env var for synthesis endpoint resolution
- Add `OLLAMA_HOST` fallback to the synthesis resolution chain (parity with embedding)
- Mirror the `embed.ResolveOllamaEndpoint()` pattern with scheme normalization
- Close the test gap — assert endpoint values in synthesis config tests
- Document the new env var in `AGENTS.md`

### Non-Goals
- Modifying `config.yaml` parsing — `synthesis.endpoint` already works correctly
- Adding synthesis diagnostics to `dewey doctor` — separate concern (noted by SRE triage but out of scope for this bug fix)
- Refactoring embedding and synthesis endpoint resolution into a shared helper — premature abstraction for two call sites

## Decisions

### D1: Mirror pattern, don't share code

**Decision**: Create `ResolveSynthesisEndpoint()` in `llm/config.go` that mirrors `embed.ResolveOllamaEndpoint()`, rather than extracting a shared helper.

**Rationale**: The embedding and synthesis packages are intentionally decoupled (Constitution I: Composability First). The `llm` package already imports `embed` only for `DefaultOllamaEndpoint`. Adding a dependency on an `embed.ResolveEndpoint()` helper would couple the synthesis config to the embedding package's internal resolution logic. Two simple functions in two packages is cleaner than one shared function with a cross-package dependency.

The two functions read different env vars (`DEWEY_EMBEDDING_ENDPOINT` vs `DEWEY_SYNTHESIS_ENDPOINT`) but share the `OLLAMA_HOST` fallback and `DefaultOllamaEndpoint` default. If a third provider emerges, extraction can be revisited.

### D2: Fallback chain matches embedding

**Decision**: `DEWEY_SYNTHESIS_ENDPOINT` → `OLLAMA_HOST` → `http://localhost:11434`

**Rationale**: Consistency with the embedding path reduces cognitive load. The `OLLAMA_HOST` fallback is the ecosystem-standard way to configure Ollama's endpoint. Users who set `OLLAMA_HOST` expect it to apply to all Ollama interactions unless specifically overridden. This was the explicit intent of the `ollama-host-fallback` OpenSpec, which deferred synthesis to this issue.

### D3: Reuse `embed.DefaultOllamaEndpoint` constant

**Decision**: Continue importing `embed.DefaultOllamaEndpoint` for the default value rather than duplicating the constant.

**Rationale**: The constant is a stable, well-named value (`http://localhost:11434`). The `llm` package already imports it. Duplicating it would create a drift risk. This import does not create behavioral coupling — it's a constant reference only.

### D4: Include scheme normalization

**Decision**: Apply the same `http://` scheme normalization that `embed.normalizeEndpoint()` performs. If `DEWEY_SYNTHESIS_ENDPOINT` or `OLLAMA_HOST` is set without a URL scheme (e.g., `0.0.0.0:11434`), prepend `http://`.

**Rationale**: Users who set `OLLAMA_HOST=0.0.0.0:11434` (without scheme) expect it to work. The embedding path handles this; the synthesis path must too for consistency. This is a minor normalization that prevents confusing HTTP client errors.

### D5: Behavioral change is intentional

**Decision**: After this change, setting `DEWEY_EMBEDDING_ENDPOINT` will no longer affect synthesis. This is the desired fix, not a side effect.

**Rationale**: The current behavior (embedding env var controlling synthesis) is the bug. Users who were unknowingly relying on this will now need to set `DEWEY_SYNTHESIS_ENDPOINT` separately. This aligns with Constitution III (Observable Quality) — env var names should accurately describe what they control.

## Risks / Trade-offs

### R1: Behavioral change for existing users (LOW)

Users who set `DEWEY_EMBEDDING_ENDPOINT` and rely on it also configuring synthesis will see synthesis revert to the default endpoint. Mitigation: document the change in release notes. The `config.yaml` path (`synthesis.endpoint`) is unaffected and remains the recommended approach for multi-instance setups.

### R2: Two nearly-identical resolution functions (ACCEPTED)

`ResolveSynthesisEndpoint()` and `embed.ResolveOllamaEndpoint()` will have similar structure but read different env vars. This is intentional duplication to avoid cross-package coupling (see D1). The functions are short (~15 lines each) and the risk of drift is low — both are simple env var reads with a fallback chain.

### R3: No `dewey doctor` synthesis section (DEFERRED)

The SRE triage noted that `dewey doctor` has no synthesis diagnostics. This is a valid gap but out of scope for a bug fix. A separate issue should be filed if needed.
