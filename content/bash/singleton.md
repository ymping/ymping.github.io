+++
title = 'Bash最佳实践：孤独行者'
date = 2023-11-18T20:34:00+08:00
keywords = ['bash', 'lock', 'filelock', '单实例', '防重']
tags = ['linux', 'bash']
draft = false
+++

## 摘要

在开发程序时，我们会尽量规避冲突 (conflicts) 和竞态条件 (race conditions) ，编写**bash**脚本时同理。
为脚本中增加防止重复运行机制，使脚本进程单实例的运行，可以有效避免冲突和竞态条件的发生。

本文介绍了在**bash**脚本中基于进程名匹配，**PID**查找，文件和文件锁的**防重**机制的实现。

## 背景

设想有一个备份脚本的功能是把家目录的内容 rsync 到远程服务器，该脚本通过 crontab 设置为每间隔一小时运行一次，
到目前为止一切工作正常直到某天用户拷贝了几个体积很大的 ISO 镜像到家目录，巨量的数量增加导致了 rsync 所需的同步时间远超一个小时。
在第一个实例同步完成前，cron 又启动了第二个同步实例，网络带宽是固定的，这进一步降低了单个实例的同步速率。之后 cron
继续调度了第三个实例，
接着是第四个，第五个......管理员在知晓该问题前，服务器已经不可用了。

## 防重机制实现

**防重**机制在**bash**脚本中，一般通过进程名匹配，**PID**查找，文件或者是文件锁的机制实现，下文给出了基于这四种实现的示例，解释和分析了实现的优劣势。

### 进程名匹配

新建示例程序`singleton-by-name.sh`，该示例演示如何使用进程名匹配的方式来实现防重机制。

```shell
#!/usr/bin/env bash

set -euo pipefail

SCRIPT_NAME=${0##*/}
if pgrep -f "${SCRIPT_NAME}" >/dev/null; then
    echo "another process is running now, exit"
    exit 0
fi

# Do something anything here
echo "I'm doing something here, sleep 30s"
sleep 30
```

执行效果：

```text
bash-5.2$ bash ./singleton-by-name.sh &
[1] 7523
bash-5.2$ I'm doing something here, sleep 30s

bash-5.2$ 
bash-5.2$ bash ./singleton-by-name.sh
another process is running now, exit
bash-5.2$ 
```

可以看到，在重复运行`singleton-by-name.sh`时，检测到了之前的脚本还在执行，当前执行的脚本主动退出了。
该实现的优点是代码简单易懂，缺点是当脚本名称的辨识度低时，容易出现误判。

### PID查找

新建示例程序`singleton-by-pid.sh`，该示例演示如何使用**PID**查找的方式来实现防重机制。

```shell
#!/usr/bin/env bash

set -euo pipefail

if pgrep -F /var/tmp/my-script-singleton.pid 1>/dev/null 2>&1; then
    echo "another process is running now, exit"
    exit 0
fi
echo $$ >/var/tmp/my-script-singleton.pid

# Do something anything here
echo "I'm doing something here, sleep 30s"
sleep 30
```

执行效果：

```text
bash-5.2$ bash ./singleton-by-pid.sh &
[1] 15877
bash-5.2$ I'm doing something here, sleep 30s

bash-5.2$ 
bash-5.2$ bash ./singleton-by-pid.sh
another process is running now, exit
bash-5.2$ 
```

该脚本每次运行前查找文件中的**PID**是否存在，如果不存在，则把自身的**PID**写入到文件，用于脚本下次执行时查找。
该实现依然很简单易懂，相比于进程名匹配的实现，降低了误判的概率。但由于**PID**由操作系统分配，仍然存在误判的可能性。

### 文件

新建示例程序`singleton-by-file.sh`，该示例演示如何通过判断一个文件存在与否来实现防重机制。

```shell
#!/usr/bin/env bash

set -euo pipefail

# Check is lock file exists, if not create it and set trap on exit
if {
    set -C
    echo 2>/dev/null $$ >/var/tmp/my-script-singleton.lock
}; then
    trap 'rm -f /var/tmp/my-script-singleton.lock' EXIT
else
    echo "another process is running now, exit"
    exit 0
fi

# Do something anything here
echo "I'm doing something here, sleep 30s"
sleep 30
```

执行效果：

```text
bash-5.2$ bash ./singleton-by-file.sh &
[1] 8449
bash-5.2$ I'm doing something here, sleep 30s

bash-5.2$ 
bash-5.2$ bash ./singleton-by-file.sh
another process is running now, exit
bash-5.2$
```

