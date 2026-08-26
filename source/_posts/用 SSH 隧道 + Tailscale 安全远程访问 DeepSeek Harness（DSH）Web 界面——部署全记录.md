---
title: 用 SSH 隧道 + Tailscale 安全远程访问 DeepSeek Harness（DSH）Web 界面——部署全记录
date: 2026-08-26 18:00:00
updated: 2026-08-26 18:00:00
description: 记录在 Windows 服务器运行 DeepSeek Harness（DSH）时，如何通过 SSH 隧道、Tailscale 与密钥认证跨网络安全访问本机 Web 界面。
keywords: [DeepSeek Harness, DSH, SSH, Tailscale, OpenSSH, 端口转发, 密钥认证, Windows]
tags:
  - DeepSeek Harness
  - SSH
  - Tailscale
  - Windows
  - 网络
categories:
  - [技术, 工具]
cover: /img/bg3.png
top_img:
---

## 前言

服务器上运行的 **DeepSeek Harness（DSH）** Web 界面默认只监听本机回环地址 `127.0.0.1:3080`。当主力机与服务器不在同一网络、且两个网段无法互通时，单纯开放监听地址和防火墙端口并不能解决问题，反而会带来明显的安全风险。

本文记录一套最终可用的方案：**Tailscale 虚拟内网 + SSH 本地端口转发 + SSH 密钥认证**。它既能跨网络访问，又始终保持 DSH Web 服务仅对服务器本机监听；配置完成后，主力机双击脚本即可免密打开页面。

> 本文中的用户名、网段、Tailnet IP 与文件路径均已使用示例值替代，请按自己的环境调整。

---

## 一、环境与目标

| 角色 | 系统与网络 | 关键信息 |
|---|---|---|
| 服务器 | Windows，受管有线网络 | DSH Web 监听 `127.0.0.1:3080` |
| 主力机 | Windows，另一无线网络 | 与服务器网段不同，无法直接访问 |
| 虚拟内网 | Tailscale | 两台设备登录同一个 Tailnet |

目标是让主力机浏览器访问：

```text
http://127.0.0.1:3080
```

但实际请求会通过加密隧道转发到服务器的 `127.0.0.1:3080`，不把 DSH 直接暴露到局域网或公网。

---

## 二、先理解 DSH Web 的安全边界

### 2.1 默认只监听回环地址

DSH 的 `dsh web` 会启动 HTTP 服务。虽然相关配置层可以指定监听地址，但命令行显式限制将服务直接绑定到 `0.0.0.0`。这是有意的安全设计：DSH Web 页面能够驱动 Agent 执行命令，直接暴露服务会扩大远程代码执行风险。

因此，推荐保持：

```text
127.0.0.1:3080
```

而不是将 Web 界面直接开放到所有网卡。

### 2.2 Web 页面不是“登录后才可用”的管理后台

DSH 的浏览器信任限制、受信 Host 校验等机制可以缓解 DNS Rebinding、跨站请求等风险，但它们不是用户认证。

谁能连接这个端口，谁就可能操控 Agent。因此应当把访问控制放到网络传输层：例如 Tailscale 的设备身份控制、SSH 的账号与密钥认证，以及防火墙的来源限制。

---

## 三、方案对比与最终选择

| 方案 | 原理 | 优点 | 主要问题 |
|---|---|---|---|
| SSH 隧道 | 主力机将服务器回环端口转发到本机 | 服务端无需开放 DSH 端口；加密且有认证 | 需要可访问 SSH 服务 |
| Tailscale | 两台设备加入同一虚拟内网 | 跨网段、设备身份认证、链路加密 | 需要安装客户端并完成授权 |
| 局域网直连 | DSH 监听 `0.0.0.0` 并放行端口 | 配置最直接 | HTTP 明文且服务本身无登录认证 |
| frp/ngrok 等内网穿透 | 通过公网中转暴露端口 | 无路由也可访问 | 会增加公网暴露与被扫描风险 |

最终采用：

```text
Tailscale 解决网络不通
        +
SSH 隧道保留 DSH 的回环监听
        +
SSH 密钥认证实现免密连接
```

