# Git Stash

#### stash current changes
`git stash save "A message about the changes you are saving"`

#### see the stash list
`git stash list`

#### apply the last saved stash (keeps the stash)
`git stash apply`

#### apply a specific stash when more than one stash exists
`git stash apply <stash index>`

#### apply the last saved stash and delete it
`git stash pop`

#### stash, create a new branch, and apply the stash to it (without removing the stash)
Use this when you have uncommitted changes and want to move them onto a fresh branch.
`git stash apply` keeps the stash intact so you can re-apply it to multiple branches if needed.
```
git stash save "my changes"        # stash current changes
git checkout -b <new-branch>       # create and switch to a new branch
git stash apply                    # apply the stash — stash is kept
```
To apply a specific stash instead of the latest:
```
git stash list                     # find the index, e.g. stash@{2}
git checkout -b <new-branch>
git stash apply stash@{2}
```

#### delete a specific stash only
`git stash drop <stash index>`

#### delete all saved stashes
`git stash clear`
