# Syncing the fork with upstream

How the security-audit fork
([chunqing286/superpowers-dsh](https://github.com/chunqing286/superpowers-dsh))
tracks its upstream
([LayneChai/superpowers-dsh](https://github.com/LayneChai/superpowers-dsh)),
and what to do when upstream cuts a new release.

Commands are Windows **PowerShell**.

## The sync model

The fork's **`main` is the security line**: upstream's commits plus this
fork's security commits, on one branch. A fresh clone of the fork gives you
the fork's code — there is no separate "security patch branch" to apply.

```powershell
git clone https://github.com/chunqing286/superpowers-dsh.git
cd superpowers-dsh
git remote add upstream https://github.com/LayneChai/superpowers-dsh.git
```

## The sync loop (when upstream releases)

```powershell
git fetch upstream
git switch main
git rebase upstream/main
# resolve conflicts — see the next section

npm run verify                 # must report: 15 skills registered, 84 files checked
# re-check the SKILL.md byte cap — see "The byte-cap trap" below

# Bump package.json "version" to <new upstream version>-sec.N
# (e.g. upstream 0.5.0 -> 0.5.0-sec.1; re-syncing the same upstream
#  version increments N: 0.5.0-sec.2, ...)

# Update the docs (README sections, DEPLOY.md) to match the release BEFORE
# publishing — docs land before the npm publish, so the released artifact
# always matches the docs.

git add -A
git commit -m "chore: sync upstream v<version> and publish @chunqing286/superpowers-dsh@<version>-sec.N"
npm publish

git tag sec-v<upstream version>-<N>        # e.g. sec-v0.5.0-1
git push --force-with-lease origin main    # see below
git push origin sec-v<upstream version>-<N>

# Finally, update your own machine(s) — see DEPLOY.md, "Upgrade":
dsh plugin --profile <name> add @chunqing286/superpowers-dsh@<version>-sec.N
```

**About the force-push.** The first push of the fork's `main` was a
fast-forward. After the first rebase sync, the branch's commits have been
rewritten (the security commits now sit on top of the *new* upstream
history), so a plain `git push` is rejected. That is expected — use
`git push --force-with-lease origin main`. `--force-with-lease` (not plain
`--force`) refuses to overwrite the remote if someone else pushed to it in
the meantime. Never force-push anywhere else, and never push to `upstream`.

## Resolving conflicts

The fork touches upstream code only through minimal pointer edits inside
**three upstream files**, so conflicts can only land there:

- `skills/subagent-driven-development/SKILL.md` — the security bullet
  (constraints passed verbatim / `No security constraints bind this task.` /
  pointers to `references/security-review-part.md` and
  `references/security-loop.md`)
- `skills/subagent-driven-development/task-reviewer-prompt.md` — the
  security-part pointer (`references/security-review-part.md`)
- `skills/writing-plans/SKILL.md` — the `## Security Constraints` section
  and the `references/threat-modeler-prompt.md` pointer

Resolution principle: **re-point at the `references/security-*.md` files**
against upstream's new text. Specifically:

- **Never "keep upstream's version"** of the hunk — that drops the pointer
  and silently removes the security dimension.
- **Never "keep ours" blindly** — the upstream text around the pointer may
  have moved or been reworded, and our edit may no longer make sense there.
- Instead: read upstream's new version of the section, then re-attach the
  security pointer at its new home, rewording the pointer sentence to fit.

After resolving: `git rebase --continue`, then `npm run verify` before
committing anything else.

## The byte-cap trap

Every `SKILL.md` must stay under **20,000 bytes** (upstream's size gate;
Claude Code truncates skills at 20,000 characters after compaction).
`skills/subagent-driven-development/SKILL.md` currently sits at **19,883
bytes — 117 bytes of headroom**. Any upstream release that adds more than
that to this file breaks the size gate when our security bullet is re-added.

After **every** upstream sync, re-measure:

```powershell
(Get-Item skills\subagent-driven-development\SKILL.md).Length   # must be < 20000
```

If it is at or over 20,000, re-squeeze the security bullet (shorten the
pointer sentence; the bulk of the security content lives in `references/`
files, which have no cap) until the file is under the limit again.

## Version and tag naming

- npm version: **`<upstream version>-sec.N`** (N resets to 1 for each new
  upstream version, increments on re-syncs of the same one).
- Git tag: **`sec-v<upstream version>-<N>`**, mirroring the npm version.

The first release is npm `0.4.0-sec.1`, tag **`sec-v0.4.0-1`**. (The tag
carries the counter because a counter-less `sec-v0.4.0` would collide on the
second publish of the same upstream line.)

## Publish order

Docs first, package second: update and commit the READMEs / DEPLOY.md /
UPSTREAM-SYNC.md changes for a release **before** running `npm publish`, so
the released artifact always matches the documentation.

## If upstream implements the security capability itself

If a future upstream release ships its own security auditing, evaluate
converging instead of reflexively keeping the fork:

1. Compare upstream's mechanism against the fork's four insertion points:
   - threat model before planning (`writing-plans`,
     `references/threat-modeler-prompt.md`),
   - the plan's `## Security Constraints` section,
   - per-task security review (constraints passed verbatim into every task
     brief, `references/security-review-part.md`),
   - branch-level security review (final review security dimension,
     `references/security-final-review.md` / `references/security-loop.md`).
2. If upstream covers all four points with equal or stronger semantics,
   retire the fork: switch installs back to the unscoped `superpowers-dsh`
   package, note the convergence in the README, and archive the fork
   repository.
3. If upstream covers only some of them, keep the fork and drop only the
   insertion points that became redundant.

Until that evaluation is explicitly done and recorded, the fork stays the
security line.
