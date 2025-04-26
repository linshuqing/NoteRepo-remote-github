
官方资源：https://git-scm.com/book/zh/v2

查看远程仓库：
```bash
git remote
```
> - `git remote -v` 查看详细信息
> - `git remote add <远程主机名> <url>` 添加新的远程仓库（一个本地仓库可以对应多个远程）
> - `git remote rm <远程主机名>` 删除远程仓库
> - `git remote  rename <旧主机名> <新主机名>` 重命名远程仓库

查看状态：
```bash
git status
```
> - `git status -s` 简化输出

查看两个分支的区别：
```bash
git diff
```
> - `git diff` 查看工作区和暂存区的区别
> - `git diff <本地分支名> <远程主机名>/<远程分支名>` 查看本地分支和远程分支的区别，如果省略远程分支名，表示和远程分支同名。
> - `git diff --cached` 查看暂存区和上一次提交的区别
> - `git diff HEAD` 查看工作区和上一次提交的区别
> - `git diff --stat` 查看简化输出


查看配置
```bash
git config
```
> - `git config --list` 查看所有配置
> - 配置用户名和邮箱，`git config --global user.name "Name"` 和 `git config --global user.email "Email"`，这里的 "" 号内填写用户名和邮箱，可以是任意的，不一定是真实信息。


向指定的远程仓库推送：
```bash
git push <远程主机名> <本地分支名>:<远程分支名>
```
> 一些简写：
> - 如果本地分支和远程分支同名，可以省略 `:<远程分支名>`，写为 `git push <远程主机名> <本地分支名>`，如果远程主机中不存在这个分支，会自动创建。
> - 如果本地分支已经和远程分支关联，可以省略 `<本地分支名>:<远程分支名>`，写为 `git push <远程主机名>`。
>   - 可以使用 `git branch -vv` 查看本地分支和远程分支的关联情况。
>   - 可以使用 `git branch --set-upstream-to=<远程主机名>/<远程分支名> <本地分支名>` 设置本地分支和远程分支的关联。
> - 如果只有一个远程仓库，可以省略后面所有的，直接写为 `git push`。


拉取远程仓库的代码：
```bash
git fetch <远程主机名> <远程分支名>
```
> 这个命令只是将远程仓库的代码拉取到本地，不会合并到本地仓库（也就是不会修改本地目录下的文件），需要手动合并。

合并远程仓库的代码：
```bash
git merge <远程主机名>/<分支名>
```
> 这个命令会将远程仓库的代码合并到本地仓库。
> 注意：对于非默认远程分支，需要手动指定分支名。

使用 `git pull` 命令可以一次性拉取远程仓库的代码并合并到本地仓库：
```bash
git pull <远程主机名> <远程分支名>
```
> 注意：对于非默认远程分支，需要手动指定分支名。

查询版本历史：
```bash
git log
```
> 查看 git fetch 的取回信息：`git log -p FETCH_HEAD`

处理分支：
```bash
git branch
```
> 查看分支：`git branch`，可以显示所有的本地分支，包含当前分支（前面有 * 号）和其他分支。
> 查看本地分支 + 上一次提交信息：`git branch -v`
> 查看所有远程分支：`git branch -r`
> 查看本地和远程所有分支：`git branch -a`
> 删除分支：`git branch -d <分支名>`，删除本地分支
> 创建分支：`git branch <分支名>`，创建本地分支


切换分支：
```bash
git checkout <分支名>
```
> `git checkout -b <新的分支名>`：创建并切换到新的分支
> `git checkout -b <新的分支名> <commit>`：在指定的提交基础上创建并切换到新的分支。

暂存但是不 add 工作流程：
```bash
1. 暂存当前工作 git stash save "<描述信息>"
2. 切换到其他分支 git checkout <其他分支名>
3. 完成其他分支的工作，切换回原来的分支 git checkout <原来的分支名>
4. 恢复暂存的工作 git stash pop
```
> `git stash list` 查看所有的 stash
> `git stash apply <stash@{n}>` 恢复指定的 stash，不指定后面的 `<stash@{n}>`，默认恢复最近的 stash
> `git stash drop <stash@{n}>` 删除指定的 stash


取消文件的追踪：
1. 从暂存区中删除文件：`git rm --cached <文件名>`，这个命令只是取消文件的追踪，不会删除文件。
2. 从工作区中删除文件：`git rm <文件名>`，这个命令会删除文件。
3. 此时再在 `.gitignore` 文件中添加文件名，后续就不会再追踪这个文件了。

## QA
1. 什么是主机名、什么是分支名？
    - 主机名：仓库的名字，可以随便指定。
    - 分支名：仓库的分支名，就是 branch 的名字
    - 一个仓库可以有多个分支，

1. 关于 git merge，存在三种情况：
    - Fast-forward：直接合并，不会产生新的 commit，只是移动指针
    - Auto-merging：自动合并，会产生新的 commit（修改了不同的文件）
    - CONFLICT：冲突，需要手动解决（修改了同一个文件）

2. git pull = git fetch + git merge

3. 善用 stash。少用 rebase。
