---
name: release-gate-chair
description: >-
  Runs the three pre-launch passes in order — web-preflight for external exposure,
  pre-ship-security-red-team for application behaviour in an isolated environment,
  pre-ship-performance-audit for measured speed — records each lane's own verdict and
  findings into one ledger, and computes a single release posture (HOLD or ready) with a
  script rather than judgement. Use when someone asks "is this safe to launch", "ready to
  ship", "should we deploy", "give me a go/no-go", or before any deploy of a site or client
  app you own or are authorized to test. Three verdicts stay separate; a pass in one lane
  never covers another. Variant: this fork runs the lanes itself and copies their finding
  tables verbatim; it is the default release gate. Not for a single-lane check (invoke that
  skill directly), for reports that already exist (use release-gate-validator), for
  penetration testing of production, or for performing the deploy.
argument-hint: <production URL> [--app <isolated/local URL>] [--project <path>]
version: 0.1.0
---

# Release gate chair

Three lanes, three verdicts, one posture. The gate adds nothing to what the three skills can
detect; what it adds is that the arithmetic is redone by `scripts/check_gate.mjs`, coverage is
stated, and a lane that did not run cannot be mistaken for a lane that passed.

`<skill-dir>` is this skill's directory.

## Inputs

| Input | Needed for | If absent |
|---|---|---|
| Production URL | exposure lane | Ask. A gate with no target is a template. |
| Isolated or local app URL | app-tests lane | Record the lane `untested`, or `descoped_by` with the user's reason. Never run red-team against production. |
| Project path | performance lane | Perf lane runs measurement-only and says so |

Multiple hosts are gated independently and keep separate rows, then any redirect or
canonical-host relationship is named.

## 1. Profile the target, then say so

Classify before dispatching: `static-site` (no auth, no datastore), `web-app` (auth, roles, quota,
datastore), `api-or-worker` (no page to measure in a browser), `preview-only` (not yet deployed).
Write the profile into the run directory. The profile decides which lanes run in full and which
rows are `not-applicable` — a static marketing site has no login, role or quota to attack, and
dispatching seven subagents at it is why gates get skipped.

`not-applicable` is the gate's claim about the target, so the gate writes it and names the profile
that justified it. A lane is never quietly shortened.

## 2. Run the lanes in order

Do not start the next lane on the strength of the previous one's pass.

**Exposure.** `web-preflight`. Bounded GET/HEAD, so a live host is in scope. Emit its host verdict
in its own precedence `BLOCK > INCONCLUSIVE > PASS WITH FIXES > PASS`, and its `PREFLIGHT-<nn>`
findings.

**App-tests.** `pre-ship-security-red-team`. Local or isolated only, synthetic accounts. Accept
this lane **only on mechanical evidence**:

```
node "<red-team-skill-dir>/scripts/check_results.mjs" <run-dir>
node "<red-team-skill-dir>/scripts/check_results.mjs" <run-dir> --require-fixes
```

Both must exit 0. Record the second one's exit code in the ledger. Findings are `REDTEAM-S<nn>`.

**Perf.** `pre-ship-performance-audit`. Four read-only subagent audits, findings `PERF-<nn>`,
`fixed-verified` only with an after-median taken under the same conditions.

## 3. Record the ledger

`.release-gate/<run-id>/` in the audited repo, gitignored first. Format and every column rule in
[references/ledger.md](references/ledger.md).

This fork is the **chair**: it populates the ledger itself, copying each lane's finding table
**verbatim** — ids, routes, severities, remediation states as that lane wrote them. Never
re-summarise a finding, never soften a remediation, never derive one lane's verdict from another's.
After copying a lane, count the rows you wrote against the rows in that lane's own report; a
mismatch is fixed by re-reading the report, not by adjusting the count.

Fill `touches` per finding from what the finding says, not from what you hope. Blanket `none` is a
claim; `unknown` on a CRITICAL or HIGH row holds the gate, which is the cheaper mistake.

## 4. Compute the posture

```
node "<skill-dir>/scripts/check_gate.mjs" .release-gate/<run-id> --sha <current-head-sha>
```

Exit `0` ready, `1` HOLD, `2` contract break. Report the rule codes that fired — the rule used is
part of the answer, not a footnote. `ready` means ready **for the coverage stated**, and that line
is part of the verdict.

A `HOLD` is not overridden by urgency. Deploying anyway is the user's call, made in the open, and
the gate records that it was made.

## 5. Deliver, and re-check after deploy

Print the checker's block, then write the same register to `docs/release/<YYYY-MM-DD>-<target>-gate.md`
and commit it if the user wants it in history. Re-runs reuse `finding-id`s; an id that changed
meaning is a contract break — say so rather than quietly renumbering.

Then offer the **post-deploy re-check**: one `exposure` lane row for the live host in a new run
directory, reusing existing `PREFLIGHT-*` ids. It can move `ready` to `HOLD`. It cannot move a
`HOLD` to `ready`, because it adds no coverage the pre-deploy pass lacked.

## Does not

- Deploy anything, or widen any lane's scope. New checks belong to the lane's own skill; the gate cites it.
- Merge scores, average severities, or produce one number for three different kinds of claim.
- Turn an unrun lane into a pass, or report a partial run as complete.
- Run red-team against production, or reproduce a secret in the register.

If a `caffeinity-stack` skill is installed, read it before reporting Caffeinity work (deploy runbook, no-fabricated-evidence).
Brew 3 · Espresso in `caffeinity-routing` is the sequence this skill implements.

## Verify the gate before trusting it

```
node "<skill-dir>/tests/posture.mjs"
```

20 fixtures, 7 of which must come back `ready`. A checker that always answered HOLD would pass the
HOLD cases by accident, so the `ready` cases are the real test. Run this after any edit to
`check_gate.mjs`, and before publishing.

## Known weakness of this fork

Transcription is model-mediated. The mitigation is verbatim copying plus a row-count reconciliation,
which limits the damage but does not eliminate it. If a lane's report and the ledger ever disagree,
the report wins and the ledger is rewritten.
