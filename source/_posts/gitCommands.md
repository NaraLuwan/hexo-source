---
title: git常用统计命令
date: 2017-02-20 19:33:00
tags: Git
categories: Git
---
上周要做个汇报PPT涉及到个人对项目贡献量，在网上搜集了些常用统计命令，总结如下：

## 统计代码提交量
git log --author="$(gitconfig--getuser.name)" --pretty=tformat: --numstat | gawk '{add += $1;subs += $2;loc += $1 - $2} END {printf "added lines:%s removed lines : %s total lines: %s\n",add,subs,loc}' -

## 统计提交量排名
git log --pretty='%aN' | sort | uniq-c | sort -k1 -n -r
<!-- more -->
## git log参数说明：

- --author：指定作者

- --stat：显示每次更新的文件修改统计信息，会列出具体文件列表

- --shortstat：统计每个commit的文件修改行数，包括添加、删除，但不列出文件列表 

- --numstat：统计每个commit的文件修改行数，包括添加、删除，并列出文件列表

- -p：选项展开显示每次提交的内容差异，用-2则仅显示最近的两次更新

- --name-only：仅在提交信息后显示已修改的文件清单

- --name-status：显示添加、修改、删除的文件清单

- --graph：显示ASCII图形表示的分支合并历史

- --pretty：使用其他格式显示历史提交信息。可用的选项包括oneline、short、full、fuller和format

- --pretty=tformat: 可以定制要显示的记录格式，这样的输出便于后期编程提取分析

<hr />

 