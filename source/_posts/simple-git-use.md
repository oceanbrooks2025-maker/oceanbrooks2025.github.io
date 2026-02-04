---
title: 简单的 Git 同步方法
date: 2026-02-04 12:00
categories: 'test'
tags: 
  - 'test'
description: 
---

1. 安装 Git

   安装本身没什么需要注意的，这里主要是介绍一下如何在无管理员权限的情况下，将 portable 版本的 Git Bash 添加到右键菜单目录。

   **一言以蔽之，借助"发送到"这个功能，执行自定义脚本**

   *注：为什么不直接用"发送到"打开 Git Bash 呢？因为"发送到"和 Git Bash 接口不兼容（"发送到"传递的是目录地址，而 Git Bash 默认接收脚本文件）所以我们写一个简单的脚本来充当中间人*

   - 创建启动脚本

   就在 Git 安装目录创建一个 OpenGitBash.bat 的脚本，内容如下：

   ~~~
   @echo off
   :: 1. 切换到目标目录
   cd /d %1
   
   :: 2. 设置临时代理环境变量
   set http_proxy=http://127.0.0.1:7897
   set https_proxy=http://127.0.0.1:7897
   
   :: 3. 启动 Git Bash
   start "" "...\git-bash.exe"
   ~~~

   *注：这里的第二步设置了代理，顺便解决了 Git 的网络问题。不过这个环境变量有时候不对 npm 生效，所以如果你需要用到 npm ，最好是修改一下 npm 的 config*

   再创建一个快捷方式 OpenGitBash 指向这个脚本

   - 添加到“发送到”

     WIN + R ，打开 shell:sendto （“发送到”目录），将刚才创建的快捷方式 OpenGitBash 放进去

   **如何使用？** 使用方法与原生 Git Bash 有所不同，不能在目录内右键空白的菜单中找到“发送到”选项；而要返回上一级，右键选中目录->"显示更多选项"->"发送到"->"OpenGitBash.bat"

2. 拉取仓库

   第一次拉取

   ~~~
   git clone https://github.com/user/repo.git
   cd repo
   ~~~

   平时拉取

   ~~~
   git pull 
   ~~~

3. 推送仓库

   ~~~
   git add .
   git commit -m "description"
   git push
   ~~~

4. SSH身份认证

   正文...

