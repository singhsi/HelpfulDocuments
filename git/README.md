# Git — New Developer Guide

A practical reference for everyday Git workflows. Start here, then explore the other files in this folder.

> **See also:** [commands.md](commands.md) · [stash.md](stash.md) · [undoing.md](undoing.md)

---

## Table of Contents

- [Merge vs Rebase](#merge-vs-rebase)
- [Commit Best Practices](#commit-best-practices)
- [Branch Naming](#branch-naming)
- [Working with Remotes](#working-with-remotes)
- [Undoing Things](undoing.md)
- [Useful Log Tricks](#useful-log-tricks)
- [.gitignore Tips](#gitignore-tips)

---

## Merge vs Rebase

These are two different ways to integrate changes from one branch into another. Knowing when to use each will save you a lot of confusion.

### Merge

Merging takes all the changes from one branch and creates a **new merge commit** joining them together. The original history of both branches is fully preserved.

```
      A---B---C  feature
     /         \
D---E---F---G---H  main  (H is the merge commit)
```

```
git checkout main
git merge feature
```

**When to use merge:**
- Bringing a feature branch into `main` or `develop`
- You want to preserve the full history of when and how branches diverged
- Working in a team where others may have already based work on your branch

**Pros:** Safe, non-destructive, easy to understand  
**Cons:** Can clutter history with many merge commits on a busy project

---

### Rebase

Rebasing takes your commits and **replays them on top of another branch** as if you had started from there. It rewrites commit history to produce a clean, straight line.

```
Before:               After rebase onto main:
      A---B  feature        A'--B'  feature (replayed on top)
     /                     /
D---E---F  main      D---E---F  main
```

```
git checkout feature
git rebase main
```

**When to use rebase:**
- Keeping your feature branch up to date with `main` before opening a pull request
- Cleaning up a messy local commit history before pushing
- You want a linear, easy-to-read project history

**Pros:** Clean linear history, easier to read `git log`  
**Cons:** Rewrites commits — **never rebase a branch others are working on**

---

### The Golden Rule of Rebasing

> **Never rebase a shared/public branch.**  
> Once you've pushed a branch and someone else has based work on it, rebasing will rewrite history and cause conflicts for everyone. Only rebase your own local or unshared branches.

---

### Quick Decision Guide

| Situation | Use |
|---|---|
| Merging a finished feature into `main` | `merge` |
| Updating your feature branch with latest `main` | `rebase` |
| Cleaning up commits before a pull request | `rebase -i` (interactive) |
| Someone else has based work on your branch | `merge` |

---

### Interactive Rebase — clean up commits before a PR

Interactive rebase lets you squash, reword, reorder, or drop commits before sharing your work.

```
git rebase -i HEAD~3   # interactively edit the last 3 commits
```

In the editor that opens, change `pick` to one of:

| Command | What it does |
|---|---|
| `pick` | Keep the commit as-is |
| `reword` | Keep the commit but edit the message |
| `squash` | Combine with the previous commit |
| `fixup` | Like squash but discard the commit message |
| `drop` | Remove the commit entirely |

---

## Commit Best Practices

Good commit messages make reviewing and debugging much easier.

#### format
```
<type>: <short summary in present tense>

<optional longer description>
```

Common types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`

#### examples
```
feat: add user login page
fix: correct null check in payment handler
docs: update README with setup instructions
```

#### rules of thumb
- Keep the summary under 72 characters
- Use present tense: "add feature" not "added feature"
- Commit one logical change at a time — don't bundle unrelated changes
- Don't commit broken code to a shared branch

---

## Branch Naming

Consistent branch names make it easy to understand what's being worked on at a glance.

#### common patterns
```
feature/<short-description>     # new feature
fix/<short-description>         # bug fix
chore/<short-description>       # maintenance, dependency updates
hotfix/<short-description>      # urgent production fix
```

#### examples
```
feature/user-login
fix/null-pointer-checkout
chore/upgrade-node-18
hotfix/payment-crash
```

---

## Working with Remotes

#### see all configured remotes
`git remote -v`

#### fetch latest changes without merging
`git fetch origin`

#### pull and rebase instead of merge (keeps history clean)
`git pull --rebase origin main`

#### push a new local branch to remote for the first time
`git push -u origin <branch-name>`

#### sync a fork with the original upstream repo
```
git remote add upstream <original-repo-url>   # one-time setup
git fetch upstream
git rebase upstream/main
```

---

## Undoing Things

#### undo the last commit but keep the changes staged
`git reset --soft HEAD~1`

#### undo the last commit and unstage the changes (changes stay in working directory)
`git reset HEAD~1`

#### undo the last commit and throw away the changes entirely
`git reset --hard HEAD~1`

#### discard all uncommitted changes in the working directory (cannot be undone)
`git checkout -- .`

#### undo a specific file back to its last committed state
`git checkout -- <filename>`

#### safely undo a commit that has already been pushed (creates a new revert commit)
`git revert <commit>`

> **Tip:** Prefer `git revert` over `git reset` for commits that have already been pushed to a shared branch. `reset` rewrites history; `revert` adds a new commit that undoes the change safely.

---

## Useful Log Tricks

#### see a compact one-line log
`git log --oneline`

#### see a visual graph of branches and merges
`git log --oneline --graph --all`

#### search commits by message keyword
`git log --grep="<keyword>"`

#### see what changed in the last N commits
`git log -p -<N>`

#### see who changed each line of a file
`git blame <filename>`

---

## .gitignore Tips

A `.gitignore` file tells Git which files not to track. Put it at the root of your repo.

#### common things to ignore
```
# dependencies
node_modules/

# build output
dist/
build/

# environment files (never commit secrets!)
.env
.env.local

# OS files
.DS_Store
Thumbs.db

# editor config
.vscode/
.idea/
```

#### check why a file is being ignored
`git check-ignore -v <filename>`

#### force-add a file that is being ignored (use sparingly)
`git add -f <filename>`

> **Never commit `.env` files or secrets.** If you accidentally do, rotate the secrets immediately — simply deleting the file in a later commit is not enough because the secret remains in the Git history.
