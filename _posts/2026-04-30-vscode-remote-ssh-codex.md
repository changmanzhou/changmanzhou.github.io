---
title: 'VSCode + Remote-SSH 远程服务器部署 Codex'
date: 2026-04-30
permalink: /posts/2026/04/vscode-remote-ssh-codex/
tags:
  - VSCode
  - Remote-SSH
  - Codex
  - proxy
---

在远程服务器上使用 VSCode Remote-SSH 开发时，Codex 或相关 OpenAI 服务通常需要访问外网。如果服务器本身无法直接访问，可以让远程服务器通过 SSH 隧道使用本地电脑的代理。

这篇笔记记录一种相对稳定、自动化的做法：在本地 SSH 配置中使用 `RemoteForward`，把本地代理端口转发到远程服务器。

背景
======

VSCode Remote-SSH 的实际工作环境在远程服务器上。也就是说，扩展、终端命令和远程 VSCode 的网络请求，大多从服务器侧发出。

如果本地电脑可以通过 Clash Verge 等代理访问相关服务，而远程服务器不行，就需要把本地代理能力转发给远程端。相比每次手动开端口转发，在 `~/.ssh/config` 中配置 `RemoteForward` 更适合长期使用。

核心思路
======

核心链路如下：

```text
远程服务器 127.0.0.1:10801
    -> SSH RemoteForward
    -> 本地电脑 127.0.0.1:7897
    -> 本地代理软件
    -> 目标服务
```

其中 `10801` 是远程服务器上看到的代理端口，可以自定义；`7897` 是本地代理软件监听的端口。

需要特别注意：本地代理端口一定要以 Clash Verge 设置里的“端口设置”为准，不要直接照抄示例端口。不同机器、不同代理软件、不同配置文件中的端口都可能不一样。

本地 SSH 配置 RemoteForward
======

在本地电脑打开 SSH 配置文件：

```bash
code ~/.ssh/config
```

Windows 用户通常是：

```text
C:\Users\你的用户名\.ssh\config
```

在对应远程主机配置中加入 `RemoteForward`：

```sshconfig
Host 你的远程主机别名
    HostName 远程服务器IP或域名
    User 你的用户名
    RemoteForward 10801 127.0.0.1:7897
```

这里的含义是：远程服务器访问 `127.0.0.1:10801` 时，流量会通过 SSH 连接转发到本地电脑的 `127.0.0.1:7897`。

如果使用 Clash Verge，可以在“设置 -> Clash 设置 -> 端口设置”中查看真实监听端口。确认后，把示例里的 `7897` 替换成自己的端口。

保存配置后，需要完全断开当前 VSCode Remote-SSH 连接，再重新连接远程服务器。只 Reload 窗口有时不够，因为 SSH 隧道是在连接建立时创建的。

远程 VSCode 设置代理
======

重新连接远程服务器后，在远程 VSCode 中打开设置 JSON：

```text
Ctrl+Shift+P -> Preferences: Open Remote Settings (JSON)
```

加入或修改以下配置：

```json
{
  "http.proxy": "http://127.0.0.1:10801",
  "http.proxyStrictSSL": false
}
```

保存后执行：

```text
Developer: Reload Window
```

这样远程 VSCode 扩展和部分网络请求会优先使用远程端的 `127.0.0.1:10801`，也就是通过 SSH 隧道连接到本地代理。

curl 验证方法
======

在远程服务器终端中执行：

```bash
curl -I -x http://127.0.0.1:10801 https://chatgpt.com
```

如果返回中出现：

```text
HTTP/1.1 200 Connection established
```

说明代理 CONNECT 隧道基本已经成功建立。后续即使 `chatgpt.com` 返回 `403` 或 Cloudflare challenge，也不一定代表代理失败；这通常是目标站点的访问控制或浏览器挑战，并不等同于网络不通。

可以继续测试 OpenAI API 域名：

```bash
curl -I -x http://127.0.0.1:10801 https://api.openai.com
```

如果看到 `401`、`403`、`421` 等 HTTP 状态，也不一定是失败。对这类验证来说，重点不是拿到业务成功响应，而是确认请求能经过代理到达目标服务。没有带 API Key 时，`401` 反而可能说明网络链路已经通了。

常见错误与排查
======

`Proxy CONNECT aborted`

通常是代理类型或端口不匹配。检查 VSCode 和 curl 中使用的是 `http://127.0.0.1:10801`，再确认本地代理端口是否写对。

`Connection to proxy closed`

常见原因是本地代理端口写错，或者本地代理软件没有在该端口监听。回到 Clash Verge 的“端口设置”核对真实端口。

远程端口没有生效

确认已经完全断开 Remote-SSH 并重新连接。`RemoteForward` 依赖 SSH 连接创建，旧连接不会自动带上新配置。

`codex command not found`

这说明远程服务器里没有安装 Codex CLI，和代理隧道不是同一个问题。即使 CLI 未安装，VSCode 里的 IDE 扩展仍可能可以使用。

Node 版本过低

Codex CLI 需要较新的 Node.js。如果安装或运行 CLI 时报 Node 版本错误，先升级远程服务器上的 Node.js，再重新安装或执行 Codex。

总结
======

在 VSCode Remote-SSH 场景下，推荐把代理配置放在本地 `~/.ssh/config` 中，通过 `RemoteForward` 自动创建远程端口。这样每次连接服务器后，远程 VSCode 和终端都可以通过 `127.0.0.1:10801` 使用本地代理。

排查时优先确认三件事：本地 Clash Verge 的真实端口、SSH 是否已经重连、`curl` 是否出现 `HTTP/1.1 200 Connection established`。只要隧道建立成功，后续的 `403`、`401` 或 Cloudflare challenge 未必代表网络失败，需要结合具体服务和认证状态判断。
