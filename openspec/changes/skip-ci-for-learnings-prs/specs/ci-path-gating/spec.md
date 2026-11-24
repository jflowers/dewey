## ADDED Requirements

### Requirement: Pull Request Path Detection

Each pull-request-triggered workflow (`ci.yml` and `mega-linter.yml`) MUST include a `changes` job that determines whether the pull request changes any file outside `.uf/dewey/learnings/` by diffing the pull request head against its merge base with the base branch. The job MUST output `code: 'true'` when any changed path lies outside that directory and `code: 'false'` otherwise.

#### Scenario: Learnings-only diff yields code=false

- **GIVEN** a pull request that adds and edits files only under `.uf/dewey/learnings/`
- **WHEN** the `changes` job diffs the head against the merge base
- **THEN** it outputs `code: 'false'`

#### Scenario: Any file outside learnings yields code=true

- **GIVEN** a pull request that also changes `.github/workflows/ci.yml`
- **WHEN** the `changes` job diffs the head against the merge base
- **THEN** it outputs `code: 'true'`

#### Scenario: Learnings deletions stay learnings-only

- **GIVEN** a pull request whose only change deletes `.uf/dewey/learnings/old-learning.md`
- **WHEN** the `changes` job diffs the head against the merge base
- **THEN** it outputs `code: 'false'`, because the deleted path still lies under `.uf/dewey/learnings/`

#### Scenario: Moving a file into learnings from elsewhere counts as code

- **GIVEN** a pull request whose only change moves `docs/notes.md` to `.uf/dewey/learnings/notes.md`
- **WHEN** the `changes` job diffs the head against the merge base
- **THEN** it outputs `code: 'true'`, because the move touches a path outside `.uf/dewey/learnings/`

### Requirement: Path Detection Event and Failure Semantics

On events other than `pull_request`, the `changes` job MUST output `code: 'true'` unconditionally, so non-PR runs behave exactly as before this change. If the diff cannot be computed, the job MUST fail rather than guess.

#### Scenario: Push events yield code=true

- **GIVEN** a `push` event on `main`
- **WHEN** the `changes` job runs
- **THEN** it outputs `code: 'true'` without requiring pull-request context

#### Scenario: Detection failure fails the job

- **GIVEN** a pull request for which the diff command errors
- **WHEN** the `changes` job runs
- **THEN** the job exits non-zero without emitting a `code` output, so path detection can never silently guess

### Requirement: Skip Build, Test, and Lint for Learnings-Only Pull Requests

For a pull request whose changed files all lie under `.uf/dewey/learnings/`, the workflows MUST skip the build, test, and lint jobs without building the project, running tests, or running any linter, and every required check MUST still report a passing conclusion. For any pull request that changes at least one file outside `.uf/dewey/learnings/` — alone or together with learnings files — the workflows MUST run the same jobs as before this change.

#### Scenario: Learnings-only PR skips build, test, and lint

- **GIVEN** a pull request whose changed files (additions, edits, and deletions) all lie under `.uf/dewey/learnings/`
- **WHEN** the `CI` and `MegaLinter` workflows run for that pull request
- **THEN** the `changes` job completes with output `code: 'false'`, the `build-and-test` and `megalinter` jobs are skipped, and no step of either workflow builds the project, runs a test, or runs a linter

#### Scenario: Required checks pass on a learnings-only PR

- **GIVEN** a learnings-only pull request
- **WHEN** both workflows finish
- **THEN** the `ci-ok` check reports success and the skipped `build-and-test` check reports a skipped conclusion, which satisfies a branch-protection requirement on that check name — the pull request shows all required checks passing and waits on no expected status

#### Scenario: Learnings mixed with code runs full CI

- **GIVEN** a pull request that changes both `.uf/dewey/learnings/authentication-notes.md` and `server.go`
- **WHEN** both workflows run
- **THEN** the `changes` job outputs `code: 'true'` and `build-and-test` and `megalinter` run with the same steps as before this change

#### Scenario: Code-only PR runs full CI

- **GIVEN** a pull request that changes only files outside `.uf/dewey/learnings/`
- **WHEN** both workflows run
- **THEN** the `changes` job outputs `code: 'true'` and every job runs exactly as before this change

