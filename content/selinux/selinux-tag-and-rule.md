+++
title = 'SELinux: Tag and Rule'
date = 2024-01-07T14:40:00+08:00
keywords = ['linux', 'selinux']
tags = ['linux', 'selinux', 'policy']
draft = false
+++

## 摘要

本文介绍了 SELinux 的标签、策略这两个核心基本概念和三种访问控制机制（TE、MLS、MCS）。

## 名词解释

1. SELinux: 全称 **Security Enhanced Linux**，中文译作**安全增强型 Linux**。
   基于**最小化权限原则**设计开发的 Linux 安全机制，下文将做详细介绍。
2. DAC: 全称 **Discretionary Access Control**，中文译作**自主访问控制**。
   用户对属于自己的资源有完全控制权限，如 UNIX 和 Windows 中的文件系统权限，其文件的所有者可以不受限的设置文件的访问权限，包括读、写和执行。
3. MAC: 全称 **Mandatory Access Control**，中文译作**强制访问控制**。
   在 MAC 中，系统管理员定义了资源和主体之间的访问规则，这些规则是强制性的，无法由主体自行更改。SELinux 就是一个典型的 MAC 实现。

## SELinux 是什么

SELinux 是一个标签系统。每个进程、文件、目录，甚至网络端口和设备，都有一个标签，这些标签称作安全上下文 (Security Context)。
可以编写规则控制带某个标签的进程对带某个标签的的对象（如文件）的访问，这些控制规则称作 SELinux 策略。
Linux 内核强制执行这些规则，这种强制执行称作强制访问控制（MAC）。

对象的所有者对对象的安全属性没有决定权。标准的 Linux 权限控制，所属用户/组和权限标志如 RWX，通常被称作自主访问控制（DAC）。
SELinux 没有 UID 或文件所有权的概念，一切都由标签控制。

在 SELinux 开启后，并不意味着可以跳过 DAC 的权限控制，SELinux 和 DAC 是同时生效的。应用程序必须得到 SELinux 和 DAC
的允许才能执行某些活动。这可能会导致管理员在看到 **Permission Denied** 时会感到困惑，因为通常看到 DAC 的权限是允许的，而忽略了
SELinux 标签有问题。

## SELinux 访问控制机制

TE (Type Enforcement)，MCS (Multi-Category Security) 和 MLS (Multi-Level Security) 是 SELinux 提供的三种访问控制机制。

TE 提供了基础的访问控制机制，通过标签标识主体（通常是进程）和对象（如文件），通过定义标签之间的访问规则来实现主体对对象的访问控制。
MCS 和 MLS 建立在 TE 的基础上的另外两个安全特性，通过为对象和主体分配多个安全级别来实现多级安全，提供了更高级别的安全控制机制。
MCS 和 MLS 这两个特性是可选的。

### Type enforcement

TE（Type enforcement，类型强制控制）是 SELinux 主要的访问控制机制。通常根据进程和文件的类型来定义其标签。

简单举一个例子来说明下什么是 TE。

想象你养了一只猫🐱和一只🐶，并从网上购买了猫粮和狗粮。平时你没有怎么管，狗经常去吃猫粮，结果把猫粮吃完了，害的猫也只能吃狗粮。
经常这样总是不好的，你希望猫就只能吃猫粮，狗就只能吃狗粮，所以你买了一个可以配置策略的自动投食机，遇到猫就投喂猫粮，遇到狗就投喂狗粮，
这样就做到了自食其粮。

为了达成这一目的，需要给对象打上标签，以便自动投食机识别，故给猫打了名为 cat 的标签，给狗打了名为 dog 的标签，相应的猫狗粮也分别打了标签
cat_chow 和 dog_chow，并给自动投食机设置了“吃粮策略”:

```text
allow    cat    cat_chow:food    eat
allow    dog    dog_chow:food    eat
```

