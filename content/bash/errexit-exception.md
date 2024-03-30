+++
title = 'Bash最佳实践：errexit 的例外'
date = 2023-12-10T20:05:00+08:00
keywords = ['bash', 'error', 'exit', 'errexit', 'if func']
tags = ['linux', 'bash']
draft = false
+++

## 摘要

介绍 bash 脚本 `errexit` 机制的例外情况和解决方案，如使用 `if` 调用函数时，`errexit` 机制在 `if` 上下文[^1]中不会生效。

[^1]: 此处的上下文仅指形如`if func1; then func2; else func3; fi`中的 func1 部分，不是整个 `if` 上下文。

## 背景

在之前的[文章](/bash/stop-when-error)中有介绍到 bash 脚本的 `errexit` 机制，讲到在 bash 脚本中开启 `errexit` 选项后
在遇到非零的返回状态码时，bash 脚本会停止执行并退出，同时也介绍了几种不会退出的特殊情况。

关于 `-e` 选项的描述，摘录GNU[文档](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html)如下：

> Exit immediately if a pipeline (see Pipelines), which may consist of a single simple command (see Simple Commands), a
> list (see Lists of Commands), or a compound command (see Compound Commands) returns a non-zero status. The shell does
> not exit if the command that fails is part of the command list immediately following a while or until keyword, part of
> the test in an if statement, part of any command executed in a && or || list except the command following the final &&
> or ||, any command in a pipeline but the last, or if the command’s return status is being inverted with !. If a
> compound command other than a subshell returns a non-zero status because a command failed while -e was being ignored,
> the shell does not exit. A trap on ERR, if set, is executed before the shell exits.
>
> This option applies to the shell environment and each subshell environment separately (see Command Execution
> Environment), and may cause subshells to exit before executing all the commands in the subshell.
>
> If a compound command or shell function executes in a context where -e is being ignored, none of the commands executed
> within the compound command or function body will be affected by the -e setting, even if -e is set and a command
> returns a failure status. If a compound command or shell function sets -e while executing in a context where -e is
> ignored, that setting will not have any effect until the compound command or the command containing the function call
> completes.

简单总结开启 `errexit` 机制后，命令失败但不退出的场景有：

1. 在 `while` 或 `until` 关键字后的命令
2. 在 `if` 语句中的条件测试语句部分
3. 在 `||` 或 `&&` 表达式的非结尾部分语句
4. 在管道 `|` 的非结尾部分语句
5. 使用的退出状态码使用 `!` 反转
6. subshell 中（可以使用命令 `shopt -s inherit_errexit` 避免 subshell 中不生效的情况）

在实践中，如果编写复杂脚本，必然大量使用函数封装，在调用函数时，常在 `if` 的条件测试语句中，通过对函数 `return` 值的判定来进行错误处理。
故本文主要探讨函数在使用 `if` (if func; then ...; fi)调用时，errexit 不生效的情况。
如示例`errexit-in-if.sh`：

```shell
#!/usr/bin/env bash

set -e

foo() {
    echo "foo running"
    return 99
}

bar() {
    foo
    # If not in `if` context, due to foo return non-zero exit code,
    # bar will abort execution when errexit option is set.
    # Therefore, the echo command is never executed.
    echo "call foo success"
}

if bar; then
    echo "bar success"
else
    echo "bar fail, exit code: $?"
fi
```

执行脚本`errexit-in-if.sh`：

```text
bash-5.2$ bash  ./errexit-in-if.sh 
foo running
call foo success
bar success
bash-5.2$ 
```

可以看到，在 `bar` 函数通过 `if` 调用时，虽然 `foo` 函数返回了非零退出状态码 99，但脚本并没有停止执行！
使用 `if` 调用函数时，errexit 机制没有生效！

