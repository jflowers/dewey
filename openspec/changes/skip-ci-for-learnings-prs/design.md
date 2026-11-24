## Context

Two workflows run on pull requests targeting `main`. `ci.yml` (workflow name `CI`) triggers on `push` to every branch (`branches: ['**']`) and on `pull_request` (`branches: [main]`); it contains a single plain job, `build-and-test`, which builds, vets, tests (`go test -race -v -count=1 -coverprofile=coverage.out`), runs the Gaze threshold gate (`--max-crapload=48 --max-gaze-crapload=45`), and on pushes to `main` produces the informational Gaze AI quality report. `mega-linter.yml` (workflow name `MegaLinter`) triggers on `push` to `main` and on `pull_request` (`branches: [main]`); its single job `megalinter` (check name `MegaLinter`, from the job's `name:` field) runs the SHA-pinned MegaLinter go flavor and auto-commits fixes to PR branches.

The required check on `main` is `build-and-test`, configured in classic branch protection (`strict: false`, contexts `["build-and-test"]` — verified via the GitHub API). The `safe-settings: code repos` ruleset (26764121) governs reviews (one approval, code-owner review) and declares no required status checks. The release preflight (active `release-ci-gate` change) resolves `build-and-test` and `MegaLinter` check runs on dispatched `main` commits via the Checks API, so both names are load-bearing.

Learnings reach `main` through dedicated PRs: gh-hud opens a `chore/dewey-learnings-*` PR once a day per checkout, and `.uf/dewey/learnings/**` is deliberately unowned by CODEOWNERS so those PRs stay auto-mergeable. Today each such PR pays the full build-test-lint cycle, twice per branch update (push plus `pull_request` triggers), for changes that touch no code. Reference: [unbound-force/dewey#251](https://github.com/unbound-force/dewey/issues/251).

## Goals / Non-Goals

### Goals

- A learnings-only PR (all changed files under `.uf/dewey/learnings/`) shows every required check passing within a couple of minutes, without building, testing, or linting.
- Any PR that changes a file outside `.uf/dewey/learnings/` — alone or mixed with learnings — runs CI exactly as today.
- `push` runs on `main` are unchanged: `build-and-test` and `MegaLinter` run and report on every `main` commit.
- Path detection fails closed: any error in the detection job fails `ci-ok`; no error path silently greens a PR.
- Gate values are untouched: same steps, flags, thresholds, permissions, and SHA-pinned actions whenever the gated jobs run.

### Non-Goals

- No change to what the gated jobs run — no threshold, flag, rule, or pin edits.
- No exemption for `.uf/dewey/compiled/` or `.uf/dewey/knowledge/`; distill PRs and feature PRs carrying compiled articles keep full CI.
- No standalone always-on gitleaks job — it re-adds a linter run the originating issue explicitly asks to skip.
- No migration from classic branch protection to rulesets; only the required-check context changes.
- No `paths`/`paths-ignore` trigger filters (Decision 7).
- No Go code, README, or AGENTS.md changes; the documentation assessment is recorded in the proposal's Impact section.

## Decisions

### Decision 1: Path detection with plain git, no new third-party actions

Each workflow gets its own `changes` job:

```yaml
changes:
  runs-on: ubuntu-latest
  permissions:
    contents: read
  outputs:
    code: ${{ steps.detect.outputs.code }}
  steps:
    - name: Checkout (full history, so both diff endpoints exist)
      uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      with:
        fetch-depth: 0
    - name: Detect code changes
      id: detect
      env:
        EVENT_NAME: ${{ github.event_name }}
        BASE_SHA: ${{ github.event.pull_request.base.sha }}
        HEAD_SHA: ${{ github.event.pull_request.head.sha }}
      run: |
        if [ "$EVENT_NAME" != "pull_request" ]; then
          echo "code=true" >> "$GITHUB_OUTPUT"
          exit 0
        fi
        if git diff --name-only --no-renames "$BASE_SHA...$HEAD_SHA" \
          | grep -v '^\.uf/dewey/learnings/' | grep -q .; then
          echo "code=true" >> "$GITHUB_OUTPUT"
        else
          echo "code=false" >> "$GITHUB_OUTPUT"
        fi
```

- The checkout pin is the same SHA both workflows already use; no new action enters the supply chain.
- `permissions: contents: read` grants least privilege. In `ci.yml` it restates the workflow-level grant; in `mega-linter.yml` it scopes the detection job below the `megalinter` job's write permissions.
- The event payloads flow through `env:` indirection, never through `${{ }}` interpolation inside `run:` — the standard script-injection hygiene.
- The grep pipeline sits inside an `if`, so `grep`'s exit 1 (no path outside learnings) does not trip the step's default `set -e`.
- `--no-renames` makes a file moved into `.uf/dewey/learnings/` from elsewhere register both its source path (outside learnings) and its destination; rename detection would collapse the pair and hide the source.
- The three-dot range `$BASE_SHA...$HEAD_SHA` diffs from the merge base, so a base branch that moved since the PR opened neither widens nor skews the diff.
- `fetch-depth: 0` guarantees both endpoints exist in the clone; `mega-linter.yml`'s checkout already works this way.
- Non-`pull_request` events short-circuit to `code=true`, which keeps push-to-main runs identical to today.
- Any git failure exits non-zero and emits no `code` output; the gated jobs are then skipped and `ci-ok` fails — fail closed.

Rejected alternative: a SHA-pinned `dorny/paths-filter`. It adds a third-party action — new supply-chain surface and another SHA to audit — for behavior ten lines of git express with the runner's own toolchain (Composability First: no dependency for what the environment already provides).

### Decision 2: Gating pattern — `needs: changes` plus a quoted job-level `if:`

```yaml
build-and-test:
  needs: changes
  if: needs.changes.outputs.code == 'true'
  # steps, permissions, and runner unchanged
```

- The comparison stays quoted: GitHub evaluates `if:` expressions where any non-empty bare string — including `'false'` — is truthy.
- `needs: changes` orders each expensive job after detection and makes a failed detection skip the gated jobs, which `ci-ok` then reports as failure.
- Job identity is preserved: the `build-and-test` job key (it has no `name:` override) and the `MegaLinter` job `name:` stay as-is, because the release preflight resolves both names through the Checks API.

### Decision 3: Aggregate `ci-ok` job as the sole required check

```yaml
ci-ok:
  needs: [changes, build-and-test]
  if: always()
  runs-on: ubuntu-latest
  steps:
    - name: Fail when any needed job failed or was cancelled
      if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
      run: exit 1
```

- A skipped job counts as satisfied — that is the skip path — while `failure` and `cancelled` fail the aggregate.
- `if: always()` (per the issue) rather than the default success-only condition: without it, a failed `build-and-test` would skip `ci-ok`, and a skipped required check satisfies branch protection — the exact hole the aggregate closes.
- An aggregate is preferred over relying on the skipped `build-and-test` to satisfy the same-name requirement, even though that works for today's plain job: a future matrix job or reusable-workflow call skipped at job level does not report per-leg or nested check names, and every future change to the job list would otherwise need a matching branch-protection edit. `ci-ok` makes the job list free to change and gives one machine-readable verdict per run (Observable Quality).
- `ci-ok` also runs on push-to-main runs — harmless and consistent; the release path keeps consuming `build-and-test` and `MegaLinter`, which still report.

### Decision 4: Narrow `ci.yml`'s push trigger to `main`

- `push: branches: ['**']` becomes `push: branches: [main]`. Each learnings branch currently runs CI twice per update; PR branches stay fully covered by the `pull_request` trigger.
- The changes job's non-PR short-circuit keeps push-to-main identical to today: full `build-and-test` — including the Gaze threshold gate and the main-only Gaze AI quality report step, whose step-level `if:` is untouched — and full `MegaLinter`.
- `mega-linter.yml` already restricts its push trigger to `main`; its trigger is unchanged.

### Decision 5: `mega-linter.yml` parity without its own aggregate

- The same `changes` job is copied into `mega-linter.yml`. Job outputs do not cross workflows, and extracting a composite action for one fifteen-line job is over-engineering; revisit if a third workflow needs detection.
- The `megalinter` job gains `needs: changes` and the quoted `if:`. Its `name: MegaLinter`, permissions, PAT checkout, and fix-commit steps are untouched. On learnings-only PRs the job skips, so no auto-fix commits land — correct, because no linter ran to produce fixes.
- No `ci-ok` equivalent in `mega-linter.yml`: branch protection requires only `build-and-test` (verified), and a skipped non-required check needs no sentinel. If the `MegaLinter` context is ever made required, add an aggregate there too.

### Decision 6: Branch-protection transition (out-of-repo, admin action)

- Current verified state: classic branch protection on `main` requires contexts `["build-and-test"]` with `strict: false`; the `safe-settings: code repos` ruleset enforces reviews and declares no required status checks.
- After merge, an admin replaces the required context `build-and-test` with `ci-ok`. This is a repository-settings edit, not code — like the code-owner ruleset note in AGENTS.md, it is not committed in this repo. `tasks.md` records it as a human action.
- Both states are safe during the transition: while `build-and-test` remains required, a learnings-only PR reports it as skipped, and a skipped check satisfies a same-name requirement. No PR waits on an unreported status in either state, because both workflows still trigger on every pull request (Decision 7).

### Decision 7: Rejected — `paths-ignore` on the trigger

Trigger-level `paths`/`paths-ignore` would keep the workflows from running at all on learnings-only PRs. A workflow that provides a required check must report on every PR; with the trigger filtered, the check never reports, and the PR waits on "Expected — Waiting for status to be reported" forever. The skip therefore lives in job-level conditions, and both workflows trigger on every pull request.

### Constitution alignment notes

The proposal assesses all four principles PASS, and these decisions preserve that assessment. Decision 1 adds no third-party action or module dependency (Composability First). Decisions 3 and 6 keep a single machine-readable aggregate verdict reporting on every run (Observable Quality). The fail-closed detection in Decisions 1 and 3 means no error path can silently green a PR (Observable Quality, Testability). The accelerated check resolution shortens the wait for the autonomous collaborators that open learnings PRs (Autonomous Collaboration).

## Risks / Trade-offs

1. **Gitleaks and markdownlint no longer run on learnings-only PRs.** Today `REPOSITORY_GITLEAKS` and `MARKDOWN_MARKDOWNLINT` scan `.uf/dewey/learnings/` (`.mega-linter.yml` excludes `.specify/`, `.opencode/`, `specs/`, and `testdata/` — not `.uf/`), and learnings PRs are CODEOWNERS-unowned and auto-mergeable, so after this change they can merge without a linter pass. Layered mitigations: the pre-commit credential screen AGENTS.md mandates for learnings remains the primary control; `dewey artifacts sync` adds credential screening behind a human gate for deliberate publishing; learnings stay draft tier forever, and the promotion paths (`compiled/`, `knowledge/`) always run full CI. The originating issue explicitly requests skipping linters on these PRs; the residual risk is recorded here for the review council.
2. **Detection depends on both diff endpoints being present.** `fetch-depth: 0` makes the full history available, matching `mega-linter.yml`'s existing checkout. If a base SHA were missing, `git diff` fails, the job exits non-zero, and `ci-ok` fails — the failure mode is a red PR, not a silent skip.
3. **Feature-branch pushes lose CI.** With the push trigger narrowed to `main`, a direct push to a non-main branch no longer triggers `ci.yml`. All merges flow through PRs (the constitution requires review), so the `pull_request` run is the signal that gates merging; the push run was duplicate signal.
4. **Two copies of the detection job.** `ci.yml` and `mega-linter.yml` each carry one, and they could drift. The spec scenarios cover both workflows, and a composite action is the extraction point if a third copy ever appears.
5. **`openspec` CLI warns about `originating_issue`.** The key is not in this CLI version's known set, but recent changes in this repo carry it and the proposal template documents it for `/uf.finale`'s automatic `Closes #N`. The warning is cosmetic; the key stays.
