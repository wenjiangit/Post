## Git常用命令

```shell
$ git commit                       //提交到本地仓库
```

```shell
$ git branch <branch-name>         //新建新分支
```

```shell
$ git checkout <branch-name>       //检出某个已存在的分支
```

```shell
$ git checkout -b <branch-name>    //新建分支并切换到新建的分支
```

```shell
$ git branch -D <branch-name>      //删除某个分支
```

```shell
$ git merge <branch-name>          //将某个分支的内容合并到当前分支
```

```shell
$ git rebase <branch-name>         //合并分支,并保持清晰线性的提交记录
```

```shell
$ git checkout <commit-hash>       //分离HEAD到指定commit位置
```

```shell
$ git branch -f <branch-name> HEAD         //将分支移动到指定HEAD位置
```

```shell
$ git branch -f <branch-name> HEAD~4       //在上面的基础上向前移动4步
```

```shell
$ git reset HEAD~n         //撤销本地最新的n次提交记录
```

```shell
$ git revert HEAD        //撤销当前最新提交,并同步到远程分支
```

```shell
$ git cherry-pick <commit-hash>...   //将指定的几次提交合并到当前分支
```

```shell
$ git rebase -i HEAD~n             //对当前提交的n步进行删除排序操作
```

```shell
$ git tag <tag-name> <commit-hash>  //基于某个提交记录设置一个tag(默认为当前提交)
```

```shell
$ git clone <remote-url>           //将远程仓库克隆到本地
```

```shell
$ git fetch                       //下载远程仓库内容,并不更新本地文件
```

```shell
$ git pull           ==   git fetch + git merge   //拉取远程代码并合并到当前分支
```

```shell
$ git pull --rebase  ==   git fetch + git rebase  //拉取远程代码并以rebase方式合并
```

```shell
$ git push origin :<branch-name>           //删除远程分支
```

```shell
$ git push origin <local-branch>:<remote-branch>  //将本地分支提交到远程分支上
```

```shell
$ git pull origin <remote-branch>:<local-branch>  //将远程分支合并到本地分支
```

