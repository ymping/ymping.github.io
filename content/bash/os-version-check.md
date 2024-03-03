+++
title = 'Bash常用脚本一：操作系统版本检查'
date = 2024-03-03T09:58:00+08:00
keywords = ['bash', 'os version', 'distribution']
tags = ['linux', 'bash']
draft = false
+++

## 摘要

本文提供一个检查 Linux 发行版本的 bash 脚本示例，一般用于程序对运行的操作系统版本有限制的情况下，使用引导脚本在程序启动前对操作的版本系统做检查。

## 检查脚本示例

```shell
#!/usr/bin/env bash

set -eo pipefail

os_check() {
    # Prevent running on AIX and Darwin
    if [[ $(uname) != "Linux" ]]; then
        echo "unsupported OS $(uname), Linux ONLY"
        return 1
    fi

    declare -A distrib
    # support pattern matching, all Ubuntu distributions can pass check
    distrib["Ubuntu"]="*"
    # on RH 6.X 7.X, it's name is Red Hat Enterprise Linux Server
    # on RH 8.X 9.X, it's name is Red Hat Enterprise Linux
    distrib["Red Hat Enterprise Linux"]="6.* 7.* 8.* 9.*"
    distrib["Red Hat Enterprise Linux Server"]=${distrib["Red Hat Enterprise Linux"]}
    distrib["CentOS Linux"]="7 8"
    distrib["SLES"]="12.2 12.5"
    distrib["openEuler"]="22.03"
    distrib["Kylin Linux Advanced Server"]="V10"
    distrib["TencentOS Server"]="2.4 3.1"

    declare os_name os_version
    if [[ -f /etc/os-release ]]; then
        os_name=$(awk -F'"' '$1=="NAME="{print $2;exit}' /etc/os-release)
        os_version=$(awk -F'"' '$1=="VERSION_ID="{print $2;exit}' /etc/os-release)
    elif [[ -f /etc/redhat-release ]]; then
        # RH 6.x missing file /etc/os-release
        # file /etc/redhat-release example: Red Hat Enterprise Linux Server release 6.4 (Santiago)
        os_name="Red Hat Enterprise Linux Server"
        os_version=$(grep -oP 'release \K\S+' /etc/redhat-release)
    else
        echo "cannot get the OS distribution"
        return 1
    fi

    declare ver
    for ver in ${distrib["${os_name}"]}; do
        # shellcheck disable=SC2053
        if [[ ${os_version} == ${ver} ]]; then
            return 0
        fi
    done

    echo "unsupported OS distribution: ${os_name}, version: ${os_version}"
    return 1
}

os_check

# exec to your program here
# exec <bin file> [arguments]
echo "sleep 10 seconds ..."
sleep 10

```

## 常见Linux系统release文件整理

### Ubuntu

#### Ubuntu 12.04

##### /etc/os-release

```text
NAME="Ubuntu"
VERSION="12.04.5 LTS, Precise Pangolin"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu precise (12.04.5 LTS)"
VERSION_ID="12.04"
```

##### /etc/lsb-release

```text
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=12.04
DISTRIB_CODENAME=precise
DISTRIB_DESCRIPTION="Ubuntu 12.04.5 LTS"
```

#### Ubuntu 14.04

##### /etc/os-release

```text
NAME="Ubuntu"
VERSION="14.04.5 LTS, Trusty Tahr"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 14.04.5 LTS"
VERSION_ID="14.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
```

##### /etc/lsb-release

```text
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.5 LTS"
```

#### Ubuntu 22.04

##### /etc/os-release

```text
PRETTY_NAME="Ubuntu 22.04.3 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.3 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
```

##### /etc/lsb-release

```text
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=22.04
DISTRIB_CODENAME=jammy
DISTRIB_DESCRIPTION="Ubuntu 22.04.3 LTS"
```

### Red Hat

#### Red Hat 6.4

##### /etc/os-release

no such file

##### /etc/redhat-release

```text
Red Hat Enterprise Linux Server release 6.4 (Santiago)
```

#### Red Hat 7.4

##### /etc/os-release

```text
NAME="Red Hat Enterprise Linux Server"
VERSION="7.4 (Maipo)"
ID="rhel"
ID_LIKE="fedora"
VARIANT="Server"
VARIANT_ID="server"
VERSION_ID="7.4"
PRETTY_NAME="Red Hat Enterprise Linux Server 7.4 (Maipo)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:redhat:enterprise_linux:7.4:GA:server"
HOME_URL="https://www.redhat.com/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"

REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 7"
REDHAT_BUGZILLA_PRODUCT_VERSION=7.4
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="7.4"
```

##### /etc/redhat-release

````text
Red Hat Enterprise Linux Server release 7.4 (Maipo)
````

#### Red Hat 8.4

##### /etc/os-release

