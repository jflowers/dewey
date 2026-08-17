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

## 1. Add DCP Configuration

- [x] 1.1 Create `.opencode/dcp.jsonc` with `compress.protectTags` enabled and `$schema` reference

## 2. Verification

- [x] 2.1 Verify `.opencode/dcp.jsonc` is valid JSONC and contains the correct `$schema` and `compress.protectTags: true` setting
- [x] 2.2 Verify no DCP-related fields exist in `opencode.json` (configuration isolation)
- [x] 2.3 Verify constitution alignment: change maintains composability (file is optional) and introduces no runtime coupling
<!-- spec-review: passed -->
