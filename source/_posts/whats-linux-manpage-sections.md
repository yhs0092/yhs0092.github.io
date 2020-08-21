---
title: man page section 的打开方式
tags: [ Linux, 运维能力 ]
keywords: [ Linux, 运维能力 ]
date: 2020-04-09 20:39:21
categories:
- 运维能力
index_img:
banner_img:
---
你知道 man page 后面的括号和数字是什么意思吗？
<!-- more -->

使用 Linux 的时候免不了会忘记某些命令的用法，这时候， man page ，或者说 `man` 命令就能帮助我们快速查找命令的正确使用姿势。但有时候，我们在 man page 里能看到命令后面带着括号和数字的内容，像是某种引用一样。比如，我们输入 `man sleep`，在 man page 开头能看到这样的内容：
```Text
SLEEP(1)                         User Commands                        SLEEP(1)

NAME
       sleep - delay for a specified amount of time
```
而在结尾的`SEE ALSO`部分，又能看到这样的内容：
```Text
SEE ALSO
       sleep(3)
```

那么， `sleep(1)` 和 `sleep(3)` 有什么区别呢？而且，似乎我现在看的是 `sleep(1)` ，如果我想看 `sleep(3)`，我该怎么打开它呢？

其实我们查阅一下 `man` 命令自己的 man page 就能看到相关说明，括号加数字表示的是 man page 的 "section" ，不同 section 代表的含义如下：
```Text
       1   Executable programs or shell commands
       2   System calls (functions provided by the kernel)
       3   Library calls (functions within program libraries)
       4   Special files (usually found in /dev)
       5   File formats and conventions eg /etc/passwd
       6   Games
       7   Miscellaneous  (including  macro  packages  and  conventions), e.g.
           man(7), groff(7)
       8   System administration commands (usually only for root)
       9   Kernel routines [Non standard]
```

所以一般我们查 Linux shell 命令时应该看的是 section 1 部分。当我们使用`man`命令时，系统会根据`1 n l 8 3 2 3posix 3pm 3perl 3am 5 4 9 6 7`的默认优先级顺序返回一个匹配上的 man page section 展示出来。

如果要看其他 section 的 man page ，打开方式有两种：
- `man <section> <cmd>`，例如 `man 3 sleep`
- `man <cmd>.<section>`，例如 `man sleep.3`

