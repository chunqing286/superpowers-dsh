# Security Part of the Final Whole-Branch Review

This file is the security dimension of the **final whole-branch review** —
the branch-level counterpart of [security-review-part.md](security-review-part.md),
which covers the per-task review. It is spliced into the final reviewer's
dispatch prompt the same way the task-review security part is spliced into
the task reviewer's prompt: the dispatcher merges these blocks into its own
copy of requesting-code-review's
[code-reviewer.md](../../requesting-code-review/code-reviewer.md) at the
points named below, and never edits that template file itself.

The final review's output format is **not** changed: its findings keep going
through `finishing-a-development-branch`'s adjudication in the existing
**Critical / Important / Minor** levels — no new level, no new section.

The read-only rule applies here exactly as it does to the task reviewer:
the final reviewer must not mutate the working tree, the index, HEAD, or
branch state in any way.

## Block 1 — Expand "What to Check"

The base template's `## What to Check` already carries a single
`Security concerns?` line under **Architecture**. Replace that one line with
these three branch-level questions:

```
**Security:**
- Do the trust boundaries in the security constraints hold at the branch
  level, not just per file?
- Does any change here compose with another to create a path the per-task
  reviews could not see (e.g. one task widened a path, another trusted it)?
- Are the branch's bound security constraints all met, and each covered by
  a test that would fail if the control regressed?
```

## Block 2 — Insert After "Git Range to Review"

Insert this section immediately after the base template's
`## Git Range to Review` block:

```
## Security Constraints for This Branch

Read: [SECURITY_CONSTRAINTS_FILE]

Verify every constraint against the whole-branch diff, not per file. A
constraint met in isolation but defeated by a change elsewhere in the
branch is VIOLATED.
```

`[SECURITY_CONSTRAINTS_FILE]` is the branch's constraint file — the plan's
`## Security Constraints` section, or the threat-model output the plan
copied it from. If the branch has no constraint file at all, that is itself
a finding: the threat model never ran or its output was dropped, and the
final reviewer should say so rather than treat the dimension as vacuously
satisfied.

## What the Final Reviewer Returns

The output format stays the base template's: Strengths, Issues
(Critical / Important / Minor), Recommendations, Assessment. Branch-level
security findings enter `### Issues` at the severity their attack path
implies, with the path stated — source → the control that is missing or
broken → the consequence. Nothing about the verdict headings changes.
