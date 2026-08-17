## ADDED Requirements

### Requirement: Slash Command DCP Protection

Each slash command definition in `slash_commands.go` MUST wrap its Instructions section with `<protect>` and `</protect>` tags to prevent DCP context compression from removing behavioral content.

#### Scenario: DCP compression preserves protected sections

- **GIVEN** a slash command with `<protect>` tags around its Instructions section
- **WHEN** DCP compresses the conversation context during a long session
- **THEN** the content within `<protect>...</protect>` tags MUST be preserved verbatim

#### Scenario: Commands function normally without DCP

- **GIVEN** a slash command with `<protect>` tags rendered in an environment without DCP `protectTags` enabled
- **WHEN** an agent processes the slash command
- **THEN** the agent SHOULD ignore the `<protect>` tags and execute the command normally

### Requirement: Protection Scope

The `<protect>` tag MUST be placed immediately before the `## Instructions` heading in each command. The `</protect>` tag MUST be placed after the last line of the Instructions section, before the closing backtick of the Go string literal.

#### Scenario: dewey-store.md protection coverage

- **GIVEN** the `dewey-store.md` slash command definition
- **WHEN** `<protect>` tags are applied
- **THEN** the protected region MUST include all 5 instruction steps: Parse Input, Determine Mode, Analyze and Suggest, Multi-Learning Extraction, and Post-Store

#### Scenario: Simple command protection coverage

- **GIVEN** a simple slash command (`dewey-index.md`, `dewey-reindex.md`, `dewey-compile.md`, `dewey-curate.md`, or `dewey-lint.md`)
- **WHEN** `<protect>` tags are applied
- **THEN** the protected region MUST include all instruction content between the `## Instructions` heading and the end of the command

### Requirement: No Behavioral Change

The addition of `<protect>` tags MUST NOT alter the semantic content, structure, or behavior of any slash command. Tags are additive metadata only.

#### Scenario: Existing content preserved

- **GIVEN** the current content of each slash command in `slash_commands.go`
- **WHEN** `<protect>` tags are added
- **THEN** all existing text, headings, code blocks, and formatting MUST remain unchanged

## MODIFIED Requirements

None.

## REMOVED Requirements

None.