类比于 SELinux 的场景中，你是管理员，制定了猫狗吃粮的策略，称为 TE 规则，自动投食机负责执行这个策略，是 Linux 内核，猫和狗都是进程，
它们受“吃粮策略”的控制，只能吃自己对应的粮食，即进程访问相应资源受 TE 规则的约束，没有在策略中允许的访问都会被拒绝。

现实中，我们有一个 Web 服务器，将 Nginx 进程标记为 `httpd_t`，将 Nginx 需要访问的文件标记为 `httpd_sys_content_t` 或
`httpd_sys_content_rw_t`，同时该主机还运行的 MySQL 数据库，存储了一些敏感机密数据，将这些数据文件标记为 `mysqld_data_t`。
如果 Nginx 进程被黑客劫持，可以读取 `httpd_sys_content_t` 标记的内容和读写 `httpd_sys_content_rw_t` 标记的内容。
即使 Nginx 进程以 root 身份运行，黑客也不被允许读取 MySQL 存储的敏感数据（`mysqld_data_t`）。
在这种情况下，SELinux 缓解了受到入侵的严重情况。

TE rules:

```text
allow httpd_t httpd_sys_content_t:file read;
allow httpd_t httpd_sys_content_rw_t:file { read write };
```

### MLS enforcement

MLS 的全称是 Multi Level Security。MLS 的主要思想是根据进程的层级来划分所使用到的数据，低层级的进程无法访问高层级的数据，
旨在控制系统上不同进程之间的信息流动。

继续 TE 章节中关于宠物的例子，为了方便区分，我们给养的小狗取名为旺财，假设再养了一只茶杯犬，取名豆豆。这个茶杯犬豆豆比较娇气，
需要吃比较精细的狗粮，它不能吃普通的狗粮，怕被噎死，只能给它单独购买狗粮。但旺财可以吃豆豆的狗粮，因为不会被噎死😭。

这便有了层级（Level）的概念，旺财和豆豆都是狗，但为了给它们制定吃狗粮的策略，把它们区分开来，可以让它们处于不同的层级，同时把狗粮和划分层级。
比如我们把旺财和普通狗粮划分到层级 L1，把豆豆和单独给豆豆购买的狗粮划分到层级 L2。故此可以给旺财和豆豆分别打上标签 `dog:l1`
与 `dog:l0`，普通的狗粮和豆豆的狗粮打上标签 `dog_chow:l1` 与 `dog_chow:l0`。
这时给自动投食机设置的“吃粮策略”便是高层级的狗只能吃低于或等于自己层级的食物，反之则不行。

为了实现 MLS，SELinux 使用 Bell-La Padula Model (BLP) 模型，该模型根据附加到每个主体和客体的标签来决定信息如何在系统内流动。BLP
的基本 原则是 **No read up, no write down**（下读，上写），这意味着用户只能读取等于或低于自己敏感级别的文件，
数据只能从较低级别流向较高级别，而不能反相流动。

在RHEL中，MLS 是基于修改过的 BLP 原则即 **Bell-La Padula with write equality** 实现。在该基本原则下，
用户可以读取等于或低于自己敏感级别的文件，但只能写入与自己敏感级别相同文件。这可以防止低权限用户将内容写入到更高级别。

MLS 默认有16个敏感度级别，其中 `s15` 是最高敏感度级别，`s0` 是最低敏感度级别。

在现实的计算机中，假设我们有两个 Nginx 进程，一个进程的标签是 `httpd_t:s15`，另一个进程是 `httpd_t:s10`。如果 Nginx 进程
`httpd_t:s10` 被劫持了，攻击者可以读取 `httpd_sys_content_t:s10` 的资源，但无法读取 `httpd_sys_content_t:s15` 的资源。
但如果标签是 `httpd_t:s15` 的 Nginx 进程被劫持了，攻击者不仅可以读取 `httpd_sys_content_t:s15` 的资源，还可以读取
`httpd_sys_content_t:s10` 的资源。

