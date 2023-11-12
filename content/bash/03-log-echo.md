+++
title = 'Bash最佳实践：日志打印指南'
date = 2023-11-11T11:55:00+08:00
keywords = ['bash', 'log', '日志']
tags = ['bash']
draft = false
+++

## 摘要

在 Bash 脚本开发中，日志打印是最要的一环，尤其在处理大型或复杂脚本时。
本文探讨了使用函数进行日志打印的最佳实践，以提高脚本的可扩展性和可维护性。
通过引入日志打印函数，我们不仅能够轻松添加时间戳和日志级别等信息，还能更灵活地控制日志输出的目标，例如将日志信息输出到文件或同时输出到标准输出。
本文演示了如何创建一个简单而强大的日志打印函数，并展示了如何通过环境变量自定义日志输出的级别。

## 背景

一般写 bash 脚本时，我们都习惯于使用 `echo` 命令来打印日志，这个方式用于简单，代码量小的脚本完全没有问题，
但对于功能复杂，代码量大的脚本，这个方式就有些力不从心了。 比如以下场景：

1. 需要给打印的日志统一加上时间戳，日志级别等信息；
2. 需要把打印的日志输出到指定文件，或同时输出 **stdout**。

直接使用 `echo` 命令来打印日志也可以实现上述两个场景的需求，但非常不利于后期的脚本维护。

比如有一个应用的启动脚本，启动前会做各种的环境检查和初始化工作，某天脚本的用户告诉你，这个脚本打印了太多的信息，
干扰了用户查看应用的日志打印，希望优化。对于这种情况，启动脚本的错误信息打印肯定是要保留的，用于在报错时快速定位问题。
但当你编辑脚本时，面对几十甚至上百的 `echo` 命令调用，还要看哪些 `echo` 命令打印的是错误信息，这时心里肯定是崩溃的。

## 解决方案

### 日志打印函数

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

### 功能更丰富的日志打印函数

示例 **demo.sh**：

```shell
#!/usr/bin/env bash

HOSTNAME=$(hostname)

# LOG_LEVEL(default 1):    info: 1    warn: 2    error: 3
LOG_LEVEL=${LOG_LEVEL:-1}

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
    [[ ${LOG_LEVEL} -gt 1 ]] && return 0

    echo -e "$(date '+%F %T') ${HOSTNAME} ${COLOR_GREEN}info  - $*${COLOR_END}"
}

warn() {
    [[ ${LOG_LEVEL} -gt 2 ]] && return 0

    echo -e "$(date '+%F %T') ${HOSTNAME} ${COLOR_YELLOW}warn - $*${COLOR_END}"
}

error() {
    [[ ${LOG_LEVEL} -gt 3 ]] && return 0

    echo -e "$(date '+%F %T') ${HOSTNAME} ${COLOR_RED}error - $*${COLOR_END}"
}

info "this is info log"
warn "this is warn log"
error "this is error log"
```

该示例有以下功能：

1. 打印日志中默认含有主机名
2. 可以通过环境变量设置 `LOG_LEVEL`，不设置的情况下默认打印所有级别的日志
3. 支持通过 **tty** 打印的不同级别的日志带不同的颜色

![03-log-echo-1.png](/static/images/bash/03-log-echo-1.png)

对于有多个脚本文件的情况下，不需要在每个 bash 脚本文件中都申明日志打印函数。
一般只需要把日志打包函数放在一个文件，如 `log.sh`，其它脚本直接 `source log.sh` 即可。

## 总结

代码量少的脚本可以直接使用 `echo` 命令打印日志，对于代码量大，功能复杂，有较多日志打印的脚本，
建议使用函数封装日志打印功能，以便于后期的脚本的扩展和维护。
希望本文能有抛砖引玉的效果，你完全能根据实际需求改写本文示例的日志打印函数。
