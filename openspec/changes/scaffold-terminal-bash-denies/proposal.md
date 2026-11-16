## Why

`dewey init` scaffolds `.opencode/agents/curator.md` and
`.opencode/agents/distiller.md` whose `permission.bash` maps end with
`allow` entries and carry no terminal metacharacter denies at all
(dewey #215). OpenCode permission maps resolve last-match-wins, so a
trailing allow that matches a command prefix cannot be overridden for
injection payloads: `dewey gate auto-answer; curl …` still matches the
`dewey gate auto-answer` allow, and `gh issue comment $(…)` still
matches the `gh issue comment` allow. Without deny rules for shell
metacharacters placed **after** every allow, the curated "only the gate
commands" bash surface is not actually closed against command
injection.

The failure is externally observable: the consumer re-scaffold PR
unbound-force/unbound-force#1139 fails that repository's
instruction-lint preflight, which requires the last seven
`permission.bash` entries of every scaffolded persona to be the
terminal denies `*>*`, `*<*`, `*|*`, `*;*`, `*&*`, `*$(*`, and
`` *`* ``. The root cause is here, in Dewey's embedded scaffold map:
the `deweyAgentDefinitions` entries for `curator.md` and `distiller.md`
in `agents.go` end their bash blocks with allow entries, and Dewey's
own tracked `.opencode/agents/` mirrors carry the same broken block.

None of the allowed commands (`dewey gate auto-answer`,
`dewey gate distill-unattended`, `gh issue comment`) contain shell
metacharacters, so appending the terminal denies after the allows
leaves every legitimate command permitted while refusing every
metacharacter-bearing injection form.

## What Changes

- Reorder the `permission.bash` map of the embedded
  `deweyAgentDefinitions["curator.md"]` template in `agents.go` to:
  1. `"*": deny`
  2. the single allow `dewey gate auto-answer`
  3. the seven terminal metacharacter denies placed last so they win
     over every allow: `*>*`, `*<*`, `*|*`, `*;*`, `*&*`, `*$(*`,
     `` *`* ``
- Reorder the `permission.bash` map of the embedded
  `deweyAgentDefinitions["distiller.md"]` template the same way:
  `"*": deny`, then the three allows (`dewey gate auto-answer`,
  `dewey gate distill-unattended`, `gh issue comment`), then the same
  seven terminal denies last.
- Refresh the repo's tracked mirrored copies
  `.opencode/agents/curator.md` and `.opencode/agents/distiller.md`
  to the updated embedded payloads with fresh `dewey:scaffold`
  sha256 markers (the existing tool-owned-scaffold refresh path).
- Extend the drift/parity guards in `cli_test.go` that pin the
  mirrored copies:
  - `assertCuratorFrontmatterShape`'s exact ordered `wantBash` pin
    grows from two entries to the full deny → allow → terminal-deny
    sequence.
  - `assertDistillerFrontmatterShape` gains the terminal-deny
    membership, value, and terminal-position (ordering) assertions.
  - the doctored-document negative paths gain cases for a deleted or
    misordered terminal deny.
- Because the embedded payloads are Go raw string literals and the
  `` *`* `` key contains a backtick, the deny tail is introduced as a
  shared interpreted-string constant spliced into both map entries,
  and the `agents.go` doc comment forbidding backticks in agent
  definitions is updated to describe the splice (see design.md).

No allowed command changes; no `tools:` block, prompt body, or other
frontmatter entry changes; the seven denies are pure hardening.

## Capabilities

### New Capabilities

- `distiller-agent`: The unattended distiller agent's bash permission
  map shape — deny-all first, the three documented allows, and the
  seven terminal metacharacter denies placed last — asserted on both
  shipped copies (the embedded map entry and the tracked file).

### Modified Capabilities

- `curator-agent`: The curator's bash permission map gains the seven
  terminal metacharacter denies after its single gate allow, extending
  the least-privilege boundary pinned by CA-FR-002 so metacharacter
  injection payloads are refused by the map itself.

### Removed Capabilities

None.

## Impact

- **Files**: `agents.go` (embedded templates + doc comment),
  `.opencode/agents/curator.md`, `.opencode/agents/distiller.md`
  (tracked mirrors + markers), `cli_test.go` (shape guards, doctored
  document cases). Possibly `scaffold_test.go` if a fixture pins bash
  block bytes (current fixtures are payload-agnostic).
- **Behavior**: scaffolded agent files gain seven deny entries; the
  set of runnable shell commands is unchanged (no allowed command
  contains a metacharacter). Consumer repos running `dewey init` (or
  re-init) pick up the hardened templates through the existing
  untouched-drifted refresh path, which unblocks
  unbound-force/unbound-force#1139.
- **Docs**: AGENTS.md Recent Changes entry; README curator/distiller
  agent security-posture sentences updated to mention the terminal
  denies; website sync issue per the Documentation Gate.
- **No schema, storage, MCP tool, or CLI surface changes.**

## Constitution Alignment

Assessed against the Unbound Force org constitution.

### I. Autonomous Collaboration

**Assessment**: PASS

The change hardens the artifact that governs agent capability — the
scaffolded agent frontmatter — without adding runtime coupling. The
bash maps remain self-describing YAML artifacts consumed by the OpenCode
host; no Dewey-internal API, shared state, or direct function call is
introduced. Communication between Dewey and its agents stays entirely
through the scaffolded files and MCP tool calls.

### II. Composability First

**Assessment**: PASS

Dewey remains independently installable and usable without any other
Unbound Force tool. The change touches only embedded scaffold content
and its guards; `dewey serve`, the 37 inherited graphthulhu tools, and
all 55 MCP tools are untouched. Consumer repos that never scaffold
agents (no `.opencode/` directory) see no difference at all.

### III. Observable Quality

**Assessment**: PASS

The hardened shape is machine-verifiable and verified: the extended
frontmatter guards assert the exact ordered bash map on both shipped
copies, the doctored-document negative paths prove the guards reject a
deleted or misordered deny, and the scaffold markers make the updated
payload auditable (sha256 digest over the canonical body). No result
schema or provenance metadata changes.

### IV. Testability

**Assessment**: PASS

Every requirement is testable in isolation with the standard library
`testing` package: the shape guards parse the shipped copies with
`yaml.Node` (order-preserving), the negative paths doctor fixtures in
memory, and the marker tests verify digests against independently
computed SHA-256. No external service, running host, or network access
is required. Runtime matcher semantics (last-match-wins over the
metacharacter globs) are verified once at implementation time against
the installed host, following the CA-FR-002 precedent.
