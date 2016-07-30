---
title: Shell拷贝文件
layout: post
toc: false
comments: true
date: 2015-08-06 16:17:20
tags:
---
工作中需要拷贝资源，但是不需要拷贝文件夹中的「**.svn**」隐藏文件。

解决方法如下：

```bash
rsync --exclude=.svn $SRC_DIR/ $DST_DIR/ -r
```

- -r, --recursive 对子目录以递归模式处理。
- -v, --verbose 详细模式输出。

在[另一篇blog](http://zohead.com/archives/linux-copy-directory-ignore-files/)中有给出2种方法，但是Mac中不支持**-regextype**。

- find: -regextype: unknown primary or operator
