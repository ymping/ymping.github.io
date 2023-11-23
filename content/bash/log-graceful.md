+++
title = 'Bash最佳实践：日志优雅打印'
date = 2023-11-11T11:55:00+08:00
keywords = ['bash', 'log', '日志']
tags = ['linux', 'bash']
draft = false
+++

## 摘要

在 Bash 脚本开发中，日志打印是重要的一环，尤其在处理大型或复杂脚本时。
本文探讨使用函数进行日志打印的最佳实践，以提高脚本的可扩展性和可维护性。
通过引入日志打印函数，我们不仅能够轻松添加时间戳和日志级别等信息，还能更灵活地控制日志输出的目标，例如将日志信息输出到文件或同时输出到标准输出。
本文演示了如何创建一个简单而强大的日志打印函数，如何通过环境变量自定义日志输出级别等示例。

## 背景

在 Bash 脚本编写的过程中，我们都习惯于使用 `echo` 命令来打印日志，这在小型脚本中足够的简单高效。
但当脚本逐渐变得复杂，功能不断扩展时，直接采用 `echo` 的方式可能显得力不从心。比如以下场景：

1. 需要给打印的日志统一加上时间戳，日志级别等信息；
2. 需要把打印的日志输出到指定文件，或同时输出 **stdout**。

直接使用 `echo` 命令来打印日志也可以实现上述两个场景的需求，但非常不利于后期的脚本维护。

以一个启动脚本为例，假设需要在初始化和环境检查时会打印大量信息。用户可能希望仅保留错误信息以便快速排查问题，要实现该需求的话，
需要人工甄别哪些 `echo` 的信息是错误信息以保留其内容的打印，这个甄别过程是相当繁琐和易出错的。
这就是我们引入日志打印函数的背景所在。

## 解决方案

在这一部分，我们将探讨如何通过引入函数来打印脚本的日志输出。
这包括了添加时间戳、主机名等信息，以及控制日志输出级别的实用技巧。

### 基本日志打印函数

可以通过日志打印函数来解决直接使用 `echo` 打印日志带来的不灵活和维护差问题。
以下是一个最基本的日志打印函数示例 **demo.sh**：

```shell
#!/usr/bin/env bash

info() {
    echo -e "$(date '+%F %T') info - $*"
}

warn() {
    echo -e "$(date '+%F %T') warn - $*"
}

error() {
    echo -e "$(date '+%F %T') error - $*"
}

info "this is info log"
warn "this is warn log"
error "this is error log"
```

给 **demo.sh** 添加可执行权限，执行脚本：

```text
bash-5.2$ ./demo.sh 
2023-11-12 09:29:26 info - this is info log
2023-11-12 09:29:26 warn - this is warn log
2023-11-12 09:29:26 error - this is error log
bash-5.2$
```

### 强化功能的日志打印函数

示例 **demo.sh**：

```shell
#!/usr/bin/env bash

HOSTNAME=$(hostname)

# LOG_LEVEL(default 1):    info: 1    warn: 2    error: 3
LOG_LEVEL=${LOG_LEVEL:-1}

# default log to stdout
LOG_TO_STDOUT=${LOG_TO_STDOUT:-1}

# log to file is LOG_FILE is set
LOG_TO_FILE=${LOG_TO_FILE:-1}
if [[ -z ${LOG_FILE} ]]; then
    LOG_TO_FILE=0
fi

if [[ -t 1 ]]; then
    COLOR_END="\033[0m"
    COLOR_YELLOW="\033[34m"
    COLOR_GREEN="\033[32m"
    COLOR_RED="\033[31m"
else
    COLOR_END=""
    COLOR_YELLOW=""
    COLOR_GREEN=""
    COLOR_RED=""
fi

info() {
    if [[ ${LOG_LEVEL} -gt 1 ]]; then
        return 0
    fi

    if [[ ${LOG_TO_STDOUT} -eq 1 ]]; then
        echo -e "$(date '+%F %T') ${HOSTNAME} ${COLOR_GREEN}info  - $*${COLOR_END}"
    fi

    if [[ ${LOG_TO_FILE} -eq 1 ]]; then
        echo -e "$(date '+%F %T') ${HOSTNAME} info  - $*" >>${LOG_FILE}
    fi
}

warn() {
    if [[ ${LOG_LEVEL} -gt 2 ]]; then
        return 0
    fi

    if [[ ${LOG_TO_STDOUT} -eq 1 ]]; then
        echo -e "$(date '+%F %T') ${HOSTNAME} ${COLOR_YELLOW}warn - $*${COLOR_END}"
    fi

    if [[ ${LOG_TO_FILE} -eq 1 ]]; then
        echo -e "$(date '+%F %T') ${HOSTNAME} warn - $*" >>${LOG_FILE}
    fi
}

error() {
    if [[ ${LOG_LEVEL} -gt 3 ]]; then
        return 0
    fi

    if [[ ${LOG_TO_STDOUT} -eq 1 ]]; then
        echo -e "$(date '+%F %T') ${HOSTNAME} ${COLOR_RED}error - $*${COLOR_END}"
    fi

    if [[ ${LOG_TO_FILE} -eq 1 ]]; then
        echo -e "$(date '+%F %T') ${HOSTNAME} error - $*" >>${LOG_FILE}
    fi
}

info "this is info log"
warn "this is warn log"
error "this is error log"
```

该示例实现了以下功能：

1. 打印日志中默认含有主机名
2. 可以通过环境变量设置 `LOG_LEVEL`，不设置的情况下默认打印所有级别的日志
3. 支持通过 **tty** 打印的不同级别的日志带不同的颜色
4. 通过环境变量设置 `LOG_TO_STDOUT` 决定日志是否输出到 **tty**
5. 通过环境变量设置 `LOG_FILE` 决定日志是否输出到文件

```text
bash-5.2$ 
bash-5.2$ export LOG_LEVEL=2
bash-5.2$ export LOG_FILE=./demo.log
bash-5.2$ export LOG_TO_STDOUT=0
bash-5.2$ 
bash-5.2$ ./demo.sh 
bash-5.2$ 
bash-5.2$ cat ./demo.log 
2023-11-14 19:12:04 admindeMacBook-Pro.local warn - this is warn log
2023-11-14 19:12:04 admindeMacBook-Pro.local error - this is error log
bash-5.2$
```

对于有多个脚本文件的情况下，不需要在每个 bash 脚本文件中都申明日志打印函数。
只需要把日志打印功能相关的脚本放在一个文件，如 `log.sh`，其它脚本直接 `source log.sh` 即可。

## 总结

在脚本开发中，打印日志使用 `echo` 命令适用于代码较少的情况，但当脚本变得庞大而复杂时，采用函数封装日志打印功能会更为明智。
这不仅能提高代码的可读性，还使得脚本后期的扩展和维护变得更加轻松。
希望本文不仅为你提供了实用的日志打印函数示例，同时也能激发更多的想法和创新，后续能根据实际需求封装日志打印函数，助力更好地完成脚本开发工作。
