## Context

The curator and distiller agent definitions shipped by Dewey declare
their shell surface in the nested `permission.bash` map of the agent
frontmatter. Both shipped copies currently end that map with `allow`
entries and carry no metacharacter denies at all:

```yaml
  bash:
    "*": deny
    "dewey gate auto-answer": allow          # curator
```

```yaml
  bash:
    "*": deny
    "dewey gate auto-answer": allow          # distiller
    "dewey gate distill-unattended": allow
    "gh issue comment": allow
```

OpenCode permission maps resolve ordered and last-match-wins (the same
property the `tools:` deny-all-then-allow shape of
curator-tool-allowlist D2 relies on). A trailing allow that matches a
command prefix therefore wins for any payload sharing that prefix:
`dewey gate auto-answer; curl …` matches the gate allow,
`gh issue comment $(…)` matches the notice allow. Issue #215 reports
the downstream consequence — the consumer re-scaffold PR
unbound-force/unbound-force#1139 fails that repository's
instruction-lint, which requires the last seven `permission.bash`
entries of every scaffolded persona to be the terminal denies `*>*`,
`*<*`, `*|*`, `*;*`, `*&*`, `*$(*`, `` *`* ``.

The shipping path is the one established by
tool-owned-scaffold-updates: the canonical payloads are the
`deweyAgentDefinitions` raw string literals in `agents.go`; the
tracked `.opencode/agents/curator.md` and
`.opencode/agents/distiller.md` are byte-identical mirrors carrying a
final-line `dewey:scaffold` marker whose sha256 covers the canonical
payload; `dewey init` scaffolds or refreshes copies from the map.
`assertCuratorFrontmatterShape` in `cli_test.go` pins the curator's
bash map to exactly two ordered entries; `assertDistillerFrontmatterShape`
checks the distiller's bash map by key without any ordering assertion
and without asserting the `gh issue comment` allow at all.

One hard constraint shapes the implementation: the payloads are Go
raw string literals (backtick-delimited), and the `` *`* `` deny key
contains a backtick byte. The `agents.go` doc comment currently says
an agent definition MUST NOT contain a backtick for exactly this
reason.

Constitution alignment is assessed in the proposal (all four
principles PASS). The decisions below keep it that way: the boundary
stays a self-describing scaffolded artifact (I), Dewey stays fully
usable without any host or sibling tool (II), the hardened shape is
machine-verified by order-preserving YAML guards and marker digests
(III), and every requirement is testable with the standard library
alone (IV).

## Goals / Non-Goals

### Goals

- End both scaffolded agents' `permission.bash` maps with the seven
  terminal metacharacter denies, placed after every allow so they win
  under last-match-wins resolution, in the order enumerated by
  dewey #215.
- Keep every allowed command runnable: no allow entry changes, and no
  allowed command contains a metacharacter.
- Make the exact ordered map mechanically checked on both shipped
  copies of both agents, with negative paths proving the guards
  reject a deleted or misordered deny.
- Refresh the tracked mirrors so their `dewey:scaffold` markers verify
  against the updated canonical payloads.
- Unblock unbound-force/unbound-force#1139: consumers running
  `dewey init` (first init or untouched-drifted re-init) receive the
  hardened templates through the existing machinery.

### Non-Goals

- Converting agent files to the V2 `permissions` rule format (stays
  V1, like every sibling agent — the curator-tool-allowlist D1
  rationale).
- Touching any agent other than the two Dewey-scaffolded ones. The
  repo's `.opencode/agents/divisor-*.md` and friends are scaffolded
  by the unbound-force CLI, not by `dewey init`; several use a flat
  `bash: deny` with no map at all. Their posture belongs to the
  unbound-force repository.
- Changing any `tools:` block entry, prompt body, slash command,
  workflow template, gate subcommand, or MCP tool.
- Verifying consumer repos' customized agent copies. A customized
  copy is preserved byte-for-byte by design; consumers that edited
  their curator/distiller keep their edits (and their exposure) until
  they reconcile.
- Runtime enforcement of bash-map semantics inside Dewey itself. The
  map is host policy; Dewey ships and guards the artifact.

## Decisions

### D1 Bash map shape: broad deny, allows, terminal denies last

Each map takes the three-part order issue #215 prescribes —
(1) `"*": deny`, (2) the agent's specific allows, unchanged,
(3) the seven terminal denies as the final entries:

```yaml
  bash:
    "*": deny
    "dewey gate auto-answer": allow
    "*>*": deny
    "*<*": deny
    "*|*": deny
    "*;*": deny
    "*&*": deny
    "*$(*": deny
    "*`*": deny
```

(the distiller inserts its two additional allows between the broad
deny and the tail). Only one ordering constraint is semantically
load-bearing — the mirror image of curator-tool-allowlist D2: there
the broad deny had to precede the allows so unmatched commands fall
to `deny`; here the metacharacter denies must FOLLOW the allows so an
injection payload that matches both an allow prefix and a
metacharacter glob resolves to `deny`. The order among the seven
denies is not load-bearing (all carry `deny`), but it is pinned to
the issue's enumeration for determinism, so Dewey's guards and the
consumer repo's instruction-lint agree on one canonical sequence.
YAML requires the quoted key form for every entry (a bare leading
`*` parses as an alias marker); the file's existing `"*": deny`
already establishes that convention.

### D2 Embedding the backtick key: one shared constant spliced into both raw literals

The `` "*`*" `` key requires a literal backtick byte in the payload,
which a Go raw string literal cannot contain. Options considered:

