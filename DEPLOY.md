# Deploying `@chunqing286/superpowers-dsh`

Operational guide for installing, verifying, upgrading, and rolling back the
security-audit fork of the upstream port
[LayneChai/superpowers-dsh](https://github.com/LayneChai/superpowers-dsh)
on any machine.

- Fork repository: <https://github.com/chunqing286/superpowers-dsh>
- npm package: **`@chunqing286/superpowers-dsh`** (first release `0.4.0-sec.1`,
  based on the upstream port's v0.4.0; skill content from obra/superpowers
  v7.0.1)
- This file ships in the GitHub repository only — it is **not** inside the npm
  tarball. If you installed the package and are reading the bundled README,
  read this doc at
  <https://github.com/chunqing286/superpowers-dsh/blob/main/DEPLOY.md>.

Commands below are Windows **PowerShell**; the `dsh` CLI is assumed to be on
`PATH`. The examples use the profile name `web` — replace it with your
profile (the desktop app's profile is `desktop`). `<…>` anywhere in a command
is a placeholder: substitute a real value before running, or PowerShell will
reject the command (`<` is a reserved character).

## Naming: the plugin row id is not the npm package name

The npm package is scoped (`@chunqing286/superpowers-dsh`), but
`cordis.patch.yml` deliberately keeps the DSH plugin **row id** as
`superpowers-dsh`. That is intentional: it lets later composition layers keep
addressing the row by the same id the upstream port uses. Do not "fix" either
side — the scoped name is correct in `package.json`, the unscoped id is
correct in `cordis.patch.yml`.

## Prerequisites

- DeepSeek Harness (DSH) installed — the desktop app, or the CLI (`dsh`).
- PowerShell (the examples are Windows-native; on other platforms adapt the
  path syntax only — every command otherwise works as written).

## Install

There are three recommended, supported **distribution** forms, and all of
them install from the npm registry — the CLI and in-session forms take the
registry spec `@chunqing286/superpowers-dsh@<version>`, while the desktop
Plugins page takes the package name and resolves the latest version:

1. **Desktop app (recommended for desktop users):** open the in-app
   **Plugins** page and add the package by name:
   `@chunqing286/superpowers-dsh`.
2. **CLI, for a named profile:**

   ```powershell
   dsh plugin --profile web add @chunqing286/superpowers-dsh@0.4.0-sec.1
   ```

   This forwards to pnpm inside that profile's own directory and initializes
   the profile on first use.
   **Warning:** the desktop build runs in its own profile (`desktop`), so
   `--profile web` can install into a *different* profile than the one the
   app actually loads. For the desktop app, use the Plugins page (option 1).
3. **Inside an agent session:** the harness's `plugin_manager` tool with
   action `install_bundle` and target
   `@chunqing286/superpowers-dsh@0.4.0-sec.1`.

> **Never install with a `link:` spec.** A `link:` spec is an absolute,
> machine-specific path that breaks on every other machine. The npm registry
> spec is the only supported **distribution** form. The READMEs'
> upstream-derived install sections ("Install from GitHub", "Install from a
> tarball or a local folder") also show GitHub-URL and tarball/local-folder
> installs: those work against a local checkout but are machine-specific and
> are **not** a supported way to install this fork on another machine. (A
> local folder or `file:` install is fine for developing the plugin on your
> own machine — it is just not a distribution method.)

### Replacing an existing `superpowers-dsh` bundle

If the profile already has the upstream port's (unscoped) bundle installed,
**remove the old bundle before installing the fork** — via the Plugins page,
or:

```powershell
dsh plugin --profile web remove superpowers-dsh
```

Otherwise two copies of the same-named skills sit in the profile and compete.
Duplicate-name resolution is rank-based — but the old package and the fork
both register at `PACKAGED_SKILL_RANK = 550`, so with equal ranks **the old
copy can win**. The concrete symptom: the old and new copies of
`subagent-driven-development` both register, and after a "successful"
install you can end up running the old, non-security copy.

Restart the profile after installing (stop and re-run the app / `dsh web`,
then refresh the browser) — the bundle layer mounts at profile startup.

## Verify

1. **The version exists on the registry:**

   ```powershell
   npm view @chunqing286/superpowers-dsh versions
   ```

   lists the published versions.

2. **The profile recorded it:** open the profile's `package.json`
   (`$DSH_HOME/profiles/web/package.json` — `$DSH_HOME` defaults to `~/.dsh`;
   the desktop app's profile is `desktop`). It must show
   `@chunqing286/superpowers-dsh` in `dependencies`, and
   `dsh.profile.bundles` must list `@chunqing286/superpowers-dsh` — and,
   if you migrated from the upstream port's package, must **no longer** list
   the old unscoped `superpowers-dsh`.

3. **The security dimension loaded:** in an agent session, load the skill
   `subagent-driven-development` with the `skill` tool. Its `SKILL.md` must
   contain a security bullet naming `references/security-review-part.md`, the
   literal `No security constraints bind this task.`, and a
   `references/security-loop.md` pointer. Then load `writing-plans`: its
   `SKILL.md` must contain a `## Security Constraints` section and a
   `references/threat-modeler-prompt.md` pointer. If either marker is missing,
   the security dimension did not install (most likely cause: an old copy of
   the bundle still registered — see *Replacing an existing bundle* above).

4. **Deep check (optional but recommended):** in a clone of the fork (or the
   installed package directory) run:

   ```powershell
   npm run verify
   ```

   The run must end with `verify-skills: OK`; this release reports
   15 skills registered, 84 files checked.

   `npm run compat` replays the DSH contract checks against a locally
   installed DSH SDK — but it looks in `$DSH_HOME/profiles/node_modules`
   (default `~/.dsh/profiles/node_modules`) and **skips by default** when it
   finds nothing there. To actually run it, install the SDK matching your
   DSH version into a temp dir and point the script at it (example: DSH
   `0.2.1-rc.5`):

   ```powershell
   npm install --prefix $env:TEMP\dsh-sdk-check @deepseek-ai/dsh-skill@0.2.1-rc.5
   node scripts\check-dsh-compat.mjs $env:TEMP\dsh-sdk-check\node_modules
   ```

   Expected output on that machine (the second line is the SDK directory the
   script prints):

   ```
   dsh SDK      : @deepseek-ai/dsh-skill 0.2.1-rc.5
                  C:\Users\Administrator\AppData\Local\Temp\dsh-sdk-check\node_modules\@deepseek-ai\dsh-skill
   mounted      : 15 skills listed and read through the real SkillRegistry
   rank         : PACKAGED_SKILL_RANK=550 is below BUNDLED_SKILL_RANK=600

   check-dsh-compat: OK — this plugin matches the installed DSH SDK
   ```

## Upgrade

An upgrade **replaces the installed plugin directory wholesale**: the profile's
`node_modules\@chunqing286\superpowers-dsh` directory is recreated from the
new version. The *directory* is a plain directory, but the *files inside it
are hardlinks into the pnpm store* — so never hand-edit an installed copy:
editing a file in place mutates the **shared store copy** as well (corrupting
every other profile that links the same file), and the next upgrade wipes the
edit anyway. In-place patching of an installed copy is never a valid
strategy — upgrading means installing a newer version spec, not editing
files. (This is exactly why the fork ships the security content inside the
package.)

```powershell
dsh plugin --profile web add @chunqing286/superpowers-dsh@0.4.0-sec.2
```

(substitute the version you are upgrading to). Then restart the profile and
re-run the verification steps above.

## Rollback

Install the previously published `-sec.N` version — the first one is
`0.4.0-sec.1`. List what exists:

```powershell
npm view @chunqing286/superpowers-dsh versions
```

Then install the one you want back, e.g.:

```powershell
dsh plugin --profile web add @chunqing286/superpowers-dsh@0.4.0-sec.1
```

Restart the profile afterwards.

## Syncing with upstream

Maintainer-side procedure (rebase onto upstream port releases, conflict
rules, version/tag naming): see
[UPSTREAM-SYNC.md](https://github.com/chunqing286/superpowers-dsh/blob/main/UPSTREAM-SYNC.md).
