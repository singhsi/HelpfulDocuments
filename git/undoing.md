# Git — Undoing Things

Mistakes happen. This file covers every common "undo" scenario in Git, what each command actually does, and which one to reach for.

> **See also:** [README.md](README.md) · [commands.md](commands.md)

---

## The Golden Rules Before You Undo

1. **Has it been pushed?** If yes, prefer `git revert` — it's safe for shared branches. If no, you have more options.
2. **Do you want to keep the changes?** Some commands discard changes permanently. Check before running.
3. **When in doubt, use `git reflog`** — Git almost never truly deletes anything. If something goes wrong, `reflog` can get it back.

---

## Quick Decision Table

| Situation | Command |
|---|---|
| Undo `git add` (unstage a file) | `git restore --staged <file>` |
| Discard uncommitted changes to a file | `git restore <file>` |
| Discard all uncommitted changes | `git restore .` |
| Delete all untracked files | `git clean -fd` |
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` |
| Undo last commit, keep changes unstaged | `git reset HEAD~1` |
| Undo last commit, discard changes entirely | `git reset --hard HEAD~1` |
| Undo a commit that's already been pushed | `git revert <commit>` |
| Recover something you accidentally deleted | `git reflog` |
| Remove a file from history (e.g. committed a secret) | see [Removing a File from History](#removing-a-file-from-history) |

---

## Undoing a `git add` (before committing)

You staged a file but changed your mind.

#### unstage a specific file
`git restore --staged <filename>`

#### unstage everything
`git restore --staged .`

The changes are still in your working directory — they're just no longer queued for the next commit.

---

## Discarding Uncommitted Changes

> ⚠️ These commands throw away local changes permanently. There is no undo.

#### discard changes to a specific file (revert it to the last committed state)
`git restore <filename>`

#### discard all uncommitted changes in the working directory
`git restore .`

#### delete all untracked files and directories (files not yet added to Git)
`git clean -fd`

> **Tip:** Run `git clean -nfd` first (the `-n` flag is a dry run) to see what would be deleted before actually deleting it.

---

## Undoing the Last Commit

These commands all move the branch pointer back one commit. The difference is what happens to your changes.

#### `--soft` — undo the commit, keep changes staged
```
git reset --soft HEAD~1
```
The commit is gone but your changes are still staged, ready to be re-committed. Use this when you want to fix the commit message or combine it with another commit.

#### `--mixed` (default) — undo the commit, keep changes but unstage them
```
git reset HEAD~1
```
The commit is gone and your changes are back in the working directory, unstaged. Use this when you want to re-do the commit differently.

#### `--hard` — undo the commit and throw away all changes
```
git reset --hard HEAD~1
```
The commit is gone and the changes are deleted. Use this when you're sure you don't need those changes at all.

> ⚠️ `--hard` is permanent for uncommitted changes. However, the commit itself is recoverable via `git reflog` for ~30 days.

#### undo multiple commits
```
git reset --soft HEAD~3   # undo the last 3 commits, keep changes staged
git reset --hard HEAD~3   # undo the last 3 commits, discard everything
```

---

## Undoing a Commit That's Already Been Pushed

**Never use `git reset` on a branch that others are working on.** It rewrites history and will cause conflicts for everyone who has pulled that branch.

Instead, use `git revert` — it creates a new commit that undoes the changes, leaving history intact.

#### revert the most recent commit
```
git revert HEAD
```

#### revert a specific commit
```
git revert <commit-hash>
```

#### revert without immediately committing (so you can edit the message or combine with other changes)
```
git revert --no-commit <commit-hash>
git commit -m "revert: describe what you're undoing"
```

#### revert a merge commit
```
git revert -m 1 <merge-commit-hash>
# -m 1 tells Git to keep the first parent (the branch you merged into)
```

---

## Fixing the Last Commit Message

Didn't push yet? You can rewrite the message without creating a new commit.

```
git commit --amend -m "corrected commit message"
```

Already pushed? Don't amend — it rewrites history. Use `git revert` with a follow-up commit instead, or simply leave it and move on.

---

## Recovering Lost Work with `git reflog`

`git reflog` is your safety net. It records every place HEAD has pointed, even after resets, rebases, or branch deletions. Git keeps this log for ~30 days.

#### view the reflog
```
git reflog
```

Output looks like:
```
abc1234 HEAD@{0}: reset --hard HEAD~1: updating HEAD
def5678 HEAD@{1}: commit: feat: add login page     <-- this is what you lost
gh9012  HEAD@{2}: commit: fix: correct null check
```

#### recover a lost commit
```
git checkout def5678          # inspect it first
git checkout -b recovered-branch   # save it as a new branch
```

Or if you just want to reset back to it:
```
git reset --hard def5678
```

---

## Undoing a Rebase

If a rebase went wrong, abort it while it's in progress:
```
git rebase --abort
```

If you already finished the rebase and want to go back to where you were before:
```
git reflog                       # find the entry just before the rebase started
git reset --hard HEAD@{<n>}      # reset back to that point
```

---

## Removing a File from History

If you accidentally committed a secret, password, or large binary file, deleting it in a later commit is **not enough** — it still exists in the Git history and anyone who clones the repo can see it.

> ⚠️ This rewrites history. Coordinate with your team before doing this on a shared branch. Everyone will need to re-clone or rebase after.

#### using `git filter-repo` (recommended — faster and safer than the older `filter-branch`)
```
# install if needed: pip install git-filter-repo

git filter-repo --path <filename> --invert-paths
```

After running this:
1. **Rotate the secret immediately** — assume it's already compromised
2. Force-push the rewritten history: `git push origin --force --all`
3. Ask all teammates to re-clone the repo

#### if the commit was only local (not yet pushed)
```
git reset --hard HEAD~1    # if it was the very last commit
```
Then remove the file, add it to `.gitignore`, and re-commit.

---

## Common Scenarios

#### "I committed to `main` by mistake instead of my feature branch"
```
git log --oneline -3          # note the commit hash you want to move
git reset --soft HEAD~1       # undo the commit on main, keep changes staged
git checkout -b feature/my-fix
git commit -m "feat: my fix"  # re-commit on the correct branch
```

#### "I accidentally deleted a branch"
```
git reflog                    # find the last commit that was on that branch
git checkout -b <branch-name> <commit-hash>
```

#### "I need to undo one specific commit in the middle of history (not the latest)"
```
git log --oneline             # find the commit hash
git revert <commit-hash>      # creates a new commit that undoes just that one
```
Note: this can cause conflicts if later commits depend on the one you're reverting.

#### "I want to unstage everything and start my commit fresh"
```
git restore --staged .        # unstage all files
# make your changes, then re-stage what you want
git add <files>
git commit
```
