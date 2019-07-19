---
title: 在bash高效匹配历史命令的技巧
date: 2019-07-19 20:58:28
tags:
 - bash


categories:
  - 技术

---


## 在bash里补全历史命令

在bash里，最常见的搜索历史命令的办法是`ctrl + r`，但是这个步骤太多，比较麻烦。

下面介绍一种非常快捷的补全方式。

执行：

```bash
curl -L http://hengyunabc.github.io/bash_completion_install.sh | sh
bind -f  ~/.inputrc
```

这样子，先输入部分命令，再按键盘的`Up/Down`就可以自动补全出历史命令了。


## 工作原理

实际上给`~/.inputrc`文件添加了下面的内容：

```bash
"\e[A": history-search-backward
"\e[B": history-search-forward
set show-all-if-ambiguous on
set completion-ignore-case on
```

前面两行自然是绑定了快捷键，后面两行是什么意思呢？

`show-all-if-ambiguous` 是指tab补全时，按一次tab就会把最长匹配的自动补全。具体参考 https://stackoverflow.com/a/42193784

`completion-ignore-case` 是指tab补全时，忽略大小写，这点也非常方便。

题外话，在arthas里也支持了`Up/Down`自动补全历史命令这个特性，所以在arthas里自动补全历史命令非常的方便。

