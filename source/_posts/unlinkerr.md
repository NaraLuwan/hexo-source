---
title: Unlink of file 'xx' failed. Should I try again? (y/n) 解决办法
date: 2017-08-02 17:16:28
tags: Git
categories: 
- Java
- Web
---

```
Unlink of file 'xx' failed. Should I try again? (y/n)
```
原因：一般遇到这个错输入y/n都不能解决问题，出现这个问题的原因可能是其他程序正在操作git目录下面的文件，导致git无法关联该文件。

比如用dos命令窗或者git bash打开当前分支的文件，不关闭的情况下再切换到其他分支，等再切回来的时候就会报这个错，怎么确认(y)都无济于事。

解决方法：这里的 Unlink of file 'xx' failed. 这个 xx 就是被占用的文件，只要找到占用这个文件的程序或进程，并把它kill掉就可以了。

如果实在找不到就直接重启电脑...<!-- more -->

<hr />