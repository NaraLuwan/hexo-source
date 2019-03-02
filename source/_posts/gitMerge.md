---
title: Git合并分支命令参数详解：git merge --ff
date: 2017-02-16 17:11:00
tags: Git
categories: Git
---
今天研究了一下git merge命令常用参数，并分别用简单的例子实验了一下，整理如下：

# 相关参数
- --ff  快速合并，这个是默认的参数。如果合并过程出现冲突，Git会显示出冲突并等待手动解决
- --ff-only  只有能快速合并的情况才合并。如果合并过程出现冲突，Git会自动abort此次merge
- --no-ff  不使用快速合并。会生成一次新的提交记录，这个记录只是标识在这里进行了一次merge操作（目前还没想到应用场景）
- --squash  压缩合并。将待合并的分支的内容压缩成一个新的提交合并进来<!-- more -->

# 应用场景
备注：C代表一次提交，合并时都是将dev分支合并到master。
## 第一种情况：master分支切出dev分支后没有新的提交，也就是说只有dev分支有更新，可以快速合并的情况：

    eg：master：C1 ← C2

　　　　　　　　　　 ↑

　　dev：　　　　   C3 ← C4

- 执行：git merge --ff dev

　　master：C1 ← C2 ← C3 ← C4

　　dev：C1 ← C2 ← C3 ←C4

　　结果：查看git log时master分支会看到dev分支上的所有提交，此时master和dev是一样的

- 执行：git merge --ff-only dev

　　结果同上。

- 执行：git merge --no-ff dev

　　git会提示让你输入此次合并的信息，然后生成一个特殊的commit。

　　master:C1 ← C2 ← C3 ← C4 ← C5 (Merge branch 'dev')

　　dev：C1 ← C2 ← C3 ←C4

　　结果：master分支会比dev分支多一条提交记录，也就是刚才输入犯人合并信息

- 执行：git merge --squash dev

　　master：C1 ← C2 ← C5 (Merge branch 'dev')

　　dev：C1 ← C2 ← C3 ←C4

　　结果：这里的C5其实是C3和C4的合并，如果只想合并dev的内容但是不需要它的提交记录就可以用这个参数

## 第二种情况，切出后master和dev分支均有更新，这种情况是最常见的。这里为了演示冲突，在C4和C5分别对一个文件进行了修改。

eg：master：C1 ← C2 ← C4

　　　　　　　　　　 ↑

　　dev：　　　　   C3 ← C5

- 执行：git merge --ff dev

　　这时Git会告诉你产生了冲突并列出冲突的文件，查看文件时会列出具体冲突内容，这时要先解决冲突（如果使用Intellij Idea或Eclipse等工具，可以直接选择use ours/theirs，ours代表被合并分支即master，theirs代表合并分支即dev），然后将这些修改的部分提交，再执行merge操作。

　　master：C1 ← C2 ← C3 ← C5 ← C4 ← C6 (解决冲突的那次提交)

　　dev：C1 ← C2 ← C3 ←C5

　　那么问题来了，Git是如何知道两个文件有冲突呢？

　　这里先说下结论，有时间再补一篇文章单独说明说明。

　　大家都知道在Git里每个文件都是一个blob对象，这里先不管合并时怎么找到同一个文件在两个分支上的blob（其实如果文件没有更新，在两个分支上是指向同一个blob），假设现在已经到了比较阶段了，Git会拿两个文件来逐行进行对比，但是判定是否修改是通过相邻行来确定的。也就是说文件a的第三行修改了，Git是通过第2行和第4行的对比来判定的，不信的可以先自己做实验验证。由于篇幅原因，这里不再赘述。

- 执行：git merge --ff-only dev

　　这时Git会检测到产生了冲突，所以提示：Not possible to fast-forward, aborting.    即取消这次merge操作。

- 执行：git merge --no-ff dev

　　结果同1，不过这里在解决了冲突执行commit操作后不用再进行merge操作了。如果再执行merge操作，它会提示：Already up-to-date.

- 执行：git merge --squash dev

　　master：C1 ← C2 ← C4 ← C6 (解决冲突的那次提交)

　　dev：C1 ← C2 ← C3 ←C5

　　这里解决了冲突并提交之后也不用再执行merge操作了。如果再执行merge操作会有两种情况：

　　a.刚才解决冲突时选用了master分支的修改，那么还是会提示有冲突需要解决。

　　b.刚才解决冲突时选用了dev分支的修改，那么会提示Already up-to-date。

　　对比发现，使用--squash参数时，如果有冲突，解决完冲突后只要两个分支不完全一样，再执行git merge --squash时还是会进行merge。但--no-ff就不会。

<hr />

　　

 