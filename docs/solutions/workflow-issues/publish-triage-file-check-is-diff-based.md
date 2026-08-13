---
title: "Publish triage's missing-files check reads the PR diff, and only for publish-process-classified PRs"
date: 2026-08-13
category: workflow-issues
module: publish-triage
problem_type: workflow_issue
component: development_workflow
severity: medium
applies_when:
  - "The triage bot marks a PR needs-author with \"missing publish-process files\" naming files that plainly exist in the repo"
  - Deciding whether an amend PR must include go.mod, SKILL.md, or dogfood-results.json in its changed files
  - Predicting which triage contract (publish-process vs objective gates) a planned PR will face
tags:
  - triage
  - publish-process
  - needs-author
  - diff-vs-tree
  - printing-press-library
  - fork-contributor
---

# Publish triage's missing-files check reads the PR diff, and only for publish-process-classified PRs

## Context

The maintainer's triage bot (external to this repo — its source is not in-tree, so everything here is empirical, per this investigation's evidence) marks PRs `needs-author` with messages like "missing publish-process files: `<cli>/go.mod`". Confusingly, those files always exist in the repository, which makes the message look like a bot bug. It is not: the check evaluates the PR's **changed-file set**, not the tree.

Decisive evidence (2026-08-13 investigation): at the exact SHAs the bot flagged on PR #1624 (`go.mod`) and PR #1625 (`SKILL.md`, `go.mod`), the GitHub contents API shows those files present in the tree — so a tree check could not have fired. Both PRs later merged with the named files in their diffs.

## Guidance

- The "missing publish-process files" complaint means: *your diff must touch these files*, not *these files must exist*. Resolve it by making the publish process's real output land in the diff (a `go mod tidy` result, a regenerated SKILL.md, a restored `dogfood-results.json`), not by arguing that the file exists.
- The publish-process contract applies only to PRs the bot **classifies** as publish/reprint. The observed classifier trigger is the diff touching the CLI's `.printing-press.json` (PRs #1624/#1625 touched it for a `contributors[]` entry and got the contract; #1636, #1674, and #1708 did not touch it and faced only the objective CI gates).
- Observed required-in-diff set by PR kind: amends under the contract needed `SKILL.md` and `go.mod`; the reprint (#1680) was flagged for `dogfood-results.json`, and reprints replace the whole tree, so there the file genuinely had to be restored. Amends were never flagged for `dogfood-results.json`.
- Practical consequence: once your `contributors[]` entry exists on main, later amends need not touch `.printing-press.json` at all — which routes them down the objective-gates path, a strictly lighter contract.

## Why This Matters

Misreading the check as tree-based sends you chasing a phantom bug or, worse, re-uploading files that already exist. Knowing the classifier trigger also lets you *choose* the contract: an amend that avoids touching `.printing-press.json` avoids the publish-process file requirements entirely, while a PR that must touch it should ship the full required file set in the diff from the first push to skip a triage round-trip (each round-trip costs a day or more on the maintainer's batch schedule).

## When to Apply

- A triage comment names "missing publish-process files" that exist in the repo.
- Planning an amend or reprint PR against this library and deciding which files to include in the diff.

Not applicable to the other triage failure reasons (review-score below the bar, missing manuscripts evidence, red CI) — those are separate checks with their own remedies.

## Examples

Verifying the diff-vs-tree question for a flagged SHA:

```bash
# Bot flagged "missing: library/<cat>/<cli>/go.mod" at SHA <flagged-sha>.
# If this returns true, the file was in the tree and the check must be diff-based:
gh api "repos/mvanhorn/printing-press-library/contents/library/<cat>/<cli>?ref=<flagged-sha>" \
  --jq '[.[].name] | contains(["go.mod"])'
```

Checking what a merged PR's diff actually contained:

```bash
gh pr view <n> --repo mvanhorn/printing-press-library --json files \
  --jq '[.files[].path] | map(select(test("go.mod|SKILL|dogfood")))'
```

## Related

- `docs/solutions/workflow-issues/greptile-policy-gate-head-sha-race-close-reopen.md` — the other gate a fork contributor hits on the same PRs; green gate and triage acceptance are separate hurdles.