GUN mailing lists 中关于此问题的描述
[why does errexit exist in its current utterly useless form?](https://lists.gnu.org/archive/html/bug-bash/2012-12/msg00093.html)
和[答复](https://lists.gnu.org/archive/html/bug-bash/2012-12/msg00094.html)。

简单来说，历史原因，当时的设计没有考虑到函数调用这种情况，为了向前兼容，所以这个行为一直都是这样。

## 解决方案

bash 脚本中使用函数调用时，某些情况下，还是需要根据函数的执行状态做错误处理的，比如函数执行失败时进行一些回滚操作等。
下面介绍两种 errexit 不生效时的处理办法。

其它 errexit 不生效的情形，如在`while` `until` `||` `&&` `|`的上下文，可参考处理。

尝试过在函数中使用`trap 'return $?' ERR`来解决该问题，但测试后发现不行，查阅`trap`的 manual page 后（`man bash`），
发现`trap ... ERR`同 errexit 选项一样，在`if`等关键字的上下文中也不生效。

> The ERR trap is not executed if the failed command is part of the
> command list immediately following a while or until keyword, part of the test in an if statement,
> part of a command executed in a && or || list except the command following the final && or ||,
> any command in a pipeline but the last, or if the command's return value is being inverted
> using !. These are the same conditions obeyed by the errexit (-e) option.

### 显式 return

**不推荐使用此方式！！！** 仅作参考

既然 errexit 不生效，在非零退出状态码时不会停止执行，那就在发生错误时显式的 `return`。
修改`errexit-in-if.sh`如下：

```shell
#!/usr/bin/env bash

set -e

foo() {
    echo "foo running"
    echo "execute the false command to simulate command failure"
    false || return $?
}

bar() {
    foo || return $?
    echo "call foo success"
}

if bar; then
    echo "bar success"
else
    echo "bar fail, exit code: $?"
fi
```

执行脚本`errexit-in-if.sh`：

```text
bash-5.2$ bash ./errexit-in-if.sh 
foo running
execute the false command to simulate command failure
bar fail, exit code: 1
bash-5.2$ 
```

示例中，使用`||`操作符在调用`foo`函数失败时，return `foo`函数的退出状态码。
同时`foo`函数中的命令也要做错误处理！

这个方法的缺点很明显，其一是对函数体有入侵性，其二是开启 errexit 的初衷是脚本发生错误时退出，尽量避免显式的写错误退出逻辑，
减少编码的心智负担。因此显式的使用 `return $?` 和不设置 errexit 区别也不大，反正都需要显式地进行错误处理。

另外值得注意的是，如前文所述，在形如 `func_foo || return $?` 的语句中，errexit 机制在 func_foo 函数中也不生效😂！！！

### wrapper function

回想一下，使用 `if` 调用函数的初始是什么？无非就是在函数没有正确执行时，进行如回滚之类的错误处理操作。
既然使用 `if` 调用函数会使函数体执行时 errexit 机制不生效，那不用 `if` 调用函数不就可以了！

只需要拿到函数的退出状态码，就判定函数执行是否成功。但在开启 errexit 的情况下，不可以直接调用函数，否则函数中的命令出错时，
函数就直接退出了，无法执行错误处理逻辑。这种情况下，可以使用一个包装函数（如 callfunc）来解决问题。

按这个思路改造后的`errexit-in-if.sh`：

```shell
#!/usr/bin/env bash

set -e

callfunc() {
  # First disable errexit in the current shell so we can get the function's exit code
  set +e
  # Then we set it again inside a subshell and run the function
  ( set -e;  "$@" )
  # Save exit status code to variable EXIT_CODE
  EXIT_CODE=$?
  # And finally turn errexit back on in the current shell
  set -e
}

foo() {
    echo "foo running"
    return 99
}

bar() {
    foo
    echo "call foo success"
}

callfunc bar
if [[ ${EXIT_CODE} -eq 0 ]]; then
    echo "bar success"
else
    echo "bar fail, exit code: $?"
fi
```

执行脚本`errexit-in-if.sh`：

```text
bash-5.2$ bash  ./errexit-in-if.sh 
foo running
bar fail, exit code: 99
bash-5.2$ 
```

可以看到避免了在 `if` 上下文中调用函数，errexit 机制如预期生效。

这个方式的优点的对函数没有入侵性，不必在函数体代码中进行错误处理，完全按开启 errexit 选项时编写脚本，降低了编码时的心智负担。
缺点的函数是在 subshell 中调用的，该函数中对全局变量或环境变量的修改仅限于在本函数中(subshell)生效，对父进程的上下文不产生影响。

## 总结

借用 [GUN mailing lists 中的一段话](https://lists.gnu.org/archive/html/bug-bash/2012-12/msg00094.html)：

> Because once you are in a context that ignores 'set -e', the historical
> behavior is that there is no further way to turn it back on, for that
> entire body of code in the ignored context. That's how it was done 30
> years ago, before shell functions were really thought about, and we are
> stuck with that poor design decision.

[bash](https://www.gnu.org/software/bash/) 从 [1989年](https://en.wikipedia.org/wiki/Bash_(Unix_shell))
发布第一个版本距今（2023年）已经34年，存在历史包袱，在编码时应了解这些“特性”并规避它带来的这些反直觉的行为。

## 参考

1. [Bash Errexit Inconsistency](https://stratus3d.com/blog/2019/11/29/bash-errexit-inconsistency/)
2. [Why is bash errexit not behaving as expected in function calls?](https://stackoverflow.com/questions/19789102/why-is-bash-errexit-not-behaving-as-expected-in-function-calls)