1. **A shared interpreted-string constant spliced into both map
   entries** (chosen): declare
   `const bashTerminalDenies` as the seven deny lines with escaped
   quotes, and write each map entry as
   `` `…payload head…` + bashTerminalDenies + `…payload tail…` ``.
2. Inline per-map splices (`` `…` + "`" + `…` ``) — duplicates the
   tail in two places and splices mid-key, inviting drift between the
   two agents' tails.
3. Migrating the payloads to `embed.FS` files — departs from the
   single-file `deweyAgentDefinitions` pattern the parity tests are
   built on, for one character.
4. Rewriting the payloads as interpreted strings — escaping every
   line of two large templates.

Rationale: the two bash maps MUST carry byte-identical deny tails, so
a single shared constant pins that structurally (DRY) instead of by
test; the splice is invisible to the payload bytes, so every
existing byte-identity, marker-digest, and parity comparison
continues to hold unchanged; and the `agents.go` doc comment's
absolute "MUST NOT contain a backtick" rule is updated to the accurate
rule — the raw literals must not contain a backtick *directly*; the
only backtick in any payload arrives through the
`bashTerminalDenies` constant. The constant's lines use the same
four-space indentation as the existing bash entries and end with a
trailing newline so the splice joins cleanly at line boundaries.

### D3 Guard strategy: exact ordered pin for the curator, terminal-position pin for the distiller

The curator guard (`assertCuratorFrontmatterShape`) already pins the
bash map as an exact ordered sequence, so it grows from two `wantBash`
entries to nine — deny, allow, then the seven denies in D1 order —
keeping the fail-closed exact pin and updating the count-mismatch
message to name the terminal denies. In Go, the `` "*`*" `` want
entry is an ordinary interpreted string (backticks are legal inside
double quotes).

The distiller guard (`assertDistillerFrontmatterShape`) currently
checks bash entries by key without order. It gains: the
`gh issue comment` allow assertion (an existing gap this change
closes while in the neighborhood), an exact key-set pin
(broad deny + three allows + seven terminal denies = eleven entries,
no inert extras), a value check (`deny`) on each terminal deny, and a
terminal-position assertion — the final seven entries, in order, are
the seven denies. Both guards keep running against both shipped
copies (tracked file and map entry), per the two-copy coverage
convention.

`TestCuratorAgent_GuardsRejectDoctoredDocuments` gains two doctor
cases: a deleted terminal deny (count/content mismatch) and a
terminal deny moved before the allow (ordered-sequence mismatch),
following the existing `strings.Replace` doctor pattern.

### D4 Tracked-copy refresh through the existing writer, not hand edits

The tracked mirrors carry markers whose digests match their own
current bodies, so after the embedded payloads change they classify as
untouched-but-drifted scaffolds: `dewey init` refreshes them to the
new payload with fresh markers (the `updated` path from
tool-owned-scaffold-updates). The refresh is produced by running the
actual writer — `go run . init --vault .` from the repository root
(the same marker-bootstrap path tool-owned-scaffold-updates used) —
never by hand-editing and recomputing digests, so the bytes provably
come from `scaffoldPayload`. The generic marker tests
(`TestInitCmd_FirstInitWritesVerifiableMarkers`,
`TestScaffoldMarker_DigestMatchesIndependentSHA256`) are
payload-agnostic and need no changes; the byte-identity and shape
tests fail until the refresh lands, which is the intended sequencing
(one task updates payload + mirrors + guards together).

### D5 Runtime matcher verification stays a manual acceptance step

Whether the installed host resolves `*;*`-style globs
last-match-wins over command strings cannot be verified from Go
tests without the host. Following the CA-FR-002 D11 precedent, the
implementation runs a one-time manual acceptance check — attempt an
injection payload (`dewey gate auto-answer; true`,
`` gh issue comment 1 --body `x` ``) under the curator/distiller in a
scratch vault and confirm refusal — and records the result. If the
host ignores the globs or resolves differently, the gap is escalated
(per the spec's escalation scenario), not silently accepted. The
consumer repo's instruction-lint is the standing downstream
enforcement that the shape, at least, is what it expects.

## Risks / Trade-offs

- **Customized consumer copies stay exposed.** A consumer that edited
  its curator/distiller keeps the injectable surface (the refresh
  path preserves customized files by design). Accepted: user-owned
  customizations are sacred in the scaffold ownership model, and the
  consumer's own lint surfaces the drift. Documented in the README
  posture sentence.
- **Exact-order pinning is stricter than the consumer lint.** A
  future reorder of the seven denies requires a guard update. Accepted
  deliberately — it is the repo's fail-closed pinning style, and the
  friction points exactly where a security-relevant sequence changes.
- **The backtick splice adds mild `agents.go` complexity.** Mitigated
  by the single shared constant, the updated doc comment stating the
  new rule, and the byte-identity guards that fail CI on any splice
  mistake.
- **Host glob semantics could differ from assumption.** The map could
  be inert under a matcher that tokenizes differently. Mitigated by
  the D5 acceptance step and the spec's escalation clause; the
  consumer repo's lint independently verifies the shipped shape.
