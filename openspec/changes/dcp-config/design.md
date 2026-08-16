## Context

Dewey's slash command files (`.opencode/uf/commands/`) contain `<protect>` tags around execution-critical sections -- tool registration patterns, workflow guardrails, phase boundaries, and similar content that must survive DCP context compression. These tags are embedded as Go string literals in `slash_commands.go` and written to the commands directory.

Currently, `<protect>` tags are present but inert because no `.opencode/dcp.jsonc` configuration file exists to enable the `protectTags` feature. DCP defaults to ignoring these tags without explicit opt-in.

## Goals / Non-Goals

### Goals
- Enable DCP `protectTags` via `.opencode/dcp.jsonc` so `<protect>` blocks in slash command files are preserved during compression
- Use JSONC format (JSON with comments) to allow inline documentation of the configuration

### Non-Goals
- Modifying any slash command content or `<protect>` tag placement
- Adding `<protect>` tags to files that don't already have them
- Configuring any other DCP settings beyond `protectTags`
- Modifying `opencode.json` (DCP config must be in its own file)

## Decisions

### D1: Use `.opencode/dcp.jsonc` not `opencode.json`

DCP configuration lives in `.opencode/dcp.jsonc`, separate from `opencode.json`. The `opencode.json` schema does not include DCP-specific fields, and adding them causes validation errors. DCP reads its own config file independently.

### D2: Minimal configuration

Only `compress.protectTags` is set to `true`. No other DCP settings are configured. This follows the composability principle -- configure only what is needed, rely on defaults for everything else.

### D3: Include `$schema` reference

The JSONC file includes a `$schema` field pointing to the DCP schema for editor validation and discoverability.

## Risks / Trade-offs

- **Low risk**: This is a single static configuration file with no runtime logic. If DCP is not present, the file is ignored entirely.
- **Trade-off**: Using a separate `.opencode/dcp.jsonc` file adds one more config file to the `.opencode/` directory, but this is the correct and only supported location for DCP configuration.
