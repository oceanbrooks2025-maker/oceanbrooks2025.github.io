---
title: 使用 Git + GitHub 实现版本控制和多端同步
date: 2026-02-04 12:00
categories: 'skills'
tags: 
  - 'git'
  - 'github'
description: 单人独立开发场景下，基于 Git 与 GitHub 的版本控制与多设备代码同步的方法指南
---

本文主要介绍 Git 的基础工作流与配置方法，适用场景为：单人开发、不涉及复杂的分支合并、依靠 GitHub 实现多台设备间的代码和文档同步。

## 1. 初始环境配置

在使用 Git 前，需要完成本地身份校验、署名以及必要的网络代理配置。推荐使用 SSH 协议连接 GitHub，以兼顾安全性与便捷性。

### 1.1 SSH 密钥配置

**基本原理**：SSH 采用非对称加密。在本地生成一对密钥（公钥和私钥），私钥保留在本地，公钥上传至 GitHub。连接时，GitHub 通过公钥验证本地私钥的签名，从而完成免密身份认证。

**配置步骤**：

1. 生成 SSH 密钥对：

打开任意终端，执行以下命令生成基于 ed25519 算法的密钥：

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

​一路回车按默认设置即可。

2. 添加公钥至 GitHub：
* 在用户目录下的 `.ssh` 文件夹中找到公钥文件 `id_ed25519.pub`。
* 用文本编辑器打开，全选并复制全部内容。
* 登录 GitHub，进入 `Settings` -> `SSH and GPG keys` -> `New SSH key`。
* 填写 Title（如设备名称），将复制的公钥粘贴至 Key 框中，点击 `Add SSH key`。


3. 测试连通性：
```bash
ssh -T git@github.com
```


若返回 `Hi <username>! You've successfully authenticated...`，则表示配置成功。此后该设备与 GitHub 的通信将自动完成验证。

### 1.2 全局署名配置

署名用于记录代码变更的提交者信息，与身份验证无关。只需配置一次，即可全局生效。

```bash
git config --global user.name "your_name"
git config --global user.email "your_email@example.com"
```

### 1.3 代理配置（非必要）

若访问 GitHub 存在网络障碍，可配置代理。注意 HTTPS 和 SSH 协议使用的底层通信机制不同，需要分别配置。

**对于 HTTPS 连接**（通过 Git 内部配置）：

```bash
# 如果代理仅支持 HTTP 协议
git config --global http.proxy http://127.0.0.1:<port>
git config --global https.proxy https://127.0.0.1:<port>

# 如果代理支持 SOCKS5 协议
git config --global http.proxy socks5://127.0.0.1:<port>
git config --global https.proxy socks5://127.0.0.1:<port>
```

**对于 SSH 连接**（通过 SSH 配置文件）：
在用户目录的 `.ssh` 文件夹下，编辑或新建 `config` 文件，写入以下内容：

```text
Host github.com
    HostName github.com
    User git
    # Linux / macOS 使用 nc (netcat) 命令
    # ProxyCommand nc -X 5 -x 127.0.0.1:<port> %h %p
    
    # Windows (Git Bash 内置了 connect 命令)
    ProxyCommand connect -S 127.0.0.1:<port> %h %p

```

---

## 2. 建立本地与远程仓库的连接

准备好本地目录后，有以下两种方式与 GitHub 建立连接。推荐使用“直接克隆”方式以避免初始冲突。

### 方式一：直接克隆（推荐）

适用于 GitHub 上已创建好仓库（无论是否为空），本地从零开始。

```bash
git clone git@github.com:<username>/<repo>.git
```

该命令会自动将云端仓库下载至本地，并建立好所有的追踪关系。

### 方式二：本地初始化并关联远程

适用于本地已有项目文件，需要将其推送至新建的 GitHub 空仓库。

```bash
# 1. 初始化本地仓库
git init

# 2. 添加远程仓库地址，origin 为约定的远程仓库别名
git remote add origin git@github.com:<username>/<repo>.git

# 3. 将当前目录下的所有文件暂存并提交
git add .
git commit -m "Initial commit"

# 4. 统一主分支名称为 main（兼容 GitHub 默认标准）
git branch -M main

# 5. 推送并绑定远端分支
git push -u origin main
```

*注：`-u` 参数（`--set-upstream`）用于在本地 `main` 分支和云端 `main` 分支间建立默认的跟踪通道。后续推送只需执行 `git push`。*

---

## 3. 日常同步工作流

在多设备（例如公司电脑与家中电脑）上同步工作时，必须严格遵守**“先拉取，再修改，后推送”**的原则，以避免产生版本冲突。

```bash
# 1. 开始工作前，先同步云端最新代码
git pull

# 2. 在本地进行增删改...

# 3. 将修改推送至云端仓库
git add .
git commit -m "descriptions"
git push
```

---

## 4. Git 版本控制基础

单人使用时，最常用的版本控制操作是查看历史和回溯修改。

### 4.1 查看提交历史

```bash
# 查看详细历史
git log

# 查看精简历史（仅包含哈希值和提交信息）
git log --oneline
```

### 4.2 丢弃本地未提交的修改

如果文件修改后尚未执行 `git commit`，想要恢复到上次提交的状态：

```bash
# 撤销单个文件的修改
git restore <file_name>

# 撤销当前目录下所有文件的修改
git restore .
```

### 4.3 版本回退（撤销已提交的修改）

使用 `git reset` 命令可以进行版本回退，常用有两种模式：

* **Soft 模式（软回退）**：撤销 `commit` 记录，但保留本地代码的修改内容。适用于提交信息写错或遗漏文件的情况。
```bash
# 回退到上一个版本
git reset --soft HEAD~1
```


* **Hard 模式（硬回退）**：撤销 `commit` 记录，且**强制丢弃**本地的所有修改，代码完全恢复至指定版本。操作需谨慎。
```bash
# 回退到指定的 commit 节点（通过 git log 查看节点哈希值）
git reset --hard <commit_hash>
```

---

## 5. 补充技巧
### 5.1 将 Portable Git Bash 添加至右键菜单

在无管理员权限的环境下，可通过 Windows 的“发送到”功能，将免安装（Portable）版本的 Git Bash 加入右键菜单。由于“发送到”传递的是目录路径，而 Git Bash 默认接收脚本文件，需要通过自定义批处理脚本进行中转。

**操作步骤**：

1. 创建启动脚本：
在 Git 目录下创建一个 `OpenGitBash.bat` 文件，内容如下：
```bat
@echo off
:: 1. 切换到右键选中的目标目录
cd /d %1

:: 2. 设置临时终端代理环境变量（按需保留）
set http_proxy=http://127.0.0.1:<port>
set https_proxy=http://127.0.0.1:<port>

:: 3. 启动 Git Bash (请将路径替换为实际的 git-bash.exe 路径)
start "" "git-bash.exe"
```

*注：此处的终端代理对多数命令行工具生效，但部分工具（如 npm）可能仍需通过其自身的 config 进行配置。*

2. 创建快捷方式：
为 `OpenGitBash.bat` 创建一个快捷方式。

3. 加入“发送到”目录：
按下 `Win + R`，输入 `shell:sendto` 打开目录，将刚刚创建的快捷方式放入该文件夹。

**使用方法**：
返回目标文件夹的上一层目录，鼠标右键点击目标文件夹 -> 选择“发送到” -> 点击 `OpenGitBash` 即可在目标路径打开 Git Bash。
