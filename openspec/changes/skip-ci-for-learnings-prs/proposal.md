## Why

Dewey learnings PRs add Markdown knowledge files under `.uf/dewey/learnings/` and nothing else: gh-hud opens a `chore/dewey-learnings-*` PR once a day per checkout, and learnings must never ride a feature branch (AGENTS.md, Commit Scope). Such a PR cannot affect the Go binary, its tests, or the linted source tree. Throughout this proposal, a **learnings-only PR** means exactly that: a pull request whose changed files all lie under `.uf/dewey/learnings/`.

Today every learnings-only PR runs the full CI cycle before it can merge: the `build-and-test` job in `ci.yml` (build, vet, `go test -race -v -count=1 -coverprofile=coverage.out`, the Gaze threshold gate) plus the `MegaLinter` job in `mega-linter.yml` (golangci-lint, markdownlint, yamllint, gitleaks, git-diff). That consumes minutes of runner time per PR and delays gh-hud's auto-merge for changes that touch no code.

The cost also doubles: `ci.yml` triggers on `push` to every branch (`branches: ['**']`) as well as on `pull_request`, so each learnings branch runs CI twice per update.

Reference: [unbound-force/dewey#251](https://github.com/unbound-force/dewey/issues/251).

## What Changes

- Add a cheap `changes` job to both `ci.yml` and `mega-linter.yml` (each workflow needs its own; job outputs do not cross workflows). It diffs the pull request against its base and outputs `code: 'true'` when any changed file lies outside `.uf/dewey/learnings/`. On non-`pull_request` events it outputs `code: 'true'` unconditionally.
- Gate every expensive job behind that output: `needs: changes` plus a job-level `if: needs.changes.outputs.code == 'true'`. A job skipped by its `if:` still reports and satisfies a required check of the same name.
- Add one aggregate `ci-ok` job to `ci.yml` — `if: always()`, `needs:` every gated job, failing when any of them failed or was cancelled — and adopt it as the required check. The aggregate stays correct if the workflow later grows matrix jobs or reusable-workflow calls, where a job-level skip does not report under per-leg or nested check names, and it makes future changes to the job list free of branch-protection edits.
- Narrow `ci.yml`'s `push` trigger from `branches: ['**']` to `main`, removing the duplicate per-branch run. `mega-linter.yml` already narrows its `push` trigger to `main`.
- Push runs on `main` keep the full suite: the `changes` job outputs `code: 'true'` on push events, so `build-and-test` and `MegaLinter` still run and report on every `main` commit, which the release preflight depends on.
- No workflow-level `paths-ignore`: a workflow that provides a required check must report on every PR, or the PR waits forever on "Expected — Waiting for status to be reported".
- The gated jobs' contents are untouched — same steps, same `go test -race -v -count=1` flags, same Gaze thresholds (`--max-crapload=48`, `--max-gaze-crapload=45`), same SHA-pinned actions, same job keys and check names (`build-and-test`, `MegaLinter`).

## Capabilities

### New Capabilities

- `ci-path-gating`: CI jobs run conditionally based on whether a pull request changes files outside the exempt learnings path, with an aggregate `ci-ok` job reporting overall CI health as the required check.

### Modified Capabilities

- None.

### Removed Capabilities

- None.

## Impact

- **Files**: `.github/workflows/ci.yml` and `.github/workflows/mega-linter.yml` only. No Go code, no `.mega-linter.yml` changes, no lint-rule changes.
- **Repository settings (out-of-repo, admin action)**: after merge, branch protection's required check moves from `build-and-test` to `ci-ok`. The change is safe in both states: while `build-and-test` remains required, a learnings-only PR shows it as skipped, which satisfies the requirement, so no PR is stranded during the transition.
- **Release preflight (active `release-ci-gate` change)**: unaffected. The preflight verifies `build-and-test` and `MegaLinter` check runs on the dispatched `main` commit via the Checks API; this change preserves both job keys and check names and keeps both jobs running on every push to `main`.
- **Accepted trade-off**: MegaLinter's gitleaks and markdownlint passes no longer run on learnings-only PRs. The primary credential control remains the pre-commit human screen AGENTS.md mandates for learnings; `dewey artifacts sync` adds credential screening behind a human gate when publishing deliberately. A standalone always-on gitleaks job was considered and rejected — it re-adds a linter run the originating issue explicitly asks to skip.
- **Quality-gate values**: unchanged. Any PR that changes a file outside `.uf/dewey/learnings/` — alone or mixed with learnings — runs CI exactly as today, and every push to `main` runs the full suite.
- **Documentation**: AGENTS.md's CI/CD description ("Build + vet + test with `-race -count=1` + Gaze quality report with threshold enforcement on push/PR") stays accurate at its level of detail, so no documentation update rides this change; a one-line refinement noting the learnings-only skip can land in a future docs pass, keeping this change within the issue's declared file scope. No website sync issue: CI orchestration is not user-facing behavior.

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: PASS

The learnings-only PRs this change accelerates are opened by autonomous collaborators (gh-hud's daily `chore/dewey-learnings-*` runs). Today they block on checks that cannot fail for Markdown-only additions; after this change the required checks resolve within a couple of minutes, so the autonomous pipeline waits less. No MCP tool contract, artifact format, or runtime coupling changes.

### II. Composability First

**Assessment**: PASS

Dewey remains independently installable; the change touches CI orchestration only. The `changes` job adds no third-party action — it runs plain git in the runner shell and reuses the already-pinned `actions/checkout`. No new Go module dependencies.

### III. Observable Quality

**Assessment**: PASS

The quality gates keep their values and still run on every code change and every push to `main`. Skips are visible as skipped check runs, and the new `ci-ok` check run provides a single machine-readable aggregate verdict per PR.

### IV. Testability

**Assessment**: PASS

Workflow YAML has limited unit-testability; the verification strategy matches `release-ci-gate`'s: yamllint validation (already part of MegaLinter), local reproduction of the `changes` job's diff logic with the exact git command it runs, and live verification PRs (learnings-only, code-only, mixed learnings-plus-code) as the integration test.
