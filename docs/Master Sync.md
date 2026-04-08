# Master Sync

## Overview

This fork uses the **patches branch strategy** to stay in sync with the upstream
SiYuan repository while preserving local modifications:

- **`master`** is kept as a pristine mirror of `siyuan-note/siyuan@master`. It
  is never modified directly.
- **`patches`** holds all local changes (the `IsPaidUser()` bypass, the
  Windows-only CI matrix, this documentation). It is rebased on top of `master`
  whenever upstream is updated.

This separation makes it easy to:

- Track upstream changes without conflicts polluting the modification history
- Drop all modifications by checking out `master`
- See exactly what is different from upstream with `git diff master..patches`

## One-time Setup

The fork itself is `origin`. The real upstream is added as a separate remote:

```bash
git remote add upstream git@github.com:siyuan-note/siyuan.git
git fetch upstream
```

After running this, `git remote -v` should show both remotes:

```
origin    git@github.com:felipesantoos/siyuan.git (fetch)
origin    git@github.com:felipesantoos/siyuan.git (push)
upstream  git@github.com:siyuan-note/siyuan.git (fetch)
upstream  git@github.com:siyuan-note/siyuan.git (push)
```

## Routine Sync

Run this whenever you want to pull in the latest SiYuan changes — typically
once every few weeks, or before triggering a new release build:

```bash
git checkout master
git fetch upstream
git merge --ff-only upstream/master
git push origin master
```

The `--ff-only` flag is critical. It refuses to merge if a fast-forward isn't
possible, which protects `master` from accidentally diverging from upstream.
If the merge fails because `master` has its own commits, master is no longer a
pristine mirror and needs to be reset (see **Recovery** below).

## Rebasing Patches on Top of New Master

After updating `master`, replay the `patches` branch on top of the new upstream:

```bash
git checkout patches
git rebase master
```

If there are no conflicts, push the rebased branch:

```bash
git push origin patches --force-with-lease
```

`--force-with-lease` is a safer alternative to `--force`. It refuses to push if
someone else has updated the remote in the meantime. For a personal fork this
is rare, but the safety check costs nothing.

## Handling Conflicts

If `git rebase master` reports conflicts, the most likely culprit is
`kernel/model/conf.go` if upstream modified `IsPaidUser()` or its surroundings.
As of 2026-04-08, this function has been stable since 2023-09-25, so conflicts
should be rare.

To resolve a conflict:

1. Open the conflicted file and find the `<<<<<<<` markers
2. Resolve manually, preserving the `return true` bypass while incorporating
   any new logic upstream added
3. Stage the file and continue: `git add <file> && git rebase --continue`
4. Repeat for any further conflicts

If a conflict is too complex to resolve by hand, abort with `git rebase --abort`
and inspect the upstream changes first:

```bash
git log --oneline master --not patches@{1}
git show <suspicious-commit>
```

## Recovery: Resetting Master If It Diverged

If `master` ever has its own commits and is no longer a pristine mirror,
reset it to upstream:

```bash
git checkout master
git fetch upstream
git reset --hard upstream/master
git push origin master --force-with-lease
```

This is destructive — it discards any commits that were on `master`. Confirm
nothing valuable lives there before resetting.

## What is NOT Affected by a Sync

- The `patches` branch is only touched when you explicitly rebase it
- Tags created from previous builds (`v3.6.3-dev1`, etc.) remain pinned to
  their original commits
- Existing GitHub Releases on the fork are independent of branch state
