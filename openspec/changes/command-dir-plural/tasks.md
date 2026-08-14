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

- [ ] 1.1 In `cli.go`, change the `opencodeCmdDir` path from `filepath.Join(vaultPath, ".opencode", "command")` to `filepath.Join(vaultPath, ".opencode", "commands")` (line 336)
- [ ] 1.2 In `cli.go`, add migration logic before the scaffolding loop: detect `.opencode/command/` (old), move Dewey-owned files (those in `deweySlashCommands` map) to `.opencode/commands/`, skip files already present in new directory, remove old file after move, remove old directory if empty after migration. Log migration actions at info level via `charmbracelet/log`
- [ ] 1.3 [P] In `slash_commands.go`, update the comment on line 4 from `.opencode/command/` to `.opencode/commands/`

## 2. Update tests

- [ ] 2.1 In `cli_test.go`, update all 4 test functions (`TestInitCmd_ScaffoldsSlashCommands`, `TestInitCmd_SkipsExistingSlashCommands`, `TestInitCmd_NoOpenCodeDir`, `TestInitCmd_ReInitScaffoldsNewCommands`) to use `.opencode/commands/` instead of `.opencode/command/`
- [ ] 2.2 In `cli_test.go`, add a new test `TestInitCmd_MigratesOldCommandDir` that verifies migration: create old dir with a Dewey slash command file, run init, assert file moved to new dir, assert old dir removed if empty
- [ ] 2.3 In `cli_test.go`, add a test `TestInitCmd_MigrationSkipsExistingInNewDir` that verifies: old dir has a file, new dir already has the same filename, run init, assert new dir version is preserved, old copy is removed

## 3. Repository cleanup

- [ ] 3.1 [P] Delete the stale `.opencode/command/` directory from version control (`git rm -r .opencode/command/`)

## 4. Verification

- [ ] 4.1 Run `go build ./...` — must pass
- [ ] 4.2 Run `go vet ./...` — must pass
- [ ] 4.3 Run `go test -race -count=1 ./...` — must pass
- [ ] 4.4 Verify constitution alignment: Composability First (Dewey only migrates its own commands), Observable Quality (migration logged), Testability (all scenarios covered by tests)
