+++
title = 'SSH Port Forwarding'
date = 2024-12-15T14:01:00+08:00
keywords = ['ssh', 'port forwarding', 'tunneling']
tags = ['linux', 'ssh']
draft = false
+++

# SSH Port Forwarding

## 简介

SSH Port Forwarding 是一种通过安全的 SSH （Secure Shell）连接转发网络流量的技术，通常用于在不安全的网络环境中安全地传输数据，
访问受限资源或绕过防火墙规则。
在此技术中，会绑定一个本地端口，并且所有访问该端口的网络流量都将透明的被加密并被转发到远端主机，此时报文在 SSH Client 和
SSH Server 建立的加密通道中流动，因此也被称作 SSH Tunneling（隧道）。

```text
                                          SSH Tunneling
                                                │
                 SSH Client Side                │                SSH Server Side
                                                ▼
                               ┌─────────────────────────────────┐
                  ┌────────────┤                                 ├──────────────┐
Application ────► └────────────┤                                 ├──────────────┘ ────► Server
                               └─────────────────────────────────┘
```

## 工作模式

SSH Port Forwarding 有三种工作模式：

1. Local Port Forwarding
2. Remote Port Forwarding
3. Dynamic Port Forwarding

### Local Port Forwarding

客户端主机（如执行 curl 命令或浏览器所在的主机）通过访问 SSH Client 监听的端口，从而实现访问远端某个无法直连的服务器主机（如
Nginx 服务），中间的流量通过 SSH Server 进行转发。

在 SSH Client 执行的命令形式如：

```bash
ssh -CNf -L [bind_address:]port:target_host:target_port username@ssh_server
```

- -L：设置 SSH 工作在 Local Port Forwarding 模式。
- -C：对数据流启用压缩（请注意，在高速或局域网环境中，压缩的开销可能反而导致性能下降）。
- -N：指示 SSH 不执行远程命令，只建立隧道或端口转发，用于纯粹的端口转发或隧道建立，不需要启动交互式 shell 或远程命令的情况。
- -f：后台运行，在 SSH 会话建立后，将客户端进程转到后台运行

样例：

```bash
ssh -CNf -L 0.0.0.0:1080:httpbin.org:443 root@Proxy2
```

在此样例中，SSH Client `Proxy1` 监听本机的 1080 端口，并将访问该端口的流量都通过 SSH Server `Proxy2` 这台主机转发，最终访问
`httpbin.org` 的 443 端口。

```
ssh -CNf -L 0.0.0.0:1080:httpbin.org:443 root@Proxy2
 Lan1                           │               │     Lan2
┌───────────────────────────────┼──────────┐    │    ┌────────────────────────────────────────┐
│                               ▼          │    │    │        local random port               │
│                         ┌──────────┐     │    │    │     ┌─────────┐                        │
│       ┌────────────────►│ Proxy1   ├─────┼────┼────┼────►│ Proxy2  ├───────────────┐        │
│       │            1080 │          │     │    │    │ 22  │         │               │        │
│       │                 └──────────┘     │    │    │     └─────────┘               │        │
│       │                 local random port│    │    │                               │        │
│       │ local random port                │    │    │                          443  │        │
│  ┌────┴─────┐                            │    │    │                         ┌─────▼─────┐  │
│  │ Client1  │                            │    │    │                         │httpbin.org│  │
│  └──────────┘                            │    │    │                         └───────────┘  │
│       ▲                                  │    │    │                                        │
└───────┼──────────────────────────────────┘    │    └────────────────────────────────────────┘
     curl https://proxy1:1080                   │
```

### Remote Port Forwarding

在 Local Port Forwarding 模式模式中，是 SSH Client 监听本地端口并转发到 SSH Server，
Remote Port Forwarding 与 Local Port Forwarding 刚好相反，是 SSH Server 听本地端口并转发到 SSH Client。

在 SSH Client 执行命令启动该模式后，所有访问 SSH Server 本地端口的流星，都会被转发到 SSH Client。

如果把 Local Port Forwarding 定义为“正向防火墙穿透”的话，那么 Remote Port Forwarding 就是“反向防火墙穿透”。

在 SSH Client 执行的命令形式如：

```bash
ssh -CNf -R [bind_address:]port:target_host:target_port username@ssh_server
```

- -R：设置 SSH 工作在 Remote Port Forwarding 模式。

样例：

```bash
ssh -CNf -L 0.0.0.0:1080:LocalServer1:443 root@Proxy2
```

在此样例中，SSH Server `Proxy2` 监听本机的 1080 端口，并将访问该端口的流量都通过 SSH Client `Proxy1` 这台主机转发，最终访问
`LocalServer1` 的 443 端口。

需要注意的的，SSH 建立连接的方向是 `Proxy1` 到 `Proxy2`，但数据流（主动发起请求）却相反，是从 `Proxy2` 到 `Proxy1`。

