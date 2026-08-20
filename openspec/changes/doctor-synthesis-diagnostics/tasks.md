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

## 1. Implement Synthesis Layer section in dewey doctor

- [ ] 1.1 Add Synthesis Layer section to `runDoctorChecks()` in `cli.go`
  - Read synthesis config via `llm.ReadSynthesisConfig(deweyDir)`
  - Insert new section between Embedding Layer (line 1518) and MCP Server (line 1520)
  - Handle three provider cases: ollama, vertex, unconfigured
  - **Ollama path**: section header with model+endpoint, `ensureOllama(synthEndpoint, false, nil)` for connectivity, `OllamaSynthesizer.Available()` for model check
  - **Vertex path**: section header with model+provider, report config completeness (project, region, model), no live API calls
  - **Unconfigured path**: report PASS "not configured (optional)"
  - Files: `cli.go`

## 2. Add tests for Synthesis Layer diagnostics

- [ ] 2.1 [P] Add test for unconfigured synthesis provider
  - Verify "Synthesis Layer" section appears in doctor output
  - Verify "not configured" message with PASS status
  - Use `t.TempDir()` vault with no config.yaml synthesis section
  - Files: `cli_test.go`

- [ ] 2.2 [P] Add test for Ollama synthesis provider
  - Configure synthesis with ollama provider in config.yaml
  - Use `newMockOllamaServer()` for connectivity check
  - Verify section header includes model and endpoint
  - Verify connectivity and model availability checks
  - Files: `main_test.go`

- [ ] 2.3 [P] Add test for Vertex synthesis provider
  - Configure synthesis with vertex provider in config.yaml
  - Verify section header includes model and "vertex" provider type
  - Verify config completeness check (project + region present)
  - Verify no live API calls are made
  - Files: `cli_test.go`

- [ ] 2.4 [P] Add test for Vertex synthesis provider misconfigured
  - Configure synthesis with vertex provider but missing project/region
  - Verify FAIL check for missing required fields
  - Files: `cli_test.go`

## 3. Verification

- [ ] 3.1 Run CI-equivalent checks (`go build ./...`, `go vet ./...`, `go test -race -count=1 ./...`)
- [ ] 3.2 Verify constitution alignment: Composability First (graceful degradation when unconfigured), Observable Quality (full provider stack diagnostics), Testability (no external services in tests)
- [ ] 3.3 Verify documentation impact (AGENTS.md doctor section, GoDoc comments)
<!-- scaffolded by uf vdev -->
