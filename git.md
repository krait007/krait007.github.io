# git manual

- *demo* is branch name of your working repo. 
- *main* is the remote name of origin master repo(which you forked from).
- *origin* is the remote name of working repo.

## github PR with conflicts
```shell
#rebase
git pull --rebase main master

#edit conflict files and fix conflicts
vi bala_bala_confict_file

#add fixed files
git add confict_file

#Restart the rebasing process after having resolved a merge conflict
git rebase --continue

#push to server
git push -f origin demo
```

## git  merge  from  remote tag
```
git checkout -b newbranch

#fetch  tags
git fetch -t  main

# show latest parent
git merge-base HEAD tagname
#get same parent commitid

#count how many commit to merge
git log --oneline commitid...tagname | nl

# cherry-pick all
git cherry-pick commitid...tagname

#fix  conflict files and  git add  these files
git  add  xxx

#cherry-pick continue
git cherry-pick --continue

```
```
$ 
#git cherry-pick --continue
The previous cherry-pick is now empty, possibly due to conflict resolution.
If you wish to commit it anyway, use:

    git commit --allow-empty

and then use:

    git cherry-pick --continue

to resume cherry-picking the remaining commits.
If you wish to skip this commit, use:

    git cherry-pick --skip

git commit --allow-empty
git cherry-pick --continue
```
