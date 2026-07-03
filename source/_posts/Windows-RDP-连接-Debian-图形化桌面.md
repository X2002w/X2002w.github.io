---
title: Windows 下使用 RDP 远程连接 Debian 图形化桌面
date: 2026-05-27 12:00:00
tags:
  - Windows
  - Debian
  - RDP
  - xrdp
  - 远程桌面
categories: Linux
---

## 前言

在日常工作中，经常需要在 Windows 主机上远程操作 Linux 服务器。虽然 SSH 命令行已经能满足大部分需求，但有时图形化桌面环境（GUI）会让操作更加直观高效——比如调试前端页面、使用 GUI 工具、或者为不熟悉命令行的同事提供远程环境。

在众多远程桌面方案中，**RDP（Remote Desktop Protocol）最大的优势在于它是 Windows 原生支持的协议**——Windows 系统自带了远程桌面连接客户端（`mstsc`），无需像 VNC、TeamViewer、AnyDesk 等方案那样额外下载安装第三方软件。这意味着：

- **零安装成本**：在任何一台 Windows 电脑上开箱即用
- **无安全顾虑**：不需要从第三方网站下载远程工具，规避捆绑软件和供应链风险
- **域环境友好**：在企业 AD 域环境中，RDP 与 Windows 认证体系无缝集成

本文将介绍如何在 Debian 上搭建 `xrdp` 服务，并通过 Windows 自带的远程桌面连接工具直接登录 Debian 的图形化桌面。

<!-- more -->

## 环境说明

| 角色    | 系统版本            |
| ------- | ------------------- |
| 客户端  | Windows 10 / 11     |
| 服务端  | Debian 13 (Trixie)  |

> 本文以 Debian 13 为例，Debian 12 及 Ubuntu 22.04/24.04 同样适用。

## 整体流程

```
Windows 远程桌面客户端  ──RDP──>  xrdp  ──>  Xorg / Xvnc  ──> 桌面环境 (XFCE)
```

1. 在 Debian 上安装一个轻量桌面环境（推荐 XFCE）
2. 安装并配置 `xrdp` 作为 RDP 协议的中转服务
3. 从 Windows 自带远程桌面客户端发起连接

## 第一步：统一安装所需软件包

以下所有需要在 Debian 端安装的包，一次性安装完成：

```bash
sudo apt update
sudo apt install -y \
    xfce4 xfce4-goodies \
    lightdm \
    xrdp \
    fonts-noto-cjk fonts-wqy-microhei \
    autocutsel
```

各软件包作用说明：

| 软件包                | 用途                                     |
| --------------------- | ---------------------------------------- |
| `xfce4 xfce4-goodies` | 轻量桌面环境及附加工具                   |
| `lightdm`             | 轻量显示管理器，替换默认的 gdm3，避免黑屏 |
| `xrdp`                | RDP 协议中转服务，监听 3389 端口         |
| `fonts-noto-cjk fonts-wqy-microhei` | 中文字体，避免中文显示为方块 |
| `autocutsel`          | 剪贴板同步工具，实现双向复制粘贴         |

> **为什么使用 lightdm？** Debian 13 默认使用 gdm3 作为显示管理器，但在 xrdp 环境下 gdm3 容易导致连接后黑屏。lightdm 更加轻量简洁，与 xrdp 配合更稳定，可以有效规避黑屏问题。

安装完成后，将 lightdm 设为默认显示管理器：

```bash
sudo systemctl enable lightdm
sudo systemctl set-default graphical.target
```

> 如果系统提示选择默认显示管理器，选择 `lightdm` 即可。

## 第二步：配置桌面环境与 xrdp

### 2.1 配置 XFCE 为默认会话

当 xrdp 连接时，需要指定启动哪个桌面环境。编辑 xrdp 启动脚本：

```bash
echo "startxfce4" > ~/.xsession
chmod +x ~/.xsession
```

> 注意：每个需要使用 RDP 登录的用户都需要在其家目录下执行此命令。

### 2.2 修复 startwm.sh 防止黑屏/闪退

编辑 `/etc/xrdp/startwm.sh`，在 `exit 0` 之前插入启动 XFCE 的命令：

```bash
# 先备份
sudo cp /etc/xrdp/startwm.sh /etc/xrdp/startwm.sh.bak

# 在 exit 0 前插入以下内容
sudo sed -i '/^exit 0$/i\
# Start XFCE4 session for xrdp\
unset DBUS_SESSION_BUS_ADDRESS\
unset XDG_RUNTIME_DIR\
test -x /usr/bin/startxfce4 && exec /usr/bin/startxfce4\
' /etc/xrdp/startwm.sh
```

### 2.3 创建专用于远程桌面的用户（解决登录即闪退）

**问题现象：** RDP 连接后输入凭据，点击登录立即闪退，回到连接界面。

**原因：** 该用户可能已经在 Debian 本地登录了桌面环境，同一个用户不能同时拥有两个图形化桌面会话。

**解决方法：** 创建一个专门用于 RDP 远程连接的用户，与本地登录用户隔离开。

