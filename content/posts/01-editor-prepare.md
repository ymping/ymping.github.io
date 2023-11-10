+++
title = 'Bash最佳实践：工欲善其事，必先利其器'
date = 2023-09-23T22:39:48+08:00
keywords = ['bash', 'plugin', '插件']
tags = ['bash']
draft = false
+++

## 摘要

本文将介绍如何在两个流行的集成开发环境中，即 Visual Studio Code 和 JetBrains IDE ，安装开发 Bash
脚本所必须的插件，用于提供代码格式化，语法高亮等功能，以便在开发 Bash 脚本时获得更好的开发体验，提升开发效率。

## IDE 设置建议

1. 开启显示空白字符功能，用于显示 `空格` `制表符` 等空白字符；
2. 设置制表符为 4 个空格，并用空格代替制表符；
3. 设置显示文件的换行符类型，是 `LF` `CR` 还是 `CRLF`，bash 脚本的换行符需指定为 `LF`。

## Visual Studio Code 插件

### 插件

1. [Bash IDE](https://marketplace.visualstudio.com/items?itemName=mads-hartmann.bash-ide-vscode) 提示语法高亮，代码补全，定义跳转等功能
2. [shfmt](https://marketplace.visualstudio.com/items?itemName=mkhl.shfmt) 脚本格式化。

### 插件安装

正常可能通过 Visual Studio Code 的插件管理功能搜索自动安装。
如果设备网络受限，可以在网络正常的设备，下载好安装介质后，拷贝到目标电脑手动安装。

1. 到 Visual Studio Code 官网 [https://marketplace.visualstudio.com/vscode](https://marketplace.visualstudio.com/vscode)
   搜索需要使用的插件名称。
2. 在插件的详情页面右侧，找到插件的 `Download Extension` 链接，下载离线安装包，文件格式是 `.vsix`。
3. 拷贝插件离线安装包到要安装的设备，打开 Visual Studio Code 按下 `⌘Cmd` `⇧Shift` `P` on Mac 或者 `Ctrl` `Shift` `P` on
   Windows 打开全局搜索，搜索 `install from VSIX` 选择 `Extensions: Install from VSIX...`，然后选择相应的离线插件安装包。

## JetBrains IDE 插件

### 插件

1. [shfmt](https://github.com/mvdan/sh) 插件提供代码格式化功能，
   在菜单 `Code | Reformat` 格式化代码或者使用快捷键 `⌘Сmd` `⌥Opt` `L` on Mac or `Ctrl` `Alt` `L` on Windows。
2. [shellcheck](https://github.com/koalaman/shellcheck) 插件提供语法检查功能

### 插件安装

正常情况下在打开 `.sh` 文件后，JetBrains 会自动提示安装以下的插件。
如果没有自动提示安装或者设备网络受限，可以在网络正常的设备，下载好安装介质后，拷贝到目标电脑手动安装。

- shfmt 下载链接：[https://github.com/mvdan/sh/releases](https://github.com/mvdan/sh/releases)，
  在 JetBrains 的安装设置路径为 `Preferences | Editor | Code Style | Shell Script`，
  在该页面下方的 `Shfmt formatter` 可以看到安装的路径，在安装路径指定 shfmt 文件路径。网络允许也可以选择 `Download`
  自动下载安装。
- shellcheck
  下载链接：[https://github.com/koalaman/shellcheck/releases](https://github.com/koalaman/shellcheck/releases)，
  在 JetBrains 的安装设置路径为 `Preferences | Editor | Inspections` ，搜索 `Shell script`,
  选择 `ShellCheck`，在该页面右下角的 `Options` 区域，可以看到安装的路径，在安装路径指定 shellcheck 文件路径（下载的压缩包需要提交解压）。
  网络允许也可以选择 `Download` 自动下载安装。

## 开发参考

1. 语法解释 explainshell [https://explainshell.com/](https://explainshell.com/)
2. 语法检查 shellcheck [https://github.com/koalaman/shellcheck](https://github.com/koalaman/shellcheck)
3. 代码格式化 shfmt [https://github.com/mvdan/sh](https://github.com/mvdan/sh)
4. 语法快速参考 [https://quickref.me/bash](https://quickref.me/bash)
5. Google Shell Style Guide [https://google.github.io/styleguide/shellguide.html](https://google.github.io/styleguide/shellguide.html)
6. Bash Reference Manual [https://www.gnu.org/software/bash/manual/bash.html](https://www.gnu.org/software/bash/manual/bash.html)
