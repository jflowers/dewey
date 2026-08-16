## Why

`dewey init` scaffolds `.opencode/command/` slash commands with `<protect>` tags (added by the `protect-tags-slash-commands` change), but the DCP configuration file that activates those tags is not scaffolded. Without `.opencode/dcp.jsonc` containing `compress.protectTags: true`, the `<protect>` tags in slash commands are inert — DCP ignores them during context compression.

PR #98 (`dcp-config` change) added the config file to the Dewey repository itself, but every project that runs `dewey init` also needs this file. This change makes `dewey init` scaffold `.opencode/dcp.jsonc` automatically, following the same idempotent pattern used for slash commands.

Analogous to unbound-force/unbound-force#502.

## What Changes

1. **`dewey init` scaffolds `.opencode/dcp.jsonc`** — After scaffolding slash commands, create `.opencode/dcp.jsonc` with `compress.protectTags: true` and a `$schema` reference. Idempotent: skip if a DCP config with `protectTags` already exists; warn (don't overwrite) if a DCP config exists but lacks the setting.

2. **`dewey doctor` checks DCP config** — After the existing `opencode.json` check, verify that `.opencode/dcp.jsonc` or `.opencode/dcp.json` contains `protectTags`. Report PASS/WARN with actionable fix hints.

## Capabilities

### New Capabilities
- `dewey init` DCP scaffolding: Automatically creates `.opencode/dcp.jsonc` with `compress.protectTags: true` when `.opencode/` directory exists
- `dewey doctor` DCP check: Reports whether DCP `protectTags` configuration is present and correct

### Modified Capabilities
- `dewey init`: Gains additional scaffolding step after slash commands
- `dewey doctor`: Gains additional diagnostic check after `opencode.json` check

### Removed Capabilities
- None

## Impact

- **Files modified**: `cli.go` (init command + doctor command), `cli_test.go` (8 new tests)
- **No new packages or dependencies**
- **No schema or data model changes**
- **Backward compatible**: Existing `dewey init` invocations gain the new file; re-running is idempotent

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: N/A

This change modifies CLI scaffolding behavior within Dewey. No inter-hero artifact communication is affected. The scaffolded file is consumed by DCP (an external tool), not by Dewey itself.

### II. Composability First

**Assessment**: PASS

The DCP config is only scaffolded when `.opencode/` already exists — Dewey does not create it. This preserves the composability guard: projects without OpenCode are unaffected. The scaffolded file is optional and has no effect when DCP is absent.

### III. Observable Quality

**Assessment**: PASS

`dewey doctor` gains a new diagnostic check that reports DCP config status with PASS/WARN output and actionable fix hints, maintaining the observable quality pattern established by existing doctor checks.

### IV. Testability

**Assessment**: PASS

Eight new tests cover all scaffolding paths (create, skip, warn) and doctor output (present, missing, partial config, silent skip). Tests use `t.TempDir()` for isolation — no external services required.
