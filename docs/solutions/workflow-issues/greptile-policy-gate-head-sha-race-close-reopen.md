---
title: "Greptile policy gate stale-summary race: wait for the re-review, then close/reopen the PR"
date: 2026-08-13
category: workflow-issues
module: ci/greptile-policy-gate
problem_type: workflow_issue
component: development_workflow
severity: medium
applies_when:
  - "A push to a PR makes the Greptile policy gate fail with \"Latest Greptile confidence-score comment does not reference the current head SHA\""
  - "Greptile has not yet re-reviewed after a push, so its in-place-edited summary comment still carries the previous head SHA"
  - Contributing from a fork account with no workflow_dispatch rights and no label permissions
  - The gate needs re-running without changing the head commit
symptoms:
  - Greptile policy gate check fails on every push even though the code is unchanged and previously passed
  - "Gate python step error \"Latest Greptile confidence-score comment does not reference the current head SHA\""
  - Re-running the failed check before Greptile's async re-review lands fails again with the same error
root_cause: async_timing
resolution_type: workflow_improvement
tags:
  - greptile
  - ci-gate
  - pull-request-target
  - head-sha
  - close-reopen
  - fork-contributor
  - race-condition
---

# Greptile policy gate stale-summary race: wait for the re-review, then close/reopen the PR

## Context

The repo's CI check "Greptile policy gate" is defined in `.github/workflows/greptile-policy-gate.yml`. It runs on `pull_request_target` for the types `opened`, `reopened`, `synchronize`, and `ready_for_review` (`.github/workflows/greptile-policy-gate.yml:4-9`), plus a `workflow_dispatch` that fork contributors cannot use on upstream (`.github/workflows/greptile-policy-gate.yml:10-14`).

The gate waits for the Greptile Review check to complete, then pulls the PR's issue comments and filters Greptile confidence-score comments down to those whose body contains the current head SHA (`.github/workflows/greptile-policy-gate.yml:152-154`). If none match, it fails with `::error::Latest Greptile confidence-score comment does not reference the current head SHA.` (`.github/workflows/greptile-policy-gate.yml:157`). When a matching comment exists, the score must be at least `MIN_GREPTILE_SCORE`, set to `"4"` (`.github/workflows/greptile-policy-gate.yml:37`, threshold check at `:173-175`).

The failure mode: Greptile edits its summary comment **in place** and re-reviews asynchronously, landing minutes after a push. On every push, the `synchronize`-triggered gate run reads the still-stale summary that cites the previous head SHA and fails. This is a structural race between two asynchronous consumers of the same push event, not a defect in the PR's code. Runs also share a concurrency group with `cancel-in-progress: true` (`.github/workflows/greptile-policy-gate.yml:32-34`), so each push cancels the previous gate run — there is never a lingering run that could catch the refreshed summary later, and the gate has no comment-event trigger to self-heal.

Verified three times on PR #1708 (2026-08-13), across heads `27a605bd07` and `2442810a42` (historical evidence only); final state was all checks green with the summary citing 5/5 for head `2442810a42`.

## Guidance

When the gate fails with the stale-summary error after a push:

1. **Do nothing to the branch.** Do not re-push (that restarts the race via a new `synchronize` event and cancels any in-flight run per the concurrency group), do not argue in review threads, and do not wait for the gate to recover on its own — it has no trigger for comment updates.
2. **Wait for Greptile's re-review to land.** Detect it by polling the PR's issue comments: the summary comment is edited in place, so watch its `updated_at` and check whether the body now contains the **current** head SHA. Never use comment count as a signal — editing in place means the count does not change.
3. **Close and reopen the PR** (`gh pr close N && gh pr reopen N`). The `reopened` trigger type (`.github/workflows/greptile-policy-gate.yml:7`) re-runs the gate against the now-current summary without touching the head commit and without requeuing a Greptile review.
4. **Do not conflate the gate with triage.** The gate passes at score >= 4 (`MIN_GREPTILE_SCORE`, `.github/workflows/greptile-policy-gate.yml:37`), but the repo's human triage bot has historically required 5/5 for maintainer-ready on publish-process PRs — a green gate does not imply triage acceptance.

## Why This Matters

Without this recipe, the natural reactions all make things worse: re-pushing restarts the race and burns another Greptile review cycle; waiting passively never resolves because no comment event re-triggers the gate; and misreading the failure as a code problem wastes debugging time on a PR whose diff is fine. The close/reopen move is the only lever a fork contributor has — `workflow_dispatch` requires write access on upstream — and it is cheap, idempotent, and leaves the head commit and review state untouched.

## When to Apply

- The "Greptile policy gate" check fails on a `printing-press-library` PR (or any repo using this workflow) with the message "does not reference the current head SHA".
- You pushed to an open PR within the last few minutes and the gate ran before Greptile finished re-reviewing.
- You are on a fork without write access, so re-dispatching the workflow manually is not an option.

Not applicable when the gate fails for a different reason: no confidence-score comment at all (`.github/workflows/greptile-policy-gate.yml:148-150`), an unparsable score (`:164-167`), or a genuine score below 4 (`:173-175`) — those need a real review response, not a close/reopen.

## Examples

Poll until the Greptile summary cites the current head SHA (edit-in-place aware — checks body content, not comment count):

```bash
PR=1708
HEAD=$(gh pr view "$PR" --json headRefOid -q .headRefOid)
until gh api "repos/mvanhorn/printing-press-library/issues/$PR/comments" --paginate \
  --jq "[.[] | select(.user.login == \"greptile-apps[bot]\" and (.body | contains(\"Confidence Score:\")) and (.body | contains(\"$HEAD\")))] | length" \
  | grep -qv '^0$'; do
  echo "summary still stale for $HEAD; waiting..."; sleep 30
done
```

Then re-trigger the gate without touching the branch:

```bash
gh pr close "$PR" && gh pr reopen "$PR"
```

The `reopened` event re-runs the gate, which now finds the summary comment containing `$HEAD` and evaluates the fresh score. On PR #1708 this sequence turned the gate green all three times it was needed.

## Related

- `docs/solutions/security/2026-05-supply-chain-hardening.md` — why the Greptile-rule + deterministic-gate defense stack exists; this doc covers the operational race inside that stack.
- Issue #1622 — adjacent Greptile CI tooling friction (`gh pr checks --json` unsupported on Ubuntu's packaged gh).
