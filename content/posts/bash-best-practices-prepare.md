+++
title = 'Bash最佳实践：工欲善其事，必先利其器'
date = 2023-09-23T22:39:48+08:00
draft = false
+++

## 摘要

本文将介绍如何在两个流行的集成开发环境中，即 Visual Studio Code 和 JetBrains IDE ，安装开发 Bash 脚本所必须的插件，用于提供代码格式化，语法高亮等功能，以便在开发 Bash 脚本时获得更好的开发体验，提升开发效率。

## IDE 设置建议

1. 开启显示空白字符功能，用于显示 `空格` `制表符` 等空白字符；
2. 设置制表符为 4 个空格，并用空格代替制表符；
3. 设置显示文件的换行符类型，是 `LF` `CR` 还是 `CRLF`，bash 脚本的换行符需指定为 `LF`。

## Visual Studio Code 插件

建议安装 `Bash IDE` （提示语法高亮，代码补全，定义跳转等功能） 和 `shell-format` （提供代码格式化功能）插件。

## JetBrains IDE 插件

### shfmt

[shfmt](https://github.com/mvdan/sh) 插件提供代码格式化功能，在 JetBrains 的设置路径为 `Preferences | Editor | Code Style | Shell Script` ，在该页面可以看到安装的路径，如果没有安装，可以选择 `Download` 自动安装或者指定已经下载的 [shfmt](https://github.com/mvdan/sh/releases) 路径。

### shellcheck

[shellcheck](https://github.com/koalaman/shellcheck) 插件提供语法检查功能，在 JetBrains 的设置路径为 `Preferences | Editor | Inspections` ，在该页面可以看到安装的路径，如果没有安装，可以选择 `Download` 自动安装或者指定已经下载的 [shellcheck](https://github.com/koalaman/shellcheck/releases) 路径。


## 开发参考

1. 语法解释 [explainshell https://explainshell.com/](https://explainshell.com/)
2. 语法检查 [shellcheck https://github.com/koalaman/shellcheck](https://github.com/koalaman/shellcheck)
3. 代码格式化 [shfmt https://github.com/mvdan/sh](https://github.com/mvdan/sh)
4. 语法快速参考 [https://quickref.me/bash](https://quickref.me/bash)
5. Google Bash Styleguide [https://google.github.io/styleguide/shellguide.html](https://google.github.io/styleguide/shellguide.html)
