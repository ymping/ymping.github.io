+++
title = 'Bash最佳实践：脱缰的野马'
date = 2023-10-29T15:43:58+08:00
keywords = ['bash', 'error', 'exit', 'errexit', 'pipefail', '错误停止']
tags = ['linux', 'bash']
draft = false
+++

## 摘要

默认情况下，Bash 脚本会在命令执行失败（返回非零的 **exit code**）或者引用未绑定变量时，并不会停止执行。
这可能会导致非预期的结果发生。
可以使用 `set -e` 或 `set -o errexit` 命令让 bash 解释器在命令执行失败时立即退出，
使用 `set -u` 或者 `set -o nounset` 在引用未绑定变量时立即退出。

在使用管道时，由于管道的 **exit code** 默认为管道最后一个命令的 **exit code**，这将导致管道中其它命令的 **exit code** 被掩盖，
使用 `set -o pipefail` 命令来避免管道中的错误被掩盖，即管道中的命令全部成功才会返回 **0** 的 **exit code**。

一般使用命令 `set -euo pipefail` 来同时启用这两个特性。

> 在脚本编程领域，是通过命令/脚本的 **exit code** （退出状态码）来判断命令/脚本的执行状态，
> 通常 **exit code** 为 **0** 表示命令/脚本成功执行，**非0** 表示命令/脚本执行失败，有错误发生。

---

## 默认行为

为测试 bash 在错误发生时的默认行为，创建一个示例脚本 **demo.sh**, 代码内容如下，由于路径 `/path/not` 不存在，
命令 `mkdir /path/not/exist` 会报错，在该行命令的前后都有相关打印，用以验证在命令报错后，脚本是否会继续执行。

```shell
#!/usr/bin/env bash

echo "before error occurs"

mkdir /path/not/exist
echo "error code: $?"

echo "after error occurs"
```

给 **demo.sh** 添加可执行权限后，执行脚本：

```text
bash-5.2$ ./demo.sh 
before error occurs
mkdir: /path/not: No such file or directory
error code: 1
after error occurs
bash-5.2$ 
```

可以打印的输出可以看到，命令 `mkdir /path/not/exist` 报错了，但是这行命令报错后，后面的命令也继续执行了，
打印了 `mkdir` 命令的 **exit code: 1** 和提示信息 **after error occurs**。

**bash 脚本这种发生错误仍继续执行脚本的行为的，会导致后续命令执行所依赖的前提条件不存在，从而导致非预期的问题，
存在很大的安全风险。**
尤其是 bash 脚本本身的应用场景是服务器的运维自动化操作，在脚本代码质量不佳的情况下，该特性会使脚本像一匹脱僵野马一样，
冲破安全围栏，导致生产事故。

设想在数据库备份场景，该场景一般分为两个步骤：

1. 创建备份文件
2. 删除过期备份文件
3. 发送备份成功通知

假如某天 DBA 修改了数据库密码，但是忘记同步了运维人员，步骤 1 执行失败，但步骤 2 和 3 会正常执行（该脚本是一个简单的备份删除脚本，
没有错误处理逻辑），可以预见的情形是，你一直都认为备份是成功执行的，当某一天需要备份文件还原的时候，一个备份文件也找不到了！

---

下面验证在 bash 脚本中引用未绑定变量时，脚本仍继续执行。修改 **demo.sh** 为

```shell
#!/usr/bin/env bash

echo "the command is: rm -f ${parent_dir}/${sub_dir}"
```

执行脚本：

```text
bash-5.2$ ./demo.sh 
the command is: rm -f /
bash-5.2$
```

脚本在引用未绑定变量时仍继续执行，这可能会导致比上个场景更严重的后果。
如脚本中有一个删除命令 `rm -rf "${parent_dir}/${sub_dir}"`，在 `sub_dir` 变量未绑定时，
这行命令被展开为 `rm -rf "${parent_dir}/"`，这会导致其它数据被误删除。如果 `parent_dir`
变量也未绑定，此时的命令被展开为 `rm -rf /`，这将会是一场灾难！

## 解决方案

