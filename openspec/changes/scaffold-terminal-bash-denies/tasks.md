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

  Phases 1–3 are one atomic unit (design D4): the
  byte-identity and shape guards are intentionally red
  from the first agents.go payload edit until the
  phase-3 tracked-mirror refresh lands. The phase-3
  checkpoint is the first green gate — do not "fix" the
  intermediate red by hand-editing the tracked mirrors.

  Phase checkpoints run the exact Build/Vet/Test steps from
  .github/workflows/ci.yml (see each checkpoint). The Gaze
  threshold gate and MegaLinter run only at the Phase 5.1
  CI-parity gate, which is the sole full CI-parity checkpoint.
-->

## 0. Preconditions

- [ ] 0.1 Confirm the change artifacts under
  `openspec/changes/scaffold-terminal-bash-denies/` are committed and
  pushed on `opsx/scaffold-terminal-bash-denies` (Spec Commit Gate), with
  `originating_issue: 215` recorded in the change's `.openspec.yaml`.

## 1. Embedded Payloads (agents.go)

- [ ] 1.1 Declare the shared `bashTerminalDenies` constant above
  `deweyAgentDefinitions` (design D2): an interpreted string
  concatenation, one deny line per operand with escaped quotes,
  four-space indentation matching the existing bash entries, every
  line newline-terminated so the splice joins cleanly at line
  boundaries, in the dewey #215 enumeration order:

  ```go
  const bashTerminalDenies = "    \"*>*\": deny\n" +
  	"    \"*<*\": deny\n" +
  	"    \"*|*\": deny\n" +
  	"    \"*;*\": deny\n" +
  	"    \"*&*\": deny\n" +
  	"    \"*$(*\": deny\n" +
  	"    \"*`*\": deny\n"
  ```

  The final operand's backtick is legal inside an interpreted string;
  the constant is the only route a backtick byte reaches any payload.
  Add a GoDoc comment on the constant stating exactly that rule.
- [ ] 1.2 Restructure the `deweyAgentDefinitions["curator.md"]` map
  value into the three-part splice (design D2): a raw-string head
  ending immediately after the `"dewey gate auto-answer": allow`
  line's newline, then `+ bashTerminalDenies +`, then a raw-string
  tail beginning at the `---` frontmatter terminator. The resulting
  `permission.bash` map is exactly nine entries in the CA-FR-015
  order: `"*": deny`, `"dewey gate auto-answer": allow`, then the
  seven terminal denies. No `tools:` entry, `permission:` sibling,
  prompt-body byte, or any other line of the payload changes
  (CA-FR-015 adds no allow and removes none); the byte-level result
  is the current payload plus the seven inserted deny lines, so every
  existing byte-identity comparison keeps its meaning.
- [ ] 1.3 Apply the same splice to
  `deweyAgentDefinitions["distiller.md"]`: raw-string head ending
  after the `"gh issue comment": allow` line's newline,
  `bashTerminalDenies`, tail from `---`. The map is exactly eleven
  entries in the DA-FR-001 order: `"*": deny`, the three allows
  (`dewey gate auto-answer`, `dewey gate distill-unattended`,
  `gh issue comment`), then the seven terminal denies.
- [ ] 1.4 Update the `agents.go` doc comment's backtick rule (design
  D2): it currently states an agent definition MUST NOT contain a
  backtick because the payloads are raw string literals. Restate it
  as — the raw string literals MUST NOT contain a backtick directly;
  the only backtick in any payload arrives through the
  `bashTerminalDenies` constant splice.

## 2. Frontmatter Shape Guards (cli_test.go)

- [ ] 2.1 Extend `assertCuratorFrontmatterShape` (design D3): grow the
  ordered `wantBash` pin from two entries to the full nine-entry
  sequence — `{"*", "deny"}`,
  `{"dewey gate auto-answer", "allow"}`, then the seven terminal
  denies each `{"<glob>", "deny"}` in the D1 order (the `"*` + "`" +
  `*"` want entry is an ordinary interpreted string in Go). Update
  the count-mismatch message to name the terminal denies instead of
  only `the "*" deny and the single gate allow`. Keep the fail-closed
  exact pin and both shipped copies running through the guard.
