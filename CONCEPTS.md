# Concepts

Shared domain vocabulary for this project — entities, named processes, and status concepts with project-specific meaning. Seeded with core domain vocabulary, then accretes as ce-compound and ce-compound-refresh process learnings; direct edits are fine. Glossary only, not a spec or catch-all.

## Publish workflow

**Printed CLI** — a generated command-line tool produced by the CLI Printing Press from an API spec, living as one directory under the library tree; the unit everything else in this repo versions, reviews, and releases.

**Publish-process PR** — a pull request the triage automation classifies as carrying a printed CLI's publish or reprint lifecycle, which subjects it to the full publish contract (required files in the diff, manuscripts evidence, top review score); other PRs face only the objective CI gates.

**Reprint** — regenerating an existing printed CLI from scratch under a newer Printing Press, replacing the whole CLI tree while carrying prior research, novel features, and patch records forward as reconciliation context.

**Amend** — a scoped post-publish change to an already-released printed CLI (bug fix, new flag, new command), shipped as its own PR without regenerating the tree.

**Patch record** — a per-CLI ledger entry describing a hand-authored deviation from generated output, kept so a future reprint can re-apply the intent instead of silently dropping it. Convention: one entry per PR.

**Manuscripts evidence** — the proofs (plans, build logs, acceptance markers) stored alongside a CLI's generation run that demonstrate a PR's claims were actually verified; triage treats their presence in the diff as the evidence of record.

**Release ledger** — the per-CLI record of released versions and changes, owned by post-merge automation; contributors never hand-bump it, and the runtime version vars in source are stamped from it.

**Policy gate** — the CI check that converts the external review bot's confidence score into a pass/fail status, requiring the score summary to reference the PR's current head commit before it counts.

**Triage states** — the maintainer bot's labels for a PR's readiness: needs-author (contributor must act), awaiting-maintainer (queued for human merge). A green policy gate does not by itself advance a PR past triage.

## Flagged ambiguities

- "Gate" alone is ambiguous in this domain — the policy gate (review-score CI check) and the live gate (a CLI's phase-5 acceptance dogfood) are distinct mechanisms.
