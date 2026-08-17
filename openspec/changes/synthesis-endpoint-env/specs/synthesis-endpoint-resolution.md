## ADDED Requirements

### Requirement: Synthesis endpoint env var

The synthesis provider MUST support a dedicated `DEWEY_SYNTHESIS_ENDPOINT` environment variable for endpoint resolution, independent of `DEWEY_EMBEDDING_ENDPOINT`.

#### Scenario: Synthesis endpoint override

- **GIVEN** `DEWEY_SYNTHESIS_ENDPOINT` is set to `http://gpu-b:11434`
- **WHEN** `ReadSynthesisConfig()` resolves the synthesis endpoint
- **THEN** the resolved endpoint MUST be `http://gpu-b:11434`

#### Scenario: Synthesis endpoint does not affect embedding

- **GIVEN** `DEWEY_SYNTHESIS_ENDPOINT` is set to `http://gpu-b:11434`
- **AND** `DEWEY_EMBEDDING_ENDPOINT` is not set
- **WHEN** `embed.ResolveOllamaEndpoint()` resolves the embedding endpoint
- **THEN** the embedding endpoint MUST NOT be `http://gpu-b:11434`

#### Scenario: Embedding endpoint does not affect synthesis

- **GIVEN** `DEWEY_EMBEDDING_ENDPOINT` is set to `http://gpu-a:11434`
- **AND** `DEWEY_SYNTHESIS_ENDPOINT` is not set
- **WHEN** `ReadSynthesisConfig()` resolves the synthesis endpoint
- **THEN** the resolved endpoint MUST NOT be `http://gpu-a:11434`
- **AND** the resolved endpoint MUST be the default (`http://localhost:11434`)

### Requirement: Synthesis endpoint fallback chain

The synthesis endpoint resolution MUST follow this precedence chain (highest to lowest):

1. `DEWEY_SYNTHESIS_ENDPOINT` env var
2. `OLLAMA_HOST` env var (ecosystem-standard fallback)
3. `http://localhost:11434` (default)

#### Scenario: OLLAMA_HOST fallback for synthesis

- **GIVEN** `DEWEY_SYNTHESIS_ENDPOINT` is not set
- **AND** `OLLAMA_HOST` is set to `http://remote:11434`
- **WHEN** `ReadSynthesisConfig()` resolves the synthesis endpoint
- **THEN** the resolved endpoint MUST be `http://remote:11434`

#### Scenario: DEWEY_SYNTHESIS_ENDPOINT wins over OLLAMA_HOST

- **GIVEN** `DEWEY_SYNTHESIS_ENDPOINT` is set to `http://gpu-b:11434`
- **AND** `OLLAMA_HOST` is set to `http://remote:11434`
- **WHEN** `ReadSynthesisConfig()` resolves the synthesis endpoint
- **THEN** the resolved endpoint MUST be `http://gpu-b:11434`

#### Scenario: Default when nothing set

- **GIVEN** neither `DEWEY_SYNTHESIS_ENDPOINT` nor `OLLAMA_HOST` is set
- **WHEN** `ReadSynthesisConfig()` resolves the synthesis endpoint
- **THEN** the resolved endpoint MUST be `http://localhost:11434`

### Requirement: Synthesis endpoint scheme normalization

When `DEWEY_SYNTHESIS_ENDPOINT` or `OLLAMA_HOST` is set without a URL scheme (e.g., `0.0.0.0:11434`), the resolver MUST prepend `http://` automatically.

#### Scenario: No-scheme normalization

- **GIVEN** `DEWEY_SYNTHESIS_ENDPOINT` is set to `0.0.0.0:11434`
- **WHEN** `ReadSynthesisConfig()` resolves the synthesis endpoint
- **THEN** the resolved endpoint MUST be `http://0.0.0.0:11434`

#### Scenario: HTTPS preserved

- **GIVEN** `DEWEY_SYNTHESIS_ENDPOINT` is set to `https://secure:11434`
- **WHEN** `ReadSynthesisConfig()` resolves the synthesis endpoint
- **THEN** the resolved endpoint MUST be `https://secure:11434`

#### Scenario: Empty treated as unset

- **GIVEN** `DEWEY_SYNTHESIS_ENDPOINT` is set to `""`
- **AND** `OLLAMA_HOST` is set to `http://fallback:11434`
- **WHEN** `ReadSynthesisConfig()` resolves the synthesis endpoint
- **THEN** the resolved endpoint MUST be `http://fallback:11434`

### Requirement: Both config paths use new resolver

Both synthesis config paths — the legacy `compile_model` path and the `synthConfigFromEnv()` fallback — MUST use `ResolveSynthesisEndpoint()` for endpoint resolution.

#### Scenario: Legacy compile_model path

- **GIVEN** a config file with `compile_model: llama3.2:3b` (legacy format)
- **AND** `DEWEY_SYNTHESIS_ENDPOINT` is set to `http://gpu-b:11434`
- **WHEN** `ReadSynthesisConfig()` is called
- **THEN** the resolved endpoint MUST be `http://gpu-b:11434`

#### Scenario: Env-only fallback path

- **GIVEN** no config file exists
- **AND** `DEWEY_SYNTHESIS_ENDPOINT` is set to `http://gpu-b:11434`
- **WHEN** `ReadSynthesisConfig()` is called
- **THEN** the resolved endpoint MUST be `http://gpu-b:11434`

## MODIFIED Requirements

### Requirement: Synthesis endpoint resolution precedence

The synthesis endpoint MUST be resolved with the following precedence (highest to lowest):

1. `config.yaml` `synthesis.endpoint` field (per-vault, then global)
2. `DEWEY_SYNTHESIS_ENDPOINT` env var (app-specific override)
3. `OLLAMA_HOST` env var (ecosystem-standard fallback)
4. `http://localhost:11434` (default)

Previously: The synthesis endpoint was resolved using `DEWEY_EMBEDDING_ENDPOINT` instead of a dedicated env var, and had no `OLLAMA_HOST` fallback.

Note: `config.yaml` takes highest precedence because it is the most explicit, user-intentional configuration. Env vars serve as the fallback for config-file-less deployments (CI, containers).

## REMOVED Requirements

None.