- [ ] 2.2 Extend `assertDistillerFrontmatterShape` (design D3): add
  the missing `gh issue comment` allow assertion to the existing loop
  over the documented commands (an existing gap this change closes);
  pin the exact bash key set — eleven entries, broad deny + three
  allows + seven terminal denies, no inert extras, mirroring the
  curator guard's exact key-set pins; assert every terminal deny
  carries the value `deny`; and add the terminal-position assertion —
  the final seven `bashPairs` entries, in order, are the seven denies
  in the D1 order.
- [ ] 2.3 Add two doctored-document cases to
  `TestCuratorAgent_GuardsRejectDoctoredDocuments` (design D3; the
  CA-FR-015 guard-rejection scenarios): one deletes a terminal deny
  from a copy of the tracked curator.md (expects the count/content
  mismatch problem), one moves a terminal deny to precede the gate
  allow (expects the ordered-sequence mismatch problem), both via the
  existing `strings.Replace` doctor pattern through the
  `problemRecorder`.
- [ ] 2.4 Confirm no other test file needs changes:
  `scaffold_test.go` fixtures are payload-agnostic (verified — no
  fixture pins bash-block bytes);
  `TestInitCmd_ScaffoldsCuratorAgentFromMap` and the distiller
  byte-identity coverage assert tracked = map entry = scaffolded
  copy, so the CA-FR-015 / DA-FR-001 scaffolded-copy scenarios are
  covered transitively once the payloads and mirrors update; the
  allowlist-parity guards compare tool sets, not bash entries, and
  are unaffected. Extend a test only if the new shape reveals a gap;
  do not weaken any existing assertion.

## 3. Tracked-Mirror Refresh (design D4)

