# Git

## notes

0. git command some-path and git command -- some-path are equivalent in all cases except when some-path could be interpreted as a commit reference. The most common case is a branch that has the same name as a file.

   For example, imagine your repository has a file named master in its root. Then git checkout master would checkout the branch master. But git checkout -- master would check out the file master of the current HEAD and replace the local master file with the version of that revision.

   git语法中--的意义是表示命令本身的结束，也就是说--后面的文字将被强制解释成文件或某个区。

   比如`git diff  -- cached readme.txt`表示cached会被解释成暂存区。

   `git diff HEAD -- readme.txt`中HEAD只能放在--前，不能放在--后，因为HEAD表示工作区顶的指针，也就是上面英文说的commit reference，如果放在--后面将被解释成文件或某个区，此时这个命令会被理解成比较工作区和缓存区中叫做HEAD和叫做readme.txt的两个文件的区别，由于git diff命令的性质，即使找不到叫做HEAD的文件，git diff命令也不做出反应。
   
1. `git log `查看commit的历史记录

   `git log --pretty=oneline`单行显示历史记录

   `git reflog`查看执行过的命令的历史记录

2. `git reset --hard HEAD^`使HEAD^的内容覆盖工作区和暂存区，HEAD^也可以换成commit id前几位。

3. `git diff`查看暂存区和工作区的区别

   `git diff HEAD -- readme.txt`查看HEAD即最近一次提交与工作区的区别。

   `git diff  -- cached readme.txt`查看暂存区与工作区的区别。

   （注意HEAD写在--前而cached写在后）

   ***git语法中--的意义是表示命令本身的结束，也就是说--后面的文字将被强制解释成文件或某个区。***

   ***比如`git diff  -- cached readme.txt`表示cached会被解释成暂存区。***

   ***`git diff HEAD -- readme.txt`中HEAD只能放在--前，不能放在--后，因为HEAD表示工作区顶的指针，如果放在--后面将被解释成文件或某个区，此时这个命令会被理解成比较工作区和缓存区中叫做HEAD和叫做readme.txt的两个文件的区别，由于git diff命令的性质，即使找不到叫做HEAD的文件，git diff命令也不做出反应。***

4. `git branch dev` 创建名为dev的分支

   `git checkout dev`切换到名为dev的分支
   
   `git checkout -b dev origin/bulabula`从远程叫做bulabula的分支创建名为dev的本地分支并切换到本地dev分支

## 我对pull request协作的理解

1. 管理员创建主仓库，称为上游仓库，创建dev分支。
2. 协作者fork上游仓库，进行开发，一个功能开发结束后，申请pull request合并到上游仓库的dev分支。
3. 每隔一段时间之后，当管理员处理完当前所有pull request时，通知协作者更新自己的仓库为上游仓库，这个操作可以降低管理员处理pull request的困难程度。（如果管理员处理了当前所有pull request，就可以保证协作者无需处理冲突（因为协作者的仓库已经没有意义），可以直接以上游仓库为准）。
4. 一段开发结束后，管理员将dev分支合并到master分支。并可以根据需要创建一个release version。