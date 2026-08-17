## Why

Dewey's slash command files embed `<protect>` tags around execution-critical sections (tool registration patterns, workflow steps, guardrails) to preserve them during DCP context compression. However, these tags are inert without a DCP configuration file that enables the `protectTags` feature. Without `.opencode/dcp.jsonc`, DCP has no way to know it should honor `<protect>` blocks, and critical slash command content may be compressed away during long sessions.

This was identified in GitHub issue #97.

## What Changes

Add a `.opencode/dcp.jsonc` configuration file that enables the `compress.protectTags` setting. This is the sole mechanism for activating `<protect>` tag support in DCP -- the setting cannot be placed in `opencode.json` (which causes validation errors due to schema differences).

## Capabilities

### New Capabilities
- `dcp-protect-tags`: DCP will honor `<protect>` tags in slash command files, preserving execution-critical sections during context compression.

### Modified Capabilities
- None

### Removed Capabilities
- None

## Impact

- **Files added**: `.opencode/dcp.jsonc` (single new configuration file)
- **Affected systems**: DCP context compression pipeline -- `<protect>` tags in `.opencode/commands/` slash command files will now be respected
- **No code changes**: This is a configuration-only change. No Go source, tests, or CI modifications required.
- **Backward compatible**: Adding a DCP config file has no effect on environments that don't use DCP.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: N/A

This change adds a configuration file for DCP tooling. It does not affect artifact-based communication or self-describing outputs between heroes.

### II. Composability First

**Assessment**: PASS

The `.opencode/dcp.jsonc` file is an optional configuration file. Dewey remains fully functional without it -- the file only affects DCP's compression behavior when DCP is present. No mandatory dependencies are introduced.

### III. Observable Quality

**Assessment**: N/A

This change does not affect machine-parseable output or provenance metadata. It configures an external tool's compression behavior.

### IV. Testability

**Assessment**: N/A

This is a static configuration file with no runtime logic. No components are affected that would require isolation testing.
