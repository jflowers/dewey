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

## 1. Synthesis Endpoint Resolution

- [x] 1.1 Add `ResolveSynthesisEndpoint()` function in `llm/config.go` mirroring `embed.ResolveOllamaEndpoint()`. Implement the fallback chain: `DEWEY_SYNTHESIS_ENDPOINT` → `OLLAMA_HOST` → `embed.DefaultOllamaEndpoint`. Include scheme normalization (prepend `http://` when no scheme present) and treat empty string as unset. Also add a helper `normalizeSynthesisEndpoint()` for scheme handling.
- [x] 1.2 Update the legacy `compile_model` path (line 85) to call `ResolveSynthesisEndpoint()` instead of `os.Getenv("DEWEY_EMBEDDING_ENDPOINT")`.
- [x] 1.3 Update `synthConfigFromEnv()` (line 117) to call `ResolveSynthesisEndpoint()` instead of `os.Getenv("DEWEY_EMBEDDING_ENDPOINT")`.

## 2. Tests

- [x] 2.1 Add endpoint resolution tests in `llm/provider_test.go` mirroring `embed/config_test.go` patterns. Required test cases: (1) `DEWEY_SYNTHESIS_ENDPOINT` overrides all, (2) `OLLAMA_HOST` fallback when `DEWEY_SYNTHESIS_ENDPOINT` unset, (3) `DEWEY_SYNTHESIS_ENDPOINT` wins over `OLLAMA_HOST`, (4) defaults to `http://localhost:11434` when nothing set, (5) `DEWEY_EMBEDDING_ENDPOINT` does NOT affect synthesis (regression test), (6) no-scheme normalization (e.g., `0.0.0.0:11434` → `http://0.0.0.0:11434`), (7) HTTPS preserved, (8) empty string treated as unset, (9) `OLLAMA_HOST` without scheme normalized for synthesis (e.g., `0.0.0.0:11434` → `http://0.0.0.0:11434`), (10) `config.yaml` `synthesis.endpoint` wins over `DEWEY_SYNTHESIS_ENDPOINT`.
- [x] 2.2 Extend existing `TestReadSynthesisConfig_EnvFallback` and `TestReadSynthesisConfig_BackwardCompatible` to also assert `cfg.Endpoint` values. Expected values: `TestReadSynthesisConfig_EnvFallback` should assert `cfg.Endpoint` equals `embed.DefaultOllamaEndpoint` (default, since no synthesis env var is set); `TestReadSynthesisConfig_BackwardCompatible` should assert `cfg.Endpoint` equals `embed.DefaultOllamaEndpoint` (default, since only `compile_model` is set). Note: Scenarios for legacy and env-only paths with `DEWEY_SYNTHESIS_ENDPOINT` set are covered by test cases in task 2.1.

## 3. Documentation

- [x] 3.1 [P] Update `AGENTS.md` to document the synthesis endpoint resolution chain alongside the existing embedding endpoint documentation. Add a new section under Provider Configuration with the precedence: `DEWEY_SYNTHESIS_ENDPOINT` → `OLLAMA_HOST` → `http://localhost:11434`. Note the precedence asymmetry: unlike embedding (where env var overrides config.yaml), the synthesis path gives config.yaml highest precedence — env vars are consulted only when config.yaml does not specify a synthesis section.
- [x] 3.2 [P] File a GitHub issue in `unbound-force/website` for documentation sync (new `DEWEY_SYNTHESIS_ENDPOINT` env var).
- [x] 3.3 [P] File a GitHub issue in `unbound-force/dewey` tracking the `dewey doctor` synthesis diagnostics gap (no synthesis layer in doctor output). Reference this OpenSpec and GitHub #71.

## 4. Verification

- [x] 4.1 Run `go build ./...`, `go vet ./...`, and `go test -race -count=1 ./...` to verify all tests pass.
- [x] 4.2 Verify constitution alignment: Composability First (independent endpoint config), Observable Quality (env var name matches behavior), Testability (all scenarios covered by tests).

<!-- spec-review: passed -->
<!-- code-review: passed -->
