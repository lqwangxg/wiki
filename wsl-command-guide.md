---
title: 🐧 WSL 常用命令大全
description: Windows Subsystem for Linux (WSL) 常用操作命令汇总。
published: true
date: 2025-10-31T05:05:20.173Z
tags: docker, linux, windows, wsl
editor: markdown
dateCreated: 2025-10-07T05:16:40.549Z
---

# 🐧 WSL 常用命令大全

Windows Subsystem for Linux (WSL) 常用操作命令汇总。

---

## 📋 查看已安装发行版

```bash
# 查看所有已安装的发行版（含状态与版本）
wsl --list --verbose
# 或简写
wsl -l -v
```

示例输出：
```
  NAME                   STATE           VERSION
* Ubuntu-22.04           Running         2
  docker-desktop         Running         2
  docker-desktop-data    Stopped         2
```

| 列名 | 含义 |
|------|------|
| NAME | 发行版名称 |
| STATE | 当前状态（Running / Stopped） |
| VERSION | WSL 版本（1 或 2） |
| `*` | 表示默认发行版 |

---

## 🧱 安装发行版

### 1️⃣ 查看可安装发行版列表
```bash
wsl --list --online
# 或简写
wsl -l -o
```

示例输出：
```
NAME            FRIENDLY NAME
Ubuntu          Ubuntu
Debian          Debian GNU/Linux
Ubuntu-20.04    Ubuntu 20.04 LTS
Ubuntu-22.04    Ubuntu 22.04 LTS
```

---

### 2️⃣ 安装指定发行版
```bash
wsl --install -d <发行版名称>
```

例如：
```bash
wsl --install -d Ubuntu-22.04
```

> 💡 若系统首次使用 WSL，该命令会自动安装 WSL 核心与虚拟机环境。

---

### 3️⃣ 手动导入 `.tar` 或 `.appx` 发行版
```bash
wsl --import <发行版名称> <安装路径> <rootfs路径.tar>
```

示例：
```bash
wsl --import Arch C:\WSL\Arch archlinux.tar
```

---

## ⚙️ 设置默认发行版与版本

```bash
# 设置默认发行版
wsl --set-default <发行版名称>

# 设置默认 WSL 版本为 2
wsl --set-default-version 2

# 将已安装发行版切换为 WSL 2
wsl --set-version <发行版名称> 2
```

---

## 🚀 启动发行版

```bash
# 启动默认发行版
wsl

# 启动指定发行版
wsl -d <发行版名称>

# 启动并执行命令
wsl -d <发行版名称> -- <命令>
```

示例：
```bash
wsl -d Ubuntu-22.04 -- uname -a
```

---

## 🛑 停止或关闭 WSL 实例

### 1️⃣ 停止所有 WSL 实例（包括 Docker Desktop）
```bash
wsl --shutdown
```

### 2️⃣ 停止指定发行版
```bash
wsl --terminate <发行版名称>
# 或简写
wsl -t <发行版名称>
```

示例：
```bash
wsl -t Ubuntu-22.04
```

### 3️⃣ 强制重启 WSL 服务
（仅在系统卡死时使用）
```bash
wsl --shutdown
net stop LxssManager
net start LxssManager
```

---

## 🧹 卸载 / 删除发行版

```bash
wsl --unregister <发行版名称>
```

示例：
```bash
wsl --unregister Ubuntu-20.04
```

> ⚠️ 注意：该命令会永久删除该发行版的所有文件与配置。

---

## 🧰 其他实用命令

```bash
# 查看 WSL 版本信息
wsl --version

# 查看 WSL 状态
wsl --status

# 更新 WSL 核心
wsl --update
```

---

## 💡 常见操作示例

```bash
# 查看所有 WSL 实例
wsl -l -v

# 安装 Ubuntu 22.04
wsl --install -d Ubuntu-22.04

# 停止所有实例
wsl --shutdown

# 只关闭 Ubuntu 实例
wsl -t Ubuntu-22.04

# 设置 Ubuntu 为默认发行版
wsl --set-default Ubuntu-22.04

# 删除旧版本 Ubuntu
wsl --unregister Ubuntu-20.04
```

---
## 重命名
wsl --install 发行版名 # 安装之后不能修改名称，只能通过导出，删除，再导入来修改名。
```bash
#💡 C:\WSL 是导出目录，可自定义路径。
wsl --export Ubuntu-22.04 C:\WSL\ubuntu_backup.tar

#⚠️ 这会删除原实例（包括用户、数据），但你已有备份 .tar。
wsl --unregister Ubuntu-22.04

wsl --import Ubuntu-RPA C:\WSL\Ubuntu-RPA C:\WSL\ubuntu_backup.tar --version 2
```
### 解释：
- Ubuntu-RPA → 新的发行版名称；
- C:\WSL\Ubuntu-RPA → 安装路径；
- C:\WSL\ubuntu_backup.tar → 导出的备份文件；
- --version 2 → 指定使用 WSL 2。

## 📖 参考文档
- [Microsoft 官方文档：WSL 命令参考](https://learn.microsoft.com/windows/wsl/basic-commands)
- [Microsoft Store: 各发行版下载](https://aka.ms/wslstore)
