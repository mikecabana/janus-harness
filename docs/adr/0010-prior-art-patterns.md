# ADR 0010 — Patterns adopted from industry practice

**Status:** Accepted 2026-11-13 (by merging PR #2).
**Affects:** §2, §5.2, §5.3, §8.6, §8.8 (new), §9.2, §9.6, §12.1 E, §14.3 item 4, §18.1, §20

## Context

Agent-driven issue-to-PR systems, and the CI/CD and code-review practice around them, have converged on several control-plane patterns that fill gaps this spec leaves open:

- **Stage results lead with a fixed verdict**, and the orchestrator branches only on that verdict.
- **A single state writer refuses any stage change** unless the evidence that justifies it exists and matches the current commit.
- **A triage step runs before any plan exists.** It can cut an oversized issue down to a first slice, ask the author questions, or reject the issue, and it rates risk and the surfaces a change touches.
- **A table in config routes reviewers** by that risk rating.
- **Checks run again against the latest base branch** just before merge.
- **Escalations ask a human one specific question.**

The same practice shows two failure modes to design around:

- **Protecting test files can lead to a dead end.** A task that needs a protected test edited can never pass, so repair attempts are spent before anyone escalates.
- **A branch can weaken the checks it is judged by.** A protected check script still runs package scripts and test config that the branch itself controls.

Only the ideas are adopted here. No third-party code is copied.

## Decision

1. The patterns are added to the spec as **PROPOSED** text, in the sections listed above. A "Prior art" note in §2 indexes them.
2. They are adapted to this spec rather than copied:
   - **Verdict names:** the implement stage keeps M1.5's `done | blocked` vocabulary.
   - **Triage isolation:** triage is isolated the way ADR 0003 isolates review (separate sandbox, result-only), not through tool restrictions.
   - **Comment filter:** triage reads comments through ADR 0009 §3's filter. A reply from an author who is not a collaborator gets through only after the operator acknowledges it.
   - **Routing selects agent IDs, not models.** Routing can only add review on top of a configured floor. `pathSurfaces`, evaluated on the actual diff, can add reviewers but never remove them.
   - **`protectedPaths` extends the existing deny list.** It builds on ADR 0009 §7's changed-path deny list. A hit goes to `BLOCKED` with a decision request instead of failing validation.
   - **Re-testing against the current base is left to GitHub.** Branch protection requiring up-to-date branches, or a merge queue, covers it, because ADR 0008 forbids the factory from rebasing.
3. They are built in Milestone 3 as work packages M3.9–M3.12, plus additions to M3.4.

## Consequences

- **This is an exception to ADR 0009 §11, which says the spec is not edited.** The operator asked for the patterns to go into the spec itself. This ADR is the record that §21 item 12 requires, and the spec text points back to it.
- **Milestone 3 grows from 8 to 12 work packages.** Its exit condition is unchanged.
- **The completion contract from M1.5 becomes a per-stage-kind envelope in M3.9.** Results written by earlier milestones still parse, because `done | blocked` is unchanged.
- **Target repositories need one more branch-protection setting from Milestone 3:** require branches to be up to date before merging, or use a merge queue.

## Not chosen

- **Auto-merge on approval, as some existing systems do.** The spec requires human merge approval (§0, §14.3 item 4).
- **Treating a protected-path change as a validation failure.** That sends the agent into repair attempts that cannot succeed.
- **Letting triage's risk rating reduce review below a floor.** The rating is model output derived from untrusted issue text.
- **Reading cost from undocumented harness transcripts.** §9.4 and §20 already take a more careful line on cost.
