+++
title = 'Bash最佳实践：脱缰的野马'
date = 2023-10-29T15:43:58+08:00
keywords = 'bash, -e, error, exit, errexit, 错误停止'
draft = false
+++

## 摘要

默认情况下，Bash 脚本会在错误发生时继续执行。这意味着如果脚本中的某个命令失败（返回非零的`exit code`），脚本将继续执行后续的命令。
这可能会导致问题或不正确的结果，要想 Bash 脚本在发生错误时停止执行，可以使用 `set -e` 或 `set -o errexit` 命令，
它会在脚本的执行过程中，命令返回任何非零退出代码的情况下使脚本立即停止执行。

> 在脚本编程领域，一般是通过命令/脚本的 `exit code` （退出状态码）来判断命令/脚本的执行状态，
> 通常 `exit code` 为 `0` 表示命令/脚本成功执行，`非0` 表示命令/脚本执行失败，有错误发生。

## 默认行为

为测试 bash 在错误发生时的默认行为，创建一个示例脚本 `demo.sh`, 代码内容如下，由于路径 `/path/not` 不存在，
命令 `mkdir /path/not/exist` 会报错，在该行命令的前后都有相关打印，用以验证在命令报错后，脚本是否会继续执行。

```shell
#!/usr/bin/env bash

echo "before error occurs"

mkdir /path/not/exist
echo "error code: $?"

echo "after error occurs"
```

给 `demo.sh` 添加可执行权限后，执行脚本：

```text
bash-5.2$ ./demo.sh 
before error occurs
mkdir: /path/not: No such file or directory
error code: 1
after error occurs
bash-5.2$ 
```

可以打印的输出可以看到，命令 `mkdir /path/not/exist` 报错了，但是这行命令报错后，后面的命令也继续执行了，
打印了 `mkdir` 命令的 `exit code: 1` 和提示信息 `after error occurs`。

---

bash 脚本这种发生错误仍继续执行脚本的行为的，会导致后续命令执行所依赖的前提条件不存在，从而导致非预期的问题，存在很大的安全风险。
尤其是 bash 脚本本身的应用场景是服务器的运维自动化操作，在脚本代码质量不佳的情况下，该特性会使脚本像一匹脱僵野马一样，
冲破安全围栏，导致生产事故。

设想在数据库备份场景，该场景一般分为两个步骤：

1. 创建备份文件
2. 删除过期备份文件
3. 发送备份成功通知

假如某天 DBA 修改了数据库密码，但是忘记同步了运维人员，步骤 1 执行失败，但步骤 2 和 3 会正常执行（该脚本是一个简单的备份删除脚本，
没有错误处理逻辑），可以预见的情形是，你一直都认为备份是成功执行的，当某一天需要备份文件还原的时候，一个备份文件也找不到了！

## 解决方案

