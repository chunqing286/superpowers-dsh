# Syncing the fork with upstream

How the security-audit fork
([chunqing286/superpowers-dsh](https://github.com/chunqing286/superpowers-dsh))
tracks the upstream port
([LayneChai/superpowers-dsh](https://github.com/LayneChai/superpowers-dsh)),
and what to do when the upstream port cuts a new release.

Two different "upstreams" appear in this document, always qualified:

- **the upstream port** — LayneChai/superpowers-dsh, the DSH plugin
  repository this fork is rebased on;
- **the upstream skill content** — obra/superpowers, the Claude-Code skills
  library the port (and therefore this fork) takes its skill text from.

Commands are Windows **PowerShell**. `<…>` anywhere in a command is a
placeholder: substitute a real value before running, or PowerShell will
reject the command (`<` is a reserved character).

## The sync model

The fork's **`main` is the security line**: the upstream port's commits plus
this fork's security commits, on one branch. A fresh clone of the fork gives
you the fork's code — there is no patch set to apply on top. (The branch
`local/security-audit` also exists on `origin` and is kept identical to
`main`; `main` is the canonical line.)

```powershell
git clone https://github.com/chunqing286/superpowers-dsh.git
cd superpowers-dsh
git remote add upstream https://github.com/LayneChai/superpowers-dsh.git
```

## The sync loop (when the upstream port releases)

```powershell
git fetch upstream
git switch main
git rebase upstream/main
# resolve conflicts — see the next section

npm run verify                 # must print "verify-skills: OK"
# re-check the SKILL.md byte cap — see "The byte-cap trap" below

# Bump package.json "version" to <new upstream version>-sec.N
# (e.g. upstream 0.5.0 -> 0.5.0-sec.1; re-syncing the same upstream
#  version increments N: 0.5.0-sec.2, ...)

# Update the docs (README sections, DEPLOY.md) to match the release BEFORE
# publishing — docs land before the npm publish (see "Publish order").

git add -A
git commit -m "chore: sync upstream v<version> and publish @chunqing286/superpowers-dsh@<version>-sec.N"
npm publish                  # prerequisites below

git tag sec-v<upstream version>-<N>        # e.g. sec-v0.5.0-1
git push --force-with-lease origin main    # see below
git push --force-with-lease origin main:local/security-audit
                                           # keep the mirror branch identical to main
git push origin sec-v<upstream version>-<N>

# Finally, update your own machine(s) — see DEPLOY.md, "Upgrade":
dsh plugin --profile web add @chunqing286/superpowers-dsh@<version>-sec.N
```

**`npm run verify` output.** The invariant is that the run prints
`verify-skills: OK`. The counts are a snapshot, not an invariant: this
release reports 15 skills registered, 84 files checked, and the next
upstream release will change both numbers.

**`npm publish` prerequisites.** The package is owned by the npm account
`chunqing286`; the publisher must be logged in — `npm whoami` prints
`chunqing286` — with publish rights for the `@chunqing286` scope, using the
account's configured 2FA / automation token. `publishConfig.access =
"public"` in `package.json` is what makes the scoped package public; no
`--access public` flag is needed.

**About the force-push.** The first push of the fork's `main` was a
fast-forward. After the first rebase sync, the branch's commits have been
rewritten (the security commits now sit on top of the *new* upstream port
history), so a plain `git push` is rejected. That is expected — use
`git push --force-with-lease origin main`. `--force-with-lease` (not plain
`--force`) refuses to overwrite the remote if someone else pushed to it in
the meantime. Never force-push anywhere else, and never push to `upstream`.
The one exception is the mirror push already in the loop,
`git push --force-with-lease origin main:local/security-audit`: that branch
is a **mirror** of `main`, so after `main` is rebased it must be rewritten
to the new `main` — and `--force-with-lease` limits that rewrite to the
state you last fetched, so a concurrent change on the branch is refused
rather than silently overwritten.

## Resolving conflicts

`git diff --stat v0.4.0..HEAD` shows the fork's **full delta** relative to
the upstream port: 17 paths. Only the **nine files that already exist
upstream** are conflict-bearing — **four upstream skill files** plus the
**five more upstream files the fork modifies**, listed below. The other
eight are fork-only
additions (`DEPLOY.md`, `UPSTREAM-SYNC.md`, and the six new
`skills/subagent-driven-development/references/*.md` files); a file that
exists only in the fork can never conflict. (An earlier version of this
document listed only three skill files; that was wrong, and the correction
matters because `references/final-review.md` carries the branch-level
security review.)

The four upstream skill files, each carrying a minimal security pointer:

- `skills/subagent-driven-development/SKILL.md` — the security bullet
  (constraints passed verbatim / `No security constraints bind this task.` /
  pointers to `references/security-review-part.md` and
  `references/security-loop.md`)
- `skills/subagent-driven-development/task-reviewer-prompt.md` — the
  security-part pointer (`references/security-review-part.md`)
- `skills/subagent-driven-development/references/final-review.md` — the
  4-line splice pointing the final whole-branch review at
  `references/security-final-review.md` (dropping it silently removes the
  branch-level security review)
- `skills/writing-plans/SKILL.md` — the `## Security Constraints` section
  and the `references/threat-modeler-prompt.md` pointer

The five upstream files the fork modifies (they exist upstream — the
fork-only additions are the other group):

- `package.json` — upstream will bump `version` and may change any other
  field. Keep the scoped name `@chunqing286/superpowers-dsh`, the `-sec.N`
  version, the fork's `description` (the security-audit sentence), and the
  fork's `repository` / `homepage` / `bugs` URLs; take upstream's other
  fields.
- `README.md` / `README.en.md` — keep this fork's section (the
  security-audit line) and the fork-pointing install commands; take
  upstream's prose changes everywhere else.
- `cordis.patch.yml` — upstream keeps the row `name: 'superpowers-dsh'`
  because upstream's package is unscoped. Keep **our** row `name` exactly in
  lockstep with `package.json`'s `name`
  (`@chunqing286/superpowers-dsh`) — the row `id: superpowers-dsh` stays as
  it is. Taking upstream's `name` here breaks activation: the package
  installs, then the row fails to import. This happened once already — the
  package was moved to the `@chunqing286` scope and the stale `name` was not
  updated in the same commit.
- `lib/index.js` — keep our scoped provider `name`
  (`@chunqing286/superpowers-dsh`) and `PACKAGED_SKILL_RANK` if upstream
  changes it; take upstream's other changes.

Resolution principle for all of them: **re-point at the
`references/security-*.md` files** against upstream's new text.
Specifically:

- **Never "keep upstream's version"** of a skill-file hunk — that drops the
  pointer and silently removes the security dimension.
- **Never "keep ours" blindly** — the upstream text around the pointer may
  have moved or been reworded, and our edit may no longer make sense there.
- Instead: read upstream's new version of the section, then re-attach the
  security pointer at its new home, rewording the pointer sentence to fit.
  For `references/final-review.md`, re-apply the 4-line
  `references/security-final-review.md` splice after resolving upstream's
  text — keep the splice and attach it at its new home.

After resolving: `git rebase --continue`, then `npm run verify` before
committing anything else.

## Never run the upstream sync script in this fork

**`scripts/sync-from-upstream.mjs <obra-checkout>` (equivalently
`npm run sync -- <checkout>`) is destructive in this fork — do not use it to
pull upstream.** (Bare `npm run sync` without a checkout path only prints its
usage and exits without writing anything.) The script serves the upstream
*port* repo's own re-porting from obra/superpowers, not this fork's sync
with the upstream port. Its source is **obra/superpowers** (plus
`port/overrides/`), and it **deletes and replaces the whole `skills/`
directory** (`rm -r`, then rename a staging copy into place). Running it
here would:

- delete this fork's six added security files, all under
  `skills/subagent-driven-development/references/`: `security-loop.md`,
  `security-review-part.md`, `security-kit.md`, `security-final-review.md`,
  `threat-modeler-prompt.md`, `plan-security-section.md`;
- restore the four thin-shell-modified upstream files to their upstream text
  (`skills/subagent-driven-development/SKILL.md`,
  `skills/subagent-driven-development/task-reviewer-prompt.md`,
  `skills/subagent-driven-development/references/final-review.md`,
  `skills/writing-plans/SKILL.md`) — erasing the security dimension;
- and **not detect the loss**: the script's assertions only check that each
  transform matched its anchor, not that the fork's files still exist, so it
  exits successfully leaving a `skills/` tree with no security
  dimension.

The correct way to pull the upstream port is the **git rebase** loop above
(`git fetch upstream` → `git rebase upstream/main`). If a re-port from obra
is genuinely needed (rare), first confirm the working tree is clean, do it
on a separate branch, and afterwards check with `git diff` that the security
files above are all still present.

## The byte-cap trap

Every `SKILL.md` must stay under **20,000 bytes**. The limit comes from the
upstream skill content — obra/superpowers v7.0.1's
`tests/skill-size/test-skill-size.sh` (`LIMIT=20000`, measured in bytes,
because Claude Code re-attaches a skill truncated to 20,000 characters after
compaction and bytes are stricter than characters). Two things do **not**
protect you here: the upstream port repo (LayneChai/superpowers-dsh) has no
`tests/` directory, and this repo's `npm run verify` does **not** enforce
the cap — so you must measure the file yourself.

`skills/subagent-driven-development/SKILL.md` currently sits at **19,982
bytes — 18 bytes of headroom**. Any upstream release that adds more than
that to this file breaks the cap when our security bullet is re-added.

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

The current release is npm `0.4.0-sec.2`, tag **`sec-v0.4.0-2`**. (The first
release, `0.4.0-sec.1` / tag `sec-v0.4.0-1`, is deprecated: it installed but
did not activate. The tag carries the counter because a counter-less
`sec-v0.4.0` would collide on the second publish of the same upstream line.)

## Publish order

Docs first, package second: update and commit the READMEs / DEPLOY.md /
UPSTREAM-SYNC.md changes for a release **before** running `npm publish`.
Only the two READMEs ship in the npm tarball, so "the artifact matches the
docs" means the **published README** matches the release; DEPLOY.md and
UPSTREAM-SYNC.md live in the repository only and are versioned alongside in
the same commit.

## If the upstream port implements the security capability itself

If a future upstream port release ships its own security auditing, evaluate
converging instead of reflexively keeping the fork:

1. Compare the upstream port's mechanism against the fork's four insertion
   points:
   - threat model dispatched by writing-plans just before it writes the
     `## Security Constraints` section (`writing-plans`,
     `references/threat-modeler-prompt.md`),
   - the plan's `## Security Constraints` section,
   - per-task security review (constraints passed verbatim into every task's
     dispatch and the reviewer's input, `references/security-review-part.md`),
   - branch-level security review (final review security dimension,
     `references/security-final-review.md` / `references/security-loop.md`).
2. If the upstream port covers all four points with equal or stronger
   semantics, retire the fork: switch installs back to the unscoped
   `superpowers-dsh` package, note the convergence in the README, and
   archive the fork repository.
3. If the upstream port covers only some of them, keep the fork and drop
   only the insertion points that became redundant.

Until that evaluation is explicitly done and recorded, the fork stays the
security line.
