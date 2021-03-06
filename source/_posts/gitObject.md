---
title: git 4种对象的理解
date: 2017-02-19 19:46:00
tags: Git
categories: Git
---
### Git中有四种基本对象类型，可以说Git的所有操作都是通过这四种对象完成的。
- blob对象。blob文件是一种二进制文件，当把一个文件add进暂存区时就生成一个blob对象，它包含了这个文件的所有数据，但不包括文件名字、路径、格式等信息，之后每次提交只要文件有变更都会生成一个新的blob对象，但原有的blob对象也会保存下来，也就是说只要内容相同的文件在Objects库中只存在一份，这也就是Git和其他版本控制系统的区别。<!-- more -->还要说明的是，每个blob对象的id是对文件内容进行hash得到的，这样在比较文件是否修改了的时候只用比较对应的blob对象名是否相同就可以了。
- tree对象。可以把它想象成文件目录，每个tree对象包含0个或多个tree对象和blob对象。
- commit对象。每次提交都会生成一个commit对象，而每个commit对象都对应一个tree对象，通过这个tree对象可以查看这次提交的所有信息。
- tags对象。tags对象主要是为了解决每次提交id太长不好记的问题，可以对每次提交标注一个tag。
顺便吐槽下,《Git版本控制管理》中文第二版真的翻译的一般...
 
<hr />