根据 [bash 文档](https://www.gnu.org/software/bash/manual/bash.html) 中有关
[set 命令](https://www.gnu.org/software/bash/manual/bash.html#The-Set-Builtin) `-e` 选项的部分。

> Exit immediately if a pipeline (see Pipelines), which may consist of a single simple
> command (see Simple Commands), a list (see Lists of Commands), or a compound
> command (see Compound Commands) returns a non-zero status.

改造 `demo.sh` 为：

```shell
#!/usr/bin/env bash

set -e

echo "before error occurs"

mkdir /path/not/exist
echo "error code: $?"

echo "after error occurs"
```

执行 `demo.sh` 脚本：

```text
bash-5.2$ ./demo.sh 
before error occurs
mkdir: /path/not: No such file or directory
bash-5.2$
```

可以看到脚本在 `mkdir` 命令报错后，停止执行了，没有打印后续的 `exit code` 和提示信息。

当然，也可以不改 `demo.sh` 脚本，直接在命令行把 `-e` 参数传递给 bash 解释器。
执行命令 `bash -e ./demo.sh` 与在 `demo.sh` 增加 `set -e` 命令有相关的效果。

```text
bash-5.2$ bash -e ./demo.sh 
before error occurs
mkdir: /path/not: No such file or directory
bash-5.2$ 
```

### `-e` 选项的作用域

`-e` 选项的作用域是单个 bash 进程，如果 bash 进程创建另外一个新的 bash 进程，新的 bash 进程不会继承 `-e` 选项。

比如在 `demo.sh` 脚本中设置了 `set -e`, `demo2.sh` 中没有，如果 `demo.sh` 脚本中调用了（或者 `exec demo2.sh`）`demo2.sh`
脚本。哪 `demo2.sh` 脚本在执行遇到错误时会停止吗？ 答案是不会，在执行时 `demo.sh` 和 `demo2.sh` 是两个 bash 进程。
如果是使用 `exec demo2.sh` 的方式调用的 `demo2.sh` 脚本，`exec` 创建的新的 bash 运行时不会有之前 bash 运行时的参数。

## 错误处理

在使用 `-e` 选项后，脚本在遇到执行错误时，就直接停止退出了，如果我们想针对该错误，做一些错误处理，
类型于高级编辑语言中的 `try...catch` 机制。

在 [set 命令](https://www.gnu.org/software/bash/manual/bash.html#The-Set-Builtin) `-e` 选项的部分，
如果设置了 `-e` 选项，有明确写明在某些命令或者代码片段中，bash 解释器遇到错误不会退出。`if` 是其中的一个命令。

> The shell does not exit if the command that fails is part of the command list immediately
> following a while or until keyword, part of the test in an if statement, part of any command
> executed in a && or || list except the command following the final && or ||, any command in
> a pipeline but the last, or if the command’s return status is being inverted with !.

故此，对于 `demo.sh` 脚本，我们有收下方式来做处理处理

### 方式一: `||`

使用 `||`，在错误时仅打印简单的提示，脚本继续执行，适用于可以忽略命令执行失败的情况。

```shell
#!/usr/bin/env bash

set -e

mkdir /path/not/exist || echo "failed to make directory!"

echo "after error occurs"
```

执行结果：

```text
bash-5.2$ ./demo.sh
mkdir: /path/not: No such file or directory
failed to make directory!
after error occurs
bash-5.2$ 
```

### 方式二: `if`

使用 `if` 关健字，相比于 `||` ，`if` 有更多的灵活性，可以适用于更复杂的情形，`if command` 的语法形式是

```shell
if command; then
    # when command exit code is zero
    other command1 
    other command2
    ...
    other commandN
else
    # when command exit code is NOT zero
    other command1
fi

```

使用 `if command` 来写 `demo.sh` 的错误处理逻辑：

```shell
#!/usr/bin/env bash

set -e

if mkdir /path/not/exist; then
    echo "directory created successfully"
else
    # do your error handle here
    echo "failed to create directory!"
fi

echo "after error occurs"
```

执行结果：

```text
bash-5.2$ ./demo.sh
mkdir: /path/not: No such file or directory
failed to create directory!
after error occurs
bash-5.2$ 
```

### 方式三: `$?`

如果未设置 `-e` 选项，仍可以通过判断 `exit code` 的方式来进行错误处理。

```shell
#!/usr/bin/env bash

mkdir /path/not/exist
if [[ $? -ne 0 ]]; then
    # do your error handle here
    echo "failed to create directory!"
fi

echo "after error occurs"
```

执行结果：

```text
bash-5.2$ ./demo.sh
mkdir: /path/not: No such file or directory
failed to create directory!
after error occurs
bash-5.2$ 
```

## 总结

脚本中**错误则停止**（exit immediately if command returns a non-zero status）参数是默认不开启的，
但强烈建议在脚本中使用 `set -e` 命令开启该参数，能有效规避脚本中命令的非预期执行带来的风险。为脱僵的野马套上缰绳。