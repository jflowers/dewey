<!-- spec-review: passed -->
<!-- code-review: passed -->
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

## 1. Add protect tags to slash commands

All tasks modify the same file (`slash_commands.go`), so no parallel execution.

- [x] 1.1 Add `<protect>` / `</protect>` tags around the Instructions section of `dewey-store.md` (lines 36-83 in Go string literal) — wrap from before `## Instructions` through the end of Step 5
- [x] 1.2 Add `<protect>` / `</protect>` tags around the Instructions section of `dewey-index.md` (lines 105-116)
- [x] 1.3 Add `<protect>` / `</protect>` tags around the Instructions section of `dewey-reindex.md` (lines 137-145)
- [x] 1.4 Add `<protect>` / `</protect>` tags around the Instructions section of `dewey-compile.md` (lines 166-171)
- [x] 1.5 Add `<protect>` / `</protect>` tags around the Instructions section of `dewey-curate.md` (lines 194-198)
- [x] 1.6 Add `<protect>` / `</protect>` tags around the Instructions section of `dewey-lint.md` (lines 219-227)

## 2. Verification

- [x] 2.1 Run `go build ./...` to verify string literals compile correctly
- [x] 2.2 Run `go vet ./...` to verify no static analysis issues
- [x] 2.3 Verify constitution alignment: Composability (no new dependencies), Testability (existing tests pass with `go test -race -count=1 ./...`)
