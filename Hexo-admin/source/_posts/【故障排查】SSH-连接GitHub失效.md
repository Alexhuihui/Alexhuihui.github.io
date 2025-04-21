---
title: 【故障排查】SSH 连接GitHub失效
date: 2025-04-21 17:03:59
categories: 故障排查
tags: [故障排查, ssh]
---

# Git SSH 无法连接 GitHub 的排查与代理配置（Clash 环境）

在开发过程中，使用 Git SSH 连接 GitHub 是常见的操作，最近在本地开发中就遇到了类似的问题，执行 `git pull` 报错：

```bash
ssh: connect to host ssh.github.com port 443: Connection refused
```

<!-- more --> 

## 排查方向（客户端配置、SSH 使用方式、本地网络、代理等维度）

### 一、客户端配置检查

- 是使用 SSH 协议还是 HTTPS？

  > 通过 `git remote -v` 查看远程地址是否为 `git@github.com:xxx`（SSH）

- 本地是否存在 SSH Key？GitHub 是否配置了公钥？

  > 检查 `~/.ssh/id_rsa.pub`，并确认 GitHub 设置中添加了对应的 SSH 公钥

- SSH 配置文件 `~/.ssh/config` 是否有误？

------

### 二、GitHub SSH 服务是否可用

#### 📡 使用 telnet 或 nc 测试是否能访问 `ssh.github.com:443`

```bash
nc -vz ssh.github.com 443
```

如果显示连接失败，有可能是被防火墙拦截或网络限制。

------

### 三、本地网络或代理问题

#### Clash 是否已开启？

Clash 是本地代理工具，通常运行在 127.0.0.1:7890（HTTP）或 7897（SOCKS5），可以代理所有出站流量。

若未开启或未启用系统代理，则 Git SSH 不会自动走代理。

------

### 四、配置 SSH 使用 Clash 的 SOCKS5 代理

为了解决 SSH 连接无法通过网络直连 GitHub 的问题，我们可以使用 `ProxyCommand` 方式，配置 Git 的 SSH 使用 Clash 的代理。

#### ✅ 推荐使用 connect.exe 作为代理工具

1. Git 自带：
2. 修改 `~/.ssh/config` 文件如下：

```ssh
Host github.com
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_rsa
  TCPKeepAlive yes
  ProxyCommand "C:\Program Files\Git\mingw64\bin\connect.exe" -S 127.0.0.1:7897 -a none %h %p

Host ssh.github.com
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_rsa
  TCPKeepAlive yes
  ProxyCommand "C:\Program Files\Git\mingw64\bin\connect.exe" -S 127.0.0.1:7897 -a none %h %p
```

> 这里指定了通过 Clash 的本地 SOCKS5 代理（127.0.0.1:7897）连接 GitHub 的 SSH 服务器。

------

### 五、验证连接是否成功

```bash
ssh -T git@github.com
```

如果输出：

```bash
Hi your-username! You've successfully authenticated...
```

## 总结与建议

- 如果你用的是 SSH 协议（`git@github.com`），就需要配置 SSH 的 `ProxyCommand` 才能走代理。
- 推荐使用 `connect.exe` 工具，兼容性更好。

如果你也遇到 Git SSH 无法连接的问题，不妨参考以上方式配置代理，大概率就能解决了。
