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
另外一外方式就是使用参数替换，可以看到，使用字符串 `'\n'` 并没有发生替换，此时就必须要使用到 `$'\n'` 才可以完成换行符到逗号
`,` 的替换。

## Process Substitution

[Process Substitution](https://www.gnu.org/software/bash/manual/html_node/Process-Substitution.html)
是一种将命令的输入或输出模拟为文件的技术，使得需要文件参数的命令可以直接使用其他命令的输出。
其核心目的是解决管道 `|` 无法直接传递多个命令的输出作为文件参数的问题。

```bash
# 输入形式：<(command)
# 将 command 的输出作为“文件”传递给另一个命令。例如比较动态生成的输出：
diff <(grep "error" log1.txt) <(grep "error" log2.txt)

# 输出形式：>(command)
# 将当前命令的输出作为“文件”传递给 command 的输入。例如将 `date` 命令的输出同时计算 sha256sum, md5sum 和 base64 编码：
date > >(sha256sum) > >(md5sum) > >(base64)
```

## Stdout Redirect

bash 使用 `exec` 可以操作 `stdout` `stderr` 重定向，在 bash 脚本中，可以将整个脚本的输出重定向到文件。
关于 `exec` 的更多使用方法请参考 [linux exec与重定向](http://xstarcd.github.io/wiki/shell/exec_redirect.html)。

```bash
# 将整个脚本的 stdout 重定向到文件 stdout.txt，脚本 stdout 不再打印内容
exec > stdout.txt

# 将整个脚本的 stdout 备份到文件 stdout.txt，脚本 stdout 仍输出内容
exec > >(tee stdout.txt)
```
