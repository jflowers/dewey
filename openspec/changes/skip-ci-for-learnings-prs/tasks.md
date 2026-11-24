<!--
  [P] marks tasks eligible for parallel execution.
  Add [P] when a task: (a) touches different files from
  other [P] tasks in the group, (b) has no dependency on
  prior tasks in the group, (c) can safely execute without
  ordering constraints.
  Do NOT add [P] when tasks modify the same file —
  parallel workers will cause merge conflicts.
  Tasks without [P] run sequentially first, then [P]
  tasks run in parallel.
-->

## 1. `ci.yml` Changes

All tasks modify `.github/workflows/ci.yml` — no parallel execution.

- [x] 1.1 Narrow the `push` trigger from `branches: ['**']` to `branches: [main]`. Leave `pull_request: branches: [main]`, the top-level `permissions`, and the `concurrency` block untouched (design Decision 4).
- [ ] 1.2 Add the `changes` detection job per design Decision 1: `runs-on: ubuntu-latest`, `permissions: contents: read`, `outputs.code` wired from the `detect` step, `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2` with `fetch-depth: 0`, `EVENT_NAME`/`BASE_SHA`/`HEAD_SHA` passed through `env:` (never `${{ }}` interpolation inside `run:`), a non-`pull_request` short-circuit to `code=true`, and the `git diff --name-only --no-renames "$BASE_SHA...$HEAD_SHA" | grep -v '^\.uf/dewey/learnings/' | grep -q .` pipeline inside an `if` so `grep`'s exit 1 does not trip `set -e`.
- [ ] 1.3 Gate `build-and-test`: add `needs: changes` and `if: needs.changes.outputs.code == 'true'` (quoted — a bare `'false'` string is truthy in `if:` expressions). Change nothing else in the job: same steps, same runner, and the main-only Gaze AI quality report step keeps its existing step-level `if:` (design Decision 2).
- [ ] 1.4 Add the `ci-ok` aggregate job per design Decision 3: `needs: [changes, build-and-test]`, `if: always()`, and one step that fails when `contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')`.

## 2. `mega-linter.yml` Changes

Both tasks modify `.github/workflows/mega-linter.yml` — no parallel execution.

- [ ] 2.1 Add the same `changes` job as task 1.2 (design Decisions 1 and 5). Leave the triggers (`push: [main]`, `pull_request: [main]`), the `env` fix settings, and the `concurrency` block untouched.
- [ ] 2.2 Gate the `megalinter` job: add `needs: changes` and the quoted `if: needs.changes.outputs.code == 'true'`. Preserve the job's `name: MegaLinter`, its `permissions`, the PAT checkout, and both fix-commit steps exactly. Do not add an aggregate job to this workflow (design Decision 5).

## 3. Local Validation

- [ ] 3.1 Validate both workflow files parse: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"` and the same for `mega-linter.yml`. If `actionlint` is installed, run it on both for GitHub Actions schema validation. YAML syntax is the only locally checkable surface; the first workflow execution is the real validation (same note as `release-ci-gate` task 2.1).
- [ ] 3.2 [P] Reproduce the detection logic locally in a throwaway clone (for example under `/tmp/opencode/`): build a base commit and head commits covering the spec scenarios, then run the exact pipeline from task 1.2 and assert `code=false` for a learnings-only diff (additions plus edits), `code=false` for a learnings-only deletion, `code=true` for a change touching any file outside `.uf/dewey/learnings/`, `code=true` for a diff mixing a learning with a code file, and `code=true` for a file moved into `.uf/dewey/learnings/` from elsewhere. The push short-circuit is an Actions expression — it is covered live by task 5.3, not here.
- [ ] 3.3 [P] CI Parity Gate (AGENTS.md): run locally the exact commands `build-and-test` runs when it runs — `go build -v ./...`, `go vet ./...`, `go test -race -v -count=1 -coverprofile=coverage.out ./...`, and the Gaze threshold gate `gaze crap --format=json --coverprofile=coverage.out --max-crapload=48 --max-gaze-crapload=45 ./...` (install gaze first if needed: `go install github.com/unbound-force/gaze/cmd/gaze@latest`). Any failure is a blocking error.
- [ ] 3.4 [P] Scope check: `git diff --stat main` (or the merge base) shows only `.github/workflows/ci.yml` and `.github/workflows/mega-linter.yml` changed besides the `openspec/changes/skip-ci-for-learnings-prs/` artifacts.

## 4. Constitution Alignment and Documentation Gate

- [ ] 4.1 Verify constitution alignment against the proposal's PASS assessments: Observable Quality (gates unchanged for every code change and `main` push; `ci-ok` gives one machine-readable verdict per run), Composability First (no new third-party actions or Go dependencies — detection reuses the pinned `actions/checkout` and plain git), Autonomous Collaboration (autonomous learnings PRs resolve checks faster; no tool contract changes), Testability (fail-closed detection proven by task 3.2; live behavior by group 5).
- [ ] 4.2 [P] Documentation validation gate (AGENTS.md): confirm the assessment recorded in the proposal's Impact section still holds — AGENTS.md's CI/CD line stays accurate at its level of detail, GoDoc is untouched (no Go changes), and no website sync issue is needed (CI orchestration is not user-facing behavior). Record the outcome in the PR body; a one-line AGENTS.md refinement may ride a future docs pass, outside this change's file scope.

## 5. Live Verification

- [ ] 5.1 This change's own PR is the code-path test: it changes `.github/workflows/`, so confirm `changes` reports `code: 'true'` and `build-and-test` plus `megalinter` run in full on it.
- [ ] 5.2 After merge, verify a learnings-only PR — the next gh-hud `chore/dewey-learnings-*` PR, or a throwaway test PR adding a file under `.uf/dewey/learnings/`: `changes` reports `code: 'false'`, `build-and-test` and `megalinter` report skipped, `ci-ok` reports success, and every required check passes within a couple of minutes.
- [ ] 5.3 After merge, observe a push to `main`: the `changes` job outputs `code: 'true'`, and `build-and-test` and `MegaLinter` run and report under their existing names — the release preflight's Checks API lookups stay intact.

## 6. Repository Settings (admin, out-of-repo)

- [ ] 6.1 Replace the required status check context `build-and-test` with `ci-ok` in classic branch protection on `main` (Settings → Branches → `main` → required status checks). Human/admin action, not committed to this repo. Safe at any time after merge: before the swap, learnings-only PRs pass via the skipped `build-and-test` (design Decision 6).
