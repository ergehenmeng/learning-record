> 创建分支
* `git checkout -b 新分支名称` 相当于 
   * `git branch 分支名称` 创建新分支
   * `git checkout 分支名称` 切换到分支
* `git branch -r` 查看远程库分支
* `git push origin bendi_new_branch:remote_new_branch` (本地分支与远程分支同名) 将本地分支推送到远程分支
* 删除远程分支
   * 方法一 `git push origin  :new_branch` (推送一个空分支到远程分支)
   * 方法二 `git push origin --delete new_branch`
* `git status`查看分支状态 
* `git add 文件名(支持多个文件)` 提交代码
* `git commit -m "注释信息" ` 本地代码提交
* `git diff HEAD -- readme.txt` 当前最新已提交与本地readme.txt比较
* `git reflog ` 显示操作的命令日志
* `git log --pretty=oneline` 格式化显示日志
* `git reset --hard commitId` 回退到指定提交节点上
* `git checkout -- readme.txt` 撤销修改(暂存区回退到工作区) 其实是用版本库里的版本替换工作区的版本，无论工作区是修改还是删除，都可以“一键还原”。
* `git rm readme.txt` 删除文件
* `git branch -d branchName` 删除分支
* `git log --graph --abbrev-commit` 显示图形和简写commit
* `git merge 分支` 合并分支  合并分支时，如果可能，Git会用Fast forward模式，但这种模式下，删除分支后，会丢掉分支信息
* `git merge --no-ff -m "xxxx" dev`
* `git stash` 将工作区临时保存
* `git stash pop` 恢复工作区同时删除stash区的内容
* `git stash apply` 恢复工作区
* `git stash list` stash保存的列表
* `git cherry-pick commitId` 将修改的bug同步到其他分支上
* `git remote` 查询远程仓库
* `git branch --set-upstream bendi_branch-name origin/remote_branch-name` 本地分支与远程分支关联
* `git tag v1.2 commitId` 在指定的commitId上创建标签
* `git tag ` 查询tag
* `git tag -a tagName -m "备注"` 指定标签及信息
* `git push origin :refs/tags/<tagname>` 删除远程标签
* `git tag -d <tagname>`可以删除一个本地标签
* `git tag -d <tagname>`可以删除一个本地标签
* `git push origin --tags` 推送全部标签
* `git config --global alias.st status` 配置别名 git status == git st