### Requirement: Aggregate CI Verdict Check

`ci.yml` MUST provide an aggregate `ci-ok` job that runs on every workflow run after the `changes` job and every gated job, using `if: always()` with `needs` covering all of them. `ci-ok` MUST succeed when every needed job either succeeded or was skipped, and MUST fail when any needed job failed or was cancelled. `ci-ok` MUST report a conclusion on every pull request run, making it suitable as the repository's sole required check.

#### Scenario: ci-ok passes when gated jobs are skipped

- **GIVEN** a learnings-only pull request run where `build-and-test` was skipped
- **WHEN** `ci-ok` runs
- **THEN** it reports success

#### Scenario: ci-ok fails when a gated job fails

- **GIVEN** a pull request run where `build-and-test` failed
- **WHEN** `ci-ok` runs
- **THEN** it reports failure

#### Scenario: ci-ok fails when a gated job is cancelled

- **GIVEN** a pull request run where `build-and-test` was cancelled
- **WHEN** `ci-ok` runs
- **THEN** it reports failure

#### Scenario: ci-ok fails when path detection fails

- **GIVEN** a pull request run where the `changes` job failed
- **WHEN** `ci-ok` runs
- **THEN** it reports failure, so a detection error can never produce a green pull request

### Requirement: Push-to-Main Runs Unchanged

Every push to `main` MUST run the full CI suite exactly as before this change: the `changes` job MUST output `code: 'true'`, `build-and-test` MUST run all of its steps and report under its existing check name, and the `MegaLinter` check MUST run and report under its existing check name. `ci.yml` MUST NOT trigger on pushes to branches other than `main`.

#### Scenario: Push to main runs the full suite

- **GIVEN** a push to `main`
- **WHEN** the workflows run
- **THEN** `build-and-test` executes every step — including `go test -race -v -count=1 -coverprofile=coverage.out` and the Gaze threshold gate — and the `MegaLinter` job executes, with both reporting under the same check names as before this change

#### Scenario: Push to a non-main branch triggers no CI run

- **GIVEN** a push to a branch other than `main`, with no pull request open
- **WHEN** the push lands
- **THEN** `ci.yml` does not run for that push, because pull request branches are covered by the `pull_request` trigger

### Requirement: Gated Jobs Run Unchanged When They Run

When the gated jobs run, they MUST execute the same steps, commands, flags, thresholds, permissions, and SHA-pinned actions as before this change. The workflows MUST NOT use trigger-level `paths-ignore` to implement the skip, because a workflow that provides a required check must report on every pull request.

#### Scenario: build-and-test runs its identical step list

- **GIVEN** a pull request that changes Go code
- **WHEN** `build-and-test` runs
- **THEN** it executes the same steps with the same commands — including `go test -race -v -count=1 -coverprofile=coverage.out` and the Gaze threshold gate with `--max-crapload=48` and `--max-gaze-crapload=45` — and the same SHA-pinned action versions as before this change

#### Scenario: megalinter runs with identical configuration

- **GIVEN** a pull request that changes linted files
- **WHEN** the `megalinter` job runs
- **THEN** it runs the same SHA-pinned MegaLinter action with the same job name (`MegaLinter`), permissions, and fix-commit behavior as before this change

#### Scenario: Workflows report on every pull request

- **GIVEN** a learnings-only pull request
- **WHEN** the workflows trigger
- **THEN** both workflows create check runs for that pull request — the skip comes from job-level conditions, not from the trigger — so no required check is left unreported

Note: the check names are load-bearing. `build-and-test` is the job key in `ci.yml` (it has no `name:` override) and `MegaLinter` is the `name:` field of the `megalinter` job in `mega-linter.yml`; the release preflight (active `release-ci-gate` change) looks up both names on dispatched `main` commits via the Checks API. A plain job skipped by a job-level `if:` reports a skipped conclusion under its own name, which satisfies a required check of that name; matrix jobs and reusable-workflow calls do not report per-leg or nested names when skipped, which is why the aggregate `ci-ok` job is the required check rather than any gated job.