访问链路如下：

```text
主力机浏览器
    ↓
主力机 127.0.0.1:3080
    ↓ SSH 本地端口转发（加密）
Tailscale 虚拟内网（加密）
    ↓
服务器 127.0.0.1:3080
```

---

## 四、服务器：启用 OpenSSH Server

以管理员身份打开 PowerShell，安装并启动 OpenSSH Server：

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

如需在物理局域网内测试 SSH，可以添加严格限定来源的入站规则：

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' `
  -Enabled True -Direction Inbound -Protocol TCP -Action Allow `
  -LocalPort 22 -RemoteAddress 10.32.88.0/23
```

### 一个常见坑：不要想当然写网段

最初按 `/24` 写规则，而服务器真实掩码是 `/23`（`255.255.254.0`），会导致同一逻辑网段另一半地址的连接被防火墙静默丢弃。

先在服务器查看实际网络配置：

```powershell
ipconfig
```

再按真实掩码设置来源范围。测试连通性时也应优先使用 TCP 测试，而不是仅依赖 `ping`：

```powershell
Test-NetConnection <服务器地址> -Port 22
```

Windows 防火墙可能默认拦截 ICMP，`ping` 不通不等于 SSH 不通。

---

## 五、第一道坎：两个物理网段无法互通

主力机尝试连接服务器局域网地址时出现：

```text
Connection timed out
```

即使扩大防火墙允许范围，连接仍然超时。通过 `tracert` 可以发现数据包进入了受管网络的中间路由，但无法到达目标终端。

这类场景通常不是本机配置错误，而是网络之间存在终端访问策略或回程路由限制。若无法管理中间网关，就不应该继续尝试静态路由方案，而应改用覆盖网络。

---

## 六、第二道坎：受管网络的出口过滤

受管网络中还可能存在域名白名单或 DPI 过滤：部分 API、控制平面可访问，不代表 GitHub Release、CDN 或软件包下载地址也可访问。

可以将网络可达性分为三层：

```text
TCP 可达 ≠ TLS 可达 ≠ 文件内容可达
```

遇到服务器无法下载 Tailscale 安装包时，可在网络正常的主力机下载官方 MSI，再通过已授权的远程控制软件文件传输到服务器。安装前应校验下载文件的来源、大小和哈希值，避免把错误页或不完整文件当作安装包。

服务器安装完成后执行：

```powershell
tailscale up
```

命令会给出设备授权链接。在主力机浏览器中用同一 Tailscale 账号完成授权后，服务器会加入 Tailnet。可用以下命令确认 IPv4 地址：

```powershell
tailscale ip -4
```

---

## 七、主力机：建立 SSH 隧道

主力机同样安装 Tailscale 并登录同一账号。确认两台设备都在线：

```powershell
tailscale status
```

随后在主力机运行下面的命令，将本地 `3080` 转发到服务器回环地址：

```powershell
ssh -N -L 3080:127.0.0.1:3080 <服务器用户名>@<服务器 Tailnet IP>
```

参数含义：

| 参数 | 含义 |
|---|---|
| `-N` | 不执行远程命令，只建立隧道 |
| `-L 3080:127.0.0.1:3080` | 将本机 3080 转发到服务器回环地址的 3080 |

保持该终端窗口运行，然后在主力机浏览器访问：

```text
http://127.0.0.1:3080
```

此时页面虽然显示为本地地址，但实际流量已经经由 SSH 与 Tailscale 到达服务器。

---

## 八、使用 SSH 密钥实现免密认证

每次建立隧道都输入密码既麻烦，也不适合将密码写入脚本。更可靠的方案是 SSH 密钥认证。

### 8.1 生成密钥对

在主力机生成 ed25519 密钥。私钥只保留在主力机，公钥写入服务器：

```powershell
ssh-keygen -t ed25519 -C "dsh-main-pc" -f "$env:USERPROFILE\.ssh\id_ed25519"
```

建议为私钥设置安全的口令；若使用 Windows OpenSSH Agent，可避免每次重复输入口令。

