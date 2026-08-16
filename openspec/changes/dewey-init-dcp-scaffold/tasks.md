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

## 1. Implementation (cli.go)

- [x] 1.1 Add `dcpJSONCContent` const with the DCP config template string (matching PR #98 content: `$schema`, comment, `compress.protectTags: true`)
- [x] 1.2 Add `fileExists(path string) bool` and `fileContains(path, substr string) bool` private helper functions
- [x] 1.3 Add DCP config scaffolding in `newInitCmd()` after slash command scaffolding (after line 349), inside the `.opencode/` existence guard. Three-way logic: skip if `dcp.jsonc`/`dcp.json` has `protectTags`; warn if file exists without `protectTags`; create `dcp.jsonc` if neither exists
- [x] 1.4 Add DCP config doctor check in `newDoctorCmd()` after `opencode.json` check (after line 1450). PASS if config has `protectTags`; WARN if file exists without it; WARN if `.opencode/` exists but no DCP config; silent skip if no `.opencode/`

## 2. Tests (cli_test.go)

- [x] 2.1 [P] `TestInitCmd_ScaffoldsDCPConfig` — verify file creation when `.opencode/` exists but no DCP config
- [x] 2.2 [P] `TestInitCmd_SkipsExistingDCPConfigWithProtectTags` — verify no overwrite when config already has `protectTags`
- [x] 2.3 [P] `TestInitCmd_WarnsOnDCPConfigMissingProtectTags` — verify warn + no overwrite when config lacks `protectTags`
- [x] 2.4 [P] `TestInitCmd_NoDCPConfigWhenNoOpenCodeDir` — verify composability guard (no `.opencode/` = no scaffolding)
- [x] 2.5 [P] `TestDoctorCmd_DCPConfigPresent` — verify PASS output when config is correct
- [x] 2.6 [P] `TestDoctorCmd_DCPConfigMissing` — verify WARN output when `.opencode/` exists but no config
- [x] 2.7 [P] `TestDoctorCmd_DCPConfigMissingProtectTags` — verify WARN output when `.opencode/dcp.jsonc` exists but does not contain `protectTags`
- [x] 2.8 [P] `TestDoctorCmd_NoDCPCheckWithoutOpenCodeDir` — verify doctor output does NOT contain DCP-related text when `.opencode/` is absent

## 3. Verification

- [x] 3.1 Run `go build ./...`, `go vet ./...`, `go test -race -count=1 ./...` — all must pass
- [x] 3.2 Verify constitution alignment: Composability First (DCP config only scaffolded when `.opencode/` exists), Observable Quality (doctor check provides PASS/WARN with fix hints), Testability (all paths covered by tests)
<!-- spec-review: passed -->
<!-- code-review: passed -->
