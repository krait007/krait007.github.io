## github PR with conflicts
- *demo* is branch name of your working repo. 
- *main* is the remote name of origin master repo(which you forked from).
- *origin* is the remote name of working repo.

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
