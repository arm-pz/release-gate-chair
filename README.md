# release-gate-chair

One answer to "can we deploy?" — from three checks that must not be averaged.

`web-preflight`, `pre-ship-security-red-team` and `pre-ship-performance-audit` each answer a
different question about the same site: what is exposed, what an attacker can do, how slow it is.
Running them by hand leaves you with three reports and no decision. This skill runs the three in
sequence, records each lane's verdict and findings in one ledger, and computes the release posture
with a script instead of a judgement call.

**Three verdicts stay separate.** The posture is a gate, not a score. A preflight `PASS` never
implies the application survived penetration testing, and a lane that did not run is reported
`untested` — never a pass.

## Install

```
npx skills add arm-pz/release-gate-chair --skill release-gate-chair
```

Then `/release-gate-chair https://example.com --app http://127.0.0.1:3000 --project .`

## What it emits

```
RELEASE POSTURE: HOLD   (rule: R1)
Coverage: exposure=INCONCLUSIVE app-tests=ready perf=DESCOPED(user:deadline)

| finding-id | lane | route | severity | remediation | touches | evidence |
HOLD R1: lane exposure (example.com) verdict INCONCLUSIVE — an unassessed lane is not a passed lane
GAP: lane=perf de-scoped by user:deadline, measure next week
```

Exit codes: `0` ready, `1` HOLD, `2` contract break. An unrecognised enum in the ledger aborts
rather than defaulting to `ready`.

## Layout

| Path | What |
|---|---|
| `SKILL.md` | the skill: profile, run lanes, record, compute, deliver |
| `scripts/check_gate.mjs` | the posture rule as code — the only thing that decides |
| `references/ledger.md` | the two-file ledger format every fork shares |
| `tests/posture.mjs` | 20 fixtures; 7 must come back `ready` |

Run the tests after touching the checker: `node tests/posture.mjs`.

## The posture rule

HOLD if any of: a lane verdict is `BLOCK`, `INCONCLUSIVE`, `HOLD`, `fail` or `untested` (R1); a
CRITICAL or HIGH finding is `open` or `fixed-unproven` (R2); the app-tests lane lacks a
`check_results.mjs` exit 0 (R3); a finding touching another user's data, money or role is not
`fixed-verified` (R4); a required lane has no row (R5); a lane was recorded at a stale sha (R6);
`accepted-risk` has no attribution (R7); a CRITICAL or HIGH row leaves `touches=unknown` (R8).

Attributed `accepted-risk` is honoured even on a boundary finding: a recorded choice by the owner
outranks the gate's default, and only that.

## The three forks

Built side by side in October 2026 to compare one variable — who writes the ledger rows. The
checker, the ledger contract and the tests are byte-identical across them.

- **`release-gate-chair`** (this repo) runs the lanes itself and copies their tables verbatim.
- [`release-gate-ledger`](https://github.com/arm-pz/release-gate-ledger) has each lane append its
  own rows; ships the upstream patches that make that real, unapplied.
- [`release-gate-validator`](https://github.com/arm-pz/release-gate-validator) runs nothing and
  judges reports that already exist.

## Limits

Nothing here makes a site secure. It reports what ran, what did not, and what was chosen anyway.
It never deploys, never attacks production with the red-team lane, and never puts a secret in the
register.

MIT © 2026 Abdul Rahman P Z
