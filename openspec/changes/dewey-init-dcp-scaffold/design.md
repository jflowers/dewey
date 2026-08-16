## Context

`dewey init` scaffolds project infrastructure including `.uf/dewey/` directories, `sources.yaml`, `config.yaml`, and `.opencode/command/` slash commands. The slash commands contain `<protect>` tags that prevent DCP from compressing execution-critical sections, but DCP requires `.opencode/dcp.jsonc` with `compress.protectTags: true` to honor those tags. Without this file, the tags are inert.

PR #98 added the file to the Dewey repository itself. This change extends `dewey init` to scaffold the file into any project, and extends `dewey doctor` to verify the configuration is present.

## Goals / Non-Goals

### Goals
- Scaffold `.opencode/dcp.jsonc` during `dewey init` with the same idempotent pattern used for slash commands
- Add a `dewey doctor` check for DCP `protectTags` configuration
- Support both `.opencode/dcp.jsonc` (JSONC) and `.opencode/dcp.json` (JSON) for detection

### Non-Goals
- Merging or patching existing DCP config files (JSONC merging is fragile and error-prone)
- Scaffolding any DCP settings beyond `compress.protectTags`
- Making `dewey init` create the `.opencode/` directory (composability — only scaffold when OpenCode is already present)

## Decisions

### D1: Reuse the `.opencode/` existence guard

Scaffolding is gated on `os.Stat(filepath.Join(vaultPath, ".opencode"))` succeeding — the same guard used for slash commands (cli.go:337). This ensures DCP config is only scaffolded in projects that use OpenCode, preserving Composability First (Constitution II).

### D2: Idempotent three-way logic

The scaffolding uses three-way detection:
1. If `dcp.jsonc` or `dcp.json` already contains `protectTags` — **skip silently** (already configured)
2. If `dcp.jsonc` or `dcp.json` exists but lacks `protectTags` — **warn, don't overwrite** (user has custom DCP config; overwriting risks losing their settings)
3. If neither file exists — **create `dcp.jsonc`** from template

This mirrors the slash command pattern (skip existing files) but adds the warn-on-partial-config case because DCP configs can contain other settings the user has configured.

### D3: String-based `protectTags` detection (not JSON parsing)

Use `strings.Contains(data, "protectTags")` rather than parsing JSONC. Rationale:
- JSONC parsing requires a third-party library (Go's `encoding/json` does not support comments)
- The detection only needs to confirm the key is present, not validate its value
- This matches the existing `opencode.json` check pattern in doctor (cli.go:1442), which uses `strings.Contains(data, "dewey")`
- False positives are negligible — `protectTags` is a DCP-specific key unlikely to appear in comments

### D4: Doctor check placement

The DCP check is placed immediately after the existing `opencode.json` check (cli.go:1450), grouping all `.opencode/` configuration checks together. The check follows the same PASS/WARN pattern with actionable fix hints.

### D5: Template content matches PR #98

The scaffolded file content is identical to the `.opencode/dcp.jsonc` committed in PR #98, ensuring consistency across the Dewey repo and user projects. Content includes:
- `$schema` reference for editor validation
- Explanatory JSONC comment
- `compress.protectTags: true`

## Risks / Trade-offs

### Low: `$schema` URL is mutable

The `$schema` points to a `master` branch reference on GitHub. If the upstream schema changes, editor validation may break. However, this is editor-only convenience with zero runtime impact — DCP reads config values, not the schema.

### Low: String detection could miss edge cases

A `protectTags` key appearing only in a JSONC comment would produce a false positive (skip when it shouldn't). In practice this is extremely unlikely and harmless — the worst case is the user manually creates the setting, which they clearly intended.
