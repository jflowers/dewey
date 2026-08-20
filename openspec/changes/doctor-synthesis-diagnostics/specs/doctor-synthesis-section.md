## ADDED Requirements

### Requirement: Synthesis Layer diagnostic section

The `dewey doctor` command MUST include a "Synthesis Layer" section
that reports synthesis provider diagnostics. The section MUST appear
after the "Embedding Layer" section and before the "MCP Server"
section.

The section MUST report:
1. **Provider type** — `ollama`, `vertex`, or unconfigured
2. **Resolved endpoint** — the synthesis endpoint after precedence
   resolution
3. **Model** — the configured synthesis model identifier
4. **Connectivity status** — whether the synthesis endpoint is
   reachable (Ollama only)
5. **Model/credential availability** — whether the model is
   available (Ollama) or config is complete (Vertex)

#### Scenario: Ollama synthesis provider configured and reachable

- **GIVEN** a vault with synthesis configured as provider `ollama`
  with model `llama3.2:3b` and the Ollama instance is running
- **WHEN** the user runs `dewey doctor`
- **THEN** the output MUST include a "Synthesis Layer" section header
  with the model and endpoint
- **AND** the connectivity check MUST report PASS with "running
  (external)"
- **AND** the model availability check MUST report PASS with the
  model name and "ready"

#### Scenario: Ollama synthesis provider configured but unreachable

- **GIVEN** a vault with synthesis configured as provider `ollama`
  but the Ollama instance is not running
- **WHEN** the user runs `dewey doctor`
- **THEN** the connectivity check MUST report WARN with "not running"
  or PASS with "not installed (optional)"
- **AND** the model availability check MUST report WARN with "skipped
  (ollama not reachable)"

#### Scenario: Vertex synthesis provider configured

- **GIVEN** a vault with synthesis configured as provider `vertex`
  with project, region, and model set
- **WHEN** the user runs `dewey doctor`
- **THEN** the output MUST include a "Synthesis Layer" section header
  with the model and provider type
- **AND** the provider check MUST report PASS with "vertex" and the
  project/region
- **AND** doctor MUST NOT make live API calls to Vertex AI

#### Scenario: Vertex synthesis provider misconfigured

- **GIVEN** a vault with synthesis configured as provider `vertex`
  but missing required fields (project or region)
- **WHEN** the user runs `dewey doctor`
- **THEN** the configuration check MUST report FAIL indicating which
  required field is missing

#### Scenario: No synthesis provider configured

- **GIVEN** a vault with no synthesis configuration (no config.yaml
  synthesis section, no env vars)
- **WHEN** the user runs `dewey doctor`
- **THEN** the output MUST include a "Synthesis Layer" section
- **AND** the section MUST report PASS with "not configured
  (optional)" status
- **AND** doctor MUST NOT report this as a failure or warning

### Requirement: Token safety in diagnostic output

The `dewey doctor` synthesis section MUST NOT display OAuth tokens,
API keys, or credentials in its output. Only configuration metadata
(endpoint URLs, project IDs, region names, model names) MAY be
displayed.

#### Scenario: Vertex provider output contains no credentials

- **GIVEN** a vault with synthesis configured as provider `vertex`
- **WHEN** the user runs `dewey doctor`
- **THEN** the output MUST include project ID and region
- **AND** the output MUST NOT include any OAuth token, bearer token,
  or API key values

## MODIFIED Requirements

_None._

## REMOVED Requirements

_None._
<!-- scaffolded by uf vdev -->