```
ssh -CNf -R 0.0.0.0:1080:LocalServer1:443 root@Proxy2 
 Lan1                       │               │     Lan2
┌───────────────────────────┼──────────┐    │    ┌───────────────────────────────────────────┐
│                           ▼          │    │    │                                           │
│                     ┌──────────┐     │    │    │     ┌─────────┐                           │
│   local random port │          │     │    │    │     │         │                           │
│       ┌─────────────┤ Proxy1   ├─────┼────┼────┼────►│ Proxy2  ├◄───────────────┐          │
│       │             │          │     │    │    │ 22  │         │ 1080           │          │
│       │             └──────────┘     │    │    │     └─────────┘                │          │
│       │             local random port│    │    │                                │          │
│       ▼ 443                          │    │    │              local random port │          │
│  ┌──────────────┐                    │    │    │                          ┌─────┴─────┐    │
│  │LocalServer1  │                    │    │    │                          │Client1    │    │
│  └──────────────┘                    │    │    │                          └───────────┘    │
│                                      │    │    │                                ▲          │
└──────────────────────────────────────┘    │    └────────────────────────────────┼──────────┘
                                            │                        curl https://proxy2:1080
```

### Dynamic Port Forwarding

Dynamic Port Forwarding 本质上是 SSH Client 启动了 socks5 的代理服务来转发流量。

可以将此模式理解为一个增强型的 Local Port Forwarding，两者在流量转发形式上虽仅表现在是否限制目的地址，但原理完全不同。
在 Local Port Forwarding 中，本地端口（在 SSH Client）和目标端口（在远程主机）均已指定，仅为这些特定端口设置转发，
隧道将流量从源端口引导到目标端口。客户端不需要支持 SOCKS 协议，仅需要访问代理的地址和端口，就能访问到代理指向的地址和端口。

相比之下，Dynamic Port Forwarding 转发不指定目标端口。只指定本地端口，该端口为 SOCKS 代理服务的监听端口，
所有访问该本地端口的请求都会通过 SOCKS 代理传送到其目标。访问本地端口的客户端需要支持 SOCKS 协议，但代理不再指定要访问的目标地址。

在 SSH Client 执行的命令形式如：

```bash
ssh -CNf -D [bind_address:]port username@ssh_server
```

- -D：设置 SSH 工作在 Dynamic Port Forwarding 模式。

样例：

在此样例中，`Proxy1` 运行 socks5 代理监听在 1080 端口，`Client1` 指定 SOCKS 代理的地址为 `Proxy1`，流量经过 `Proxy1` 转发到
`Proxy2` 后，再由 `Proxy2` 转发到指定的目标地址。

```bash
ssh -CNf -D 0.0.0.0:1080 root@Proxy2
```

```
      ssh -CNf -D 0.0.0.0:1080 root@Proxy2      │
 Lan1                           │               │     Lan2
┌───────────────────────────────┼──────────┐    │    ┌───────────────────────────────────────┐
│                               ▼          │    │    │       local random port               │
│                         ┌──────────┐     │    │    │     ┌─────────┐                       │
│       ┌────────────────►│ Proxy1   ├─────┼────┼────┼────►│ Proxy2  ├──────────────┐        │
│       │            1080 │          │     │    │    │ 22  │         │              │        │
│       │                 └──────────┘     │    │    │     └─────────┘              │        │
│       │                 local random port│    │    │                              │        │
│       │ local random port                │    │    │                         443  │        │
│  ┌────┴─────┐                            │    │    │                        ┌─────▼─────┐  │
│  │ Client1  │                            │    │    │                        │httpbin.org│  │
│  └──────────┘                            │    │    │                        └───────────┘  │
│       ▲                                  │    │    │                                       │
└───────┼──────────────────────────────────┘    │    └───────────────────────────────────────┘
  curl https://httpbin.org --socks5 Proxy1:1080 │
```

## 转发限制

在 SSH Server 端，可以修改 SSH 配置文件（通常在 /etc/ssh/sshd_config）来限制端口转发目标的范围，具体的配置项的名称为
`PermitOpen`。

在 man pages 中关于 `PermitOpen` 的描述：

> Specifies the destinations to which TCP port forwarding
> is permitted. The forwarding specification must be one
> of the following forms:
>
>      PermitOpen host:port
>      PermitOpen IPv4_addr:port
>      PermitOpen [IPv6_addr]:port
>
> Multiple forwards may be specified by separating them
> with whitespace. An argument of any can be used to
> remove all restrictions and permit any forwarding
> requests. An argument of none can be used to prohibit
> all forwarding requests. The wildcard ‘*’ can be used
> for host or port to allow all hosts or ports
> respectively. Otherwise, no pattern matching or address
> lookups are performed on supplied names. By default all
> port forwarding requests are permitted.

需要注意的是，`PermitOpen` 限制的目标地址不支持 CIDR 格式的 IP 地址网段。

## 参考

1. [sshd config](https://man7.org/linux/man-pages/man5/sshd_config.5.html)
