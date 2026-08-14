## Why

UF tooling v0.15.0 (commit `3b9a878`, May 13, 2026) renamed the OpenCode slash command directory from `.opencode/command/` (singular) to `.opencode/commands/` (plural). However, `dewey init` was never updated to match — it still scaffolds Dewey-specific slash commands into the old `.opencode/command/` path.

This causes a split-brain state: `dewey init` writes to `.opencode/command/` while `/uf.init` and all UF tooling expect `.opencode/commands/`. Users end up with duplicate directories and potentially stale command files in the wrong location. The `/uf.init` command includes migration logic (Step 0) that handles this, but `dewey init` should write to the correct directory natively rather than relying on external migration.

## What Changes

- Update `dewey init` to scaffold slash commands into `.opencode/commands/` (plural) instead of `.opencode/command/` (singular).
- Add migration logic: when `dewey init` detects the old `.opencode/command/` directory with Dewey slash commands, move them to `.opencode/commands/` before scaffolding.
- Update all comments and test assertions to reflect the new path.

## Capabilities

### New Capabilities
- `dewey init` migration: Automatically migrates Dewey slash commands from `.opencode/command/` to `.opencode/commands/` when the old directory is detected.

### Modified Capabilities
- `dewey init` scaffolding: Writes slash commands to `.opencode/commands/` instead of `.opencode/command/`.

### Removed Capabilities
- None.

## Impact

- **Files affected**: `cli.go` (scaffolding path), `slash_commands.go` (comment), `cli_test.go` (4 test functions with path assertions).
- **Behavioral change**: New installations get commands in the correct directory. Existing installations with the old directory get migrated on next `dewey init` run.
- **Risk**: Low. The change is a path string update plus idempotent migration logic. No new dependencies, no schema changes, no MCP tool changes.
- **Backward compatibility**: Existing `.opencode/command/` directories are migrated, not abandoned. User customizations in the old directory are preserved during migration (files are moved, not re-scaffolded).

## Constitution Alignment

Assessed against the Dewey project constitution (v1.4.0).

### I. Composability First

**Assessment**: PASS

Dewey remains independently installable. The `.opencode/` directory detection guard is preserved — slash commands are only scaffolded when OpenCode is present. The migration only moves Dewey-owned commands (Composability First boundary). The change aligns Dewey with the directory convention used by all other UF heroes, improving cross-hero composability.

### II. Autonomous Collaboration

**Assessment**: N/A

This change does not affect artifact-based communication or MCP tool interfaces. It modifies a local filesystem scaffolding path used during project initialization.

### III. Observable Quality

**Assessment**: PASS

Migration actions are logged via `charmbracelet/log` (info level), providing observability into what was moved and where. No change to machine-parseable output or provenance metadata.

### IV. Testability

**Assessment**: PASS

All four existing test functions are updated to use the new path. Three new tests are added: migration (old directory detected, commands moved), conflict resolution (new directory version preserved), and non-Dewey file preservation (old directory retained when it contains non-Dewey files). All 8 spec scenarios have corresponding test coverage.