```bash
# 创建新用户（例：rdpuser），并设置密码
sudo useradd -m -s /bin/bash rdpuser
sudo passwd rdpuser

# 为该用户配置 XFCE 会话
echo "startxfce4" | sudo tee /home/rdpuser/.xsession
sudo chmod +x /home/rdpuser/.xsession
sudo chown rdpuser:rdpuser /home/rdpuser/.xsession
```

之后在 Windows 远程桌面连接中，使用新创建的 `rdpuser` 登录即可，不会与本地用户冲突。

### 2.4 防火墙放行 3389 端口（如果开启了防火墙）

```bash
sudo ufw allow 3389/tcp
# 或者
sudo firewall-cmd --permanent --add-port=3389/tcp
sudo firewall-cmd --reload
```

### 2.5 设置 xrdp 开机自启

```bash
sudo systemctl enable xrdp
sudo systemctl restart xrdp
```

检查服务状态：

```bash
sudo systemctl status xrdp
sudo ss -tlnp | grep 3389
```

### 2.6 配置剪贴板共享

`autocutsel` 已在第一步安装，现在添加自启动配置：

```bash
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/autocutsel.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=autocutsel
Exec=autocutsel -fork
X-GNOME-Autostart-enabled=true
EOF
```

## 第三步：从 Windows 连接

### 3.1 使用远程桌面连接（mstsc）

按下 `Win + R`，输入 `mstsc` 并回车，打开远程桌面连接。

在"计算机"一栏填入 Debian 服务器的 IP 地址，点击"连接"：

```
计算机: 192.168.1.100
用户名: your_username
```

### 3.2 登录界面说明

连接成功后会看到 xrdp 登录界面：

- **Session** 保持默认 `Xorg`（推荐，性能最好）
- **username / password** 填写 Debian 上的用户凭据

点击 OK 后会进入 XFCE 桌面环境，操作体验与本地桌面基本一致。

### 3.3 优化连接体验

在 Windows 远程桌面连接客户端中，可以调整以下设置以获得更好体验：

- **显示** → 颜色深度选择"24 位"（真彩色）
- **体验** → 根据网络状况选择连接速度，局域网可选择"LAN (10 Mbps 或更高)"
- **本地资源** → 可勾选"剪贴板"实现双向复制粘贴

## 常见问题排查

### Q1: 连接后输入凭据，点击登录立即闪退

**最常见原因：该用户已经在 Debian 本地登录了桌面环境。** 同一个 Linux 用户不能同时拥有两个图形化桌面会话。

**解决：** 按第二步 2.3 中的方法，在 Debian 上创建一个专门用于远程连接的独立用户（如 `rdpuser`），每次远程连接使用这个专用账户登录即可，不会与本地用户冲突。

如果问题依旧，可进一步排查：

```bash
# 查看 xrdp 日志
sudo tail -f /var/log/xrdp.log
sudo tail -f /var/log/xrdp-sesman.log
```

同时检查 `~/.xsession` 是否配置正确，以及 XFCE 是否正确安装。

### Q2: 认证成功但一直黑屏

**常见原因有三：**

1. **显示管理器不兼容**：Debian 默认的 gdm3 与 xrdp 配合容易黑屏。换成 lightdm 即可解决（第一步已安装并启用）。
2. **`startwm.sh` 中桌面启动命令未正确执行**：按第二步 2.2 中的方法修改 `/etc/xrdp/startwm.sh`，确保在 `exit 0` 之前调用了 `startxfce4`。
3. **用户未配置 `~/.xsession`**：确认对应用户已执行 `echo "startxfce4" > ~/.xsession`。

### Q3: 多用户同时连接

xrdp 原生支持多用户并发，每个用户登录后会拥有独立的桌面会话，互不影响。如需远程多人同时使用，在 Debian 端为每位用户创建独立账户即可。

注意：同一个用户不能同时登录多个图形化会话。如果你的场景是同一人需要在多台 Windows 终端同时远程连接，请参考第二步 2.3 为每个终端创建独立的远程用户。

## 总结

通过 xrdp 让 Windows 远程连接 Debian 桌面，整体配置流程如下：

1. 统一安装所需软件包（XFCE + lightdm + xrdp + 字体 + 剪贴板工具）
2. 配置 lightdm 为默认显示管理器（规避黑屏）
3. 配置 `~/.xsession` 指向 `startxfce4`
4. 修改 `/etc/xrdp/startwm.sh` 确保会话正确启动
5. 创建专用远程用户，避免与本地登录用户冲突（解决登录闪退）
6. 从 Windows 使用 `mstsc` 连接

这套方案的核心优势在于 **RDP 是 Windows 原生协议**——无需在客户端安装任何第三方软件，开箱即用，安全省心。其余优势包括：

- **性能优异**：RDP 协议对带宽要求低，操作流畅
- **多用户支持**：多人可同时连接，同一用户也可多终端登录，互不干扰
- **稳定可靠**：xrdp + lightdm + XFCE 组合经过大量生产环境验证，兼容性最佳

如果你在使用过程中遇到问题，欢迎在评论区留言交流。
