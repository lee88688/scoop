# AGENTS.md

## Purpose

A personal Scoop bucket (from `ScoopInstaller/BucketTemplate`) for manifests of
packages sourced mostly from GitHub releases. The `excavator` workflow auto-
updates versions on GitHub on a schedule.

## Layout

- `bucket/`                 manifest JSON files (one per app); `app-name.json.template` is the starter
- `bin/`                    maintenance scripts: `checkver.ps1`, `test.ps1`, `auto-pr.ps1`, `formatjson.ps1`, ...
- `Scoop-Bucket.Tests.ps1` Pester suite run by CI
- `.github/workflows/`     `ci.yml` (tests), `excavator.yml` (auto-update), issue/PR handlers

## Create a package

1. Copy `bucket/app-name.json.template` → `bucket/<app>.json`; fill in `version`, `url`, `hash`, `bin`, `description`, `homepage`, `license`.
2. For GitHub-release sources set `"checkver": "github"` and an `autoupdate` block using `$version` in the URL template.
3. Test: `scoop install .\bucket\<app>.json` (reads the file live). Lower `version` then `.\bin\checkver.ps1 <app> -f` to verify autoupdate; `.\bin\test.ps1` (needs `Pester`, `BuildHelpers`).
4. `git add bucket\<app>.json; git commit; git push`.

## Modify a package

- Edit `bucket\<app>.json`, retest with `scoop uninstall <app>; scoop cache rm <app>; scoop install .\bucket\<app>.json`, then commit + push.

## Delete a package

- Prefer deprecation over hard delete: `git mv bucket\<app>.json deprecated\<app>.json`; optionally add a `notes` field pointing to a replacement. Installed users then see "Deprecated" in `scoop status`/`scoop list`/`scoop info` instead of silently "Removed".
- Hard delete only when truly gone: `git rm bucket\<app>.json`. Locally `scoop uninstall <app>`.

## Update versions (manual)

- `.\bin\checkver.ps1 * -u` (all) or `.\bin\checkver.ps1 <app> -u` (one), then commit + push. The `excavator` workflow does this automatically on GitHub.

## Commit messages

Follow the official Scoop bucket convention (see
[ScoopInstaller/.github CONTRIBUTING](https://github.com/ScoopInstaller/.github/blob/main/.github/CONTRIBUTING.md)
and the Extras commit history):

| Change                         | Format                                        | Example                              |
| ------------------------------ | --------------------------------------------- | ------------------------------------ |
| New manifest                   | `<app>: Add version <version>`                | `netcatty: Add version 1.1.70`       |
| Version bump (existing)        | `<app>: Update to version <version>`          | `netcatty: Update to version 1.1.71` |
| Maintenance / chore            | `(chore): <description>`                      | `(chore): fix netcatty autoupdate`   |

- The excavator bot auto-commits version bumps as `<app>: Update to version <version>`.
- Non-manifest changes (scripts, docs, skills) use `(chore): <description>`.

## Manifest conventions

Per the official contributing guide:

- **Field order** (whichever exist): `version`, `description`, `homepage`,
  `license`, `notes`, `depends`, `suggest`, `architecture`, `url`, `hash`,
  `extract_dir`, `extract_to`, `pre_install`, `installer`, `post_install`,
  `env_add_path`, `env_set`, `bin`, `shortcuts`, `persist`, `pre_uninstall`,
  `uninstaller`, `post_uninstall`, `checkver`, `autoupdate`.
- **Indentation**: 4 spaces (run `.\bin\formatjson.ps1` to normalize).
- **License**: valid [SPDX identifier](https://spdx.org/licenses/).
- **CLI app**: add to `bin`, omit `shortcuts`.
- **GUI app without CLI args**: add to `shortcuts`, omit `bin`.
- **Single-item array**: write as a string, not an array.
- **`architecture`**: mandatory unless the app provides _only_ a 32bit download.
- **Portable config**: prefer `persist` for user data that should survive updates.

## Pre-commit checks

Before committing, run:

```powershell
.\bin\formatjson.ps1   # normalize manifest JSON (4-space indent, field order)
.\bin\test.ps1         # run the Pester suite (needs Pester, BuildHelpers)
```

The Pester suite includes a `files have no lines containing trailing
whitespace` test — ensure no staged file has trailing spaces, or CI will fail.

## Push

The `excavator` workflow auto-commits version bumps on GitHub on a schedule, so
the remote `master` may have advanced since your last fetch. **Always rebase,
never merge**, to keep a linear history:

```powershell
git pull --rebase
git push
```

If the rebase conflicts (e.g. the excavator bumped the same manifest you also
edited), resolve by taking the excavator's version for version/hash fields and
re-running `.\bin\checkver.ps1 <app> -f` to regenerate, then `git rebase --continue`.
