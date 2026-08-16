## ADDED Requirements

### Requirement: DCP protectTags Configuration

The repository MUST contain a `.opencode/dcp.jsonc` file that enables the `compress.protectTags` setting, activating DCP's support for `<protect>` tags during context compression.

#### Scenario: DCP reads protectTags configuration

- **GIVEN** the file `.opencode/dcp.jsonc` exists in the repository
- **WHEN** DCP initializes context compression for a session
- **THEN** DCP SHALL honor `<protect>` tags in slash command files, preserving their content from compression

#### Scenario: protectTags prevents compression of critical sections

- **GIVEN** `compress.protectTags` is set to `true` in `.opencode/dcp.jsonc`
- **WHEN** DCP compresses context during a long session
- **THEN** content within `<protect>...</protect>` blocks in `.opencode/commands/` files MUST be preserved verbatim

### Requirement: DCP Configuration File Format

The `.opencode/dcp.jsonc` file MUST be valid JSONC (JSON with Comments) and SHOULD include a `$schema` reference for editor validation.

#### Scenario: Valid JSONC with schema

- **GIVEN** a developer opens `.opencode/dcp.jsonc` in an editor with JSONC support
- **WHEN** the editor parses the file
- **THEN** the file MUST parse without errors and the `$schema` field SHOULD provide autocomplete and validation

### Requirement: Configuration Isolation

DCP configuration MUST NOT be placed in `opencode.json`. The `opencode.json` schema does not support DCP fields, and including them causes validation errors.

#### Scenario: Separate config files

- **GIVEN** `opencode.json` exists with OpenCode configuration
- **WHEN** DCP protectTags is configured
- **THEN** the setting MUST reside in `.opencode/dcp.jsonc`, not in `opencode.json`

## MODIFIED Requirements

None.

## REMOVED Requirements

None.
