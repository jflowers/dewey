## Context

Dewey's slash commands are embedded as Go string literals in `slash_commands.go` and scaffolded to `.opencode/command/` by `dewey init`. OpenCode's DCP plugin treats these as user messages eligible for compression. When long sessions trigger compression, critical behavioral sections (guardrails, execution steps, exit conditions) may be lost, causing agents to deviate from intended behavior.

The proposal establishes that `<protect>` tags should be added around critical sections. This design documents the technical approach for identifying and wrapping those sections.

## Goals / Non-Goals

### Goals
- Identify critical sections in each slash command that must survive DCP compression
- Define a consistent tagging strategy across all 6 commands
- Ensure tags are additive-only — no changes to existing command behavior

### Non-Goals
- Modifying DCP plugin behavior (DCP already supports `protectTags`)
- Adding new slash commands
- Changing slash command logic, wording, or structure
- Documenting DCP configuration (users configure `protectTags` in their DCP config)

## Decisions

### D1: What constitutes a "critical section"

Three categories of content are critical for correct agent behavior:

1. **Execution workflow** — Numbered steps that define the command's procedure (e.g., "Parse Input → Determine Mode → Analyze → Store"). Without these, the agent cannot follow the intended flow.
2. **Guardrails** — Constraints that prevent incorrect behavior (e.g., "Wait for user confirmation", "Warning: this deletes all external source content").
3. **Exit conditions** — What the agent should report after completion (e.g., "Display the returned identity and suggest /dewey-compile").

Non-critical content (description, usage examples, frontmatter) can safely be compressed without behavioral impact.

### D2: Tagging granularity

Each command gets a single `<protect>` region covering its Instructions section (the behavioral contract). This avoids fragmentation — wrapping individual subsections would create many small protected blocks that increase token overhead without benefit.

For `dewey-store.md` (the multi-step workflow), the entire Instructions section including all 5 steps is wrapped, since the steps form a dependent chain.

### D3: Tag placement within Go string literals

Tags are added directly inside the Go string literal content, as plain text within the markdown. No Go code changes beyond the string content. The backtick-delimited raw strings in `slash_commands.go` support arbitrary content including angle brackets.

## Risks / Trade-offs

### Token overhead
`<protect>` and `</protect>` tags add ~15-20 characters per command (6 commands × ~18 chars = ~108 chars total). This is negligible relative to the total command content.

### DCP not configured
When `protectTags` is not enabled in DCP config, the tags are inert text rendered as-is. They appear as harmless HTML-like tags in markdown. Agents are expected to ignore unrecognized tags. Acceptable trade-off for the protection benefit.

### Over-protection risk
Wrapping entire Instructions sections means DCP cannot compress any part of them, even descriptive text within steps. This is acceptable because the Instructions sections are the behavioral contract — partial compression could create ambiguous instructions worse than no compression.
