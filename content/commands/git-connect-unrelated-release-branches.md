+++
title = "Git unrelated histories: connecting dev, stg, and main branches"
date = 2026-05-23

[taxonomies]
categories = ["post"]
tags = ["git", "branch", "release", "github", "workflow"]
+++

A company release branch flow often looks like this:

```text
feature branches -> dev -> stg -> main
```

Here I call the production branch `main`. Depending on the team, this branch might be named `prd`.

In my real environment there was also an `int` branch, but the core problem was between `stg` and `main`. `dev` had branched from `stg`, so those two histories were connected. But `main` had been created separately and had no common ancestor with `dev` or `stg`.

In other words, the branch names looked like they belonged to the same release flow, but from Git's point of view they were two unrelated projects.

## Desired flow

The rule I wanted was simple.

```text
feature branches -> dev -> stg -> main
```

`main` should not receive feature branches directly. Only code validated in `dev` and promoted through `stg` should reach `main`.

Git history needs to understand that direction too.

```text
main <- stg <- dev
```

Here the arrow means "the branch on the right contains the history of the branch on the left."

- `stg` must contain the history of `main`.
- `dev` must contain the history of `stg`.
- That allows `dev -> stg` and `stg -> main` promotion PRs to work normally.

## Problem: unrelated histories

If `main` and `stg` have unrelated histories, Git will not connect them with a normal merge.

You can run into this message:

```bash
fatal: refusing to merge unrelated histories
```

If you force a PR in this state, the diff can look strange, or existing files can appear as if they are all new files. Git cannot find a shared baseline between the two branches.

## Fix direction

The goal was not to change file contents. The file contents already present in `stg` and `dev` needed to stay exactly as they were.

What I needed was one thing:

> Keep the current branch's file contents, but connect another branch as parent history.

The merge strategy for that is `-s ours`.

`-s ours` is not the same as choosing "ours" for a few conflicted files. It creates a merge commit, but the resulting tree uses the current branch's file contents.

So the following operation does not overwrite `stg` with files from `main`; it only connects `main` into `stg` history.

## 1. Connect main into stg

First, do not work directly on the real `stg`. Create a test branch.

```bash
git fetch origin
git checkout -b test-connect-main-stg origin/stg
```

Then merge `origin/main` while preserving the current branch's file contents, meaning the `stg` contents.

```bash
git merge origin/main --allow-unrelated-histories -s ours
```

Now always verify the diff.

```bash
git diff origin/stg..test-connect-main-stg
```

There should be no output. No output means the merge commit exists, but the file contents are the same as the original `stg`.

If the diff is empty, fast-forward the real `stg`.

```bash
git checkout stg
git merge --ff-only test-connect-main-stg
git push origin stg
```

After this, `origin/main` is an ancestor of `origin/stg`.

```text
main <- stg
```

## 2. Connect stg into dev

Next, connect `stg` into `dev` with the same approach.

```bash
git checkout -b test-connect-stg-dev origin/dev
```

Preserve the current `dev` file contents while connecting `origin/stg` as history.

```bash
git merge origin/stg --allow-unrelated-histories -s ours
```

Again, verify the diff.

```bash
git diff origin/dev..test-connect-stg-dev
```

There should be no output.

If everything is fine, fast-forward the real `dev`.

```bash
git checkout dev
git merge --ff-only test-connect-stg-dev
git push origin dev
```

Now the history has the desired shape.

```text
main <- stg <- dev
```

## Release flow after that

From now on, promotion PRs can be created normally.

```text
feature branch -> dev
dev -> stg
stg -> main
```

If automated, the flow looks like this:

```text
merge into dev
  -> auto PR: dev to stg
  -> after approval/merge
  -> auto PR: stg to main
```

If the real pipeline has `int`, insert it with the same principle.

```text
main <- stg <- int <- dev
```

The important part is to connect history from the production-side branch toward the development-side branch. That way PR diffs in the promotion direction, `dev -> int -> stg -> main`, are calculated naturally.

## Cautions

`-s ours` is powerful, but it can be dangerous. This strategy does not bring the other branch's file contents into the merge result.

Use it only when all of these are true:

1. The goal is connecting Git history, not changing files.
2. The current branch's file contents must remain unchanged.
3. It is clear that the other branch's file contents should not be applied to the current branch.
4. You verified on a test branch that the diff is empty.

In particular, if `main` has real production code that is missing from `stg`, do not use this method as-is. In that case, first decide which code should be the source of truth.

## Three-line summary

Branch names and environment names are a human-readable flow. Git understands the commit graph.

Naming branches `dev`, `stg`, and `main` does not automatically create a promotion relationship. PR diffs, mergeability, and automatic promotion all depend on whether the branches have a common ancestor commit.

The core of this issue was not fixing code, but connecting history correctly. The safe approach was to create an `-s ours` merge on a test branch, verify that the diff was empty, and then fast-forward the real branches.
