## ADDED Requirements

### Requirement: Migration of Dewey slash commands from old directory

When `dewey init` detects the old `.opencode/command/` directory, it MUST migrate Dewey-owned slash command files to `.opencode/commands/` before scaffolding. Only files present in the `deweySlashCommands` map SHALL be migrated. Files already present in the new directory MUST NOT be overwritten by migrated copies.

#### Scenario: Old directory exists with Dewey commands, new directory does not exist

- **GIVEN** a vault with `.opencode/command/dewey-store.md` and no `.opencode/commands/` directory
- **WHEN** `dewey init` is run
- **THEN** `.opencode/commands/` is created, `dewey-store.md` is moved there, and scaffolding proceeds in `.opencode/commands/`

#### Scenario: Both directories exist with conflicting files

- **GIVEN** a vault with `.opencode/command/dewey-index.md` (old version) and `.opencode/commands/dewey-index.md` (new version)
- **WHEN** `dewey init` is run
- **THEN** the file in `.opencode/commands/` is preserved (not overwritten), the old copy in `.opencode/command/` is removed, and scaffolding proceeds

#### Scenario: Old directory becomes empty after migration

- **GIVEN** a vault with `.opencode/command/` containing only Dewey slash command files
- **WHEN** `dewey init` migrates all files to `.opencode/commands/`
- **THEN** the empty `.opencode/command/` directory SHOULD be removed

#### Scenario: Old directory contains non-Dewey files after migration

- **GIVEN** a vault with `.opencode/command/` containing both Dewey slash commands and non-Dewey files (e.g., `speckit.plan.md`)
- **WHEN** `dewey init` migrates Dewey files to `.opencode/commands/`
- **THEN** `.opencode/command/` MUST NOT be removed (it still contains non-Dewey files)

## MODIFIED Requirements

### Requirement: Slash command scaffolding target directory

`dewey init` MUST scaffold slash commands into `.opencode/commands/` (plural).

Previously: `dewey init` scaffolded slash commands into `.opencode/command/` (singular).

#### Scenario: Fresh initialization

- **GIVEN** a directory with `.opencode/` but no `.opencode/commands/` directory
- **WHEN** `dewey init` is run
- **THEN** `.opencode/commands/` is created and all `deweySlashCommands` entries are written there

#### Scenario: Re-initialization with existing commands

- **GIVEN** a vault with `.opencode/commands/dewey-store.md` already present
- **WHEN** `dewey init` is run
- **THEN** existing `dewey-store.md` is not overwritten, and any missing commands are scaffolded

#### Scenario: No OpenCode directory

- **GIVEN** a directory without `.opencode/`
- **WHEN** `dewey init` is run
- **THEN** slash command scaffolding is skipped entirely (no `.opencode/commands/` directory is created)

## REMOVED Requirements

None.