MLS 检查发生在 DAC 和 TE 规则检查通过后。

### MCS enforcement

MCS 的全称是 Multi Category Security。

在 MLS 章节的宠物例子中，假设我们多养的一只狗不是茶杯犬豆豆，而是和旺财同品种的一个小狗来福，另外小猫取名叫加菲。
此时希望分配给旺财的狗粮只能旺财吃，分配给来福的狗粮只能来福吃，当然，加菲只能吃猫粮。

这种情况下我们应该如何打标签来让自动投食机识别呢？

一个方案是旺财和来福虽都属于狗，但都为它们分别创建新的标签，如 `wangcai_dog`/`laifu_dog` 和 `wangcai_chow`/`laifu_chow`。
但当狗的数量增多后，这样的编写的规则会变得不可维护，因为狗的权限基本上都相同。

为了解决这个问题，SELinux 提供一种新的控制策略 MCS。在 MCS 中，我们把标签的格式扩展成 `Type:Level:Category` 的格式（中间的
Level 段为 MLS 内容，此处请忽略），如旺财和其狗粮的标签为 `dog:l0:random1` 和 `dog_chow:l0:random1`，来福和其狗粮的标签是
`dog:l0:random2` 和 `dog_chow:l0:random2`。

MCS 的机制很简单，用户要访问资源，标签中用户的 Category 区间必须包含资源的 Category 区间，否则拒绝访问。
这是之所以用*Category 区间*的表述，是实际情况中，标签是写成 `dog:l0:c0-c2` 的格式，其中 `c0,c2` 表示一个区间即 `c0-c2`。
一个完整的标签如下:

```text
system_u:system_r:container_t:s0:c476,c813
    ^       ^         ^        ^     ^--- unique category
    |       |         |        |---- security level 0
    |       |         |--- SELinux type
    |       |--- SELinux role
    |------ SELinux user
```

MCS 检查发生在 DAC 和 TE 规则检查通过后。

现实中，MCS 类别取值从 c0 到 c1023。MCS 的一个典型的应用场景是在容器上。

容器启动时，容器引擎会生成一个随机且唯一的 MCS 标签（如 `c476,c813`）。然后容器引擎会在镜像挂载时，为容器镜像内的文件设置该 MCS
标签 （`c476,c813`），随后在启动容器进程时，也会为进程设置同样的 MCS 标签（`c476,c813`）。这样同一容器内的进程和文件都有相同的 MCS
标签，而不同的容器内的进程和文件的 MCS 标签不同，这样，内核就可以根据 MCS 规则，使容器内的进程仅能访问自己容器的文件系统，
阻止不同容器间对文件的越界访问。

MCS 为容器的文件系统之间的隔离性，在 mount namespace 基础上再增加了一层安全机制，这样即使某一个容器进程（标签如
`system_u:system_r:container_t:s0:c476,c813` ）被黑客劫持，逃逸出了容器的 mount namespace，
也不能对其它容器的文件（标签如 `system_u:object_r:container_file_t:s0:c109,c889`）造成破坏。

## 总结

SELinux 是一个标签系统，Linux 系统中的每个资源都会被分配一个标签，围绕这些标签制定的标签之间的访问控制规则，称作 SELinux
策略。

标签和规则，是 SELinux 两个核心的基本要点。其它更上层的概念如 TE MLS MCS 等，都是基于这两个点展开的，万变不离其宗！

## 参考

1. [Your visual how-to guide for SELinux policy enforcement](https://opensource.com/business/13/11/selinux-policy-guide)
2. [RedHat SELinux Doc](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux/index)
3. [Bell-LaPadula模型](https://zh.wikipedia.org/wiki/Bell%E2%80%93LaPadula%E6%A8%A1%E5%9E%8B)
4. [Why you should be using Multi-Category Security for your Linux containers](https://www.redhat.com/en/blog/why-you-should-be-using-multi-category-security-your-linux-containers)
