## Why

OpenCode's DCP (Dynamic Context Preservation) plugin compresses conversation context during long sessions. Slash command content is injected as user messages, making it eligible for compression — unlike skills, which are protected via DCP's `protectedTools` mechanism. When critical sections of slash commands (guardrails, execution checklists, exit conditions) are compressed, agents lose behavioral constraints and may produce incorrect results.

Adding `<protect>...</protect>` tags around these critical sections tells DCP to preserve them verbatim during compression, ensuring agent behavior remains correct throughout long sessions.

Addresses: https://github.com/unbound-force/dewey/issues/95

## What Changes

Add `<protect>` tags to the 6 slash command definitions embedded as Go string literals in `slash_commands.go`. Each command's critical behavioral sections are wrapped so DCP preserves them during context compression.

## Capabilities

### New Capabilities
- None (no new user-facing capabilities)

### Modified Capabilities
- `dewey-store.md`: Protect the multi-step workflow execution checklist, guardrails (verification requirements, error handling), and exit conditions
- `dewey-index.md`: Protect the execution steps and guardrails
- `dewey-reindex.md`: Protect the execution steps and guardrails (especially the destructive operation warning)
- `dewey-compile.md`: Protect the execution steps and guardrails
- `dewey-curate.md`: Protect the execution steps and guardrails
- `dewey-lint.md`: Protect the execution steps and guardrails

### Removed Capabilities
- None

## Impact

- **File modified**: `slash_commands.go` — Go string literals containing slash command markdown content
- **Runtime behavior**: No change when DCP `protectTags` is disabled (tags are inert markdown). When enabled, DCP preserves tagged sections during compression.
- **Backward compatibility**: Fully backward compatible. `<protect>` tags are ignored by renderers that don't understand them. No Go API, CLI, or MCP tool changes.
- **Testing**: Existing tests continue to pass unchanged. No new runtime logic is introduced — only static content changes.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: PASS

This change improves autonomous collaboration by ensuring that agent behavioral constraints survive context compression. Slash commands remain self-describing artifacts — the `<protect>` tags are additive metadata that preserves existing content fidelity.

### II. Composability First

**Assessment**: PASS

No new dependencies are introduced. The `<protect>` tags are inert when DCP is not configured with `protectTags` enabled. Dewey remains independently installable and usable without any DCP configuration.

### III. Observable Quality

**Assessment**: N/A

This change modifies static content (string literals), not runtime output. No provenance metadata or machine-parseable output is affected.

### IV. Testability

**Assessment**: PASS

The change is purely additive static content within Go string literals. Existing compilation and any content-based tests verify the strings are well-formed. No new runtime logic requires testing.
