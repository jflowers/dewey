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

## 1. Fix URL construction

- [x] 1.1 [P] Fix `rawPredictURL()` in `llm/vertex.go` to detect `region == "global"` and use `aiplatform.googleapis.com` (no region prefix) as the hostname. Regional endpoints remain unchanged.
- [x] 1.2 [P] Fix `predictURL()` in `embed/vertex.go` with the same conditional: `region == "global"` uses `aiplatform.googleapis.com`, otherwise `{region}-aiplatform.googleapis.com`.

## 2. Add test coverage

- [x] 2.1 [P] Add `TestVertexSynthesizer_RawPredictURL` in `llm/vertex_test.go` with table-driven subtests covering: (a) `region: "global"` produces URL with `aiplatform.googleapis.com` hostname (regression test — MUST fail without the fix per TC-006), (b) `region: "us-east5"` produces URL with `us-east5-aiplatform.googleapis.com` hostname.
- [x] 2.2 [P] Add `TestVertexEmbedder_PredictURL` in `embed/vertex_test.go` with table-driven subtests covering: (a) `region: "global"` produces URL with `aiplatform.googleapis.com` hostname (regression test — MUST fail without the fix per TC-006), (b) `region: "us-central1"` produces URL with `us-central1-aiplatform.googleapis.com` hostname.

## 3. Verify

- [x] 3.1 Run `go build ./...` and `go test -race -count=1 ./llm/ ./embed/` to confirm the fix compiles and all tests pass.
- [x] 3.2 Run `go vet ./...` to confirm no static analysis issues.
- [x] 3.3 Verify constitution alignment: Composability (no new imports between `llm/` and `embed/` — run `go list -f '{{.Imports}}' ./llm/ ./embed/`), Testability (new tests are isolated, no external service calls).
- [x] 3.4 Update `README.md` and `AGENTS.md` to mention `global` as a valid `region` value for Vertex AI providers (routes to nearest available region).
<!-- code-review: passed -->
