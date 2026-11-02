# The Security Dimension in the SDD Loop

Security auditing is a **dimension of the review slots SDD already has**,
not a parallel process. `subagent-driven-development` already wires three
review slots together — per-task review, scoped re-review, and the final
whole-branch review — through a fixed output format, the fix loop, the
re-review's `ADDRESSED` verdict, and the ledger. A separate "security
auditor" role would re-report the same issues the task review finds, in a
format the re-review template cannot adjudicate, doubling noise until the
fix loop drowns. So the security dimension rides the existing seats.

## Four Files, Four Insertion Points

| File | Insertion point | When it runs |
|---|---|---|
| [threat-modeler-prompt.md](threat-modeler-prompt.md) | After brainstorming, before writing-plans | Greenfield project: produce the security constraints |
| [plan-security-section.md](plan-security-section.md) | The `## Security Constraints` section of the plan writing-plans produces | Existing plan: add the section retroactively |
| [security-review-part.md](security-review-part.md) | Spliced into `task-reviewer-prompt.md` at dispatch | Every task review (injected per task) |
| [security-final-review.md](security-final-review.md) | Spliced into the final reviewer's dispatch | The final whole-branch review, once per branch |

```
Greenfield:
  brainstorming → [threat-modeler] → writing-plans (+ Security Constraints section)
    → SDD (per-task security review) → final review → review (security lens) → merge

Existing project (already underway):
  [threat-modeler once, over the existing spec/plan] → add the plan's
    Security Constraints section → continue SDD (inject from the current task on)
```

The threat model is where the constraints come from: a read-only subagent
maps the spec's entry points, trust boundaries, and assets onto precise,
testable constraints and writes them to a file. The plan's
`## Security Constraints` section is that threat model's output — a plan
without the section means the threat model never ran or its findings were
dropped.

## Three Inviolable Rules

1. **The output format does not change by one character.** A security
   finding is Critical, Important, or Minor — never a new level. Changing
   the format breaks the re-review's verdict and snaps the fix loop.
2. **The auditor is read-only.** It never touches the working tree, the
   index, HEAD, or branch state. To look at another revision,
   `git worktree add` a temporary directory.
3. **Inject per task.** A security constraint is "what this task must get
   right", not "an area this task might touch". Constraints not handed to
   a task's reviewer simply do not get audited there — that is both the
   cost saving and the noise control.

## How Security Constraints Bind to Tasks

When the plan carries a `## Security Constraints` section, the reviewer of
a task also gets the constraints that bind that task, as the same kind of
attention lens as the global constraints:

- **Binding rule:** a constraint binds a task when the component path it
  names appears in that task's `Files:` list. One component touched by
  several tasks binds all of them. The reviewer only ever audits the
  components its task actually touches.
- **Copy verbatim.** Hand the reviewer the binding constraints in the
  plan's `[SEC-0N] ...` form, exactly as written. The authoritative
  definition of a constraint lives in the plan's section alone — copies
  drift.
- **The dismissal literal.** If the plan has no `## Security Constraints`
  section, or none of its constraints name a component this task touches,
  pass exactly `No security constraints bind this task.` The reviewer then
  skips its security part and says so in its report — the dimension is
  visibly dismissed, never silently forgotten.
- **Never the whole list.** A reviewer auditing constraints for components
  it did not change produces false positives, and false positives are what
  makes a reviewer stop being read.
- **The gap rule.** If the plan has no `## Security Constraints` section
  but the task touches a trust boundary — authentication, authorization, a
  tenant boundary, an untrusted input, a secret, a dependency on an
  external service — record it in the ledger as a plan gap. Never skip
  silently. The section's format, and how to add one to an existing plan,
  is in [plan-security-section.md](plan-security-section.md); the threat
  model that produces it is
  [threat-modeler-prompt.md](threat-modeler-prompt.md), run before
  [writing-plans](../../writing-plans/SKILL.md).

## Division of Labour with the DSH `review` Tool

| | SDD task review + [security-review-part.md](security-review-part.md) | DSH `review` tool |
|---|---|---|
| When | **Inside** the SDD loop, every task | **Outside** the loop, once before merge |
| False-positive control | The constraint list bounds the scope | A **verifier** refutes each finding |
| Output format | SDD format (the fix loop needs it) | Free format |

**Never insert the `review` tool into the SDD loop** — it does not produce
findings in the SDD format and would scramble the fix loop.

The pre-merge pass (`review` with `lenses: ["security"]`, `depth: "full"`)
looks at the **whole branch's composition effects** — task A widened a
path, task B trusted that path. No per-task review can see that.

The branch level has two complementary seats, and neither replaces the
other. The SDD final-review seat, extended by
[security-final-review.md](security-final-review.md), carries the
**constraint-driven** branch check: every bound constraint verified against
the whole-branch diff, in the SDD-friendly format
`finishing-a-development-branch` adjudicates. The DSH `review` tool's
pre-merge pass is the **complementary** sweep: free-form, aimed at
whole-branch composition the constraint list never anticipated, with a
verifier refuting each finding. Run both; do not drop one because the
other exists.
