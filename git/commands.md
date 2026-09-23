# GIT Commands

> For stash commands see [stash.md](stash.md).

#### checkout a branch ignoring the changes on the current branch
`git checkout -f <branch-name>(try not to do this!)`

#### compare changes between two commits
`git diff <commit-a> <commit-b>`

#### checkout a specific commit
`git checkout <commit>`

#### display the log until the specified date by the specified author
`git log --until="<date>" --author="<author>`

#### check the differences between the remote branch and local branch
`git diff origin/develop…<branch-name>`

#### creates a new branch off of origin/develop branch
`git checkout -b <new branch> origin/develop`

#### check the commits to the current branch/<branch-name>
`git cherry -v origin/develop <branch-name>`

#### shows the changes made in that commit
`git show <commit>`

#### point the current branch to a new upstream branch (e.g. change upstream from "develop" to "release")
`git branch -u <upstream branch>`

#### point a branch to a new upstream branch
`git branch -u <upstream branch> <branch name>`

#### reflog gives the log of everything that has been done with git
`git reflog`

#### resets to the provided reflog id
`git reset --hard <reflog id>`

#### lists all the branches
`git branch -a`

#### checkout <filename> from the <remote-branch>
`git checkout origin/<remote-branch> <filename>`

#### diff <filename> from the <remote-branch>
`git diff origin/<remote-branch> <filename>`

#### check the upstream branches
`git branch -vv`

#### rebase with a different branch
`git rebase <branch-name>`

#### undo a git add
`git reset <filename>`

#### delete a local branch
`git branch -D <branch name>`

#### rename the current local branch
`git branch -m <new name>`

#### delete a remote branch
`git push origin --delete <remote branch name>`

#### revert a commit
`git revert <commit>`

#### see all the changes made to a certain file
`git log --follow -- <filename>`

#### cherry-pick a commit
`git cherry-pick <commit>`

#### open the diff tool
`git difftool`

#### merge a branch (resolve conflicts workflow)
```
git fetch
git checkout -t origin/<the-merged-to-branch>
git merge origin/<the-merged-from-branch>
# resolve conflicts
git add
git commit
# save in editor: Esc : wq
git push origin HEAD
```