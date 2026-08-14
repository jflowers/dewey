<!-- spec-review: passed -->
<!--
  [P] marks tasks eligible for parallel execution.
  Add [P] when a task: (a) touches different files from
  other [P] tasks in the group, (b) has no dependency
  on prior tasks in the group, (c) can safely execute
  without ordering constraints.
  Do NOT add [P] when tasks modify the same file —
  parallel workers will cause merge conflicts.
  Tasks without [P] run sequentially first, then [P]
  tasks run in parallel.
-->

## 1. Update scaffolding path and add migration logic

- [x] 1.1 In `cli.go`, change the `opencodeCmdDir` path from `filepath.Join(vaultPath, ".opencode", "command")` to `filepath.Join(vaultPath, ".opencode", "commands")` (line 336)
- [x] 1.2 In `cli.go`, add migration logic before the scaffolding loop: detect `.opencode/command/` (old), move Dewey-owned files (those in `deweySlashCommands` map) to `.opencode/commands/`, skip files already present in new directory, remove old file after move, remove old directory if empty after migration. Log migration actions at info level via `charmbracelet/log`. Errors during individual file migration or old directory removal SHOULD be logged at warn level and MUST NOT halt the init process
- [x] 1.3 [P] In `slash_commands.go`, update the comment on line 4 from `.opencode/command/` to `.opencode/commands/`

## 2. Update tests

- [x] 2.1 In `cli_test.go`, update all 4 test functions (`TestInitCmd_ScaffoldsSlashCommands`, `TestInitCmd_SkipsExistingSlashCommands`, `TestInitCmd_NoOpenCodeDir`, `TestInitCmd_ReInitScaffoldsNewCommands`) to use `.opencode/commands/` instead of `.opencode/command/`. Also fix pre-existing gap: add `dewey-curate.md` to the assertion list in `TestInitCmd_ScaffoldsSlashCommands` (currently checks 5 of 6 commands)
- [x] 2.2 In `cli_test.go`, add a new test `TestInitCmd_MigratesOldCommandDir` that verifies migration: create old dir with at least 2 Dewey slash command files, run init, assert all files moved to new dir, assert old dir removed when empty, assert all `deweySlashCommands` entries exist in `.opencode/commands/` after init (scaffolding proceeds after migration)
- [x] 2.3 In `cli_test.go`, add a test `TestInitCmd_MigrationSkipsExistingInNewDir` that verifies: old dir has a file with distinct content, new dir already has the same filename with different content, run init, assert new dir file retains its original content, old copy is removed
- [x] 2.4 In `cli_test.go`, add a test `TestInitCmd_MigrationKeepsOldDirWithNonDeweyFiles` that verifies: create old dir with a Dewey file AND a non-Dewey file (e.g., `speckit.plan.md`), run init, assert Dewey file migrated to new dir, assert non-Dewey file untouched in old dir, assert old dir still exists

## 3. Verification

- [x] 3.1 Run `go build ./...` — must pass
- [x] 3.2 Run `go vet ./...` — must pass
- [x] 3.3 Run `go test -race -count=1 ./...` — must pass
- [x] 3.4 Verify constitution alignment: Composability First (Dewey only migrates its own commands), Observable Quality (migration logged), Testability (all 8 spec scenarios covered by tests)
