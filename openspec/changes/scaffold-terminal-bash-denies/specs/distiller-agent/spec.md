# Delta Spec: distiller-agent

<!--
  Requirement numbering allocation
  ================================
  Capability: distiller-agent (prefix DA)
  Allocated to this change: DA-FR-001
  This is the first distiller-agent delta: the distiller agent was
  created by the Speckit change specs/018-weekly-distill-automation,
  which does not allocate OpenSpec requirement ids, so no prior DA ids
  exist and the capability starts its numbering here.
  Reserved for future distiller-agent changes: DA-FR-002 .. DA-FR-099
  Sibling capability curator-agent (prefix CA) carries the matching
  terminal-deny requirement for the curator's bash map in this
  change's specs/curator-agent/ delta (CA-FR-015); the shared deny
  tail is specified once per agent so each capability's guards stand
  alone.
  Requirement ids MUST stay inside the allocated range.
-->

## ADDED Requirements

### Requirement: Terminal Metacharacter Denies in the Distiller Bash Map

**ID**: DA-FR-001

The distiller agent's `permission.bash` map MUST end with exactly
seven terminal metacharacter deny entries, placed after every allow
entry so they win under the host's last-match-wins resolution:
`*>*`, `*<*`, `*|*`, `*;*`, `*&*`, `*$(*`, and `` *`* ``, in that
order (the enumeration from dewey #215, identical to the curator's
CA-FR-015 tail). Each terminal deny MUST carry the value `deny`. The
map's overall order MUST be: the `"*": deny` broad deny first, the
three documented allows — `dewey gate auto-answer`,
`dewey gate distill-unattended`, and `gh issue comment` — then the
seven terminal denies last. The map MUST NOT contain any other entry.
The requirement MUST hold on every shipped copy — the tracked
`.opencode/agents/distiller.md`, the embedded
`deweyAgentDefinitions["distiller.md"]` map entry in `agents.go`, and
every file `dewey init` scaffolds from that entry — and the
frontmatter shape guard in `cli_test.go` MUST assert the terminal
denies' presence, value, and terminal position on both shipped
copies.

Because none of the allowed commands contains shell metacharacters,
the terminal denies MUST NOT change which commands the distiller may
run: all three allows MUST remain permitted. The terminal denies
exist to refuse metacharacter-bearing injection payloads — such as
`gh issue comment $(curl …)` or
`dewey gate distill-unattended; git push …` — that still match an
allow entry's command prefix.

#### Scenario: Bash map shape asserted

- **GIVEN** the tracked `.opencode/agents/distiller.md` and the
  embedded `deweyAgentDefinitions["distiller.md"]` map entry
- **WHEN** the `permission.bash` map is parsed in entry order
- **THEN** a `"*": deny` entry MUST be present and MUST precede every
  allow entry
- **AND** exactly three allow entries MUST be present:
  `dewey gate auto-answer`, `dewey gate distill-unattended`, and
  `gh issue comment`
- **AND** the final seven entries MUST be the terminal denies
  `*>*`, `*<*`, `*|*`, `*;*`, `*&*`, `*$(*`, `` *`* ``, each carrying
  `deny`
- **AND** the map MUST NOT contain any other entry

#### Scenario: Guard rejects a deleted or misordered terminal deny

- **GIVEN** a copy of the distiller agent definition with one of the
  seven terminal denies removed, or with one moved to precede an
  allow entry
- **WHEN** the frontmatter shape guard runs against it
- **THEN** the guard MUST report the terminal-position or entry
  mismatch rather than pass

#### Scenario: Scaffolded copies carry the tail

- **GIVEN** a distiller agent scaffolded by `dewey init` into a fresh
  vault
- **WHEN** its `permission.bash` map is parsed
- **THEN** it MUST carry the identical ordered map — broad deny,
  three allows, seven terminal denies
- **AND** the scaffold marker digest MUST verify against the updated
  canonical payload

#### Scenario: Startup gates and notice command still permitted

- **GIVEN** the distiller agent running under the installed host with
  the extended bash map
- **WHEN** it runs `dewey gate auto-answer`,
  `dewey gate distill-unattended`, or `gh issue comment …`
- **THEN** the host MUST permit the command

#### Scenario: Injection payload refused at runtime

- **GIVEN** the distiller agent running under the installed host with
  the extended bash map
- **WHEN** it attempts to run a metacharacter-bearing payload that
  starts with an allowed prefix, such as
  `gh issue comment 123 --body $(cat /etc/passwd)` or
  `dewey gate auto-answer; git push --force origin main`
- **THEN** the host MUST refuse the command
- **AND** the refusal MUST come from the terminal denies resolving
  after the allow entries, not only from the agent declining per its
  prompt
- **AND** if the installed host does not honor the metacharacter
  globs or does not resolve last-match-wins over them, the gap MUST
  be escalated and recorded — the change MUST NOT silently accept an
  inert deny tail

## MODIFIED Requirements

None.

## REMOVED Requirements

None.
