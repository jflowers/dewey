## ADDED Requirements

### Requirement: DCP Config Scaffolding

`dewey init` MUST scaffold `.opencode/dcp.jsonc` with `compress.protectTags: true` when the `.opencode/` directory exists and no DCP config file with `protectTags` is already present.

The scaffolded file MUST contain:
- A `$schema` field referencing the DCP JSON schema
- A JSONC comment explaining the file's purpose
- `compress.protectTags` set to `true`

#### Scenario: Scaffold DCP config in new project
- **GIVEN** a vault directory with an `.opencode/` directory and no `.opencode/dcp.jsonc` or `.opencode/dcp.json` file
- **WHEN** the user runs `dewey init`
- **THEN** `.opencode/dcp.jsonc` SHALL be created with `compress.protectTags: true`

#### Scenario: Skip scaffolding when DCP config exists with protectTags
- **GIVEN** a vault directory with `.opencode/dcp.jsonc` containing `protectTags`
- **WHEN** the user runs `dewey init`
- **THEN** the existing file SHALL NOT be modified or overwritten

#### Scenario: Warn when DCP config exists without protectTags
- **GIVEN** a vault directory with `.opencode/dcp.jsonc` that does not contain `protectTags`
- **WHEN** the user runs `dewey init`
- **THEN** the command SHALL log a warning indicating `protectTags` is missing, with a fix hint describing how to add the setting
- **AND** the existing file SHALL NOT be modified or overwritten

### Requirement: Composability Guard

`dewey init` MUST NOT scaffold `.opencode/dcp.jsonc` when the `.opencode/` directory does not exist. The init command MUST NOT create the `.opencode/` directory.

#### Scenario: No DCP config when no OpenCode directory
- **GIVEN** a vault directory without an `.opencode/` directory
- **WHEN** the user runs `dewey init`
- **THEN** no `.opencode/dcp.jsonc` file SHALL be created

### Requirement: DCP Config Doctor Check

`dewey doctor` MUST include a diagnostic check for DCP `protectTags` configuration when the `.opencode/` directory exists.

The check MUST report:
- **PASS** when `.opencode/dcp.jsonc` or `.opencode/dcp.json` contains `protectTags`
- **WARN** when a DCP config file exists but does not contain `protectTags`
- **WARN** when `.opencode/` exists but no DCP config file is present, with a fix hint referencing `dewey init`

The check SHOULD be silent (no output) when `.opencode/` does not exist.

#### Scenario: Doctor reports PASS for correct DCP config
- **GIVEN** a vault directory with `.opencode/dcp.jsonc` containing `protectTags`
- **WHEN** the user runs `dewey doctor`
- **THEN** the output SHALL include a PASS check for DCP config

#### Scenario: Doctor reports WARN for missing DCP config
- **GIVEN** a vault directory with `.opencode/` but no DCP config file
- **WHEN** the user runs `dewey doctor`
- **THEN** the output SHALL include a WARN check for DCP config with a fix hint

#### Scenario: Doctor reports WARN for DCP config without protectTags
- **GIVEN** a vault directory with `.opencode/dcp.jsonc` that does not contain `protectTags`
- **WHEN** the user runs `dewey doctor`
- **THEN** the output SHALL include a WARN check indicating `protectTags` is missing

#### Scenario: Doctor is silent when no OpenCode directory
- **GIVEN** a vault directory without an `.opencode/` directory
- **WHEN** the user runs `dewey doctor`
- **THEN** the output SHALL NOT include any DCP-related checks

## MODIFIED Requirements

None.

## REMOVED Requirements

None.