从脚本输出可以看到该脚本同样有防重效果。

在该脚本中，`set -C`（同`set -o noclobber` [官方文档](https://www.gnu.org/software/bash/manual/bash.html#The-Set-Builtin)）
表示在使用`>` `>&` `<>`重定向输出时，不覆盖已经存在的文件，同时使用命令组`{}`限定`set -C`仅在命令组范围内生效。
如果文件`/var/tmp/my-script-singleton.lock`不存在，则会创建该文件并把当前进程的PID `$$`写入到文件，
在命令组中的两命令成功执行后，`trap`命令会在脚本**退出**的时候，删除掉文件`/var/tmp/my-script-singleton.lock`。

建议使用`trap`命令移除锁文件，而非在脚本的最后直接用`rm`命令移除。原因是存在脚本执行过程中可能会出错退出，或脚本进程意外被终止的情况，
这将导致锁文件没有被正确清理，会一直存在，影响脚本的二次运行。

在重复运行脚本时，由于文件存在且设置了`set -C`，`echo`命令重定向覆盖文件会失败，此时`if`语句走到`else`分支，打印提示信息并退出。

此方式也存在误判的概率，如主机突然断电或**bash**解释器被强制终止（`kill -9`）导致锁文件没有被删除的情况，
且需要外部机制介入删除锁文件后才能恢复，不具备自愈能力。
另外用到了**bash**的命令组`{}`，`noclobber`和`trap`等特性，技术难度上有提升。

### 文件锁

`flock`是一个内核级别的系统调用，同时也是一个命令行工具，命令`flock`用于管理从脚本或命令行中发起的系统调用。

新建示例程序`singleton-by-flock.sh`，该示例演示如何通过文件锁来实现防重机制。

```shell
#!/usr/bin/env bash

set -euo pipefail

# Create a file located at /var/tmp/my-script-singleton.lock
# with file descriptor number 100
exec 100>/var/tmp/my-script-singleton.lock

# Try to acquire the file lock in a non-blocking manner,
# If the lock is successfully obtained, it will be held throughout the life cycle of the process.
# If unsuccessful, it means another process has acquired the lock on this file descriptor, 
# so output a message and exit.
# The default behavior of flock is to wait indefinitely to acquire a lock,
# -n, --nb, --nonblock: Fail rather than wait if the lock cannot be immediately acquired.
if ! flock -n 100; then
    echo "another process is running now, exit"
    exit 0
fi

# Do something anything here
echo "I'm doing something here, sleep 30s"
sleep 30
```

执行效果：

```text
bash-5.2$ bash ./singleton-by-flock.sh &
[1] 13405
bash-5.2$ I'm doing something here, sleep 30s

bash-5.2$ 
bash-5.2$ bash ./singleton-by-flock.sh
another process is running now, exit
bash-5.2$ 
```

该示例使用了文件锁机制，锁与持有锁的进程的生命周期相同，不会存在误判的情况。
另一个优点是命令`flock`有很多控制锁的选项，这为脚本提供了更多的灵活性，如：

1. 可以使用 `-u` 选项释放锁，如`flock -u 100`，可以在脚本执行的过程中释放锁，这可以控制锁的粒度，仅在访问会发生冲突的资源时加锁；
2. 命令`flock`在获取锁时，提供了超时参数 `-w seconds, --timeout`，如`flock -w 5 100`表示尝试在最多5秒的时间内获取到锁；
3. 默认获取的是排它锁 exclusive lock（也称作写锁 write lock），通过参数`-s, --shared`可以获取到共享锁 shared lock（也称作读锁
   read lock）；
4. 在获取到锁的同时还可以执行命令，如 `flock -w 5 echo "successfully obtained file lock"`。

结合上一个示例的内容，也可以使用`trap`
命令在脚本执行结束时移除掉锁文件，`trap 'rm -f /var/tmp/my-script-singleton.lock' EXIT`。

由于涉及到了`exec`操作文件描述符，文件锁等**bash**脚本中的高级特性，脚本的技术难度上有更多的提升，但从功能实现，脚本编写的角度看，
该实现是这四个示例中最简洁优雅的。

## 总结

比较以上四种实现方式，**PID**查找方案实现简单易懂，没有使用到**bash**的高级特性，推荐简单场景下使用。
文件锁方案虽使用到了锁，文件描述符等较复杂的高级特性，但其实现代码简洁，且提供了更多的控制选项和灵活性，强烈推荐使用。

总体而言，选择防重机制的实现应从实际的需求场景出发，开发者根据实际情况选择相应的方案。