根据 [bash 文档](https://www.gnu.org/software/bash/manual/bash.html) 中有关
[set 命令](https://www.gnu.org/software/bash/manual/bash.html#The-Set-Builtin) `-e` 选项的部分：

> Exit immediately if a pipeline (see Pipelines), which may consist of a single simple
> command (see Simple Commands), a list (see Lists of Commands), or a compound
> command (see Compound Commands) returns a non-zero status.

[set 命令](https://www.gnu.org/software/bash/manual/bash.html#The-Set-Builtin) `-u` 选项的部分：

> Treat unset variables and parameters other than the special parameters ‘@’ or ‘*’, or
> array variables subscripted with ‘@’ or ‘*’, as an error when performing parameter
> expansion. An error message will be written to the standard error, and a non-interactive shell will exit.

### 验证 `set -e`

为 **demo.sh** 增加 `set -e` 验证命令执行失败时会停止：

```shell
#!/usr/bin/env bash

set -e

echo "before error occurs"

mkdir /path/not/exist
echo "error code: $?"

echo "after error occurs"
```

执行 **demo.sh** 脚本：

```text
bash-5.2$ ./demo.sh 
before error occurs
mkdir: /path/not: No such file or directory
bash-5.2$
```

可以看到脚本在 `mkdir` 命令报错后，停止执行了，没有打印后续的 **exit code** 和提示信息。

当然，也可以不改 **demo.sh** 脚本，直接在命令行把 `-e` 参数传递给 bash 解释器。
执行命令 `bash -e ./demo.sh` 与在 **demo.sh** 增加 `set -e` 命令有相关的效果。

```text
bash-5.2$ bash -e ./demo.sh 
before error occurs
mkdir: /path/not: No such file or directory
bash-5.2$ 
```

### 验证 `set -u`

为 **demo.sh** 增加 `set -u` 验证引用未绑定变量时会停止：

```shell
#!/usr/bin/env bash

set -u

echo "the command is: rm -f ${parent_dir}/${sub_dir}"
echo "nothing will output for reference unbound variable"
```

执行脚本：

```text
bash-5.2$ ./demo.sh 
./demo.sh: 行 5: parent_dir: 未绑定的变量
bash-5.2$ 
```

可以看到 bash 在执行第一个 `echo` 命令时因为引用未绑定的变量退出了，第二个 `echo` 命令没有执行。

### `-eu` 选项的作用域

命令 `shopt -s inherit_errexit` 可以使 subshell 继承 `-e` 选项，具体请见
[文档](https://www.gnu.org/software/bash/manual/html_node/The-Shopt-Builtin.html)中关于 inherit_errexit 的部分。

> If set, command substitution inherits the value of the errexit option, instead of
> unsetting it in the subshell environment. This option is enabled when POSIX mode is enabled.

| case                     | -e                                        | -u       |
|--------------------------|-------------------------------------------|----------|
| $(command1;command2)     | not work (unless inherit_errexit opt set) | work     |
| $(bash_script_file.sh)   | not work                                  | not work |
| exec bash_script_file.sh | not work                                  | not work |

## 错误处理

在使用 `-e` 选项后，脚本在遇到执行错误时，就直接停止退出了，如果想要脚本继续执行，并进行错误处理，
类型于高级编辑语言中的 **try catch** 机制，可以参考下面三种方式。

在 [set 命令](https://www.gnu.org/software/bash/manual/bash.html#The-Set-Builtin) `-e` 选项的部分，有明确写明在某些指令或者代码片段中，bash
解释器遇到错误不会退出。

> The shell does not exit if the command that fails is part of the command list immediately
> following a while or until keyword, part of the test in an if statement, part of any command
> executed in a && or || list except the command following the final && or ||, any command in
> a pipeline but the last, or if the command’s return status is being inverted with !.

故此，对于 **demo.sh** 脚本，我们有收下方式来做错误处理。

### 方式一: `||`

使用 **或** 运算符 `||` 连接两个命令，形如 `cmd1 || cmd2`，bash 解释器在处理时会将该两个命令视为一个整体，
该整体的退出状态码是 **cmd2** 命令的退出状态码，**cmd1** 命令的退出状态码被掩盖了。
故此，只要 **cmd2** 命令的退出状态码为 **0**，bash 解释器就会认为没有错误发生。

以下示例在错误发生时仅打印简单的提示，脚本继续执行，适用于可以忽略命令执行失败的情况。
如果不关心命令的执行情况，还可以将语句替换为 `mkdir /path/not/exist || true`。

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

使用 `if command` 来写 **demo.sh** 的错误处理逻辑：

关于使用 `if` 调用函数的副作用，请查阅[文档](/bash/errexit-within-if)。

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

如果未设置 `-e` 选项，仍可以通过判断 **exit code** 的方式来进行错误处理。

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

## pipefail

设置了 `set -e`，在使用管道（pipeline）时，管道中的命令报错了，脚本还会继续执行吗？
来看一下示例 **demo.sh**：

```shell
#!/usr/bin/env bash

set -e

grep some-string /non/existent/file | awk 'BEGIN{print "awk was executed"}'
echo "error code: $?"

echo "after error occurs"
```

执行 **demo.sh** 脚本：

```text
bash-5.2$ ./demo.sh 
grep: /non/existent/file: No such file or directory
awk was executed
error code: 0
after error occurs
bash-5.2$ 
```

从脚本执行的输出可以看到，管道中的命令执行错误不会导致脚本退出，`awk` 命令正确地执行，
**error code** 拿到的是 `awk` 的退出状态码 **0**，`grep` 命令的退出状态码被掩盖了！
原因是脚本是否继续执行是由退出状态码来决定的，而管道的退出状态码默认是管道中最后一个命令的退出状态码，
管道中其它命令的退出状态码都被掩盖了。
所以当 `set -e` 时，除了管道中的最后一个命令外，管道中的命令失败不会导致脚本退出。

对于这种情况，bash 中使用 `set -o pipefail` 来避免管道中发生的错误被掩盖。
在 [set 命令](https://www.gnu.org/software/bash/manual/bash.html#The-Set-Builtin) 中关于 **pipefail** 的描述。

> If set, the return value of a pipeline is the value of the last (rightmost) command
> to exit with a non-zero status, or zero if all commands in the pipeline exit
> successfully. This option is disabled by default.

故示例 **demo.sh** 可以修改为：

```shell
#!/usr/bin/env bash

set -e
set -o pipefail

grep some-string /non/existent/file | awk 'BEGIN{print "awk was executed"}'
echo "error code: $?"

echo "after error occurs"
```

执行 **demo.sh** 脚本：

```text
bash-5.2$ ./demo.sh 
awk was executed
grep: /non/existent/file: No such file or directory
bash-5.2$ 
```

通过脚本的输出可以看到，`awk` 命令被执行，但因为 `grep` 命令报错，bash 解释器直接退出了，在管道之后的脚本没有被继续执行。

## 总结

```shell
set -o errexit   # abort on nonzero exitstatus
set -o nounset   # abort on unbound variable
set -o pipefail  # don't hide errors within pipes
```

等效于 `set -euo pipefail`，由于脚本中**错误则停止**（exit immediately if command returns a non-zero status）不是默认的行为，
强烈建议在脚本中使用 `set -euo pipefail` 命令启用**错误则停止**功能，可以有效规避脚本中命令的非预期执行带来的风险，为脱僵的野马套上缰绳。
