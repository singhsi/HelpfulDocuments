# GIT Commands

#### checkout a branch ignoring the changes on the current branch
`git checkout -f <branch-name>(try not to do this!)`

#### compare changes between two commits
`git diff <commit-a> <commit-b>`

#### checkout a specific commit
`git checkout <commit>`

#### display the log until the specified date by the specified author
`git log --until=”<date>” --author=”<author>`

#### check the differences between the remote branch and local branch
`git diff origin/develop…<branch-name>`

#### creates a new branch off of origin/develop branch
 `git checkout -b <new branch>  origin/develop`

#### check the commits to the current branch/<branch-name>
`git cherry -v origin/develop <branch-name>`

#### shows the changes made in that commit
-git show <commit>

-git branch -u <upstream branch>
####  point the current branch to a new upstream branch.eg:say you want to change the upstream branch from “develop” to “release”
-git branch -u <upstream branch> <branch name>
#### point a branch to a new upstream branch.

-git reflog
#### reflog gives the log of everything that has been done with git.
-git reset --hard <reflog id>
#### resets to the provided reflog id.

-git branch -a
#### lists all the branches

-git checkout origin/<remote-branch> <filename>
#### checkout <filename> from the <remote-branch>

-git diff origin/<remote-branch> <filename>
#### diff <filename> from the <remote-branch>

-git branch -vv
#### to check the upstream branches

-git rebase <branch-name>
#### to rebase with a different branch

-git reset <filename>
-undo a git add

-git stash save "A message about the changes you are saving"
-stash current changes


-git stash apply 
- apply the last saved stash, it doesn't delete the stash

-git stash pop
-apply the last saved stash, and also deletes the stash

-git stash apply <stash index> 
-if we have more than one stash, and want to apply a specific stash

-git stash list
-to see the stash list

-git branch -D <branch name>    
-delete local branch

 -git stash clear 
-delete all saved stashes
  
-git stash drop <stash index> 
-delete the specified stash only

-git branch -m <new name>
-rename the current local branch

-git push origin --delete <remote branch name>
-delete a remote branch

-git revert <commit>
-revert commits 
-git log --follow -- <filename>
-see all the changes made to a certain file
-get cherry-pick <commit>
-pick the commit to be applied



-git difftool
-run gitmerge
git fetch 
git checkout -t origin/<the-merged-to-branch> 
git merge origin/<the-merged-from-branch>
merged the conflicts
git add
git commit
Esc; :; wq
git push origin HEAD or <the-merged-to-branch> 
 
 



