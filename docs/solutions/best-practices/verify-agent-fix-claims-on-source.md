---
title: Verify a delegated agent's fix claim against source before pushing it
date: 2026-08-13
category: best-practices
module: publish-workflow
problem_type: best_practice
component: development_workflow
severity: medium
applies_when:
  - An orchestrating session is about to push a fix implemented by a delegated agent
  - "An agent's completion report claims output behavior (\"the envelope keeps note and fetch_failures\") that no test asserted end-to-end"
  - A reviewer rebuts a fix that the implementing agent reported as complete
tags:
  - delegation
  - verification
  - code-review
  - agent-claims
  - stdout-contract
---

# Verify a delegated agent's fix claim against source before pushing it

## Context

During PR #1708 (scrape-creators amend, 2026-08-13), a delegated agent fixed a review finding and reported: "the returned envelope keeps `note` and all `fetch_failures`." The claim was true of the function's **return value** — and false for the user: on the combined failure path the caller returned the error before ever serializing the envelope to stdout, so the structured partial result was unreachable. The review bot rebutted the fix on the next pass, explicitly contradicting the agent's thread reply, and reading the code confirmed the bot was right. One verification grep before pushing (`does the error path reach the print call?`) would have saved a full review round-trip.

## Guidance

Before pushing a delegated fix, verify the *user-observable* claim, not the *internal* one:

- Trace the claimed data to its exit surface. "The struct contains X" is not "the user receives X" — find the line that serializes it (stdout print, HTTP write, file write) and confirm the failing path reaches that line.
- Treat a completion report's behavioral claims as hypotheses. The cheap check is reading the caller of the changed function; the thorough check is a test that asserts the actual output bytes (the eventual fix here added a test asserting stdout JSON content at non-zero exit — that shape of test is what makes the claim durable).
- When a reviewer rebuts a fix the agent reported complete, default to re-reading the code yourself before defending the fix. In this case the rebuttal was precise ("the caller still returns the helper error before serializing the populated envelope") and correct.

## Why This Matters

An orchestrator that relays agent claims unverified converts one agent's blind spot into a pushed commit, a wrong reply in a review thread, and an extra review cycle — with the orchestrator's credibility attached. The failure mode is specific to delegation: the implementing agent tested what it changed (the return value) and honestly reported that; nobody tested the layer above it. Output-surface verification is the orchestrator's job precisely because it sits between the layers.

## When to Apply

- Any time a delegated agent reports a fix whose success criterion is what a user or caller observes (output, exit codes, API responses), and the tests it added assert internal values rather than that surface.
- Especially before replying in a review thread that something is fixed — a wrong "fixed" reply invites a documented rebuttal.

## Examples

The verification that settled it (30 seconds, no build needed):

```bash
# Does the error path reach the serialization call?
grep -n 'return .*err\|printJSONFiltered' internal/cli/comments_thread.go
# return path at ~374-379, print call at ~415 → the error path exits first. Claim false.
```

The durable fix shape: a test that asserts stdout bytes on the failing path — valid JSON containing the diagnostic `note` and every recorded fetch failure — while the command still exits non-zero.

## Related

- `docs/solutions/workflow-issues/greptile-policy-gate-head-sha-race-close-reopen.md` — the same PR's other lesson; each extra review round-trip also re-runs that gate race.
