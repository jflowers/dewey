## Context

UF tooling v0.15.0 renamed the OpenCode slash command directory from `.opencode/command/` to `.opencode/commands/`. The `dewey init` command was not updated to match, creating a split-brain state where Dewey scaffolds to the old path while all other UF tooling uses the new path. See proposal for full investigation details.

Affected source locations:
- `cli.go:336` — `opencodeCmdDir := filepath.Join(vaultPath, ".opencode", "command")`
- `slash_commands.go:4` — comment referencing old path
- `cli_test.go` — 4 test functions with old path assertions (lines 408, 437, 476, 497)

## Goals / Non-Goals

### Goals
- Update `dewey init` to scaffold slash commands into `.opencode/commands/` (plural)
- Add idempotent migration logic to move existing Dewey slash commands from old to new directory
- Update all tests and comments to reflect the new path
- Delete the stale `.opencode/command/` directory from the repository

### Non-Goals
- Modifying the `/uf.init` migration logic (Step 0) — that handles the broader UF ecosystem migration and remains valid
- Changing any MCP tool behavior
- Modifying the `deweySlashCommands` map contents (only the target path changes)
- Updating old spec documents that reference `.opencode/command/` — those are historical artifacts

## Decisions

### D1: Migration scope — Dewey commands only

The migration logic in `dewey init` only moves files that exist in the `deweySlashCommands` map (currently 6 commands: `dewey-store`, `dewey-index`, `dewey-reindex`, `dewey-compile`, `dewey-curate`, `dewey-lint`). It does not attempt to migrate other commands that may exist in `.opencode/command/` — that is `/uf.init`'s responsibility.

**Rationale**: Dewey should only manage its own slash commands. Moving non-Dewey commands would violate Composability First — Dewey must not assume ownership of artifacts from other heroes.

### D2: Migration before scaffolding

Migration runs before the scaffolding loop, so moved files are not overwritten by freshly generated content. If a file already exists in the new directory, the old copy is skipped (the new-directory version is authoritative).

**Rationale**: Preserves user modifications made in the new directory. Avoids data loss.

### D3: Silent cleanup of empty old directory

After migration, if `.opencode/command/` is empty, it is removed. If it still contains non-Dewey files, it is left intact.

**Rationale**: Clean up what Dewey created without affecting other tools' files. Composability First requires Dewey not to delete artifacts it doesn't own.

### D4: Delete `.opencode/command/` from repository

The stale `.opencode/command/` directory (14 files) is deleted from version control. These files are duplicates of content that already exists in `.opencode/commands/` or were scaffolded there by `dewey init` writing to the wrong path.

**Rationale**: Zero-Waste Mandate — no orphaned files.

## Risks / Trade-offs

### Risk: Users on older Dewey with `.opencode/command/` only

**Mitigation**: The migration logic handles this gracefully — files are moved to the new directory. Users who haven't run `/uf.init` will get their Dewey commands migrated on the next `dewey init`.

### Risk: Race between `dewey init` migration and `/uf.init` Step 0

**Assessment**: Negligible. These are interactive CLI commands, not concurrent processes. The migration is idempotent — running both in any order produces the same result (files in `.opencode/commands/`).

### Trade-off: No backward-compatible fallback

After this change, `dewey init` only writes to `.opencode/commands/`. If a user somehow needs the old path, they must manually create it. This is acceptable because the entire UF ecosystem has standardized on the plural path since v0.15.0.