```text
NAME="Red Hat Enterprise Linux"
VERSION="8.4 (Ootpa)"
ID="rhel"
ID_LIKE="fedora"
VERSION_ID="8.4"
PLATFORM_ID="platform:el8"
PRETTY_NAME="Red Hat Enterprise Linux 8.4 (Ootpa)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:redhat:enterprise_linux:8.4:GA"
HOME_URL="https://www.redhat.com/"
DOCUMENTATION_URL="https://access.redhat.com/documentation/red_hat_enterprise_linux/8/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"

REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 8"
REDHAT_BUGZILLA_PRODUCT_VERSION=8.4
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="8.4"
```

##### /etc/redhat-release

```text
Red Hat Enterprise Linux release 8.4 (Ootpa)
```

#### Red Hat 9.0

##### /etc/os-release

```text
NAME="Red Hat Enterprise Linux"
VERSION="9.0 (Plow)"
ID="rhel"
ID_LIKE="fedora"
VERSION_ID="9.0"
PLATFORM_ID="platform:el9"
PRETTY_NAME="Red Hat Enterprise Linux 9.0 (Plow)"
ANSI_COLOR="0;31"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:redhat:enterprise_linux:9::baseos"
HOME_URL="https://www.redhat.com/"
DOCUMENTATION_URL="https://access.redhat.com/documentation/red_hat_enterprise_linux/9/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"

REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 9"
REDHAT_BUGZILLA_PRODUCT_VERSION=9.0
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="9.0"
```

##### /etc/redhat-release

```text
Red Hat Enterprise Linux release 9.0 (Plow)
```

### openEuler

#### openEuler 22.03

##### /etc/os-release

```text
NAME="openEuler"
VERSION="20.03 (LTS-SP4)"
ID="openEuler"
VERSION_ID="20.03"
PRETTY_NAME="openEuler 20.03 (LTS-SP4)"
ANSI_COLOR="0;31"
```

##### /etc/openEuler-release

```text
openEuler release 20.03 (LTS-SP4)
```

### Kylin

#### Kylin V10

##### /etc/os-release

```text
NAME="Kylin Linux Advanced Server"
VERSION="V10 (Lance)"
ID="kylin"
VERSION_ID="V10"
PRETTY_NAME="Kylin Linux Advanced Server V10 (Lance)"
ANSI_COLOR="0;31"
```

##### kylin-release

```text
Kylin Linux Advanced Server release V10 (Lance)
```

`/etc/system-release` link to this file.

### TencentOS

#### tencentos 2.4

##### /etc/os-release

```text
NAME="TencentOS Server"
VERSION="2.4"
ID="tencentos"
ID_LIKE="rhel fedora centos tlinux"
VERSION_ID="2.4"
PRETTY_NAME="TencentOS Server 2.4"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:tencentos:tencentos:2"
HOME_URL="https://cloud.tencent.com/product/ts"
```

##### /etc/tlinux-release

```text
TencentOS Server 2.4 (Final)
```

`/etc/centos-release` `/etc/redhat-release` `/etc/system-release` `/etc/tencentos-release` all soft link to this file
finally.

#### tencentos 3.1

##### /etc/os-release

```text
NAME="TencentOS Server"
VERSION="3.1 (Final)"
ID="tencentos"
ID_LIKE="rhel fedora centos"
VERSION_ID="3.1"
PLATFORM_ID="platform:el8"
PRETTY_NAME="TencentOS Server 3.1 (Final)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:tencentos:tencentos:3"
HOME_URL="https://cloud.tencent.com/product/ts"

TENCENT_SUPPORT_PRODUCT="tencentos"
TENCENT_SUPPORT_PRODUCT_VERSION="3"
NAME_ORIG="TencentOS Server
```

##### /etc/tlinux-release

```text
TencentOS Server 3.1 (Final)
```

##### /etc/centos-release

```text
TencentOS Server release 3.1 (Final) 
```

`/etc/redhat-release` link to this file.

##### /etc/tencentos-release

```text
TencentOS Server 3.1 (Final)
```

`/etc/system-release` `/etc/tlinux-release` link to this file.

### SUSE

#### SLES 15

##### /etc/os-release

```text
NAME="SLES"
VERSION="15-SP4"
VERSION_ID="15.4"
PRETTY_NAME="SUSE Linux Enterprise Server 15 SP4"
ID="sles"
ID_LIKE="suse"
ANSI_COLOR="0;32"
CPE_NAME="cpe:/o:suse:sles:15:sp4"
DOCUMENTATION_URL="https://documentation.suse.com/"
```

#### openSUSE 42.2

##### /etc/os-release

```text
NAME="openSUSE Leap"
VERSION="42.2"
ID=opensuse
ID_LIKE="suse"
VERSION_ID="42.2"
PRETTY_NAME="openSUSE Leap 42.2"
ANSI_COLOR="0;32"
CPE_NAME="cpe:/o:opensuse:leap:42.2"
BUG_REPORT_URL="https://bugs.opensuse.org"
HOME_URL="https://www.opensuse.org/"
```

##### /etc/SuSE-release

```text
openSUSE 42.2 (x86_64)
VERSION = 42.2
CODENAME = Malachite
# /etc/SuSE-release is deprecated and will be removed in the future, use /etc/os-release instead
```
