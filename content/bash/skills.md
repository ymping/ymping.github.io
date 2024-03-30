+++
title = 'Bash脚本示例二：巧夺天工'
date = 2024-03-30T10:47:00+08:00
keywords = ['bash', 'skills']
tags = ['linux', 'bash']
draft = false
+++

## 摘要

本文展示在 [bash](https://www.gnu.org/software/bash/manual/bash.html) 脚本中一些技巧和知识点。

## ANSI-C Quoting

`$'string'` 形式的字符序列被视为一种特殊类型的单引号。该序列扩展为字符串，并按照 ANSI C 标准的规定替换字符串中的反斜杠转义字符。
简单说就是让 bash 知道这个字符串是转义字符，如 `$'\n'` 表示一个换行符，而 `'\n'` 仅是换行符的字面量表示。

```bash
bash-5.2$ echo 'hello\nworld'
hello\nworld
bash-5.2$ 
bash-5.2$ echo -e 'hello\nworld'
hello
world
bash-5.2$ 
bash-5.2$ echo $'hello\nworld'
hello
world
bash-5.2$ 
```

在实践中的样例，如果一个字符串中含有一些转义字符，想要替换这些转义字符时，这个特性很好用。
如找出主机上面所有 `Nginx` 进程并使用逗号 `,` 分隔：

```bash
bash-5.2$ PIDS=$(pgrep nginx)
bash-5.2$ echo "${PIDS}"
42387
42472
42474
bash-5.2$ 
bash-5.2$ tr '\n' ',' <<<"${PIDS}"
42387,42472,42474,bash-5.2$ 
bash-5.2$ 
bash-5.2$ echo "${PIDS//'\n'/,}"
42387
42472
42474
bash-5.2$ 
bash-5.2$ echo "${PIDS//$'\n'/,}"
42387,42472,42474
bash-5.2$ 
```

使用命令 `tr` 可以完成转义字符的替换，但会涉及到一次外部命令调用。
另外一外方式就是使用参数替换，可以看到，使用字符串 `'\n'` 并没有发生替换，此时就必须要使用到 `$'\n'` 才可以完成换行符到逗号 `,` 的替换。