- [ ] 3.1 Refresh the tracked mirrors through the actual writer, never
  by hand: run `go run . init --vault .` from the repository root
  (the marker-bootstrap path tool-owned-scaffold-updates used) and
  confirm both agents log `updated` (they classify as
  untouched-but-drifted once the embedded payloads change — their
  markers' digests still match their own pre-change bodies). Verify
  the diff under `.opencode/` is exactly
  `.opencode/agents/curator.md` and `.opencode/agents/distiller.md`
  with the seven new deny lines and fresh final-line `dewey:scaffold`
  sha256 markers computed by `scaffoldPayload` over the updated
  canonical payloads. If any other scaffold (slash command, workflow
  template) also reports `updated`, stop and investigate before
  committing — this change must not carry unrelated scaffold drift.
- [ ] 3.2 Confirm the generic payload-agnostic marker tests stay
  green (`TestInitCmd_FirstInitWritesVerifiableMarkers`,
  `TestScaffoldMarker_DigestMatchesIndependentSHA256`).

**Checkpoint** (exact ci.yml Build/Vet/Test steps — the first green
gate; the suite is intentionally red between task 1.2 and this point,
per design D4):
`go build -v ./... && go vet ./... && go test -race -v -count=1
-coverprofile=coverage.out ./...`

## 4. Documentation

- [ ] 4.1 [P] Update `AGENTS.md`: add a Recent Changes bullet for
  `scaffold-terminal-bash-denies` (the seven terminal metacharacter
  denies closing both scaffolded agents' bash maps, the shared
  `bashTerminalDenies` splice, the guard extensions, and the
  unbound-force/unbound-force#1139 unblock); update the two bullets
  that describe the bash maps — Server-side synthesis (the curator's
  single gate carve-out) and Weekly unattended distill (the
  distiller's two-gates-plus-notice allowlist) — so each states that
  the map ends with the seven terminal metacharacter denies placed
  after every allow.
- [ ] 4.2 [P] Update `README.md`: extend the curator agent
  security-posture sentence (currently "the only shell command it may
  run is `dewey gate auto-answer`") and the distiller agent's
  corresponding bash-surface sentence to mention the seven terminal
  metacharacter denies that refuse injection payloads sharing an
  allowed prefix; note that consumer copies customized by hand are
  preserved by re-init and keep their exposure until reconciled (the
  design's accepted customized-copies risk).
- [ ] 4.3 [P] Website sync per the Documentation Gate: check whether
  the open prior issues covering the curator/distiller permission
  model (unbound-force/website#371 docs, #376 tutorial, #377 blog)
  still cover the bash-map posture; if they are open and covering,
  extend them by comment with the terminal-deny hardening; only if
  they are closed or do not cover the bash map, file
  `gh issue create --repo unbound-force/website --label docs` titled
  `docs: sync dewey agent bash permission hardening documentation`
  describing the seven terminal denies, both scaffolded agents, and
  which `content/docs/` pages need updating. Do not create a
  duplicate. Record the outcome here.

**Checkpoint** (exact ci.yml Test step):
`go test -race -v -count=1 -coverprofile=coverage.out ./...`

## 5. Verification

- [ ] 5.1 Run the full CI-parity gate, deriving every command from
  `.github/workflows/ci.yml` and
  `.github/workflows/mega-linter.yml` (this is the sole full
  CI-parity checkpoint): `go build -v ./...`, `go vet ./...`,
  `go test -race -v -count=1 -coverprofile=coverage.out ./...`,
  `gaze crap --format=json --coverprofile=coverage.out
  --max-crapload=48 --max-gaze-crapload=45 ./...` (exit non-zero on
  regression), and the MegaLinter go flavor locally via
  `npx mega-linter-runner --flavor go` (gofmt/goimports,
  golangci-lint, markdownlint, yamllint, gitleaks — the tracked
  agent mirrors are markdown the linters re-check). Failures block
  the task.
- [ ] 5.2 Perform the manual acceptance matrix (design D5) under the
  installed host with the scaffolded agents and record the results
  here: every allow remains permitted — curator `dewey gate
  auto-answer`; distiller `dewey gate auto-answer`,
  `dewey gate distill-unattended`, and `gh issue comment …`; and
  metacharacter-bearing payloads sharing an allowed prefix are
  refused by the map itself — `dewey gate auto-answer; true`,
  `` dewey gate auto-answer `id` `` (curator), and
  `` gh issue comment 1 --body `x` `` plus
  `gh issue comment 1 --body $(true)` (distiller). If the installed
  host does not honor the metacharacter globs or does not resolve
  them last-match-wins, execute the spec's escalation clause —
  record the gap and escalate rather than silently accepting an
  inert deny tail (the CA-FR-015 / DA-FR-001 runtime scenarios).
- [ ] 5.3 Verify constitution alignment (the proposal records all
  four principles PASS; confirm they hold on the implementation):
  principles I–IV each re-checked; `git status --short` shows only
  `agents.go`, `cli_test.go`, `.opencode/agents/curator.md`,
  `.opencode/agents/distiller.md`, `AGENTS.md`, `README.md`, and the
  change directory touched; no `tools:` block entry, prompt body,
  slash command, workflow template, gate subcommand, or MCP tool
  changed; no runtime code path outside the scaffold map is touched,
  so the 37 inherited graphthulhu tools and all 55 MCP tools behave
  identically.
- [ ] 5.4 Confirm the consumer unblock: both agents'
  `permission.bash` maps now end with the exact seven terminal denies
  in the order unbound-force/unbound-force#1139's instruction-lint
  requires — `*>*`, `*<*`, `*|*`, `*;*`, `*&*`, `*$(*`, `` *`* `` as
  the last seven entries — so a consumer re-scaffold through
  `dewey init` (first init or untouched-drifted re-init) passes that
  lint and #1139 unblocks.

**Checkpoint** (exact ci.yml Test step):
`go test -race -v -count=1 -coverprofile=coverage.out ./...`

## 6. Review

- [ ] 6.1 Run `/uf.review-council` and resolve findings until all
  reviewers return APPROVE; re-run the Phase 5 checks after any fix.

<!-- scaffolded by uf vdev -->
