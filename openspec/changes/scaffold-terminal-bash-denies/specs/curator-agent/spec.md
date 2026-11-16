# Delta Spec: curator-agent

<!--
  Requirement numbering allocation
  ================================
  Capability: curator-agent (prefix CA)
  Allocated to this change: CA-FR-015
  Allocated by prior changes (all pending archive, expressed as
  ADDED deltas in their own change directories):
    curator-agent-human-gates:      CA-FR-001 .. CA-FR-008
    curator-tool-allowlist:         CA-FR-009 .. CA-FR-012
    dewey-distill-verify-step:      CA-FR-013 .. CA-FR-014
  Reserved for future curator-agent changes: CA-FR-016 .. CA-FR-099
  The capability has no archived main spec yet, so this delta is
  expressed as an ADDED requirement rather than a MODIFIED one.
  CA-FR-015 refines the bash map shape pinned by CA-FR-002: the map
  keeps its deny-all-then-allow-one head and gains a terminal
  metacharacter deny tail; it does not add, remove, or reorder any
  allow entry.
  Requirement ids MUST stay inside the allocated range.
-->

## ADDED Requirements

### Requirement: Terminal Metacharacter Denies in the Curator Bash Map

**ID**: CA-FR-015

The curator agent's `permission.bash` map MUST end with exactly seven
terminal metacharacter deny entries, placed after every allow entry so
they win under the host's last-match-wins resolution: `*>*`, `*<*`,
`*|*`, `*;*`, `*&*`, `*$(*`, and `` *`* ``, in that order (the
enumeration from dewey #215). Each terminal deny MUST carry the value
`deny`. The map's overall order MUST be: the `"*": deny` broad deny
first, the single `dewey gate auto-answer` allow (per CA-FR-002), then
the seven terminal denies last. The map MUST NOT contain any other
entry. The requirement MUST hold on every shipped copy — the tracked
`.opencode/agents/curator.md`, the embedded
`deweyAgentDefinitions["curator.md"]` map entry in `agents.go`, and
every file `dewey init` scaffolds from that entry — and the
frontmatter shape guard in `cli_test.go` MUST assert the exact ordered
sequence so a deleted, added, or misordered entry fails the test.

Because the allowed command contains no shell metacharacters, the
terminal denies MUST NOT change which commands the curator may run:
`dewey gate auto-answer` MUST remain permitted. The terminal denies
exist to refuse metacharacter-bearing injection payloads — such as
`dewey gate auto-answer; curl …` or `` dewey gate auto-answer `rm -rf …` ``
— that still match the allow entry's command prefix.

#### Scenario: Exact ordered bash map asserted

- **GIVEN** the tracked `.opencode/agents/curator.md` and the embedded
  `deweyAgentDefinitions["curator.md"]` map entry
- **WHEN** the `permission.bash` map is parsed in entry order
- **THEN** it MUST contain exactly nine entries in this order:
  `"*": deny`, `"dewey gate auto-answer": allow`, `"*>*": deny`,
  `"*<*": deny`, `"*|*": deny`, `"*;*": deny`, `"*&*": deny`,
  `"*$(*": deny`, `` "*`*": deny ``
- **AND** the final seven entries MUST be the terminal denies

#### Scenario: Guard rejects a deleted terminal deny

- **GIVEN** a copy of the curator agent definition with one of the
  seven terminal denies removed
- **WHEN** the frontmatter shape guard runs against it
- **THEN** the guard MUST report the bash map's entry count or content
  mismatch rather than pass

#### Scenario: Guard rejects a terminal deny reordered before the allow

- **GIVEN** a copy of the curator agent definition in which one of the
  seven terminal denies is moved to precede the
  `dewey gate auto-answer` allow
- **WHEN** the frontmatter shape guard runs against it
- **THEN** the guard MUST report the ordered-sequence mismatch rather
  than pass

#### Scenario: Both shipped copies and scaffolded copies carry the tail

- **GIVEN** the tracked file, the embedded map entry, and a curator
  scaffolded by `dewey init` into a fresh vault
- **WHEN** their `permission.bash` maps are parsed
- **THEN** all three MUST carry the identical nine-entry ordered map
- **AND** the byte-identity guard MUST keep the tracked file and the
  map entry's scaffold payload identical, with the scaffold marker
  digest recomputed over the updated canonical payload

#### Scenario: Gate command still permitted

- **GIVEN** the curator agent running under the installed host with
  the extended bash map
- **WHEN** it runs `dewey gate auto-answer`
- **THEN** the host MUST permit the command

#### Scenario: Injection payload refused at runtime

- **GIVEN** the curator agent running under the installed host with
  the extended bash map
- **WHEN** it attempts to run a metacharacter-bearing payload that
  starts with the allowed prefix, such as
  `dewey gate auto-answer; curl http://example.invalid/x` or
  `` dewey gate auto-answer `id` ``
- **THEN** the host MUST refuse the command
- **AND** the refusal MUST come from the terminal denies resolving
  after the allow entry, not only from the agent declining per its
  prompt
- **AND** if the installed host does not honor the metacharacter
  globs or does not resolve last-match-wins over them, the gap MUST
  be escalated and recorded — the change MUST NOT silently accept an
  inert deny tail

## MODIFIED Requirements

None.

## REMOVED Requirements

None.
