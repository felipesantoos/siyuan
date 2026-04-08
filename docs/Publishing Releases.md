# Publishing Releases

## Overview

Releases are built by GitHub Actions on tagged commits of the `patches` branch.
Pushing a tag matching `*-dev*` triggers the workflow at
`.github/workflows/cd.yml`, which compiles the Go kernel, builds the frontend,
packages everything with electron-builder, and uploads the installer to a new
GitHub Release on the fork.

The build matrix is currently limited to Windows only — output is
`siyuan-VERSION-win.exe`. Re-enabling the other four platforms (Linux AppImage,
Linux tar.gz, macOS x64, macOS arm64) is documented in **Re-enabling All
Platforms** below.

## Why Builds Come From Patches, Not Master

Tags in git point at commits, not branches. When the workflow fires, GitHub
Actions checks out the commit the tag points to and uses the `cd.yml` from
that commit. As long as the tag is created while on the `patches` branch, the
build picks up:

- The IsPaidUser bypass in `kernel/model/conf.go` and `app/src/util/needSubscribe.ts`
- The Windows-only CI matrix in `cd.yml`
- Any other modifications on the `patches` branch

Tagging from `master` would build pristine upstream SiYuan with the original
five-platform matrix and **without** the paywall bypass — not what we want.

## Prerequisites

Before triggering your first build, ensure all three conditions are met:

### 1. GitHub Actions is Enabled on the Fork

Visit `https://github.com/felipesantoos/siyuan/actions`. If you see a banner
with an "I understand my workflows, go ahead and enable them" button, click it.
Forks have Actions disabled by default as a safety measure.

### 2. A Baseline Release Exists

The CI changelog action at `cd.yml:32-39` (`pozetroninc/github-action-get-latest-release`)
requires at least one non-prerelease release. A fresh fork has none, so the job
fails before the build starts.

To create the placeholder:

1. Go to `https://github.com/felipesantoos/siyuan/releases`
2. Click **Draft a new release**
3. **Choose a tag** → type `v0.0.0-baseline` → **Create new tag on publish**
4. **Target:** `master`
5. **Release title:** `baseline`
6. **Description:** `Placeholder release to unblock CI changelog lookup`
7. **Do not** check "Set as a pre-release" (the action filters out prereleases)
8. Click **Publish release**

This is a one-time setup. All future builds use this placeholder as the
"previous release" for changelog generation.

### 3. The Patches Branch is Pushed

If you've recently rebased patches on top of a new master, push first:

```bash
git push origin patches --force-with-lease
```

## Triggering a Build

From the `patches` branch:

```bash
git checkout patches
git tag -a v3.6.3-dev1 -m "Build description here"
git push origin v3.6.3-dev1
```

The tag name must contain `-dev` somewhere to match the workflow filter at
`cd.yml:6`. Increment the suffix (`-dev2`, `-dev3`, ...) for each new build.
Annotated tags (`-a`) are preferred over lightweight tags so the build is
attributable.

## What Happens After the Push

1. GitHub detects the new tag and fires the **CD For SiYuan** workflow
2. **Create Release** job runs on Ubuntu (~1-2 min):
   - Looks up the previous release (`v0.0.0-baseline` or the most recent build)
   - Generates a changelog from commits and milestones
   - Creates a new GitHub Release on the fork as a prerelease
3. **windows build win.exe** job runs on `windows-latest` (~10-15 min):
   - Sets up MSYS2 with `mingw-w64-x86_64-lua`
   - Sets up Go from `kernel/go.mod`
   - Sets up Node 20 and pnpm
   - Builds the frontend (`pnpm run build`)
   - Builds the Go kernel with `Mode=prod` and `-H=windowsgui` flags
   - Packages with electron-builder (`pnpm run dist`)
   - Uploads `siyuan-3.6.3-win.exe` to the release

Watch progress at `https://github.com/felipesantoos/siyuan/actions`.

## Downloading the Installer

When the build finishes, find the artifact at:

`https://github.com/felipesantoos/siyuan/releases`

Look for the most recent prerelease (auto-titled `vYYYYMMDDHHMM`, generated at
`cd.yml:63`) and download the `siyuan-VERSION-win.exe` asset.

## Re-enabling All Platforms

The Windows-only matrix lives in commit `b10970840` on the `patches` branch.
To restore the full five-platform build:

**Option A — Revert the matrix-restriction commit:**

```bash
git checkout patches
git revert b10970840
git push origin patches --force-with-lease
```

**Option B — Edit `cd.yml` directly:** uncomment the four matrix entries
between lines ~92-127.

After updating `patches`, push a new dev tag to trigger a multi-platform build:

```bash
git tag -a v3.6.3-dev2 -m "Full multi-platform build"
git push origin v3.6.3-dev2
```

The matrix will then produce all five installers:

- `siyuan-VERSION-linux.AppImage`
- `siyuan-VERSION-linux.tar.gz`
- `siyuan-VERSION-mac.dmg`
- `siyuan-VERSION-mac-arm64.dmg`
- `siyuan-VERSION-win.exe`

## Common Failure Modes

### Create Release Fails on Changelog Parsing

The Python script `scripts/parse-changelog.py` queries the fork's milestones
via the GitHub API. A fork with no milestones may produce empty output or
raise an exception.

Fix: either create a milestone in the fork's Issues tab matching the version
being built, or patch the script to handle the empty case gracefully.

### Windows Build Fails at MSYS2 Setup

MSYS2 package mirrors are sometimes slow or temporarily unavailable. Re-running
the workflow usually fixes this — go to the failed run page and click
**Re-run failed jobs**.

### Go Build Fails with Linker Errors

Usually means a transitive Go dependency requires CGO and the MSYS2 mingw-w64
compiler is not on PATH. Verify that `cd.yml:147-149` still installs the
`mingw-w64-x86_64-lua` package and that no upstream change altered the CI
prerequisites.

### Electron-Builder Fails Packaging

Check for missing files referenced in `app/electron-builder.yml` under
`extraResources`. Most commonly the `pandoc/pandoc-windows-amd64.zip` file is
missing or corrupted in the source tree.

### Tag Push Has No Effect

If pushing the tag does not trigger a workflow run, verify GitHub Actions is
enabled on the fork (see **Prerequisites**) and that the tag name actually
contains `-dev`.

## What is NOT Affected by a Release

- The `master` branch is never touched by the build process
- Previous releases and their tags remain unchanged
- Source files in the working tree are not modified — the build runs on a
  fresh clone in the GitHub Actions runner
