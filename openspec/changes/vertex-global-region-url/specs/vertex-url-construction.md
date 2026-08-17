## ADDED Requirements

### Requirement: Global region endpoint URL

When `region` is set to `"global"`, the Vertex AI endpoint URL MUST use `aiplatform.googleapis.com` as the hostname without a region prefix.

#### Scenario: Synthesis with global region
- **GIVEN** a `VertexSynthesizer` configured with `region: "global"`, `project: "my-project"`, and `model: "claude-opus-4-6"`
- **WHEN** `rawPredictURL()` is called
- **THEN** the returned URL MUST be `https://aiplatform.googleapis.com/v1/projects/my-project/locations/global/publishers/anthropic/models/claude-opus-4-6:rawPredict`

#### Scenario: Embedding with global region
- **GIVEN** a `VertexEmbedder` configured with `region: "global"`, `project: "my-project"`, and `model: "text-embedding-005"`
- **WHEN** `predictURL()` is called
- **THEN** the returned URL MUST be `https://aiplatform.googleapis.com/v1/projects/my-project/locations/global/publishers/google/models/text-embedding-005:predict`

#### Scenario: Regional synthesis endpoint unchanged
- **GIVEN** a `VertexSynthesizer` configured with `region: "us-east5"`, `project: "my-project"`, and `model: "claude-sonnet-4-6"`
- **WHEN** `rawPredictURL()` is called
- **THEN** the returned URL MUST be `https://us-east5-aiplatform.googleapis.com/v1/projects/my-project/locations/us-east5/publishers/anthropic/models/claude-sonnet-4-6:rawPredict`

#### Scenario: Regional embedding endpoint unchanged
- **GIVEN** a `VertexEmbedder` configured with `region: "us-central1"`, `project: "my-project"`, and `model: "text-embedding-005"`
- **WHEN** `predictURL()` is called
- **THEN** the returned URL MUST be `https://us-central1-aiplatform.googleapis.com/v1/projects/my-project/locations/us-central1/publishers/google/models/text-embedding-005:predict`

> **Note**: The `region == "global"` comparison is case-sensitive. GCP region names are conventionally lowercase. Mixed-case variants (e.g., `"Global"`, `"GLOBAL"`) are not specially handled and will produce a region-prefixed hostname, consistent with how all other non-global region strings are treated. Validating whether a region string is a real GCP region is a non-goal (see design.md).

> **Regression**: The "Synthesis with global region" and "Embedding with global region" scenarios above reproduce the original bug. Without the fix, these methods return URLs with `global-aiplatform.googleapis.com` (invalid hostname). With the fix, they return URLs with `aiplatform.googleapis.com` (correct). These scenarios serve as regression tests per TC-006.

## MODIFIED Requirements

None.

## REMOVED Requirements

None.