### 8.2 将公钥写入服务器授权表

在服务器上创建目录并追加公钥内容：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.ssh"
Add-Content "$env:USERPROFILE\.ssh\authorized_keys" `
  -Value (Get-Content "<主力机公钥路径>\id_ed25519.pub")
```

用批处理模式验证密钥是否生效：

```powershell
ssh -o BatchMode=yes -i "$env:USERPROFILE\.ssh\id_ed25519" `
  <服务器用户名>@<服务器 Tailnet IP> "whoami"
```

`BatchMode=yes` 会禁止回退到密码交互：密钥有问题时命令会直接失败，便于排查。

> 私钥绝不能上传、发送或写入仓库；服务器端只保存公钥。Windows 登录密码与 SSH 密钥也是两套独立机制，互不替代。

---

## 九、双击即用的隧道脚本

可以在主力机创建 `start-dsh-tunnel.bat`：

```bat
@echo off
set "KEY=%USERPROFILE%\.ssh\id_ed25519"

if not exist "%KEY%" (
  echo [ERROR] Missing SSH private key: %KEY%
  pause
  exit /b 1
)

start "DSH Tunnel" cmd /k "ssh -N -L 3080:127.0.0.1:3080 -i \"%KEY%\" -o BatchMode=yes -o StrictHostKeyChecking=accept-new <服务器用户名>@<服务器 Tailnet IP>"
timeout /t 4 /nobreak >nul
start "" "http://127.0.0.1:3080"
```

其中：

* `BatchMode=yes`：缺少密钥或认证失败时立即报错，不会卡在输入密码；
* `StrictHostKeyChecking=accept-new`：首次连接自动记录服务器主机指纹，后续若指纹变化仍会报警；
* 隧道窗口必须保持运行，关闭窗口即关闭转发。

---

## 十、冷启动与多设备扩展

两台电脑都重启后，恢复流程如下：

| 设备 | 操作 |
|---|---|
| 服务器 | 等待 `sshd` 与 Tailscale 服务自动启动，然后手动执行 `dsh web --no-open` |
| 主力机 | 等待 Tailscale 自动重连，确认设备在线，双击隧道脚本 |

最容易遗漏的是服务器端的 DSH 进程：SSH 隧道可以建立并不表示 Web 服务已经启动。

需要扩展到多台主力机时，每台设备加入同一 Tailnet，并为每台机器生成一套独立 SSH 密钥，在服务器的 `authorized_keys` 中各保留一行公钥。设备不用时，可直接在 Tailscale 管理后台移除。

---

## 十一、安全建议

1. **不要裸露服务。** DSH 一类 Agent Web 界面应默认视作高权限入口，不要直接暴露到公网。
2. **保持最小监听范围。** 让 DSH 继续只监听 `127.0.0.1`，通过 SSH 转发访问。
3. **采用分层防护。** Tailscale 负责设备网络身份与加密，SSH 负责登录认证和端口转发，防火墙限制额外暴露面。
4. **使用独立密钥。** 每台主力机一把密钥，遗失设备时删除对应公钥并在 Tailnet 中移除设备。
5. **按需提权。** 平时以普通权限运行 DSH；只有确实需要时才使用管理员权限，避免扩大 Agent 命令的影响范围。
6. **保护凭据。** 密码一旦进入聊天记录、日志或不可信脚本，应尽快轮换；私钥始终只保存在自己的受控设备上。

---

## 十二、关键命令速查

```powershell
# 服务器：安装并启用 SSH
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic

# 服务器：加入 Tailnet 并查看地址
tailscale up
tailscale ip -4

# 服务器：启动 DSH Web
dsh web --no-open

# 主力机：查看设备状态
tailscale status

# 主力机：建立本地端口转发
ssh -N -L 3080:127.0.0.1:3080 -o BatchMode=yes `
  <服务器用户名>@<服务器 Tailnet IP>
```

最终只需记住一句话：**网络不通交给 Tailscale，服务不外露交给 SSH 隧道，免密登录交给 SSH 密钥。**
